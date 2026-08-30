# 94 — Explicit Locks: `ReentrantLock`, `ReadWriteLock`, `StampedLock`

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: the two contended writes in `orderflow` — inventory decrement and wallet debit. This topic is where those two writes deadlock each other, and where you learn to prove it from a thread dump rather than from a hunch.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> `ReentrantLock` and `ReentrantReadWriteLock` are thin wrappers over
> **`AbstractQueuedSynchronizer` (AQS)**: one `volatile int state` plus a
> **CLH-style FIFO wait queue** of parked threads.
>
> `state` *is* the lock. For `ReentrantLock`, `state` is the reentrant hold count —
> `0` means free — and a separate `exclusiveOwnerThread` field records who owns it.
> For `ReentrantReadWriteLock`, the same 32-bit word is **split**: the high 16 bits
> are the shared (read) count, the low 16 bits are the exclusive (write) hold count.
>
> Acquiring is a **CAS on `state`**. If the CAS succeeds you own the lock and no
> operating system call happened. If it fails, you are appended to the queue and
> **parked** — `LockSupport.park`, which becomes a real OS wait (a futex on Linux).
> Releasing sets `state` back and **unparks** your successor.
>
> `StampedLock` is the odd one out. It is **not** an AQS subclass. It has its own
> hand-written 64-bit `state` word and its own queue, and it adds a third mode:
> an **optimistic read** that takes no lock at all, returns a `stamp`, and requires
> you to **`validate(stamp)` after reading**. It is **not reentrant** and has **no
> `Condition`**. Taking a `StampedLock` twice on one thread is a permanent,
> undetectable self-deadlock.

Two consequences you will use for the rest of your career:

1. A thread blocked on `synchronized` is **`BLOCKED`** in a thread dump. A thread
   blocked on `ReentrantLock` is **`WAITING (parking)`**. They are different words
   for the same business symptom, and if you only know one of them you will read
   half of your thread dumps wrong.
2. `synchronized` releases the monitor automatically when the frame unwinds, for any
   reason, including an exception. `ReentrantLock` does not. Every explicit lock you
   take is a `finally` block you are obliged to write, and there is exactly one
   correct place to put it.

---

## The bridge from what you know

### `NO TYPESCRIPT ANALOGUE.`

Say it plainly, because reaching here would cost you more than it buys.

A single-threaded runtime **has no locks**. Node's event loop gives you
run-to-completion: once your synchronous function starts, nothing else in the process
executes until it returns. That guarantee is why you have never written a mutex, and
it is why you have never had a deadlock.

There is no `Promise` pattern, no `async` idiom, no library, and no `AsyncLocalStorage`
trick that corresponds to `ReentrantLock`. There is nothing to transfer.

What you must actively **unlearn** is a habit, not a concept:

| Habit you have | Why it is safe in Node | Why it is a data-corruption bug in Java |
|---|---|---|
| Read a shared value, compute, write it back, all in one function | Nothing can run between the read and the write | The OS can suspend you between any two bytecodes. Another thread reads the stale value and both write. |
| "This function is short, so it's atomic" | Effectively true | Length has nothing to do with it. A single `count++` is three operations. |
| Guard a shared resource with a boolean flag | Works, because setting and checking the flag cannot interleave | Two threads both see `false` and both proceed. This is the check-then-act race from Topic 92. |
| Assume that if it works under test it works under load | Mostly true | Deadlocks and races are *interleaving-dependent*. The failing interleaving may occur once per million requests, which at 400 rps is once per 40 minutes. |

### The one thing that *rhymes*, and why it is not the same

You have a mental model of **await points**. In an `async` function, you know exactly
where control can leave: at each `await`. Between two `await`s, nothing interleaves.

Java has an inverse of that: with a lock held, nothing else can enter the *same*
critical section. So `lock()` … `unlock()` is roughly "the region between two awaits",
except:

- The protection is **per-lock**, not global. Code guarded by a *different* lock runs
  concurrently the whole time.
- The boundary is not visible in the syntax. `await` is a keyword you can grep for. A
  missing `unlock()` looks like ordinary code.
- Two threads each holding one lock and waiting for the other's is a state your
  runtime cannot enter and Java's can. That state is a **deadlock**, and it is
  permanent — there is no timeout, no rejection, no error. The threads simply stop
  existing as far as your service is concerned.

**Verdict: NO ANALOGUE. Do not carry the event loop into this document.**

### The database analogue that *does* transfer — from Topic 52

This is the honest bridge, and it comes from your SQL background rather than your
Node background.

In Topic 52 you learned two ways to protect a row:

- **Optimistic** — `@Version`. Read freely, check at write time that nobody else
  changed it, and retry if they did. Cheap when conflicts are rare.
- **Pessimistic** — `SELECT ... FOR UPDATE`. Take the lock up front, serialise
  writers, never retry. Cheap when conflicts are common and the critical section is
  short.

**Topic 94 is exactly the same trade, one level down.** Same names, same reasoning,
different unit of protection:

| Database (Topic 52) | JVM (Topic 94) | The unit protected |
|---|---|---|
| `SELECT ... FOR UPDATE` (`PESSIMISTIC_WRITE`) | `ReentrantLock.lock()` | a row → an object graph in one heap |
| `@Version` + retry on `OptimisticLockException` | `StampedLock.tryOptimisticRead()` + `validate(stamp)` | a row version → a lock version stamp |
| Shared read locks vs exclusive write locks | `ReentrantReadWriteLock` | a row/table lock mode → a critical section |
| `SELECT ... FOR UPDATE NOWAIT` / `SKIP LOCKED` | `tryLock()` / `tryLock(timeout)` | fail fast rather than queue |
| Postgres detecting a deadlock and killing one transaction | **nothing.** The JVM detects and *reports* deadlock; it never breaks one. | — |

That last row is the one to memorise. **Postgres is kinder than the JVM.** A database
deadlock is resolved automatically: the engine picks a victim, aborts it, and your
application sees a retryable error. A JVM deadlock is not resolved by anything. Two
threads park and stay parked until the process is killed. The JVM will *tell* you it
happened, if you ask, and that is all it will do.

---

## What is this?

`java.util.concurrent.locks` gives you lock objects you construct, hold and release
yourself, instead of the `synchronized` keyword's implicit monitor.

There are three you need to know.

### `ReentrantLock`

A mutual-exclusion lock with the same semantics as `synchronized` — one holder at a
time, re-entrant by the same thread — plus five capabilities the keyword cannot
express:

1. **`tryLock()`** — take it if free, return `false` immediately if not. No blocking.
2. **`tryLock(timeout, unit)`** — wait, but only this long. This is the primitive that
   converts a deadlock into an error you can handle.
3. **`lockInterruptibly()`** — a waiting thread can be interrupted and abandon the
   attempt. `synchronized` waiting is uninterruptible.
4. **Fairness** — `new ReentrantLock(true)` hands the lock to the longest waiter.
   `synchronized` makes no ordering promise at all.
5. **Multiple `Condition`s** — one lock can have several independent wait sets
   (`notFull`, `notEmpty`). An object monitor has exactly one.

Plus one structural capability: the lock and unlock do not have to be in the same
method or the same block. That is called **non-block-structured locking**, it is how
hand-over-hand traversal of a linked structure is written, and it is also how people
leak locks.

### `ReentrantReadWriteLock`

A pair of locks that share one state word. Many threads may hold the **read** lock at
once. Exactly one thread may hold the **write** lock, and only when no readers hold it.

The intended workload is read-dominated data: a price table read thousands of times a
second and written once a minute.

The trap is that this is **not free**. Acquiring a read lock still CASes the shared
state word, so N reader threads on N cores still fight over one cache line — you have
removed the *logical* exclusion and kept the *physical* contention. That is Topic 96,
and it is why a `ReadWriteLock` sometimes loses to a plain `ReentrantLock`.

### `StampedLock`

Three modes over one 64-bit state word:

- **Write lock** — exclusive. Returns a stamp; you pass the stamp back to unlock.
- **Read lock** — shared. Also stamped.
- **Optimistic read** — **takes no lock**. `tryOptimisticRead()` returns the current
  version stamp (or `0` if a write lock is held). You read the fields you want, then
  call `validate(stamp)`. If it returns `true`, no write occurred while you were
  reading and your values are consistent. If it returns `false`, you throw your
  values away and fall back to a real read lock.

Optimistic reading is the fastest read available in the JDK for a small, hot,
read-mostly structure, because in the success case it performs **zero writes to shared
memory**. Nothing is CASed. No cache line changes ownership. Readers on different
cores do not interfere with each other at all.

The price is a rigid discipline, which the API does not enforce:

- **Not reentrant.** Any of the three modes, taken twice on one thread, self-deadlocks.
- **No `Condition` support.** `newCondition()` does not exist.
- You must **copy field values into locals and then validate**, in that order. If you
  act on the values before validating, you have acted on a torn read.

---

## Why does it matter?

Four things, and only the first is about performance.

**1. `synchronized` cannot express "give up after 50 milliseconds".**

This is the important one. `synchronized` has one behaviour when contended: wait
forever. `tryLock(50, MILLISECONDS)` gives you a bounded wait and a `false` you can
act on. That single capability is the difference between an order that fails cleanly
with a retryable error and a request thread that never returns.

**2. A hung Java service looks identical to a slow one from outside.**

No exception. No error rate. No log line. The p99 latency graph does not spike — it
goes *flat*, because hung requests never complete and therefore never record a
latency. The first signal is usually the health check failing or the request-thread
pool saturating, and by then it has been broken for minutes. You need to read a thread
dump to find out anything at all, and reading one is a learnable skill that most
engineers never acquire.

**3. `orderflow` has exactly the shape that deadlocks.**

Two shared resources — inventory rows and wallet rows. Two code paths that touch both:
order placement (reserve stock, then debit the wallet) and refunds (credit the wallet,
then release stock). Two paths, two resources, opposite orders. That is the textbook
deadlock and it is also your actual service.

**4. Virtual threads changed the default advice.**

Until JDK 21, "use `synchronized` unless you need something it cannot do" was the
right rule. Under virtual threads (Topic 101), a thread blocking inside `synchronized`
used to **pin** its carrier thread, so a handful of blocked virtual threads could
starve the entire scheduler. A virtual thread blocking on `ReentrantLock` unmounts
cleanly. JDK 24's JEP 491 removed most of that pinning — **verify on your own JDK
before assuming either way**, with the command in the Hands-on section. But the
episode is why you will see recent codebases preferring `ReentrantLock` in places
where 2015 advice said `synchronized`.

---

## Machine-level reality

### AQS: one integer and a queue

Every lock in `java.util.concurrent.locks` except `StampedLock` is a subclass of
`AbstractQueuedSynchronizer` in disguise. The lock class itself is a facade; the real
object is a private inner `Sync extends AbstractQueuedSynchronizer`.

AQS holds two things:

```
private volatile int state;      // the entire meaning of the lock
private transient volatile Node head;   // the CLH wait queue
private transient volatile Node tail;
```

plus, from its superclass `AbstractOwnableSynchronizer`:

```
private transient Thread exclusiveOwnerThread;
```

That `exclusiveOwnerThread` field is not a debugging nicety. It is why `jcmd
Thread.print` can tell you *which thread holds* a `ReentrantLock`, and therefore why
the JVM can detect a `ReentrantLock` deadlock at all. `StampedLock` does not extend
`AbstractOwnableSynchronizer` and does not record an owner — which is why its
self-deadlock is invisible. Hold on to that.

### What `state` means, per lock

| Lock | `state` encoding | Consequence you can observe |
|---|---|---|
| `ReentrantLock` | hold count. `0` = free. `1` = held once. `5` = the owner entered five times. | Reentrancy is just an increment, so it is nearly free. |
| `ReentrantReadWriteLock` | high 16 bits = number of read holds; low 16 bits = write hold count | **Maximum 65535 of each.** Exceeding it throws `Error("Maximum lock count exceeded")` — not an exception, an `Error`. |
| `Semaphore` (Topic 97) | permits remaining | `release()` without a matching `acquire()` silently *creates* permits. |
| `CountDownLatch` (Topic 97) | remaining count | Reaching `0` is terminal. That is what "one-shot" means mechanically. |
| `StampedLock` | **not AQS.** A 64-bit word: low bits are a reader count, one bit is the write flag, the high bits are a **version counter incremented on every write-lock release**. | The version counter is what `validate(stamp)` compares against. That is the whole optimistic-read mechanism. |

Confirm the class hierarchy yourself rather than believing me:

```bash
javap java.util.concurrent.locks.ReentrantLock | head -5
javap java.util.concurrent.locks.StampedLock  | head -5
javap -p java.util.concurrent.locks.ReentrantLock | grep -i sync
```

**What to look for:** `ReentrantLock` declares a private abstract static class `Sync`
that extends `AbstractQueuedSynchronizer`. `StampedLock` extends nothing but `Object`
and implements only `Serializable`. If your JDK shows otherwise, trust your JDK — the
internals are not a public contract and have been rewritten more than once.

### The acquire path, step by step

This is what `lock.lock()` actually does. Notice how much of it is *not* a system call.

```
1.  compareAndSetState(0, 1)
      - one CAS instruction. On x86-64 this is `lock cmpxchg`.
      - SUCCEEDS -> set exclusiveOwnerThread = currentThread(); return.
        Total cost: one atomic instruction and one plain store.
        No syscall. No context switch. Tens of nanoseconds.

2.  CAS failed. Is the current owner me?
      - YES -> state = state + 1; return. (reentrancy: a plain add, no CAS needed,
        because only the owner can be executing here)
      - NO  -> continue.

3.  addWaiter(): allocate a Node, CAS it onto the tail of the queue.

4.  acquireQueued(): loop.
      - If my predecessor is the head, try the CAS once more (someone may have
        released in the meantime). Succeed -> I am now head, return.
      - Otherwise: set my predecessor's waitStatus to SIGNAL, then
        LockSupport.park(this).
      - THIS is where the syscall lives.

5.  Someone calls unlock():
      - state = state - 1. If it reaches 0, exclusiveOwnerThread = null,
        then LockSupport.unpark(successor).
      - The unparked thread wakes at step 4 and re-tries the CAS.
```

Three facts fall out of that listing, and each is an interview answer:

**Fact 1 — an uncontended `ReentrantLock` is one atomic instruction.** It is not
"slower than `synchronized`". It is the same order of magnitude, because
`synchronized` uncontended is also a CAS (on the object's mark word — Topic 85).

**Fact 2 — the expensive part is `park`/`unpark`, and it only happens under
contention.** On Linux, `LockSupport.park` reaches HotSpot's `Parker::park`, which
uses a pthread mutex and condition variable, which is implemented on a **futex**
("fast userspace mutex"). The design of a futex is: stay in userspace while you can,
and only enter the kernel to actually sleep. So the cost model is:

| Situation | What happens | Rough order of magnitude |
|---|---|---|
| Uncontended acquire | one CAS, one store | nanoseconds |
| Reentrant acquire | one add | nanoseconds |
| Contended, released almost immediately | brief spin, CAS succeeds on retry | tens of nanoseconds |
| Contended, must park | `futex(FUTEX_WAIT)`, context switch out, context switch back in on unpark | **microseconds** — three to four orders of magnitude more |

Do not memorise the numbers; memorise the **shape**. The cliff is between "spun and
got it" and "went to sleep". Everything you do to reduce lock cost is really an
attempt to stay on the left of that cliff.

**Fact 3 — non-fair locks barge.** `new ReentrantLock()` is non-fair by default.
Step 1 runs *before* anyone checks the queue, so a thread arriving at exactly the
right moment takes the lock ahead of threads that have been queued for milliseconds.
That is deliberate: barging avoids a park/unpark round trip, which is why the
throughput of a non-fair lock can be many times that of a fair one. The cost is that
a queued thread can, in principle, wait a very long time. That is **starvation**, and
it is Topic 98.

`new ReentrantLock(true)` makes it fair: `tryAcquire` first calls
`hasQueuedPredecessors()` and refuses to barge. **Fairness is expensive. Do not
enable it because it sounds safer.** Enable it when you have measured a starvation
problem, and expect to pay for it in throughput.

One asymmetry worth knowing: **`tryLock()` with no arguments always barges, even on a
fair lock.** It is documented to do so. `tryLock(0, SECONDS)` honours fairness.
If you want fair behaviour from a non-blocking attempt, you must use the timed form
with a zero timeout.

### The CLH queue, and why it is called that

The queue is a variant of the Craig, Landin and Hagersten queue lock. In the original
CLH design, each thread spins on a flag in its **predecessor's** node. Spinning on a
predecessor rather than on one shared variable is the whole point: each waiting thread
polls a different cache line, so waiting threads do not invalidate each other's caches.

AQS adapts it in two ways: the queue is doubly linked (so a cancelled node can be
spliced out), and instead of spinning, a waiting thread **parks** after a brief spin.
The node's `waitStatus` field is how a releasing thread knows whether it needs to
bother calling `unpark` at all.

You will not touch any of this directly. You need it for one reason: it explains why a
thread dump of a contended `ReentrantLock` shows a *pile* of threads all parked with
identical stacks, and exactly one thread doing work. That shape — N identical parked
stacks — is the visual signature of lock contention, and once you have seen it you
recognise it in a second.

### `park`/`unpark` versus `wait`/`notify`

Both end up sleeping the thread. They differ in a way that matters when you read dumps.

| | `Object.wait()` / `notify()` (Topic 89) | `LockSupport.park()` / `unpark()` |
|---|---|---|
| Requires holding a lock | yes — must be inside `synchronized` | **no** |
| Releases a lock on entry | yes, releases the monitor | no, releases nothing |
| Permit semantics | none — `notify` before `wait` is lost | **has a permit.** `unpark` before `park` makes the next `park` return immediately. |
| Thread state in a dump | `WAITING (on object monitor)` | `WAITING (parking)` |
| Spurious wakeup possible | yes, per the JLS | yes |

The permit is why AQS can be written correctly at all: there is no window between
"decide to park" and "actually park" in which a wakeup can be lost.

### The exact difference between BLOCKED, WAITING and TIMED_WAITING

This is the single most useful table in the document. Learn it. It is asked in
interviews as "walk me through diagnosing a hung service", and it is what you will
actually use at 3am.

| State | What the thread is doing | What puts it there | How it appears in `jcmd Thread.print` | What it implies |
|---|---|---|---|---|
| **`RUNNABLE`** | executing, or blocked in an OS call the JVM cannot see | your code; also **socket reads**, which is the trap | `java.lang.Thread.State: RUNNABLE` | *Not* proof of CPU use. A thread blocked on `SocketInputStream.read` is RUNNABLE and using zero CPU. Cross-check with `top -H`. |
| **`BLOCKED`** | waiting to **enter or re-enter a `synchronized` block** | contention on an **intrinsic monitor**, and nothing else | `BLOCKED (on object monitor)` plus a line `- waiting to lock <0x...> (a com.orderflow.Inventory)` | Monitor contention. If several BLOCKED threads name the same monitor, you have found your hot lock. If BLOCKED threads form a cycle, that is a deadlock. |
| **`WAITING`** | parked indefinitely | `Object.wait()` with no timeout; `Thread.join()` with no timeout; **`LockSupport.park()` — which is every `j.u.c` lock**; `BlockingQueue.take()` | `WAITING (on object monitor)` for `wait()`, or `WAITING (parking)` with `at jdk.internal.misc.Unsafe.park` and a line `- parking to wait for <0x...> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)` | **A `ReentrantLock` deadlock lands here, not in BLOCKED.** An idle worker waiting on an empty queue also lands here — so WAITING alone means nothing. You must read the stack. |
| **`TIMED_WAITING`** | parked with a deadline | `Thread.sleep(n)`; `wait(n)`; `join(n)`; `LockSupport.parkNanos`; **`tryLock(timeout, unit)`**; `poll(timeout, unit)` | `TIMED_WAITING (parking)` or `(sleeping)` or `(on object monitor)` | This thread **will** wake up. It cannot be part of a permanent deadlock. That is exactly why the `tryLock(timeout)` fix works: it converts a WAITING deadlock into a TIMED_WAITING that resolves. |
| **`NEW` / `TERMINATED`** | not started / finished | — | rarely seen in a dump | If you see many TERMINATED threads retained, look at Topic 79 rather than here. |

Three rules that follow from that table, which separate people who read dumps from
people who look at dumps:

- **`synchronized` produces `BLOCKED`. `j.u.c` locks produce `WAITING (parking)`.**
  Searching a dump for `BLOCKED` and finding none does not mean there is no lock
  problem. It may mean the codebase uses `ReentrantLock`.
- **`RUNNABLE` does not mean running.** The JVM cannot tell whether a thread inside a
  native call is computing or waiting on a socket. Always corroborate with per-thread
  CPU (`top -H -p <pid>`, then convert the thread id to hex and match `nid=0x...` in
  the dump).
- **`TIMED_WAITING` is a promise that this thread will return.** Deadlock is a
  property of threads that will *not*.

### Where `StampedLock`'s optimistic read gets its speed

A normal read lock **writes to shared memory**: it CASes the reader count up on
acquire and down on release. Two reader threads on two cores must therefore take
exclusive ownership of the same cache line, twice each. They do not exclude each
other logically, but they serialise physically. That is false sharing's cousin, and
it is Topic 96.

`tryOptimisticRead()` performs **a single volatile read** of the state word and
returns it. `validate(stamp)` performs another volatile read and compares. On x86-64 a
volatile read is an ordinary `MOV` — the barrier requirements are satisfied by the
hardware's memory model — so the whole optimistic read is:

```
read state          (a load)
read your fields    (loads)
read state again    (a load)
compare             (a register compare)
```

**Zero stores to shared memory.** Every core keeps the cache line in Shared state.
Nothing is invalidated. This is why an optimistic read scales essentially linearly
with cores while a `ReadWriteLock` read does not.

The rule that makes it correct: **copy the fields into local variables first, validate
second, act third.** If you act on the fields before validating, you have already used
data that may be torn — half from before a write, half from after.

### `[JAVA 25]` and virtual threads

Two version notes, both hedged deliberately:

- **JEP 491 (targeted JDK 24) removed most virtual-thread pinning inside
  `synchronized`.** If you are on JDK 25, the historical reason to prefer
  `ReentrantLock` under Loom is largely gone. It is not entirely gone — native frames
  and class-initialisation still pin. Verify on your JDK with the JFR event in the
  Measurement section rather than trusting any blog post, including this one.
- **Compact object headers** (`-XX:+UseCompactObjectHeaders`) change the mark word
  layout. That affects `synchronized`'s stack-locking encoding, not AQS — AQS lives in
  ordinary fields. Mentioned here only so you do not attribute an AQS change to it.
  It *does* matter for Topic 96's padding arithmetic.

---

## Concurrency trace

Two threads. Two locks. Opposite orders. This is the deadlock you will build in the
Failure drill, traced one step at a time **before** you see any correct code.

**The setup.** `orderflow` guards each product's stock with a `ReentrantLock` from a
map keyed by SKU, and each user's wallet with a `ReentrantLock` from a map keyed by
user id.

- `OrderPlacementService.place()` locks **inventory first**, then **wallet**. That
  ordering came from the domain: you check stock before you take money.
- `RefundService.refund()` locks **wallet first**, then **inventory**. That ordering
  also came from the domain: you give the money back, then put the stock back.

Both are individually reasonable. Neither author knew about the other. This is how
every real deadlock is written.

| Step | Thread A — `http-nio-8080-exec-17` (order placement, order 8812, SKU-1001, user 55) | Thread B — `refund-worker-3` (refund of order 4471, SKU-1001, user 55) | Locks held / outcome |
|---|---|---|---|
| 1 | `inventoryLock("SKU-1001").lock()` → CAS `state` 0→1 succeeds | — | A: {inv:SKU-1001} · state=1, owner=A |
| 2 | reads `available = 12` | `walletLock(55).lock()` → CAS `state` 0→1 succeeds | A: {inv} · B: {wallet:55} |
| 3 | computes `available - 1 = 11`, not yet written | credits wallet by £40.00 | A: {inv} · B: {wallet} |
| 4 | `walletLock(55).lock()` → CAS fails, `state`=1, owner is B → `addWaiter`, `LockSupport.park` | — | **A is `WAITING (parking)` for B's lock** |
| 5 | parked | `inventoryLock("SKU-1001").lock()` → CAS fails, `state`=1, owner is A → `addWaiter`, `LockSupport.park` | **B is `WAITING (parking)` for A's lock** |
| 6 | parked | parked | **Circular wait closed. Deadlock.** No exception. No log line. No timeout. |
| 7 | still parked | still parked | The JVM knows — `ThreadMXBean.findDeadlockedThreads()` would return both — but the JVM **does nothing about it**. |
| 8 | request thread 18 arrives for SKU-1001 → parks behind A | refund worker 4 arrives → parks behind B | Two piles of identical parked stacks growing |
| 9 | …threads 19 through 200 arrive | …workers 5 through 8 arrive | **Tomcat's 200-thread pool is fully consumed by parked threads** |
| 10 | — | — | `GET /products`, which touches **neither** lock, now returns HTTP 503: there is no free request thread to run it. |

**Outcome, in business terms.**

Order 8812 never completes and never fails. The customer's card authorisation
succeeded before the lock was taken, so money is reserved against a card with no order
row to show for it. The refund for order 4471 is half-applied: the wallet was credited
in step 3, but the stock was never returned, so that unit is permanently unsellable.

Within roughly ninety seconds, every endpoint in `orderflow` returns 503 — including
the product catalogue, which shares no data with either lock. The latency dashboard
does **not** spike; it goes flat, because requests that never complete never record a
latency. The error-rate graph shows 503s from the load balancer, not from the
application, so the application logs are silent.

The p99 you recorded in Topic 65 is not violated. It is simply no longer being
measured.

**Second trace: the same two threads, after the `tryLock` fix.**

This is what "converts a hang into an error" means, concretely. Compare the last row
of each table.

| Step | Thread A (order placement) | Thread B (refund worker) | Locks held / outcome |
|---|---|---|---|
| 1 | `invLock.tryLock(50, MILLISECONDS)` → `true` | — | A: {inv} |
| 2 | reads `available = 12` | `walletLock.tryLock(50, MILLISECONDS)` → `true` | A: {inv} · B: {wallet} |
| 3 | `walletLock.tryLock(50, MILLISECONDS)` → held by B → parks **with a deadline** | — | A: `TIMED_WAITING` |
| 4 | still waiting | `invLock.tryLock(50, MILLISECONDS)` → held by A → parks **with a deadline** | B: `TIMED_WAITING` — the cycle exists but is not permanent |
| 5 | 50 ms elapses → `tryLock` returns **`false`** | 50 ms elapses → returns **`false`** | Both threads wake up |
| 6 | `finally` → `invLock.unlock()` → releases | `finally` → `walletLock.unlock()` → releases | **Both locks free. Cycle broken.** |
| 7 | throws `LockAcquisitionTimeout` → mapped to HTTP 409 | logs a warning, requeues the refund with backoff | Request thread returned to the pool |

**Outcome, in business terms.** The customer sees "please try again" after 50 ms
instead of nothing forever. The refund is retried by the worker's own backoff and
succeeds on the next attempt because the contending order has gone. The 503 storm
never happens. Your error-rate graph goes up, which is *correct* — the failure is now
visible, bounded and attributable.

**And the failure mode you have chosen instead.** Under sustained contention, both
threads can time out, both retry, and both time out again. That is a **livelock**:
100% CPU-adjacent behaviour with no forward progress, and it is Topic 98. It is a
better failure than a deadlock because it is self-limiting and observable, but it is
still a failure. Which is why lock ordering is the primary fix and `tryLock` is the
safety net, not the other way round.

---

## Example 1 — minimal

The smallest correct `ReentrantLock`, and the one-character variations that break it.

```java
import java.util.concurrent.locks.ReentrantLock;

/** A single product's stock cell. Deliberately tiny. */
public final class StockCell {

    private final ReentrantLock lock = new ReentrantLock();
    private int available;

    public StockCell(int initial) {
        this.available = initial;
    }

    /** THE IDIOM. Memorise the shape, not the words. */
    public boolean reserve(int quantity) {
        lock.lock();                 // <-- OUTSIDE the try. Always.
        try {
            if (available < quantity) {
                return false;
            }
            available -= quantity;
            return true;
        } finally {
            lock.unlock();           // <-- runs on return, on throw, always.
        }
    }

    public int available() {
        lock.lock();
        try {
            return available;
        } finally {
            lock.unlock();
        }
    }
}
```

Three things in that fifteen lines are load-bearing.

**`lock.lock()` is outside the `try`.** This is not style. If `lock()` itself throws —
and it can, on `OutOfMemoryError` while allocating a queue node, or on a
`ReadWriteLock` at 65535 holders — then it did not acquire, and a `finally` inside the
`try` would call `unlock()` without holding the lock. That throws
`IllegalMonitorStateException`, which **replaces** the original exception. You lose
the real error and get a confusing one.

**There is nothing between `lock()` and `try`.** Not a log line, not a null check, not
a metric increment. Any statement there can throw, and if it does, the lock is held
forever by a thread that has already returned. Nothing will ever release it.

**Even the read takes the lock.** `available` is a plain `int`. Reading it without the
lock is a data race: with no happens-before edge (Topic 86), a reader can see a stale
value indefinitely — not "briefly", but *forever*, because hoisting the read out of a
loop is a legal compilation. If you want lock-free reads, make the field `volatile`
(Topic 87) or use `StampedLock`'s optimistic mode; do not simply omit the lock.

### The same thing with `StampedLock`, and the discipline it demands

```java
import java.util.concurrent.locks.StampedLock;

public final class StockSnapshot {

    private final StampedLock sl = new StampedLock();
    private int available;
    private int reserved;        // two fields that must agree with each other

    public void reserve(int quantity) {
        long stamp = sl.writeLock();
        try {
            available -= quantity;
            reserved  += quantity;      // between these two lines the object is INCONSISTENT
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    /** The optimistic read, written correctly. */
    public int free() {
        long stamp = sl.tryOptimisticRead();     // no lock taken; 0 if a writer holds it
        int a = available;                       // copy to locals
        int r = reserved;                        // copy to locals
        if (!sl.validate(stamp)) {               // did a write happen while we read?
            stamp = sl.readLock();               // yes -> fall back to a real read lock
            try {
                a = available;
                r = reserved;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return a - r;                            // only NOW is it safe to compute
    }
}
```

Read `free()` again and note the order: **copy, validate, compute.** Every incorrect
`StampedLock` usage you will ever see gets that order wrong.

If you computed `available - reserved` *before* validating, you could observe
`available` from before a write and `reserved` from after it — a value that was never
true at any instant. In `orderflow` that is a free-stock figure that is short by one
unit, which the caller then uses to reject an order that should have succeeded.

---

## Example 2 — production scenario (on the project spine)

### The constraints

From the Topic 65 baseline, the numbers you are working against:

- 100,000 products, 1,000,000 orders, 5,000,000 order lines.
- k6 open-model arrival: 70% catalogue read, 20% order read, 10% order placement.
- Skew is deliberate: a handful of products take a large share of traffic. Call the
  hottest `SKU-1001`.
- Tomcat request threads: 200.
- A separate `refund-worker` pool of 4 threads drains a refund queue.
- Recorded baseline p50/p95/p99 committed in `/docs/java/baselines/`.

The service already uses a database-level guard from Topic 52 for the durable
invariant. The in-JVM locks here protect a **write-through reservation cache** that
sits in front of it: an in-memory view of stock held for orders that are in flight but
not yet committed, so the service can reject an oversell without a database round trip
on every request.

That cache is the shared mutable state. It is the reason locks exist in this service
at all.

### The code that ships and hangs the service

```java
package com.orderflow.inventory;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

/** Per-key lock striping. Correct as far as it goes. */
public final class LockRegistry {

    private final Map<String, ReentrantLock> locks = new ConcurrentHashMap<>();

    public ReentrantLock forKey(String key) {
        // computeIfAbsent is the atomic form -- Topic 92. A get/put pair would race.
        return locks.computeIfAbsent(key, k -> new ReentrantLock());
    }
}
```

```java
package com.orderflow.orders;

import java.util.concurrent.locks.ReentrantLock;

@Service
public class OrderPlacementService {

    private final LockRegistry inventoryLocks;
    private final LockRegistry walletLocks;
    private final InventoryCache inventory;
    private final WalletCache wallets;
    private final OrderRepository orders;

    // constructor injection -- Topic 39

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {

        ReentrantLock inv = inventoryLocks.forKey(cmd.sku());          // <-- inventory FIRST
        inv.lock();
        try {
            if (!inventory.reserve(cmd.sku(), cmd.quantity())) {
                throw new OutOfStockException(cmd.sku());
            }

            ReentrantLock wallet = walletLocks.forKey(cmd.userId());   // <-- wallet SECOND
            wallet.lock();
            try {
                wallets.debit(cmd.userId(), cmd.total());
                return orders.create(cmd);
            } finally {
                wallet.unlock();
            }
        } finally {
            inv.unlock();
        }
    }
}
```

```java
package com.orderflow.payments;

@Service
public class RefundService {

    private final LockRegistry inventoryLocks;
    private final LockRegistry walletLocks;
    private final InventoryCache inventory;
    private final WalletCache wallets;

    @Transactional
    public void refund(RefundCommand cmd) {

        ReentrantLock wallet = walletLocks.forKey(cmd.userId());       // <-- wallet FIRST
        wallet.lock();
        try {
            wallets.credit(cmd.userId(), cmd.amount());

            ReentrantLock inv = inventoryLocks.forKey(cmd.sku());      // <-- inventory SECOND
            inv.lock();
            try {
                inventory.release(cmd.sku(), cmd.quantity());
            } finally {
                inv.unlock();
            }
        } finally {
            wallet.unlock();
        }
    }
}
```

Every individual thing here is correct. The lock idiom is right in all four places.
The `computeIfAbsent` is the atomic form. The `finally` blocks are properly nested.
Two different engineers wrote these two classes, each reviewed by someone who checked
the lock idiom and approved it.

**The bug is not in either file. It is in the relationship between them**, and no
single-file code review can see it.

### What actually happens under the Topic 65 baseline

The deadlock needs three things to coincide:

1. An order placement and a refund **for the same user** (`cmd.userId()`).
2. …touching the **same SKU** (`cmd.sku()`).
3. …interleaving inside the window between step 1 and step 4 of the trace above —
   a window of maybe a few microseconds.

At 400 rps with 10% placements and a refund worker running continuously, that
coincidence is rare per request and inevitable per hour. Skew makes it far worse:
because SKU-1001 takes a disproportionate share of traffic, condition 2 is satisfied
constantly.

The observable sequence, in order:

| Time | What you see |
|---|---|
| T+0 | Two threads park. Nothing in the logs. Nothing on any dashboard. |
| T+0 to T+60s | Request threads arrive for SKU-1001 and pile up behind the first. Latency for *that* SKU climbs, but it is one SKU among 100,000, so p99 across all endpoints barely moves. |
| T+90s | The 200-thread Tomcat pool is exhausted. New connections queue in the accept backlog. |
| T+95s | `GET /products` starts timing out — an endpoint that touches no lock in this document. |
| T+100s | The Kubernetes readiness probe fails. The pod is removed from the service. |
| T+100s | **Traffic shifts to the other pods, which now deadlock faster.** This is the cascade, and it is why one deadlock takes down a whole deployment. |
| T+130s | The liveness probe fails. Kubernetes restarts the pod. The evidence is destroyed. |

That last line is the operational point. **A restart makes the symptom disappear and
makes the cause unfindable.** If your liveness probe restarts the pod before you take
a thread dump, you will never diagnose this. The first thing to do when you suspect a
hang is not to restart — it is to run `jcmd <pid> Thread.print`.

### Fix 1 — global lock ordering (the primary fix)

Deadlock needs a **cycle**. Impose a total order on lock acquisition and no cycle can
exist, because a cycle requires at least one thread to acquire locks in decreasing
order.

The order must be **global** (every code path uses it) and **total** (any two locks
can be compared). Any consistent order works. Alphabetical on a namespaced key is the
easiest to enforce and the easiest to review.

```java
package com.orderflow.locking;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

/**
 * The ONLY way to take more than one lock in orderflow.
 *
 * Locks are acquired in ascending order of their namespaced key. Because every
 * caller uses the same comparator, no cycle can form: a cycle would require some
 * thread to hold a higher key while waiting for a lower one, which this class
 * makes unrepresentable.
 */
public final class OrderedLocks {

    private final Map<String, ReentrantLock> locks = new ConcurrentHashMap<>();

    public static String inventoryKey(String sku)  { return "inventory:" + sku; }
    public static String walletKey(String userId)  { return "wallet:" + userId; }

    private ReentrantLock lockFor(String key) {
        return locks.computeIfAbsent(key, k -> new ReentrantLock());
    }

    /** Acquire both locks in a globally consistent order and run the action. */
    public <T> T withBoth(String keyA, String keyB, Supplier<T> action) {

        // The total order. String.compareTo is a total order over distinct strings.
        String first  = keyA.compareTo(keyB) <= 0 ? keyA : keyB;
        String second = keyA.compareTo(keyB) <= 0 ? keyB : keyA;

        if (first.equals(second)) {                 // same lock twice: take it once
            ReentrantLock only = lockFor(first);
            only.lock();
            try {
                return action.get();
            } finally {
                only.unlock();
            }
        }

        ReentrantLock l1 = lockFor(first);
        ReentrantLock l2 = lockFor(second);

        l1.lock();
        try {
            l2.lock();
            try {
                return action.get();
            } finally {
                l2.unlock();
            }
        } finally {
            l1.unlock();
        }
    }
}
```

Both services now call the same method, and the ordering decision is made in exactly
one place:

```java
// OrderPlacementService
return orderedLocks.withBoth(
        OrderedLocks.inventoryKey(cmd.sku()),
        OrderedLocks.walletKey(cmd.userId()),
        () -> {
            if (!inventory.reserve(cmd.sku(), cmd.quantity())) {
                throw new OutOfStockException(cmd.sku());
            }
            wallets.debit(cmd.userId(), cmd.total());
            return orders.create(cmd);
        });

// RefundService -- note: the SAME call, arguments in whatever order reads best.
// The comparator, not the caller, decides acquisition order.
orderedLocks.withBoth(
        OrderedLocks.walletKey(cmd.userId()),
        OrderedLocks.inventoryKey(cmd.sku()),
        () -> {
            wallets.credit(cmd.userId(), cmd.amount());
            inventory.release(cmd.sku(), cmd.quantity());
            return null;
        });
```

Note what changed in the call sites: **nothing about their argument order matters any
more.** `RefundService` still passes wallet first because that reads naturally. The
comparator reorders. That is the property you want: correctness that does not depend
on every future author remembering a convention.

Three things this fix has, that are worth saying out loud in an interview:

- It **removes circular wait**, one of the four Coffman conditions (Topic 98). You
  break exactly one condition; you do not need to break all four.
- It has **no runtime cost**. Two string comparisons. No timeouts, no retries, no
  extra parks.
- It is **enforceable in review**. "Never call `.lock()` outside `OrderedLocks`" is a
  rule a linter can check. "Always take inventory before wallet" is a rule that
  survives until the next new joiner.

**The honest weakness:** it only works for locks this class knows about. A third lock
introduced elsewhere — a `synchronized` block in a library, a connection-pool lock
(Topic 109) — is outside the order and can still close a cycle. Lock ordering is a
discipline over a *closed set* of locks, and its integrity depends on the set staying
closed.

### Fix 2 — `tryLock` with a timeout (the safety net)

The second fix does not prevent the cycle. It bounds it.

```java
public <T> T withBothOrFail(String keyA, String keyB, Duration budget, Supplier<T> action)
        throws InterruptedException {

    ReentrantLock l1 = lockFor(keyA);
    ReentrantLock l2 = lockFor(keyB);

    long nanos = budget.toNanos();

    if (!l1.tryLock(nanos, TimeUnit.NANOSECONDS)) {
        throw new LockAcquisitionTimeout(keyA);           // did NOT acquire: do not unlock
    }
    try {
        if (!l2.tryLock(nanos, TimeUnit.NANOSECONDS)) {
            throw new LockAcquisitionTimeout(keyB);       // finally below releases l1
        }
        try {
            return action.get();
        } finally {
            l2.unlock();
        }
    } finally {
        l1.unlock();
    }
}
```

Look carefully at the two `tryLock` call sites, because this is where the idiom
differs from `lock()` and where people get it wrong:

- The **first** `tryLock` failure throws **before** entering the `try`. If you acquired
  nothing, you must unlock nothing. Putting that `throw` inside a `try/finally` that
  unlocks would give you `IllegalMonitorStateException`.
- The **second** `tryLock` failure throws **inside** the outer `try`, so the outer
  `finally` releases `l1`. That release is what breaks the cycle: the other thread's
  wait now succeeds.

**Compare the two fixes honestly.** This is the comparison the drill asks you to make,
and it is a strong interview answer.

| | Fix 1 — global lock ordering | Fix 2 — `tryLock` + timeout |
|---|---|---|
| Prevents the cycle | **yes** — a cycle is unrepresentable | no — the cycle forms and is then broken |
| Coffman condition removed | **circular wait** | **hold-and-wait** (you release what you hold and start over) |
| Failure mode when contended | none — threads queue normally | **livelock**: both time out, both retry, both time out (Topic 98) |
| Cost on the happy path | two string comparisons | a timed park instead of an untimed one; slightly more expensive |
| Thread state during contention | `WAITING (parking)` | `TIMED_WAITING (parking)` — **guaranteed to return** |
| What the customer sees on failure | nothing — it works | HTTP 409, retry-able, after a bounded delay |
| Scales to unknown third-party locks | **no** | **yes** — a timeout bounds a wait on any lock, including ones you did not write |
| What it needs from you | discipline across the whole codebase | a retry policy with **jitter** (Topic 111 — without jitter you build a retry storm) |

**The recommendation: do both, in that priority.** Order your locks so the cycle
cannot form. Then put a timeout on every acquisition anyway, because you do not
control every lock in your process — Hibernate holds locks, HikariCP holds locks, your
metrics library holds locks. Ordering is the fix; the timeout is the seatbelt for the
locks you did not know were there.

### Fix 3 — the one that is usually right: take fewer locks

Neither fix above asks the more important question: **why are two locks held at once?**

In `orderflow` the answer is that the reservation cache and the wallet cache are
separate objects. If the in-flight reservation for an order were a **single**
immutable record holding both the stock delta and the wallet delta, guarded by one
lock keyed on the order, there would be one lock and no ordering problem.

Say this in an interview before you describe either fix:

> "Before I choose a lock-ordering scheme, I'd ask whether the two locks need to be
> held simultaneously at all. Most two-lock deadlocks are a data-modelling problem
> wearing a concurrency costume. If the two pieces of state change together, they
> probably belong behind one lock; and if they genuinely don't, the second lock can
> usually be released before the first is taken."

That answer moves the conversation from "can you use `tryLock`" to "can you design",
which is the actual difference the interviewer is probing for.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `lock()` inside the `try`

**Wrong:**

```java
try {
    lock.lock();                    // <-- inside
    doWork();
} finally {
    lock.unlock();
}
```

**Exact symptom:** on the rare occasion `lock()` throws, your logs show

```
java.lang.IllegalMonitorStateException
    at java.base/java.util.concurrent.locks.ReentrantLock$Sync.tryRelease
    at java.base/java.util.concurrent.locks.AbstractQueuedSynchronizer.release
    at java.base/java.util.concurrent.locks.ReentrantLock.unlock
    at com.orderflow.inventory.InventoryCache.reserve
```

and **the original exception is nowhere**. You are debugging a monitor-state error
when the real event was an `OutOfMemoryError` or a lock-count overflow.

**Root cause:** `unlock()` on a lock you do not hold throws
`IllegalMonitorStateException`. Because it is thrown from the `finally` block, it
*replaces* the in-flight exception. Java does not chain them — the original becomes a
suppressed exception only in try-with-resources, and this is not try-with-resources.

**Fix:** `lock()` on the line before `try`. There is no situation in which the
inside-try form is correct.

```java
lock.lock();
try {
    doWork();
} finally {
    lock.unlock();
}
```

**How to enforce it:** ErrorProne has a `LockNotBeforeTry` check. IntelliJ has a
"Lock acquired but not safely unlocked" inspection. Turn one of them on; this is
exactly the class of bug a static analyser is good at and human review is bad at.

---

### Trap 2 — a statement between `lock()` and `try`

**Wrong:**

```java
lock.lock();
log.debug("reserving {} of {}", quantity, sku);       // <-- can throw
metrics.counter("reservations").increment();          // <-- can throw
try {
    doWork();
} finally {
    lock.unlock();
}
```

**Exact symptom:** intermittent, permanent hangs with **no deadlock reported**.
`jcmd <pid> Thread.print` shows a growing pile of threads in
`WAITING (parking)` on the same `ReentrantLock$NonfairSync`, and **no thread anywhere
in the dump holds it**. The dump's "Locked ownable synchronizers" sections are all
empty for that lock.

That last detail is the giveaway and it is worth planting in your memory: **a lock
with waiters and no owner is a leaked lock**, and a leaked lock is almost always a
missing `unlock()` on an exceptional path.

**Root cause:** a logging framework misconfiguration, a metrics registry throwing on
cardinality limits (Topic 118), or a `toString()` on a lazily-initialised entity
throwing `LazyInitializationException` (Topic 49) — any of these throws *after*
acquisition and *before* the `try`. The `finally` never runs. The owning thread
returns to the pool and serves other requests, cheerfully holding a lock nobody can
ever take.

**Fix:** absolutely nothing between `lock()` and `try`. Move the logging inside.

**Why the JVM cannot help you here:** the deadlock detector looks for *cycles*. One
lock held by a thread that has moved on is not a cycle. `findDeadlockedThreads()`
returns `null`. This failure is invisible to the tool that finds the other one, which
is why Trap 2 is nastier than the headline deadlock.

---

### Trap 3 — "`ReentrantLock` is faster than `synchronized`" — **dated advice**

**Wrong:** a pull request that mechanically converts `synchronized` methods to
`ReentrantLock`, justified in the description as "explicit locks are faster".

**Exact symptom:** you cannot produce one. The JMH numbers before and after are within
run-to-run variance. What you *can* observe is the code getting worse: four new
`finally` blocks, two new opportunities for Trap 1 and Trap 2, and one method where an
early `return` was added later that skipped the `unlock()`.

**Root cause: this was true, once, and it is no longer true.**

The claim comes from Java 5 (2004). At that time, `synchronized` contention went
straight to an inflated OS-level monitor with no adaptive spinning, while
`ReentrantLock` did a smarter spin-then-park. Benchmarks from that era showed
`ReentrantLock` winning by a wide margin under contention, those benchmarks were widely
cited, and the citation outlived the fact.

What changed:

- **Java 6** added adaptive spinning to intrinsic monitors and improved inflation.
- **Java 6/7** added biased locking, which made uncontended `synchronized` cheaper
  still. (Biased locking was then **deprecated in JDK 15 and removed in JDK 18** —
  so if you meet an argument that depends on biased locking, that argument is also
  dated, in the other direction.)
- Both paths now converge on the same primitives: a CAS on the fast path, a spin, then
  `park`.

Today `synchronized` and `ReentrantLock` perform comparably. Which wins a specific
microbenchmark depends on the contention level, the critical-section length and the
JDK build. **Neither answer generalises**, which is why "which is faster" is the wrong
question.

**Fix — the rule to actually follow:**

> Default to `synchronized`. Reach for `ReentrantLock` when you need a capability
> `synchronized` does not have: `tryLock`, a timeout, interruptibility, fairness,
> multiple `Condition`s, or non-block-structured locking. **Choose it for the
> capability, never for the speed.**

**How to say this in an interview:** name the era the advice came from, name what
changed, and then say what you would actually optimise instead — reducing the size of
the critical section, or removing the shared state, both of which beat swapping the
lock type by a wide margin.

---

### Trap 4 — `StampedLock` used reentrantly

**Wrong:**

```java
public void reserveAndAudit(String sku, int qty) {
    long stamp = sl.writeLock();
    try {
        available -= qty;
        auditTrail(sku);                 // <-- this method also takes the write lock
    } finally {
        sl.unlockWrite(stamp);
    }
}

private void auditTrail(String sku) {
    long stamp = sl.writeLock();         // <-- SELF-DEADLOCK. Right here.
    try {
        auditCount++;
    } finally {
        sl.unlockWrite(stamp);
    }
}
```

**Exact symptom:** one thread hangs permanently. In the dump it is
`WAITING (parking)` at `java.util.concurrent.locks.StampedLock.acquireWrite`. And —
the part that costs you an hour —

```
Found one Java-level deadlock:
```

**never appears.** `jcmd Thread.print` reports no deadlock. `ThreadMXBean.findDeadlockedThreads()`
returns `null`. The tool you reach for first tells you there is no problem.

**Root cause, precisely:** `StampedLock` is not an `AbstractOwnableSynchronizer`. It
does not record an owner thread. The JVM's deadlock detector works by building a graph
of *who waits for what, held by whom* — with no owner recorded, there is no edge to
draw, and a one-node cycle is invisible. `ReentrantLock` would have simply incremented
its hold count and proceeded; `StampedLock` queues the thread behind itself, forever.

**Fix:** two options, in order of preference.

1. **Restructure so the lock is taken once.** Have `auditTrail` take the already-held
   stamp as a parameter, or move the audit outside the critical section entirely.
2. **Use `ReentrantReadWriteLock`** if reentrancy is genuinely required. You lose
   optimistic reads; you gain reentrancy and owner tracking. That is usually the
   right trade unless you have measured the optimistic read mattering.

**The general rule:** `StampedLock` belongs in small, leaf-level, self-contained data
structures where you can see every lock acquisition on one screen. The moment a
critical section calls into code you do not control, it is the wrong tool.

---

### Trap 5 — an optimistic read whose stamp is never validated

**Wrong:**

```java
public int free() {
    long stamp = sl.tryOptimisticRead();
    return available - reserved;         // <-- stamp never used. Compiles fine.
}
```

**Exact symptom:** a free-stock figure that is occasionally wrong by exactly the
quantity of one concurrent order — never wildly wrong, never reproducible, never
caught by a test. In `orderflow` it shows up as an order rejected as out-of-stock when
the database says there was plenty, roughly once per hundred thousand requests, and
your on-call runbook eventually acquires the line "if the customer retries it works".

**Root cause:** `tryOptimisticRead()` took **no lock**. It returned a version number
and nothing else. Without `validate(stamp)`, you have written a completely unguarded
read of mutable state that *looks* guarded — the presence of a `StampedLock` field and
a `stamp` variable makes reviewers stop looking.

A writer can run entirely between your two field reads. You then compute from
`available` before the write and `reserved` after it: a state that never existed.

Worse: `tryOptimisticRead()` returns **`0`** when a write lock is currently held.
`validate(0)` always returns `false`. So the mode does not just fail to protect you,
it has a documented "I could not even try" return value that you have discarded.

**Fix:** the copy-validate-compute idiom from Example 1, every time. If you find that
tedious, that tedium is the API telling you to use a `ReadWriteLock` instead.

**How to catch it in review:** grep for `tryOptimisticRead` and check that every hit
has a `validate` within a few lines. If your codebase has more than a handful of these,
consider whether the optimistic mode is earning its complexity — measure it against a
plain `ReentrantLock` with JMH before keeping it.

---

### Trap 6 — `ReadWriteLock` on data that is not actually read-mostly

**Wrong:**

```java
private final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();

public void reserve(String sku, int qty) {
    rw.writeLock().lock();               // called on 10% of requests
    try { ... } finally { rw.writeLock().unlock(); }
}

public int available(String sku) {
    rw.readLock().lock();                // called on 90% of requests
    try { ... } finally { rw.readLock().unlock(); }
}
```

Ninety percent reads. Surely a `ReadWriteLock` wins.

**Exact symptom:** throughput is **lower** than the plain `ReentrantLock` it replaced,
and it gets relatively worse as you add cores. In JFR, `jdk.ThreadPark` events on the
read lock are numerous and short. Under sustained read load, write latency has a long
tail — some writes take hundreds of milliseconds.

**Root cause: two separate effects, and you should be able to name both.**

1. **A read lock is a write to memory.** Acquiring the read lock CASes the shared-count
   field of the same `state` word; releasing CASes it back. Eight reader threads on
   eight cores each perform two atomic read-modify-writes on **one cache line**. The
   line ping-pongs. You removed logical exclusion and kept physical serialisation, and
   you added the bookkeeping cost of a more complex acquire path on top. This is
   Topic 96's mechanism, showing up in a lock.

2. **Writer starvation.** `ReentrantReadWriteLock` in **non-fair** mode (the default)
   lets an arriving reader barge past a queued writer whenever readers already hold the
   lock. Under continuous read traffic the read count may never reach zero, so the
   writer waits. That is the long write tail. (In practice the implementation does
   apply a heuristic that discourages indefinite barging, but the tail is real and
   measurable — do not rely on the heuristic.)

**Fix, in order of what to try:**

1. **Measure against a plain `ReentrantLock` first.** A `ReadWriteLock` only wins when
   the read critical sections are **long** — long enough that concurrent execution of
   readers actually buys something. For a critical section that reads two fields, the
   lock overhead dwarfs the work and exclusion costs you nothing.
2. If reads are short and hot, use **`StampedLock`'s optimistic mode**, which performs
   no shared writes at all in the success case.
3. If writes are rare enough, consider making the state **immutable and swapping a
   `volatile` reference** — a copy-on-write of the whole structure. Readers then need
   no lock at all, only a volatile read (Topic 87). This is `CopyOnWriteArrayList`'s
   idea (Topic 92) applied by hand.
4. If you keep the `ReadWriteLock` and the write tail matters, construct it fair:
   `new ReentrantReadWriteLock(true)`. Expect to lose throughput.

**The general lesson:** `ReadWriteLock` is not "a faster lock for reads". It is a
trade that pays off only when read critical sections are long and reads genuinely
dominate. Prove both before adopting it.

---

## Hands-on proof

Everything below is a command **you** run. I do not have a JVM and will not print
output and call it real. What I can give you precisely is what to look for and what
each possible result means.

### Setup

```bash
mkdir -p ~/java-lab/94 && cd ~/java-lab/94
java --version                 # expect 21 or 25
jcmd -l                        # lists running JVMs and their pids -- you will use this constantly
```

Keep two terminals open for the whole of this section. One runs the program; the other
runs `jcmd`. That two-terminal habit is the single most useful thing in this document.

### Proof 1 — the class hierarchy, settled from your own JDK

```bash
javap java.util.concurrent.locks.ReentrantLock | head -3
javap java.util.concurrent.locks.ReentrantReadWriteLock | head -3
javap java.util.concurrent.locks.StampedLock | head -3
javap java.util.concurrent.locks.AbstractQueuedSynchronizer | head -5
```

**What to look for:**

| What you see | What it means |
|---|---|
| `ReentrantLock` implements `Lock, Serializable`, with no AQS superclass on the class line | Correct — AQS is in the private inner `Sync`. Confirm with `javap -p java.util.concurrent.locks.ReentrantLock \| grep -i sync`. |
| `StampedLock` extends only `Object` | Correct, and it is the whole point of Trap 4: no `AbstractOwnableSynchronizer`, so no owner tracking, so no deadlock detection. |
| `AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer` | Correct. That superclass is the one field that makes `jcmd` able to name a lock's owner. |

To read the actual implementation, unpack the JDK sources:

```bash
mkdir -p ~/jdk-src && cd ~/jdk-src
unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/util/concurrent/locks/*'
less java.base/java/util/concurrent/locks/ReentrantLock.java
```

**What to look for in `ReentrantLock.java`:** the `NonfairSync.tryAcquire` path
calling `compareAndSetState(0, acquires)` before consulting the queue — that single
line *is* barging. In `ReentrantReadWriteLock.java`, look for `SHARED_SHIFT = 16` and
`MAX_COUNT = (1 << SHARED_SHIFT) - 1`; that is where the 65535 limit lives.

### Proof 2 — see all three thread states in one dump

The most valuable ten minutes in this document. Build a program that puts one thread
into each state deliberately, capture a dump, and map each thread to its signature.

`ThreadStates.java`:

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class ThreadStates {

    static final Object monitor = new Object();
    static final ReentrantLock rl = new ReentrantLock();

    public static void main(String[] args) throws Exception {

        // Holder: takes both, then sleeps forever. Will appear as TIMED_WAITING (sleeping).
        Thread holder = new Thread(() -> {
            synchronized (monitor) {
                rl.lock();
                try {
                    sleep(600_000);
                } finally {
                    rl.unlock();
                }
            }
        }, "holder");

        holder.start();
        Thread.sleep(500);      // let the holder acquire both

        // Will be BLOCKED: waiting on an intrinsic monitor.
        new Thread(() -> { synchronized (monitor) { sleep(1); } }, "blocked-on-monitor").start();

        // Will be WAITING (parking): waiting on a j.u.c. lock, no timeout.
        new Thread(() -> { rl.lock(); rl.unlock(); }, "waiting-on-reentrantlock").start();

        // Will be TIMED_WAITING (parking): waiting on a j.u.c. lock WITH a timeout.
        new Thread(() -> {
            try { rl.tryLock(600, TimeUnit.SECONDS); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "timed-waiting-on-trylock").start();

        // Will be WAITING (on object monitor): Object.wait().
        new Thread(() -> {
            Object other = new Object();
            synchronized (other) {
                try { other.wait(); } catch (InterruptedException e) { }
            }
        }, "waiting-on-object-wait").start();

        System.out.println("pid = " + ProcessHandle.current().pid());
        Thread.sleep(600_000);
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

```bash
java ThreadStates.java
# in the other terminal, using the printed pid:
jcmd <pid> Thread.print -l > states.txt
grep -A3 -E '"(holder|blocked-on-monitor|waiting-on-reentrantlock|timed-waiting-on-trylock|waiting-on-object-wait)"' states.txt
```

**What to look for — fill this table in yourself from your own dump:**

| Thread name | Expected `Thread.State` line | Expected extra line |
|---|---|---|
| `holder` | `TIMED_WAITING (sleeping)` | a `Locked ownable synchronizers:` section naming the `ReentrantLock$NonfairSync` |
| `blocked-on-monitor` | `BLOCKED (on object monitor)` | `- waiting to lock <0x...> (a java.lang.Object)` |
| `waiting-on-reentrantlock` | `WAITING (parking)` | `- parking to wait for <0x...> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)` |
| `timed-waiting-on-trylock` | `TIMED_WAITING (parking)` | same `parking to wait for` line — **the state is the only difference** |
| `waiting-on-object-wait` | `WAITING (on object monitor)` | `- waiting on <0x...>` (note: *waiting on*, not *waiting to lock*) |

**How to read the result:** the pair you must be able to tell apart instantly is
`blocked-on-monitor` versus `waiting-on-reentrantlock`. Same business situation —
"cannot get the lock" — two entirely different words in the dump. And the pair
`waiting-on-reentrantlock` versus `timed-waiting-on-trylock` differ only in the state
word, which is precisely the difference between "hung forever" and "will recover".

Keep `states.txt`. It is your reference card for every future incident.

**The `-l` flag matters.** Without it you do not get the "Locked ownable
synchronizers" sections, which are the only way to see who holds a `ReentrantLock`.
Check what your JDK's `jcmd` accepts:

```bash
jcmd <pid> help Thread.print
```

### Proof 3 — reentrancy is an increment

```java
import java.util.concurrent.locks.ReentrantLock;

public class Reentrancy {
    static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) {
        System.out.println("held at start: " + lock.getHoldCount());
        lock.lock();
        System.out.println("after 1st lock: " + lock.getHoldCount());
        lock.lock();
        System.out.println("after 2nd lock: " + lock.getHoldCount());
        System.out.println("isHeldByCurrentThread: " + lock.isHeldByCurrentThread());
        lock.unlock();
        System.out.println("after 1st unlock: " + lock.getHoldCount());
        lock.unlock();
        System.out.println("after 2nd unlock: " + lock.getHoldCount());

        // Now the classic bug: one unlock too many.
        try {
            lock.unlock();
        } catch (IllegalMonitorStateException e) {
            System.out.println("third unlock threw: " + e.getClass().getSimpleName());
        }
    }
}
```

**What to look for:** the hold count climbing 0, 1, 2 and back down. `getHoldCount()`
is reading AQS's `state` directly, so this is you observing the mechanical statement.
The final `IllegalMonitorStateException` is Trap 1's symptom, produced deliberately so
that you recognise it.

### Proof 4 — `StampedLock` self-deadlock, and the detector failing to see it

```java
import java.util.concurrent.locks.StampedLock;

public class StampedSelfDeadlock {
    static final StampedLock sl = new StampedLock();

    public static void main(String[] args) throws Exception {
        System.out.println("pid = " + ProcessHandle.current().pid());
        new Thread(() -> {
            long s1 = sl.writeLock();
            System.out.println("first write lock acquired, stamp=" + s1);
            long s2 = sl.writeLock();          // hangs here, forever
            System.out.println("never printed, stamp=" + s2);
        }, "self-deadlocker").start();
        Thread.sleep(600_000);
    }
}
```

```bash
java StampedSelfDeadlock.java
# other terminal:
jcmd <pid> Thread.print -l | grep -c "Found one Java-level deadlock"
jcmd <pid> Thread.print -l | grep -A12 '"self-deadlocker"'
```

**What to look for:**

| What you see | What it means |
|---|---|
| The grep count is `0` | **Expected, and the point of the proof.** The thread is permanently stuck and the deadlock detector says nothing. |
| `self-deadlocker` is `WAITING (parking)` at `StampedLock.acquireWrite` | Confirms where it stopped. |
| The thread has an empty "Locked ownable synchronizers" list | Confirms *why* the detector is blind: `StampedLock` never registered an owner. |

Now run the same shape with `ReentrantLock` (which will simply succeed, because it is
reentrant) and with `ReentrantReadWriteLock`'s write lock (which will also
self-deadlock — it *is* reentrant for write-after-write, so try
read-lock-then-write-lock instead, which is the documented upgrade that is not
supported). Note which of these the detector finds.

### Proof 5 — measure the uncontended cost, correctly

Do **not** do this with `System.nanoTime()`. Topic 77 explains at length why; the short
version is in the Measurement section below. A correct harness is given there. Run it
before you form any opinion about which lock is faster.

### Proof 6 — is `synchronized` still pinning virtual threads on your JDK?

```java
public class PinCheck {
    static final Object monitor = new Object();

    public static void main(String[] args) throws Exception {
        Thread.ofVirtual().name("vt-sync").start(() -> {
            synchronized (monitor) {
                try { Thread.sleep(2000); } catch (InterruptedException e) { }
            }
        }).join();
        System.out.println("done");
    }
}
```

```bash
# JDK 21/22/23:
java -Djdk.tracePinnedThreads=full PinCheck.java

# JDK 24+: the system property was removed. Use JFR instead:
java -XX:StartFlightRecording=filename=pin.jfr,settings=profile PinCheck.java
jfr summary pin.jfr | grep -i pinned
jfr print --events jdk.VirtualThreadPinned pin.jfr
```

**How to read it:**

| What you see | What it means |
|---|---|
| A pinned-thread stack trace, or `jdk.VirtualThreadPinned` events | Your JDK still pins on `synchronized` in this case. The Loom argument for `ReentrantLock` applies to you. |
| Nothing at all, on JDK 24+ | JEP 491 is in effect. `synchronized` no longer pins here. Note that native frames still pin — this proof only covers the `synchronized` case. |
| The property is ignored with a warning | You are on a JDK where `jdk.tracePinnedThreads` was removed. Use the JFR path. |

**This is a genuine uncertainty and you should settle it on your own JDK rather than
quoting a version number.** It is also a strong interview move: "the answer changed in
JDK 24, and here is how I'd check which behaviour I have."

---

## Failure drill

**Assignment (Topic 94, from the master plan's drill map):** a two-lock ordering
deadlock between the wallet and inventory services. Capture `jcmd <pid> Thread.print`,
read the "Found one Java-level deadlock" section, name the two threads and two
monitors, fix by global lock ordering. Then fix a second time with `tryLock` + timeout
and compare the failure modes.

Budget ninety minutes. Do not skip Part B.

### Part A — reproduce it standalone, to learn the tool

Build the smallest thing that deadlocks, so that the dump is short enough to read
completely.

`DeadlockLab.java`:

```java
import java.util.concurrent.locks.ReentrantLock;

public class DeadlockLab {

    static final ReentrantLock inventoryLock = new ReentrantLock();
    static final ReentrantLock walletLock    = new ReentrantLock();

    public static void main(String[] args) throws Exception {
        System.out.println("pid = " + ProcessHandle.current().pid());

        Thread placement = new Thread(() -> {
            inventoryLock.lock();
            try {
                pause(200);                    // widen the window so it always hits
                walletLock.lock();             // <-- will park here
                try { /* debit */ } finally { walletLock.unlock(); }
            } finally {
                inventoryLock.unlock();
            }
        }, "order-placement-1");

        Thread refund = new Thread(() -> {
            walletLock.lock();
            try {
                pause(200);
                inventoryLock.lock();          // <-- will park here
                try { /* release stock */ } finally { inventoryLock.unlock(); }
            } finally {
                walletLock.unlock();
            }
        }, "refund-worker-1");

        placement.start();
        refund.start();

        placement.join(5_000);
        refund.join(5_000);
        System.out.println("placement alive after 5s: " + placement.isAlive());
        System.out.println("refund alive after 5s:    " + refund.isAlive());
        Thread.sleep(600_000);                 // stay up so you can take a dump
    }

    static void pause(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

Also build the `synchronized` variant, because you must be able to read **both**
formats. Replace the two locks with two plain `Object`s and the `lock()/unlock()` pairs
with nested `synchronized` blocks. Name the file `DeadlockLabMonitors.java`.

### Commands

```bash
java DeadlockLab.java
# note the pid it prints, then in the other terminal:

jcmd <pid> Thread.print -l > deadlock.txt        # the primary tool
jstack -l <pid> > deadlock-jstack.txt            # the older tool; same content
kill -3 <pid>                                    # dumps to the process's stdout instead

# programmatic, and the one you can ship as a health check:
jcmd <pid> VM.info | head -40
```

### What to capture, before reading on

Write these down from your own dump. Do not read the next section first.

1. The exact `Thread.State` line for each of the two threads.
2. Whether the dump contains the string `Found one Java-level deadlock`.
3. The two lock identities, and for each: who waits for it and who holds it.
4. Whether the format differs between the `ReentrantLock` run and the `synchronized`
   run — and if so, exactly which words differ.
5. What `jcmd <pid> Thread.print` says about the CPU state of the two threads. Then
   check `top -H -p <pid>` and confirm they are using **no CPU at all**.

### The format of the deadlock section

*Illustration of the format, not captured output.* Placeholders are shown as
`<tid>`, `0x...` and `<ClassName>` precisely because plausible-looking fabricated
values would teach you to expect specific numbers. Your own dump will have real ones.

**Shape 1 — the `synchronized` (intrinsic monitor) form.** This is the classic, and
the one most documentation shows:

```
Found one Java-level deadlock:
=============================
"order-placement-1":
  waiting to lock monitor 0x... (object 0x..., a java.lang.Object),
  which is held by "refund-worker-1"
"refund-worker-1":
  waiting to lock monitor 0x... (object 0x..., a java.lang.Object),
  which is held by "order-placement-1"

Java stack information for the threads listed above:
===================================================
"order-placement-1":
        at com.orderflow.orders.OrderPlacementService.place(OrderPlacementService.java:<line>)
        - waiting to lock <0x...> (a java.lang.Object)
        - locked <0x...> (a java.lang.Object)
        at ...
"refund-worker-1":
        at com.orderflow.payments.RefundService.refund(RefundService.java:<line>)
        - waiting to lock <0x...> (a java.lang.Object)
        - locked <0x...> (a java.lang.Object)
        at ...

Found 1 deadlock.
```

**Shape 2 — the `ReentrantLock` (ownable synchronizer) form.** Same detector,
different wording, because the resource is an AQS lock rather than a monitor:

```
Found one Java-level deadlock:
=============================
"order-placement-1":
  waiting for ownable synchronizer 0x..., (a java.util.concurrent.locks.ReentrantLock$NonfairSync),
  which is held by "refund-worker-1"
"refund-worker-1":
  waiting for ownable synchronizer 0x..., (a java.util.concurrent.locks.ReentrantLock$NonfairSync),
  which is held by "order-placement-1"

Java stack information for the threads listed above:
===================================================
"order-placement-1":
        at jdk.internal.misc.Unsafe.park(java.base@<version>/Native Method)
        - parking to wait for <0x...> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
        at java.util.concurrent.locks.LockSupport.park(...)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(...)
        at java.util.concurrent.locks.ReentrantLock.lock(...)
        at DeadlockLab.lambda$main$0(DeadlockLab.java:<line>)
        ...

Found 1 deadlock.
```

### How to read it — the four questions, in order

This is the procedure. Apply it to your own file.

**1. Does the string `Found one Java-level deadlock` appear?**

- **Yes** → you have a genuine cycle over monitors and/or AQS locks. Proceed to
  question 2.
- **No** → the service may still be hung. The detector only finds *cycles* over
  resources with a recorded owner. It will **not** find: a leaked lock (Trap 2), a
  `StampedLock` self-deadlock (Trap 4), a pool-vs-lock deadlock where the resource is a
  database connection (Topic 109), or a thread blocked on a socket read with no
  timeout. Go to Topic 98's decision table.

**2. Which threads, and what are they *for*?**

The thread names are the whole reason to name your threads. `pool-1-thread-7` tells you
nothing; `refund-worker-3` tells you which subsystem to look at. Set thread names on
every executor you create — `ThreadFactory` with a naming pattern is three lines and
saves an hour per incident.

**3. Which resources, and in what order did each thread take them?**

Read the stack sections, not just the header. For each thread, the `- locked` lines
are what it **holds**; the `- waiting to lock` / `- parking to wait for` line is what
it **wants**. Write the pairs down:

```
order-placement-1 : holds inventory, wants wallet
refund-worker-1   : holds wallet,    wants inventory
```

Two lines. That is the bug, fully diagnosed.

**4. What is the acquisition order in each code path?**

Go to the two line numbers in the stack. Read the order of `lock()` calls. You will
find one path does A-then-B and the other does B-then-A. That is the finding, and the
fix follows immediately.

### Part B — write down the diagnosis before you fix anything

In your own words, in three sentences:

1. Which two code paths acquire which two locks in which orders.
2. Which of the four Coffman conditions you intend to break, and why that one.
3. What a customer experienced during the outage — not what the JVM did.

Doing this before touching code is the habit that separates a diagnosis from a guess.

### Part C — fix one: global lock ordering

Apply the `OrderedLocks` class from Example 2. Re-run `DeadlockLab` with the ordered
version. It must complete.

Then prove the fix rather than assuming it:

- Run it 10,000 times in a loop with the artificial `pause()` removed.
- Add a third lock and a third thread that takes locks in a random order **through
  `OrderedLocks`**, and confirm it still cannot deadlock. This is the property you are
  actually claiming.
- Now deliberately bypass `OrderedLocks` in one place and confirm the deadlock returns.
  **That is the proof that the fix's integrity depends on the set of locks staying
  closed** — which is the honest limitation you should be able to state.

### Part D — fix two: `tryLock` with a timeout

Apply `withBothOrFail` with a 50 ms budget. Re-run.

Capture the dump **while it is retrying** and compare:

| | Deadlocked run | `tryLock` run |
|---|---|---|
| Thread state | `WAITING (parking)` | `TIMED_WAITING (parking)` |
| `Found one Java-level deadlock` | present | **absent** |
| CPU (`top -H`) | ~0% | non-zero, and rising with the retry rate |
| Threads eventually return | never | yes, within the budget |
| Application logs | silent | `LockAcquisitionTimeout` warnings |

Then push it: drive contention up until both threads time out repeatedly and neither
makes progress. Capture that too. **That is a livelock**, and you have now produced
both failure modes deliberately, which is the entire point of the drill.

Add jitter to the retry (`Thread.sleep(random(0, 50))` before retrying) and watch the
livelock resolve. That jitter is Topic 111's retry policy appearing one layer down.

### Part E — the same drill on `orderflow` under the Topic 65 baseline

1. Start the docker compose stack. Confirm the recorded baseline reproduces within
   ±10% — if it does not, stop and fix that first (the Phase 7 gate rule).
2. Deploy the **unordered** version of `OrderPlacementService` and `RefundService`.
3. Run the k6 scenario mix, and simultaneously drive the refund worker with refunds for
   the same hot SKU and the same user id as the order-placement traffic.
4. Watch, in this order: per-SKU latency, then request-thread-pool utilisation, then
   the 503 rate on `GET /products` — the endpoint that touches nothing in this
   document. **The blast radius is the finding**, not the deadlock itself.
5. `jcmd <pid> Thread.print -l` **before** letting Kubernetes restart the pod. Set the
   liveness probe's `failureThreshold` high enough that you get your dump. Losing the
   evidence to an automatic restart is the most common way this incident goes
   undiagnosed in real companies.
6. Deploy the ordered version. Re-run the identical k6 script. Compare p50/p95/p99
   against the recorded baseline.

### What the fix proves

Write one paragraph answering each:

- **Did lock ordering cost latency?** Compare p99 to the baseline. Two string
  comparisons should be unmeasurable. If it is measurable, your critical section is so
  short that you should question whether the lock is needed at all.
- **Did `tryLock` cost latency?** It should cost slightly more than ordering, and
  should introduce a small error rate under contention. **An error rate appearing is
  correct behaviour**, not a regression — the same lesson as Topic 90's bounded queue.
- **Which fix would you ship?** The expected answer is "ordering as the fix, timeouts
  as the seatbelt", with the reasoning about locks you do not control.

---

## Measurement

### The instrument for each claim

Every claim in this document maps to an instrument that could falsify it. That mapping
is the difference between engineering and folklore.

| Claim | Instrument that makes it falsifiable |
|---|---|
| "The service is deadlocked" | `jcmd <pid> Thread.print -l` containing `Found one Java-level deadlock`; or `ThreadMXBean.findDeadlockedThreads()` returning non-null |
| "It is lock contention, not slow code" | JFR `jdk.JavaMonitorEnter` (for `synchronized`) and `jdk.ThreadPark` (for `j.u.c` locks), aggregated by stack |
| "These threads are hung, not busy" | `top -H -p <pid>` showing ~0% CPU on the thread ids that appear as `nid=0x...` in the dump |
| "A lock is leaked, not deadlocked" | A dump with waiters on a lock and **no** thread listing it under "Locked ownable synchronizers" |
| "`ReentrantLock` is not faster than `synchronized` here" | A JMH harness with `@Threads({1,2,4,8,16,32,64})`, `@Fork(3)`, both arms in the same run |
| "The `ReadWriteLock` is losing to a plain lock" | The same harness with `@Group`/`@GroupThreads` to model the real read:write ratio |
| "Contention is on this specific lock" | JFR `jdk.ThreadPark` `parkedClass` field, or async-profiler in `lock` mode |
| "The fix did not cost latency" | **The recorded Topic 65 p50/p95/p99, re-run identically** |

That last row is the one people skip. A correctness fix that costs you p99 is a trade
you made without measuring.

### `jcmd Thread.print` — the commands you will actually type

```bash
jcmd -l                                  # find the pid
jcmd <pid> Thread.print -l               # full dump WITH ownable synchronizers
jcmd <pid> Thread.print -l > dump-1.txt  # always to a file; always more than one
sleep 10
jcmd <pid> Thread.print -l > dump-2.txt

# Three dumps ten seconds apart is the standard practice. One dump tells you the
# state; three tell you whether it is CHANGING. A stuck thread appears identically
# in all three; a slow one moves.

# Quick triage on a file:
grep -c 'java.lang.Thread.State' dump-1.txt              # total thread count
grep 'java.lang.Thread.State' dump-1.txt | sort | uniq -c | sort -rn   # state histogram
grep -A2 'Found one Java-level deadlock' dump-1.txt
grep -B2 -A8 'parking to wait for' dump-1.txt | head -60
```

**The state histogram is the fastest triage in Java.** One line, and it tells you
which of Topic 98's four failure classes you are in:

| Histogram shape | Likely class |
|---|---|
| Many `BLOCKED`, all on one monitor | Monitor contention or deadlock |
| Many `WAITING (parking)` on one `Sync`, no owner visible | Leaked `ReentrantLock` |
| Many `RUNNABLE` but ~0% CPU | Threads blocked on sockets; look downstream, not at locks |
| Many `RUNNABLE` and 100% CPU with no throughput | Livelock or a spin loop — Topic 98 |
| Total thread count rising between dumps | Thread leak — Topic 98's drill |

### The programmatic detector — ship this

`ThreadMXBean.findDeadlockedThreads()` is a public, supported API. Wire it into an
Actuator health indicator and you will find out about deadlocks from a monitor rather
than from a customer.

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;

@Component
public class DeadlockHealthIndicator implements HealthIndicator {

    private final ThreadMXBean threads = ManagementFactory.getThreadMXBean();

    @Override
    public Health health() {
        // findDeadlockedThreads()          -> monitors AND ownable synchronizers
        // findMonitorDeadlockedThreads()   -> monitors ONLY. Use the first one.
        long[] deadlocked = threads.findDeadlockedThreads();

        if (deadlocked == null) {
            return Health.up().build();
        }

        ThreadInfo[] info = threads.getThreadInfo(deadlocked, true, true);
        StringBuilder detail = new StringBuilder();
        for (ThreadInfo t : info) {
            detail.append(t.getThreadName())
                  .append(" waiting on ").append(t.getLockName())
                  .append(" held by ").append(t.getLockOwnerName())
                  .append('\n');
        }
        return Health.down().withDetail("deadlock", detail.toString()).build();
    }
}
```

Two deliberate details:

- **`findDeadlockedThreads()`, not `findMonitorDeadlockedThreads()`.** The second one
  only looks at intrinsic monitors and will miss every `ReentrantLock` deadlock in
  this document.
- **Wire this to *readiness*, not liveness.** A deadlocked pod should stop receiving
  traffic (readiness) but must **not** be restarted automatically until you have taken
  a dump. That is Topic 121's distinction, and this is one of the clearest cases for
  it.

The call is not free — it walks the thread graph — so run it on a schedule (every 30
seconds), not on every health poll.

### JFR: which event belongs to which lock

This is the detail that gets missed, and it makes the difference between finding
contention and concluding there is none.

| Lock type | JFR event | Key fields |
|---|---|---|
| `synchronized` | `jdk.JavaMonitorEnter` | `monitorClass`, `previousOwner`, `duration` |
| `Object.wait()` | `jdk.JavaMonitorWait` | `monitorClass`, `timeout`, `timedOut` |
| **`ReentrantLock`, `ReadWriteLock`, `Semaphore`, `CountDownLatch`, `BlockingQueue`** | **`jdk.ThreadPark`** | `parkedClass`, `timeout`, `duration`, `address` |

```bash
java -XX:StartFlightRecording=duration=120s,filename=locks.jfr,settings=profile \
     -jar orderflow.jar

jfr summary locks.jfr
jfr print --events jdk.JavaMonitorEnter locks.jfr | head -60
jfr print --events jdk.ThreadPark        locks.jfr | head -60

# Aggregate by the class being parked on:
jfr print --json --events jdk.ThreadPark locks.jfr \
  | jq -r '.recording.events[].values.parkedClass.name' | sort | uniq -c | sort -rn
```

**How to read it:** `jdk.JavaMonitorEnter` empty and `jdk.ThreadPark` full means your
contention is on `j.u.c` locks, and searching for `synchronized` in the codebase will
waste your afternoon. The reverse means the opposite. Check both, always.

`jdk.ThreadPark` also fires for *legitimate* waiting — an idle pool thread on an empty
queue parks too. Filter by `parkedClass` and by duration; a 4-hour park on a
`LinkedBlockingQueue` is a healthy idle worker, and a 40-millisecond park on a
`ReentrantLock$NonfairSync` happening 8,000 times a second is your problem.

The default `profile` settings apply a threshold to `jdk.ThreadPark` (short parks are
not recorded, to bound overhead). If you see nothing, lower it:

```bash
java -XX:StartFlightRecording=filename=locks.jfr,settings=profile,\
jdk.ThreadPark#threshold=1ms -jar orderflow.jar
```

### The standing rule: a naive `System.nanoTime()` loop is wrong

You will be tempted to settle "is `ReentrantLock` faster than `synchronized`" like
this. Do not.

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    lock.lock();
    counter++;
    lock.unlock();
}
System.out.println((System.nanoTime() - start) / 10_000_000 + " ns/op");
```

Five independent reasons it lies, and you cannot tell which one is lying:

1. **Lock elision (Topic 75).** C2 performs escape analysis. If the lock object does
   not escape the compiled scope, the JIT is permitted to **remove the locking
   entirely**. You have measured an empty loop and concluded locks are free. This is
   not hypothetical — it is exactly what Topic 75's drill demonstrated, and it is the
   single most common way lock benchmarks lie.
2. **Lock coarsening.** Adjacent lock/unlock pairs on the same object can be merged
   into one larger critical section. Ten million acquisitions become far fewer.
3. **Dead-code elimination.** If `counter` is never read afterwards, the whole body
   can vanish.
4. **Single-threaded means uncontended.** You have measured the CAS fast path only.
   The entire interesting behaviour — park, unpark, queue, cache-line transfer — never
   executes. The number you get says nothing about production.
5. **Cold JIT and on-stack replacement.** Your average blends interpreted, C1 and C2
   execution in a ratio determined by the loop count you happened to pick.

**Topic 77 is the full treatment.** The correct shape for this topic:

```java
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)          // ONE instance shared by all threads -- this is the point
@Fork(3)                         // three JVMs: profile pollution shows as variance
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Threads({1, 2, 4, 8, 16, 32, 64})   // the scaling curve IS the result
public class LockScaling {

    private final Object monitor = new Object();
    private final ReentrantLock reentrant = new ReentrantLock();
    private final ReentrantLock fair = new ReentrantLock(true);
    private final StampedLock stamped = new StampedLock();

    private long counter;

    @Benchmark
    public void synchronizedIncrement(Blackhole bh) {
        synchronized (monitor) {
            counter++;
            bh.consume(counter);     // stops DCE removing the body
        }
    }

    @Benchmark
    public void reentrantIncrement(Blackhole bh) {
        reentrant.lock();
        try {
            counter++;
            bh.consume(counter);
        } finally {
            reentrant.unlock();
        }
    }

    @Benchmark
    public void fairIncrement(Blackhole bh) {
        fair.lock();
        try {
            counter++;
            bh.consume(counter);
        } finally {
            fair.unlock();
        }
    }

    @Benchmark
    public void stampedWrite(Blackhole bh) {
        long s = stamped.writeLock();
        try {
            counter++;
            bh.consume(counter);
        } finally {
            stamped.unlockWrite(s);
        }
    }
}
```

Four things in that harness are doing specific work, and you should be able to say what
each is for:

- **`@State(Scope.Benchmark)`** — a single shared instance across all threads. With
  `Scope.Thread` each thread gets its own lock, there is no contention, and you have
  measured nothing. This one annotation is the difference between a contention
  benchmark and a fast-path benchmark.
- **`@Threads({1,2,4,8,16,32,64})`** — JMH reruns the whole benchmark at each thread
  count. **The shape of the curve is the result**, not any single number. Flat is
  contention-limited; rising is scaling; *falling* is the pathology you will meet in
  Topic 95.
- **`Blackhole.consume`** — prevents dead-code elimination from deleting the critical
  section.
- **`@Fork(3)`** — three separate JVMs. If two forks disagree, you have profile
  pollution or a JIT-state artefact (Topic 74), and a single-fork number would have
  hidden it.

**What to expect, stated honestly before you run it:** at one thread, all four should
be close, with the fair lock slightly behind. As threads rise, all four should
**degrade**, because a single shared counter cannot scale — that is the whole point of
Topic 95. The fair lock should degrade most sharply. If your results contradict this,
trust your machine and post the numbers; hardware, JDK build and core topology all
move these curves.

### Asymmetric read/write, with `@Group`

For the `ReadWriteLock` question in Trap 6, symmetric threads are the wrong model. You
need a 90:10 read:write ratio, which is what `@Group` is for:

```java
@State(Scope.Group)
public static class RwState {
    final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
    final StampedLock sl = new StampedLock();
    int value;
}

@Benchmark @Group("rwlock") @GroupThreads(9)
public int rwRead(RwState s) {
    s.rw.readLock().lock();
    try { return s.value; } finally { s.rw.readLock().unlock(); }
}

@Benchmark @Group("rwlock") @GroupThreads(1)
public void rwWrite(RwState s) {
    s.rw.writeLock().lock();
    try { s.value++; } finally { s.rw.writeLock().unlock(); }
}
```

Nine reader threads to one writer thread, in the same group, sharing one state object.
Now the read and write throughputs are reported separately, and you can see the writer
tail that Trap 6 describes.

### `perf` — and the honest note about macOS

On Linux you can watch the cache-line transfers directly:

```bash
perf stat -e cache-misses,cache-references,LLC-load-misses \
  -- java -jar orderflow.jar
```

**On macOS, `perf` does not exist.** There is no direct equivalent that exposes the
same hardware counters to a JVM process. Your options on a Mac:

- Run the experiment in a Linux container (`docker run --privileged` is usually
  required for PMU access, and on Apple Silicon under virtualisation the counters may
  be unavailable entirely).
- Use `xctrace record --template 'Time Profiler'` for CPU-time attribution, which is
  useful but does **not** give you cache-miss counters.
- Accept that JMH's scaling curve is the measurement you can actually get, and treat
  cache-line reasoning as an *explanation* for the curve rather than something you have
  independently observed.

**Say this out loud in an interview rather than pretending.** "I'd confirm with
`perf stat` on Linux; on my Mac I can only infer it from the JMH scaling curve" is a
better answer than a confident claim about cache misses you cannot measure.

---

## Practice exercises

### 1 — Easy: build your own thread-state reference

Extend Proof 2's program so that it also produces:

- a thread `BLOCKED` on a `synchronized` **method** (not a block) — check whether the
  dump shows anything different;
- a thread in `TIMED_WAITING (sleeping)` from `Thread.sleep`;
- a thread `WAITING` in `BlockingQueue.take()` on an empty queue;
- a thread `RUNNABLE` inside a socket read that never returns (point it at a listening
  port that never sends anything).

Produce a table with one row per thread: **thread name, `Thread.State` line, the extra
`- waiting/parking/locked` lines, and one sentence on what that state implies about
whether the thread will ever recover.**

Then answer: which two of your rows look most alike in the dump, and what single word
distinguishes them? That word is what you will be searching for at 3am.

### 2 — Medium: the audit (combines Topics 01, 13, 39, 52, 85, 87, 90, 92)

The fragment below contains **seven** distinct defects drawn from this topic and
earlier ones. Find them all, state the **exact symptom each produces in production**
(not "it's bad practice" — what does the on-call engineer see on a dashboard or in a
log?), and rewrite it correctly.

```java
@Service
public class WalletService {

    private final Map<Long, ReentrantLock> locks = new HashMap<>();
    private volatile long totalDebited = 0;
    private final ExecutorService audit = Executors.newFixedThreadPool(4);

    public void debit(Long userId, long amountMinor) {

        ReentrantLock lock = locks.get(userId);
        if (lock == null) {
            lock = new ReentrantLock();
            locks.put(userId, lock);
        }

        try {
            lock.lock();
            log.info("debiting {} from user {}", amountMinor, userId);

            Wallet w = repository.findById(userId).orElseThrow();
            if (w.balanceMinor() < amountMinor) {
                throw new InsufficientFundsException(userId);
            }
            w.setBalanceMinor(w.balanceMinor() - amountMinor);
            repository.save(w);

            totalDebited += amountMinor;

            audit.submit(() -> auditLog.record(userId, amountMinor));
        } finally {
            lock.unlock();
        }
    }

    public boolean sameUser(Long a, Long b) {
        return a == b;
    }
}
```

Hints on where to look, in no particular order: how the lock map is built and why that
matters at 400 rps; where `lock()` sits relative to `try`; what `volatile` does and
does not guarantee for `+=`; what happens to that executor over the service's lifetime;
what `a == b` does for user id 4,829,113 versus user id 42; whether the lock protects
what you think it protects given that `repository.save` may throw; and whether holding
a JVM lock while inside a transaction that holds a database connection is a good idea
(Topic 55, and a preview of Topic 109).

### 3 — Hard: production simulation on the `orderflow` baseline

**Part A — establish the ground truth.** Bring up the Topic 65 stack. Re-run the
recorded k6 baseline and confirm p50/p95/p99 within ±10%. If not, stop here.

**Part B — build the deadlock into the real service.** Implement the unordered
`OrderPlacementService` / `RefundService` pair from Example 2. Add a refund worker that
issues refunds for the hot SKU and the same user ids the load generator uses.

**Part C — instrument before you break it.** Ship the `DeadlockHealthIndicator` wired
to readiness, and start a JFR recording with `jdk.ThreadPark#threshold=1ms`. Raise the
liveness probe's `failureThreshold` so the pod is not restarted before you get a dump.

**Part D — break it, and measure the blast radius.** Run the load. Record, with
timestamps:

- time from first deadlock to request-thread-pool saturation;
- time from saturation to the first 503 on `GET /products`;
- what p99 on the *catalogue* endpoint did — and explain why the latency graph is
  misleading here;
- the state histogram from three dumps ten seconds apart.

**Part E — fix it twice and compare.** Apply lock ordering. Re-run. Apply `tryLock` +
timeout (with jitter). Re-run. Produce a table: p50/p95/p99, error rate, CPU, and the
thread-state histogram, for baseline / broken / ordered / tryLock.

**Part F — argue the other side.** Given your numbers, make the strongest case for
shipping **only** the `tryLock` fix and not the ordering. Then make the strongest case
against yourself. State the condition under which your answer flips. (Hint: the answer
involves how many locks in the process you actually control, and Topic 109 is about
one you do not.)

**Part G — the design question.** Redesign the reservation cache so that the two-lock
path does not exist. Estimate what that costs in complexity and what it buys in failure
modes removed. This is the answer a principal engineer gives, and it is Phase 12's
whole axis.

---

## Interview questions

### Q1 — "When would you use `ReentrantLock` over `synchronized`?"

**Mid-level answer:** "`ReentrantLock` is more flexible and faster, and it gives you
`tryLock` and fairness."

**Senior answer:** "I default to `synchronized`, because the JVM releases the monitor
on any exit path and there is no `finally` to forget. I switch to `ReentrantLock` for a
specific capability: `tryLock` with a timeout — which is how I avoid a deadlock rather
than diagnose one; `lockInterruptibly`, so a shutdown can actually interrupt a waiter;
fairness, if I have measured starvation; multiple `Condition`s on one lock; or
non-block-structured locking such as hand-over-hand traversal.

I would push back on 'faster'. That advice is from Java 5, before adaptive spinning
and improved inflation; today they are comparable and which one wins a microbenchmark
depends on the JDK build and the contention level. There is one modern
performance-shaped reason though, and it is about virtual threads: a virtual thread
blocking on `ReentrantLock` unmounts its carrier, whereas blocking inside
`synchronized` used to pin it. JEP 491 removed most of that pinning in JDK 24, so I
would check which behaviour my JDK has rather than assume."

**What separates them:** the mid-level answer lists features. The senior answer names a
**default** and the conditions for departing from it, corrects the dated performance
claim with the reason it became dated, and hedges the Loom detail on a version boundary
instead of asserting it.

**Follow-up:** "Show me the correct `ReentrantLock` idiom." They are checking whether
`lock()` goes inside or outside the `try`, and whether you can say *why* — the
`IllegalMonitorStateException` that masks the real exception.

---

### Q2 — "Atomics and locks aside — the service is hung. Walk me through diagnosing it."

**Mid-level answer:** "I'd take a thread dump and look for deadlocks, and restart the
service to restore availability."

**Senior answer:** "First, do **not** restart. A restart destroys the only evidence and
guarantees a second incident. If availability is critical, take the pod out of the
load balancer via readiness while leaving the process alive.

Then, in order:

1. **Three thread dumps, ten seconds apart** — `jcmd <pid> Thread.print -l`, to a file.
   One dump gives state; three tell me whether anything is moving.
2. **A state histogram** of each: `grep 'Thread.State' | sort | uniq -c`. That single
   line puts me in one of four buckets: many `BLOCKED` on one monitor, many
   `WAITING (parking)` on one `Sync`, many `RUNNABLE` at zero CPU, or a rising total
   thread count.
3. **Search for `Found one Java-level deadlock`.** If it is there, the header names
   both threads and both resources and I am essentially done.
4. **If it is not there, that does not mean there is no hang.** The detector only finds
   cycles over resources with a recorded owner. It misses a leaked lock — waiters with
   no owner — it misses a `StampedLock` self-deadlock because `StampedLock` records no
   owner, it misses a thread pool blocked waiting for a database connection while
   holding one, and it misses a socket read with no timeout.
5. **Corroborate with `top -H`.** `RUNNABLE` in a dump does not mean burning CPU; a
   socket read is `RUNNABLE`. Zero CPU with `RUNNABLE` threads points downstream, not
   at my locks.
6. **Then look at what is *not* hung.** In the last incident like this, the tell was
   that an endpoint sharing no data with the lock was also failing — which said the
   request-thread pool was exhausted and the real blast radius was the pool, not the
   lock."

**What separates them:** "do not restart" as the first instruction; three dumps rather
than one; knowing the cases the automatic detector **misses**; and cross-checking
thread state against actual CPU rather than trusting the word `RUNNABLE`.

**Follow-up:** "You take the dump and there is no deadlock section, but two hundred
threads are `WAITING (parking)` on the same lock and nothing holds it. What happened?"
They want the leaked lock: an exception between `lock()` and `try`.

---

### Q3 — "How do you prevent deadlock?"

**Mid-level answer:** "Always acquire locks in the same order, and use timeouts."

**Senior answer:** "Deadlock requires **all four Coffman conditions at once**: mutual
exclusion, hold-and-wait, no preemption, and circular wait. Because all four are
needed, I only have to remove **one**, and choosing which one is a design decision with
different costs.

- **Circular wait** is the one I usually break, with a **global lock ordering**. Every
  path that takes more than one lock takes them in a total order — I enforce it by
  routing every multi-lock acquisition through one helper class rather than through a
  convention, because a convention lasts until the next new joiner. Cost: essentially
  zero at runtime. Limitation: it only covers locks I know about.
- **Hold-and-wait** is what `tryLock(timeout)` breaks: I release what I hold and start
  over. Cost: it converts a deadlock into a **livelock** if I do not add jitter, and it
  needs a retry policy. Benefit: it works against locks I do not control — a driver, a
  connection pool, a library.
- **No preemption** is what a database does: it kills a victim transaction. The JVM
  will not do this, and there is no safe way to do it yourself — `Thread.stop` is
  removed for exactly this reason.
- **Mutual exclusion** is broken by not sharing at all: immutable data, per-thread
  state, or a lock-free structure. That is the strongest fix and usually the biggest
  design change.

In practice I do ordering as the fix, timeouts as the seatbelt, and I ask first whether
two locks need to be held simultaneously at all — most two-lock deadlocks are a
data-modelling problem in disguise."

**What separates them:** naming all four conditions, knowing that you break exactly one
and that the choice is a trade, knowing that breaking hold-and-wait creates livelock,
and knowing that the JVM does not break no-preemption while a database does.

**Follow-up:** "Your global ordering is in place and it deadlocked anyway. How?" They
want: a lock outside your ordered set — a library's `synchronized`, a connection pool
(Topic 109), or a lock acquired inside a callback you did not know ran under a lock.

---

### Q4 — "What is `StampedLock`'s optimistic read, and when is it the wrong choice?"

**Mid-level answer:** "It is a faster read lock that does not block writers."

**Senior answer:** "It is not a lock at all. `tryOptimisticRead()` returns the current
version stamp and takes nothing; you read the fields into locals, then call
`validate(stamp)`, and only if that returns `true` are your locals a consistent
snapshot. If it returns `false` you discard them and fall back to a real read lock.
`tryOptimisticRead` also returns `0` when a write lock is held, and `validate(0)` is
always `false`.

The reason it is fast is precise: in the success path it performs **zero writes to
shared memory**. A `ReadWriteLock` read still CASes a counter up and down, so N readers
on N cores contend on one cache line even though they do not exclude each other. The
optimistic read is pure loads, so the line stays shared and readers genuinely scale.

It is the wrong choice in three cases. First, when the critical section calls into code
you do not control, because `StampedLock` is **not reentrant** and taking it twice
self-deadlocks — and since it does not extend `AbstractOwnableSynchronizer`, it records
no owner, so `jcmd Thread.print` will **not** report that deadlock. You get a
permanently parked thread and a tool that says everything is fine. Second, when you
need a `Condition`, which it does not support. Third, when writes are frequent enough
that validation usually fails, at which point you are paying for the optimistic attempt
and then taking the read lock anyway.

I would also say it is easy to use wrongly in a way that reviews miss: an unvalidated
stamp compiles and looks guarded."

**What separates them:** "it is not a lock"; the copy-validate-compute order; the
zero-shared-writes explanation for the speed; and the specific, memorable fact that its
self-deadlock is **invisible to the deadlock detector**, with the reason why.

**Follow-up:** "How would you catch an unvalidated stamp in review?" Grep plus a
custom static-analysis rule; and a design answer — if the discipline is hard to hold,
use `ReadWriteLock` and measure whether the optimistic mode was worth it.

---

### Q5 — "Is `ReentrantLock` faster than `synchronized`?"

**Mid-level answer:** "Yes, `ReentrantLock` scales better under contention."

**Senior answer:** "That was true in 2004 and is **dated advice** now. In Java 5,
intrinsic monitors had no adaptive spinning and inflated straight to an OS monitor,
while `ReentrantLock` spun before parking — so the published benchmarks of that era
showed a large gap. Java 6 added adaptive spinning and improved inflation, and both
paths converged on the same shape: a CAS fast path, a bounded spin, then `park`. Today
they are comparable, and which one wins a specific microbenchmark depends on the
contention level, the critical-section length, and the JDK build.

I would also push back on the framing. Both are dominated by the same thing:
whether the acquire lands on the CAS fast path or has to park. Parking is a context
switch — microseconds against nanoseconds. So if I have a lock performance problem, the
lever is shortening or removing the critical section, or striping the state, or moving
to an atomic or a `LongAdder`. Swapping the lock type moves the number far less than
any of those.

And I would measure it with JMH at `@Threads({1,2,4,8,16,32,64})` with
`@State(Scope.Benchmark)` and `@Fork(3)` — not with a `nanoTime` loop, because C2's
escape analysis can elide the lock entirely and you end up timing an empty loop."

**What separates them:** dating the claim and naming what changed; redirecting to the
lever that actually matters; and naming the specific way the naive benchmark lies —
**lock elision**, which is Topic 75.

**Follow-up:** "Then why does the JDK ship both?" Capability, not speed — and the
Loom nuance from Q1.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. AQS is one `volatile int` plus a queue. `ReentrantReadWriteLock` packs two counters
   into that one int, giving a maximum of 65535 read holds. Why did the designers
   choose to pack rather than use two separate fields — and what would break if they
   had used two `volatile int`s instead?

2. A non-fair `ReentrantLock` lets an arriving thread barge ahead of threads that have
   been queued for milliseconds. This is the default, and it is faster. Explain how
   *unfairness* can produce higher throughput, and then state the workload on which
   that trade is unacceptable.

3. The JVM detects deadlocks and reports them, but never breaks one. Postgres detects
   deadlocks and kills a victim. Argue that the JVM's choice is correct. Then make the
   strongest case against yourself.

4. `StampedLock`'s self-deadlock is invisible to `ThreadMXBean.findDeadlockedThreads()`
   because `StampedLock` records no owner. Would adding owner tracking be a strict
   improvement? What would it cost, and would it change the class's reason for existing?

5. You break exactly one Coffman condition to prevent deadlock. Lock ordering breaks
   circular wait; `tryLock` breaks hold-and-wait. Which condition does making your data
   **immutable** break, and why is that fix qualitatively different from the other two?

6. A thread blocked on a socket read shows as `RUNNABLE`. A thread blocked on a
   `ReentrantLock` shows as `WAITING`. Both are doing nothing. What does the JVM know
   in one case that it does not know in the other, and why is that distinction in the
   `Thread.State` enum at all?

7. Topic 52 gave you optimistic and pessimistic locking at the database layer. This
   topic gives you the same pair one level down. Under what circumstance would you
   choose **opposite** strategies at the two layers — optimistic in the database and
   pessimistic in the JVM, or the reverse — and what would that combination be
   protecting against?

---

## Quick reference card

### The three locks at a glance

| | `ReentrantLock` | `ReentrantReadWriteLock` | `StampedLock` |
|---|---|---|---|
| Built on AQS | yes (inner `Sync`) | yes (inner `Sync`) | **no** — own implementation |
| Reentrant | yes | yes (within a mode) | **no** — self-deadlocks |
| Records an owner | yes (`AbstractOwnableSynchronizer`) | yes | **no** |
| Deadlock detectable by `jcmd` | yes | yes | **no** |
| Supports `Condition` | yes | yes (write lock only) | **no** |
| Fairness option | yes | yes | no |
| Optimistic read | no | no | **yes** |
| `tryLock` with timeout | yes | yes | yes |
| Lock upgrade (read → write) | n/a | **not supported** — deadlocks | `tryConvertToWriteLock` |
| Thread state when waiting | `WAITING (parking)` | `WAITING (parking)` | `WAITING (parking)` |

### The idiom — the only correct shape

```java
lock.lock();                     // outside the try. nothing between this line and the next.
try {
    // critical section
} finally {
    lock.unlock();
}
```

```java
if (!lock.tryLock(50, TimeUnit.MILLISECONDS)) {
    throw new LockAcquisitionTimeout(key);      // acquired nothing -> unlock nothing
}
try {
    // critical section
} finally {
    lock.unlock();
}
```

```java
long stamp = sl.tryOptimisticRead();
int a = field1;                   // 1. copy
int b = field2;
if (!sl.validate(stamp)) {        // 2. validate
    stamp = sl.readLock();
    try { a = field1; b = field2; } finally { sl.unlockRead(stamp); }
}
return a + b;                     // 3. compute
```

### Thread states — the diagnostic table

| State | Cause | Dump marker | Will it recover? |
|---|---|---|---|
| `BLOCKED` | `synchronized` contention **only** | `- waiting to lock <0x...>` | only if the holder releases |
| `WAITING (parking)` | any `j.u.c` lock, `park()`, `take()` | `- parking to wait for <0x...>` | only if someone unparks |
| `WAITING (on object monitor)` | `Object.wait()` | `- waiting on <0x...>` | only on `notify`/`notifyAll` |
| `TIMED_WAITING` | `sleep`, `wait(n)`, `tryLock(t)`, `poll(t)` | `(sleeping)` or `(parking)` | **yes, guaranteed** |
| `RUNNABLE` | running **or** in a native call, incl. socket reads | — | check `top -H` before concluding |

### Diagnostic commands

```bash
jcmd -l                                     # find pids
jcmd <pid> Thread.print -l                  # dump WITH ownable synchronizers
jcmd <pid> help Thread.print                # what your JDK's version accepts
jstack -l <pid>                             # equivalent, older tool
kill -3 <pid>                               # dump to the process stdout

grep 'java.lang.Thread.State' d.txt | sort | uniq -c | sort -rn   # state histogram
grep -A20 'Found one Java-level deadlock' d.txt
top -H -p <pid>                             # per-thread CPU; convert tid to hex, match nid=
printf '0x%x\n' <tid>                       # tid -> the nid= value in the dump

jfr print --events jdk.JavaMonitorEnter r.jfr   # synchronized contention
jfr print --events jdk.ThreadPark        r.jfr   # j.u.c lock contention
```

### Gotchas checklist

- [ ] `lock()` goes **outside** the `try`, with nothing between it and the `try`.
- [ ] Every `lock()` has an `unlock()` in a `finally`. No exceptions.
- [ ] A failed `tryLock` must **not** reach an `unlock()`.
- [ ] `ReentrantLock` contention shows as `WAITING`, not `BLOCKED`. Search for both.
- [ ] Use `findDeadlockedThreads()`, never `findMonitorDeadlockedThreads()`.
- [ ] Use `jcmd Thread.print -l`. Without `-l` you cannot see `j.u.c` lock owners.
- [ ] `StampedLock` is not reentrant and its self-deadlock is **undetectable**.
- [ ] Never act on optimistically-read fields before `validate(stamp)`.
- [ ] `tryOptimisticRead()` returns `0` when a writer holds it; `validate(0)` is false.
- [ ] `ReadWriteLock` read locks still write to shared memory. Measure before adopting.
- [ ] `ReadWriteLock` cannot upgrade read → write. It deadlocks.
- [ ] Fairness is expensive. Enable it only after measuring starvation.
- [ ] `tryLock()` with no arguments barges even on a fair lock. Use `tryLock(0, unit)`.
- [ ] Never hold a JVM lock across an I/O call, a database call, or an HTTP call.
- [ ] Name your threads. `pool-1-thread-7` costs you an hour per incident.
- [ ] Do **not** restart a hung pod before taking a thread dump.
- [ ] "`ReentrantLock` is faster" is 2004 advice. Say so.

---

## When would I use this at work?

**1. The 3am hang.**

Pager fires: p99 flat, error rate from the load balancer only, application logs silent.
You SSH to the pod, run `jcmd <pid> Thread.print -l > /tmp/d1.txt`, wait ten seconds,
take two more, and run the state histogram. In under two minutes you know whether you
are looking at a deadlock, a leaked lock, exhausted request threads waiting on a
downstream, or a thread leak. **The value here is not the fix. It is that you did not
restart the pod and destroy the evidence** — which is what happens on most teams, and
which guarantees the same incident next week.

**2. Reviewing a pull request that introduces a second lock.**

Someone adds `walletLock` to a method that already holds `inventoryLock`. The lock
idiom is perfect and the diff looks clean. You ask one question: "is there any other
path that takes these two locks, and in what order?" That question — not a style
comment — is what catches the deadlock before it ships. Then you propose routing both
through an ordered helper so the answer stops depending on reviewer memory.

**3. Choosing the guard for a hot read-mostly cache.**

Product wants the reservation cache read on every request. Someone proposes
`ReadWriteLock` because "it's 90% reads". You ask how long the read critical section is
— two field reads — and point out that a read lock is still two atomic writes to one
shared cache line, so at 16 cores it may be slower than a plain lock. You propose three
candidates (plain `ReentrantLock`, `StampedLock` optimistic, immutable snapshot behind
a `volatile` reference), write a JMH harness with `@Threads({1,2,4,8,16,32,64})` and
`@Group` at the real read:write ratio, and let the curve decide. **The deliverable is
the curve, not the opinion** — and this is the thing that makes you the person the team
asks.

---

## Connected topics

**Prerequisites:**

- **52 — Hibernate optimistic and pessimistic locking.** The same trade, one level up.
  `@Version` + retry is `StampedLock`'s optimistic read; `PESSIMISTIC_WRITE` is
  `ReentrantLock`. The single most important difference to carry forward: Postgres
  breaks its own deadlocks by killing a victim; **the JVM never will**.
- **75 — escape analysis and lock elision.** C2 can delete an uncontended lock
  entirely when the object does not escape. This is why a naive `nanoTime` lock
  benchmark measures an empty loop, and it is the first thing to suspect when a lock
  benchmark shows an implausibly good number.
- **85 — `synchronized`, intrinsic monitors, lock inflation.** The baseline this topic
  departs from. The mark word, thin locks, inflation to a fat monitor, and the removal
  of biased locking in JDK 18 are all context for "why isn't `ReentrantLock` faster".
- **86 / 87 — the Java Memory Model and `volatile`.** A lock does two jobs: mutual
  exclusion **and** a happens-before edge. Unlock → lock is the edge. This is why
  reading a plain field outside the lock is a visibility bug even when you do not care
  about atomicity, and why `StampedLock`'s optimistic read still needs `volatile`
  semantics on the state word to be correct.
- **88 — safe publication.** `final` fields give you publication for free, which is
  the argument for immutable state that needs no lock at all — the strongest fix in
  Q3's list.
- **90 — executors and thread pools.** The request-thread pool is what turns one
  deadlocked pair into a service-wide outage. Bounded pools and named threads are
  prerequisites for the diagnosis in this document being possible.
- **92 — `ConcurrentHashMap`.** `computeIfAbsent` is how a lock registry is built
  correctly; a `get`/`put` pair races and hands two threads two different lock objects
  for the same key, which silently removes all mutual exclusion.

**This unlocks:**

- **95 — atomics, CAS and `LongAdder`.** The next step down: what happens when you
  remove the lock entirely and CAS directly. The `lock cmpxchg` retry loop is the same
  instruction AQS uses for `compareAndSetState` — this topic's fast path, examined
  under a microscope.
- **96 — false sharing.** The physical explanation for Trap 6: why a `ReadWriteLock`'s
  read path does not scale even though readers do not exclude each other.
- **97 — coordination primitives.** `CountDownLatch`, `CyclicBarrier`, `Semaphore` and
  `Phaser` are all AQS with a different meaning assigned to `state`. Once you know AQS,
  they are four readings of one integer.
- **98 — the concurrency bug taxonomy.** The diagnostic capstone. This topic gives you
  deadlock in depth; Topic 98 gives you the decision table that tells deadlock from
  livelock from starvation from a thread leak, and the `tryLock` livelock you produced
  in Part D of the drill is its worked example.
- **99 — jcstress.** How you would *prove* the ordered version correct rather than
  failing to reproduce the bug and calling it fixed.
- **100 — `ForkJoinPool`.** Blocking on a lock inside a work-stealing pool worker
  removes that worker from the stealing set — a different and worse failure shape than
  blocking on a plain pool.
- **101 — virtual threads.** The modern reason to prefer `ReentrantLock`: unmounting
  versus pinning. Verify the JEP 491 behaviour on your JDK with Proof 6.
- **109 — the HikariCP pool deadlock.** **The same four Coffman conditions with
  database connections as the resource.** Ten threads each holding one connection and
  each waiting for a second from a pool of ten. No `ReentrantLock` is involved and
  `findDeadlockedThreads()` returns `null`, because a connection pool is not a
  monitor — which is exactly the limitation of the detector that Q2's follow-up probes.
- **111 — resilience patterns.** `Semaphore` as a bulkhead, and the retry-with-jitter
  policy without which the `tryLock` fix becomes a retry storm.
- **121 — liveness versus readiness probes.** A deadlocked pod should be removed from
  the load balancer and **not** restarted, so you can take the dump. This document is
  the clearest argument for why those are two different questions.

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately
hedged rather than asserted: whether your JDK still pins virtual threads inside
`synchronized` (JEP 491 landed in 24; Proof 6 settles it on your machine in two
minutes), the exact internal class structure of `StampedLock` on your build (it has
been rewritten more than once and is not a public contract; `javap` settles it), and
whether your platform exposes hardware cache counters at all — it does not on macOS,
and pretending otherwise would be the exact fabrication this curriculum refuses.
Everything else — AQS being a `volatile int` and a CLH queue, the difference between
BLOCKED and WAITING in a dump, the four Coffman conditions, and the fact that the JVM
reports deadlocks without ever breaking one — has been stable for many years and will
still be true the next time a service goes quiet at 3am.*
