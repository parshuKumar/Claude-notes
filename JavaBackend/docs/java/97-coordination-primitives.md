# 97 — Coordination Primitives: `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`

## Phase: 9 — Concurrency
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: `orderflow`'s payment-gateway bulkhead — the `Semaphore` that limits how many requests may be in flight to a fragile downstream at once, and the latch that makes the catalogue warm-up deterministic at startup.

---

## Before anything else — what is and is not in this document

**I do not have a JVM. Nothing in this document is captured output.** No thread dumps I
ran, no JFR events I observed, no latency numbers, no permit counts read from a live
process.

Specifically, you will not find:

- a thread dump presented as something I captured,
- a throughput or latency figure attributed to a run,
- a permit-availability graph with values on it,
- a "the latch added 3 ms" claim.

Every claim that requires running something is given as the **exact command**, **WHAT TO
LOOK FOR**, and a **"what you see → what it means"** table.

### The one labelled exception

To teach you to *read* a thread dump, I show the **structure** of a `jcmd <pid>
Thread.print` section — the frame ordering and the `parking to wait for` line — with every
value replaced by `<tid>`, `0x...` or `<n>`, carrying the inline label:

> *illustration of the format, not captured output*

Placeholders only.

### What I will assert without running anything

That all four classes in the title are built on `AbstractQueuedSynchronizer`; that a
`CountDownLatch` cannot be reset; that `Semaphore.release()` can push the permit count
above the initial value; that `CyclicBarrier` throws `BrokenBarrierException` when a party
is interrupted. These are API contracts and JDK source facts, stable for over a decade,
and each comes with a command that settles it on your own JDK. If a command disagrees with
me, the command is right.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **All four are `AbstractQueuedSynchronizer` with a different meaning assigned to one
> `volatile int state` — plus, for `Phaser`, its own 64-bit word.**
>
> - **`CountDownLatch`** — `state` is a **remaining count**. `countDown()` decrements it;
>   `await()` blocks while it is greater than zero; at zero **every waiter is released at
>   once and stays released forever**. It is **one-shot, counting DOWN**. There is no
>   reset. Once open, always open.
> - **`CyclicBarrier`** — `state` is a **count of parties still to arrive** at this
>   generation. `await()` both arrives *and* blocks; when the last party arrives, all are
>   released together, an optional barrier action runs, and **the barrier resets itself for
>   the next round**. It is **reusable, counting UP to a party count**.
> - **`Semaphore`** — `state` is a **number of available permits**. `acquire()` takes one,
>   blocking if none is available; `release()` returns one. It is **a permit counter**, and
>   nothing else. It does not know who holds a permit, and the count is not capped at the
>   initial value.
> - **`Phaser`** — a **dynamic-party, multi-phase barrier**: parties may register and
>   deregister at runtime, and it advances through numbered phases. It is `CyclicBarrier`
>   with a variable party count and a phase number you can query and wait on.
>
> Choose by **lifecycle** first: *is this a one-time event or a repeating rendezvous?*
> Then by **what is being counted**: *events, participants, or capacity?*

Two consequences you will use for the rest of your career:

1. **A `CountDownLatch` reused for a second round is a bug that does not throw.** Its
   second `await()` returns instantly, because the latch is already at zero. Your code
   proceeds as if the second round completed. This is the single commonest error in this
   topic and it produces wrong results, not an exception.
2. **A `Semaphore` is a *concurrency* limiter, not a *thread* limiter.** That distinction
   is invisible on platform threads, where in-flight work and threads are the same number.
   It becomes the entire point under virtual threads (Topic 101), where you may have a
   million threads and still want at most 20 concurrent calls to a payment provider. The
   permit count is a statement about the **downstream's** capacity, not yours.

---

## The bridge from what you know

### `PARTIAL ANALOGUE` — `Promise.all` is a latch, `p-limit` is a semaphore

Two real bridges here, and both need one specific unlearning.

**`Promise.all` ≈ `CountDownLatch`.**

```js
// Node — you have written this
await Promise.all([warmProducts(), warmInventory(), warmPricing()]);
console.log('caches warm, ready to serve');
```

```java
// Java — the same shape
CountDownLatch warm = new CountDownLatch(3);
pool.execute(() -> { warmProducts();  warm.countDown(); });
pool.execute(() -> { warmInventory(); warm.countDown(); });
pool.execute(() -> { warmPricing();   warm.countDown(); });
warm.await();
log.info("caches warm, ready to serve");
```

What transfers: "wait until N things finish before proceeding", the fan-out shape, and the
fact that it is **one-shot** — you never reuse a `Promise.all`. That last property is the one
people forget about latches, and your Promise instinct is already right.

Where it breaks, and these differences matter:

| | `Promise.all` | `CountDownLatch` |
|---|---|---|
| What waiting costs | yields the event loop; zero threads blocked | **blocks a real OS thread** in `await()` |
| Results | returns an array of resolved values | **carries no values at all** — a latch is a signal, not a channel |
| Errors | rejects immediately if any promise rejects | **no error propagation whatsoever**. If a task throws before `countDown()`, the latch never reaches zero and `await()` blocks forever |
| Cancellation | none built in, but rejection short-circuits the await | none. `await()` with no timeout is a permanent hang waiting to happen |
| Counting | implicit, from the array length | explicit, in the constructor — and if it disagrees with the number of `countDown()` calls, you hang or you proceed early |

**The error row is the one that will bite you.** `Promise.all` fails loudly; a latch does
not know a task exists, let alone that it failed. That is why every `countDown()` belongs in
a `finally` and every `await()` has a timeout.

**`p-limit` ≈ `Semaphore`.** This is the closest analogue in the topic.

```js
// Node
const limit = pLimit(20);
await Promise.all(orders.map(o => limit(() => gateway.charge(o))));
```

```java
// Java
Semaphore permits = new Semaphore(20);
permits.acquire();
try { gateway.charge(order); }
finally { permits.release(); }
```

Identical concept: at most N of this operation in flight regardless of how many queue
behind it. The differences are the ones you now expect — `acquire()` blocks a real thread
rather than yielding, and **you own the `release()`**, which `p-limit` does for you when the
promise settles. A missing `release()` is a permanent capacity leak (Trap 3); `p-limit` has
no equivalent failure because there is nothing to forget.

**`CyclicBarrier` and `Phaser`: no analogue.** There is no JavaScript construct for "N
participants repeatedly meet, all proceed together, then meet again" — a single-threaded
runtime rarely needs participants advancing in lockstep.

### Now be exact — the distinction the analogy cannot carry

Say this precisely, because interviewers ask it and the analogies blur it:

> - **A latch is one-shot, counting DOWN.** The count is a number of *events*. It reaches
>   zero once and stays there. Waiters and counters are **different sets of threads**.
> - **A barrier is reusable, counting UP to a party count.** The count is a number of
>   *participants*. Every party both signals and waits — `await()` does both. It resets
>   after each round.
> - **A semaphore is a permit counter.** The count is *capacity*. It has no notion of
>   rounds, participants, or completion. Permits go down on acquire and up on release,
>   and the same thread need not do both.
> - **A phaser is a dynamic-party, multi-phase barrier.** Parties join and leave at
>   runtime; phases are numbered and you can wait on a specific one.

The compressed form, which is what you say in an interview:

> **Latch: wait for events, once. Barrier: participants meet, repeatedly. Semaphore:
> capacity. Phaser: a barrier whose party count changes.**

---

## What is this?

Four classes in `java.util.concurrent` that coordinate *when* threads proceed, as opposed
to the locks of Topic 94, which coordinate *who* may touch data.

### `CountDownLatch`

```java
CountDownLatch latch = new CountDownLatch(3);
latch.countDown();                      // decrement; never blocks; no-op at zero
latch.await();                          // block until zero
latch.await(5, TimeUnit.SECONDS);       // block until zero or timeout -> returns boolean
latch.getCount();                       // diagnostics only
```

**One-shot.** No reset, no reuse, no way back above zero; `countDown()` past zero is silently
ignored. Waiters are released together, and any `await()` *after* zero returns immediately.

Two distinct uses: **wait for N tasks** (count = N, workers `countDown()`, coordinator
`await()`s — the `Promise.all` shape), and **start N tasks simultaneously** (count = 1,
workers `await()`, coordinator `countDown()`s once — a starting gun, and the only way to
write a concurrency test that creates real contention rather than measuring thread
creation). Serious tests use both at once.

### `CyclicBarrier`

```java
CyclicBarrier barrier = new CyclicBarrier(4, () -> log.info("round complete"));
int arrivalIndex = barrier.await();     // arrives AND waits; returns arrival index
barrier.await(1, TimeUnit.SECONDS);     // or TimeoutException, which BREAKS the barrier
barrier.reset();                        // new generation; breaks current waiters
barrier.isBroken();
```

**Reusable.** When the last of the N parties arrives, the optional **barrier action** runs on
that last-arriving thread, everyone is released, and the barrier resets for the next round.
The property with no latch equivalent: **`await()` both signals and waits** — there is no
separate "I have arrived" call, so a party that never calls it stalls everyone.

**`BrokenBarrierException`** is what makes barriers hard. If any waiting party is
interrupted, times out, or the barrier is reset, the barrier **breaks** and **every other
waiting party gets `BrokenBarrierException`**. That is deliberate — a rendezvous missing a
participant cannot complete — and it means every barrier user handles a second exception
type and decides what "the round failed" means for work already done.

### `Semaphore`

```java
Semaphore permits = new Semaphore(20);            // non-fair (default)
Semaphore fair    = new Semaphore(20, true);      // FIFO, lower throughput

permits.acquire();                                 // blocks; interruptible
permits.acquireUninterruptibly();
boolean got = permits.tryAcquire();                // immediate, no blocking
boolean got2 = permits.tryAcquire(50, TimeUnit.MILLISECONDS);   // with a budget
permits.release();
permits.acquire(5); permits.release(5);            // multiple permits at once
permits.availablePermits();                        // diagnostics; instantly stale
permits.drainPermits();                            // take all remaining
```

A counter of **capacity**, with three properties that surprise people:

1. **It has no owner.** Unlike a lock, a semaphore does not record which thread holds a
   permit. Thread A may acquire and thread B may release. That is a feature (it is how you
   implement a resource pool) and a hazard (nothing detects a double release).
2. **`release()` is not capped at the initial value.** `new Semaphore(1)` followed by two
   `release()` calls leaves **two** permits. There is no exception. Your bulkhead of 20 is
   now a bulkhead of 21, permanently, and nothing anywhere reports it. This is Trap 4.
3. **A binary semaphore is not a reentrant lock.** `new Semaphore(1)` provides mutual
   exclusion, but a thread that acquires twice deadlocks against itself, because there is
   no hold count and no owner. Use `ReentrantLock` when you want a lock.

The canonical use, and the one you will ship: **a bulkhead** (Topic 111) — limit in-flight
calls to a fragile downstream.

### `Phaser`

```java
Phaser phaser = new Phaser(1);                    // register self
phaser.register();                                // add a party at runtime
phaser.arriveAndAwaitAdvance();                   // like CyclicBarrier.await()
phaser.arriveAndDeregister();                     // arrive and leave permanently
int phase = phaser.arrive();                      // arrive WITHOUT waiting
phaser.awaitAdvance(phase);                       // wait for a specific phase to pass
phaser.getPhase();                                // current phase number (wraps at MAX_VALUE)
```

`CyclicBarrier` fixes its party count at construction; `Phaser` allows registration and
deregistration at any time, separates "arrive" from "wait", tracks a phase number, and can be
tiered for scalability. Override `onAdvance(phase, parties)` for a per-phase action and to
signal termination by returning `true`.

**Honest positioning:** more capable, much less used. Most production code wants a latch or a
semaphore. Reach for `Phaser` when the party count genuinely varies at runtime or you need
multiple phases — and expect to explain it in review, because fewer people can read it.

### Also worth knowing

`Exchanger<V>` is a two-party rendezvous that swaps objects; rarely used.
`CompletableFuture.allOf` (Topic 91) and `StructuredTaskScope` (Topic 102) are the modern
answers to what a `CountDownLatch` usually does in application code: if you want the tasks'
**results** or **error propagation**, a latch is the wrong tool. Latches stay right for
pure signals — startup gates, test starting guns, shutdown.

---

## Why does it matter?

**1. The one-shot/reusable distinction is a correctness property, not a preference.**

A latch used for a repeating round does not throw; it silently stops blocking. Your batch
job's round 2 proceeds before round 2's work exists, reads partial state, and writes wrong
results. There is no exception, no log line, and no stack trace pointing anywhere near the
latch. Getting this distinction right at design time is the difference between a working
system and a silently wrong one.

**2. A `Semaphore` is the right shape for a bulkhead, and a thread pool is the wrong one.**

The common instinct is to limit concurrency with a pool size: "only 20 threads may call the
payment gateway, so at most 20 calls are in flight". That works on platform threads by
coincidence — because one blocked call occupies exactly one thread. It stops working the
moment the calls are made from virtual threads, from a reactive pipeline, or from a shared
pool, because the number of in-flight calls is no longer equal to the number of threads.

A `Semaphore` expresses the constraint **directly**: *at most 20 concurrent calls to this
downstream.* It is independent of how many threads exist, works identically on platform and
virtual threads, and — crucially — **the number 20 is a fact about the downstream, not about
you**. That is the sentence that makes it the right abstraction: it lives at the boundary it
describes.

**3. Coordination bugs are invisible where you look first.**

A latch that never reaches zero parks a thread in `await()` — indistinguishable from a
healthy idle worker. A semaphore leak parks threads in `acquire()` with no lock anywhere, so
the deadlock detector reports nothing. A broken barrier throws in a worker whose stack trace
never mentions the cause. Knowing each primitive's failure shape is most of this document's
value.

---

## Machine-level reality

### AQS: one integer, four meanings

Topic 94 established `AbstractQueuedSynchronizer`: a `volatile int state` plus a CLH-style
FIFO queue of parked threads. Acquiring is a CAS on `state`; failing that, you are appended
to the queue and parked with `LockSupport.park`. Releasing sets `state` and unparks a
successor.

Everything in this topic is that machine with a different meaning for `state`, and a
different rule for when a waiter may proceed:

| Class | What `state` means | Acquire succeeds when | Release does |
|---|---|---|---|
| `ReentrantLock` (T94) | reentrant hold count | `state == 0`, or you are the owner | decrement; at 0, unpark one successor |
| **`CountDownLatch`** | **remaining count** | **`state == 0`** | `countDown()` CAS-decrements; at 0, **unpark ALL** |
| **`Semaphore`** | **available permits** | `state >= permits requested`; CAS it down | CAS `state` up; unpark successors that now fit |
| **`CyclicBarrier`** | *(not AQS directly)* — uses a `ReentrantLock` + `Condition`, with `count` and a `Generation` object | the last party arrives | signal the condition, install a new `Generation` |
| **`Phaser`** | its own `volatile long state`: phase, parties and unarrived packed into one 64-bit word | all registered parties have arrived | advance the phase, unpark waiters |

Three structural facts fall out of that table.

**`CountDownLatch` uses AQS's *shared* mode.** `ReentrantLock` uses exclusive mode: one
winner, one successor unparked. A latch uses shared mode: when `state` hits zero the release
propagates down the queue and **every** waiter is unparked — correct, because an event has
occurred and everyone waiting for it should see it. It is also why a latch can never go back
up: shared-mode acquires succeed unconditionally once the gate is open, so "closing" it has
no coherent meaning.

**`Semaphore` is AQS shared mode with a quantity.** `tryAcquireShared` computes
`available - permits` and CASes if that is non-negative. Non-fair mode lets an arriving
thread barge past queued waiters — faster, because no handoff is needed, and a source of
starvation (Topic 98). Fair mode checks `hasQueuedPredecessors()` first.

**`CyclicBarrier` is the odd one out** — not an AQS subclass but a `ReentrantLock` plus a
`Condition` plus a `Generation`. The `Generation` is what makes it reusable and breakable:
breaking marks the current generation broken and signals everyone; resetting installs a new
one. Waiters check which generation they belong to on wake-up.

### How each appears in a thread dump

This is the diagnostic table. It is what you will actually use.

| Waiting in | Thread state | The `parking to wait for` line names | The frames above it |
|---|---|---|---|
| `CountDownLatch.await()` | **`WAITING (parking)`** | `java.util.concurrent.CountDownLatch$Sync` | `AbstractQueuedSynchronizer.acquireSharedInterruptibly` → `CountDownLatch.await` |
| `CountDownLatch.await(t, u)` | **`TIMED_WAITING (parking)`** | same | `...tryAcquireSharedNanos` |
| `Semaphore.acquire()` | **`WAITING (parking)`** | `java.util.concurrent.Semaphore$NonfairSync` (or `$FairSync`) | `...acquireSharedInterruptibly` → `Semaphore.acquire` |
| `Semaphore.tryAcquire(t, u)` | **`TIMED_WAITING (parking)`** | same | `...tryAcquireSharedNanos` |
| `CyclicBarrier.await()` | **`WAITING (parking)`** | `AbstractQueuedSynchronizer$ConditionObject` — **not** the barrier | `ConditionObject.await` → `CyclicBarrier.dowait` |
| `Phaser.arriveAndAwaitAdvance()` | **`WAITING (parking)`** or briefly `RUNNABLE` (it spins first) | `java.util.concurrent.Phaser$QNode` | `Phaser.internalAwaitAdvance` |

Three rules follow. **None of these is `BLOCKED`** — that means an intrinsic monitor, and
every class here parks instead; a dump with no `BLOCKED` threads is not a dump with no
problem. **The class in the `parking to wait for` line is your best clue, and `CyclicBarrier`
denies you it** — it names a `ConditionObject`, exactly as `BlockingQueue` does (Topic 93),
so you must read the frame above. And **`WAITING` in `acquire()` is ambiguous**: it is what
correct backpressure and a permit leak both look like. The dump cannot separate them;
`availablePermits()` can, which is why the Measurement gauge is not optional.

### The exact difference between BLOCKED, WAITING and TIMED_WAITING

Restated from Topic 94 because it is the load-bearing skill of this phase, and because this
topic adds cases:

| State | What put it there | What it implies here |
|---|---|---|
| **`RUNNABLE`** | executing, **or** in a native call the JVM cannot see (socket reads!) | Not proof of CPU use. `Phaser` briefly spins before parking, so a phaser waiter can appear RUNNABLE. |
| **`BLOCKED`** | waiting to enter a `synchronized` block — **intrinsic monitors only** | Never produced by this topic's classes. If you see it alongside them, you have a *separate* monitor problem. |
| **`WAITING`** | `Object.wait()`, `Thread.join()`, and **`LockSupport.park()` — every `j.u.c` construct** | Where every un-timed latch, semaphore, barrier and phaser waiter lives. **Can be permanent.** A latch that never reaches zero lands here forever. |
| **`TIMED_WAITING`** | `sleep`, `wait(n)`, `parkNanos`, **`await(t,u)`**, **`tryAcquire(t,u)`** | **This thread will wake up.** It cannot be part of a permanent hang. Which is exactly why every production `await` and `acquire` should have a timeout. |

**The design rule that follows, and it is the most valuable sentence in this document:**

> Every `await()` and `acquire()` in production code should have a timeout, because a
> timeout converts an unbounded `WAITING` into a bounded `TIMED_WAITING` — which converts a
> permanent hang into an error you can see, count, alert on, and retry.

### `[JAVA 25]` and virtual threads

- All four park via `LockSupport`, so under virtual threads (Topic 101) a waiter
  **unmounts** its carrier rather than blocking it. A million virtual threads may wait on 20
  permits and consume essentially no carrier time — which is precisely why `Semaphore`, not
  pool size, is the correct bulkhead in a Loom world.
- `Phaser` **spins briefly before parking** — fine on platform threads, mildly wasteful on
  virtual ones. Not a reason to avoid it; a reason to keep it off a hot path.
- Nothing in `[JAVA 25]` changes any semantics here.

---

## Concurrency trace

**Before any correct code.** This is `orderflow`'s nightly reconciliation batch — the job
that walks 1M orders in rounds, comparing each round against the payment provider's
statement before moving to the next.

**The setup.** Four worker threads process one round each night: each takes a shard of
orders, reconciles it, and the round must complete for **all four** shards before the next
round starts, because round N+1's balances depend on round N's corrections.

The author needed "wait for all four to finish", reached for `CountDownLatch`, and — being
careful — hoisted it out of the loop so it would not be re-allocated per round:

```java
// The bug. One latch, three rounds.
CountDownLatch roundDone = new CountDownLatch(4);

for (int round = 1; round <= 3; round++) {
    for (int shard = 0; shard < 4; shard++) {
        pool.execute(() -> { reconcile(round, shard); roundDone.countDown(); });
    }
    roundDone.await();                    // "wait for this round"
    applyCorrections(round);
}
```

It passed review. It passed the unit test, which runs one round.

| Step | Thread A — `batch-coordinator` | Threads B1–B4 — `recon-worker-1..4` | Latch count / outcome |
|---|---|---|---|
| 1 | round 1: submits 4 tasks, calls `roundDone.await()` | — | count = 4 · A parked, `WAITING (parking)` on `CountDownLatch$Sync` |
| 2 | parked | B1 finishes shard 0 → `countDown()` | count = 3 |
| 3 | parked | B2, B3 finish → `countDown()` ×2 | count = 1 |
| 4 | parked | B4 finishes → `countDown()` | **count = 0 — the gate opens** |
| 5 | `await()` returns · `applyCorrections(1)` | idle | **Round 1 correct.** Everything so far is right. |
| 6 | round 2: submits 4 tasks | B1–B4 begin round 2 shards | count = **still 0** — a latch never goes back up |
| 7 | calls `roundDone.await()` → **returns immediately** | still reconciling round 2, 0% done | **A does not wait.** No exception. No warning. |
| 8 | `applyCorrections(2)` runs against **round 1's** data | round 2 workers still running | **Corrections applied to results that do not exist yet** |
| 9 | round 3: submits 4 tasks, `await()` returns instantly again | rounds 2 and 3 workers now running **concurrently** | count still 0 · `countDown()` past zero is a silent no-op |
| 10 | `applyCorrections(3)` | round 2 and 3 workers interleaving writes to the same shard state | **Data race on shard state, on top of the ordering bug** |
| 11 | job logs "reconciliation complete" and exits 0 | workers still running; pool `shutdown()` is called while they work | Batch reports **success** |

**Outcome, in business terms.**

The reconciliation job reports success every night. Rounds 2 and 3 apply corrections
computed from round 1's data, so wallet balances drift from the payment provider's ledger
by a small amount each night — small enough that daily checks pass. The drift compounds.

Six weeks later, finance notices £40,000 of unexplained variance between `orderflow`'s
wallet balances and the provider's statement. There is no error to find: no exception was
ever thrown, no log line is out of place, the batch's own success metric has been green
every night, and the code — one latch, hoisted out of a loop for efficiency — reads as
careful.

The recovery is a six-week replay of reconciliation from provider statements, done by hand,
during which the wallet balances shown to customers are known to be wrong.

**The failure mode to name precisely:** a `CountDownLatch` reused across rounds does not
throw, does not warn, and does not block. It **stops being a barrier and becomes nothing**,
and code that has stopped synchronising looks exactly like code that never needed to.

### The same three rounds, with the right primitive

`CyclicBarrier` is reusable by construction: it counts *up* to a party count and resets.

| Step | Thread A — coordinator (a party) | Threads B1–B3 — workers (parties) | Barrier state / outcome |
|---|---|---|---|
| 1 | round 1 work, then `barrier.await()` | each does round 1 work, then `await()` | parties=4 · arrivals accumulate |
| 2 | parked, `WAITING` on the barrier's `ConditionObject` | B1, B2 arrive and park | 2 of 4 arrived |
| 3 | parked | B3 arrives **last** → runs the **barrier action** `applyCorrections(1)` on B3's thread | all released together |
| 4 | resumes round 2 | resume round 2 | **barrier auto-resets: new generation, arrivals back to 0** |
| 5 | round 2 work, `await()` | round 2 work, `await()` | **it blocks properly this time** |
| 6 | last arrival runs `applyCorrections(2)` | — | Round 2 correct |
| 7 | round 3, identically | — | Round 3 correct |
| 8 | if B2 is interrupted mid-round | A, B1, B3 receive **`BrokenBarrierException`** | **The round fails loudly.** No silent partial result. |

**Outcome, in business terms.** Every round waits for every shard. Corrections are computed
from the round that produced them. Balances match the provider's ledger. And when something
does go wrong — a worker interrupted by a deploy mid-round — **every party finds out** via
`BrokenBarrierException`, the batch fails visibly, and the on-call replays one night rather
than six weeks.

**Read the difference in the right terms.** The latch version was not *slower* or *less
elegant*. It had stopped synchronising anything, and there was no signal of that. The
barrier version fails loudly. **Choosing a primitive whose failure is loud is a design
decision, not a style one.**

---

## Example 1 — minimal

The two latches that every concurrency test needs, and the one-line semaphore.

```java
import java.util.concurrent.*;

public final class MinimalCoordination {

    /** The starting gun + finish line. This is how you write a contention test. */
    static void concurrentDecrements(int threads) throws InterruptedException {
        CountDownLatch start = new CountDownLatch(1);        // the gun
        CountDownLatch done  = new CountDownLatch(threads);  // the finish line
        ExecutorService pool = Executors.newFixedThreadPool(threads);

        for (int i = 0; i < threads; i++) {
            pool.execute(() -> {
                try {
                    start.await();               // every thread parks here...
                    inventory.decrement("SKU-1001", 1);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    done.countDown();            // ALWAYS in finally
                }
            });
        }

        start.countDown();                       // ...and all are released at once

        if (!done.await(10, TimeUnit.SECONDS)) { // ALWAYS with a timeout
            throw new AssertionError("workers did not finish within 10s");
        }
        pool.shutdown();
    }

    /** A bulkhead: at most 20 concurrent calls to the gateway, however many callers. */
    private static final Semaphore GATEWAY = new Semaphore(20);

    static PaymentResult charge(Order order) throws InterruptedException {
        if (!GATEWAY.tryAcquire(100, TimeUnit.MILLISECONDS)) {
            throw new DownstreamAtCapacityException("payment gateway bulkhead full");
        }
        try {
            return gateway.charge(order);
        } finally {
            GATEWAY.release();                   // ALWAYS in finally. No exceptions.
        }
    }
}
```

**What to notice:**

- **Two latches, not one.** Without the `start` latch, thread 1 finishes before thread 20
  is created and you have measured thread creation, not contention. This is the single most
  common defect in hand-written concurrency tests — including ones that "prove" a race does
  not exist.
- **`countDown()` in `finally`.** If `decrement` throws, the count must still drop or the
  coordinator waits forever for a task that has already died. **Every `countDown()` belongs
  in a `finally`.**
- **`done.await(10, SECONDS)` returns a `boolean`, and it is checked.** An un-timed
  `await()` in a test is a build that hangs in CI until someone kills it. An ignored return
  value is a test that passes when the workers never ran.
- **`release()` in `finally`, and nothing between `tryAcquire` and `try`.** If an exception
  occurs after acquiring but before entering the `try`, the permit is gone forever — the
  same discipline as `ReentrantLock` in Topic 94, and the same reason.
- **`tryAcquire` with a timeout, not `acquire()`.** `acquire()` waits forever. Under a
  downstream outage that parks every request thread indefinitely, which is Topic 93's Trap 4
  arriving through a different door.

### The three one-character variations that break it

`new CountDownLatch(0)` — the gun has already fired, so there is no contention at all.
`done.await()` without a timeout — CI hangs forever instead of failing. `GATEWAY.release()`
outside the `finally` — a permit leak on every exception path. **None of these throws**, and
all three produce a program that appears to work.

---

## Example 2 — production scenario (on the project spine)

### The constraints, before any code

| Constraint | Value | Source |
|---|---|---|
| Request rate | 400 rps baseline | Topic 65 |
| Payment gateway concurrency limit | **20 concurrent connections**, contractual | the provider's integration docs |
| Gateway latency | 200 ms p50, 2 s p99 | provider SLA |
| Order p99 budget | 250 ms | Topic 65 baseline |
| Behaviour at capacity | **shed with a clear error**, never queue unboundedly | product decision |
| Startup requirement | do not accept traffic until the product-catalogue cache is warm | operations |

**The number 20 is not ours.** It is a property of the provider's system, written into a
contract — which is what makes `Semaphore` the right abstraction: the constraint is "20
concurrent calls to *them*", not "20 threads in *us*", and those stop being the same number
the day the service moves to virtual threads.

By Little's Law, 20 permits at 200 ms sustains `20 / 0.2 = 100` charges per second. At 400
rps, if every order charged the gateway, **you are over capacity by 4× by design** — which is
exactly why the shed-versus-queue decision had to be made explicitly, in advance.

### The code that ships and takes the service down with the provider

```java
@Service
public class PaymentService {

    // No limit at all. Every request thread calls the gateway directly.
    public PaymentResult charge(Order order) {
        return gateway.charge(order);          // 200 ms p50, 2 s p99
    }
}
```

Under the Topic 65 baseline this works: 400 rps × 200 ms = 80 concurrent calls. The
provider's limit is 20. It works because the provider does not enforce the limit strictly
until it is under pressure — and then, when a spike arrives and the provider starts queueing
at *their* end, latency goes to 2 s, in-flight calls climb to `400 × 2 = 800`, Tomcat's
200 request threads are all inside `gateway.charge`, and every endpoint in `orderflow`
returns 503 — including `GET /products`, which touches no payment code.

**One slow dependency has consumed the shared request-thread pool.** That is Topic 93's
Trap 4 and Topic 111's bulkhead argument, and a `Semaphore` is the fix for both.

### Fix 1 — the bulkhead

```java
@Service
public class PaymentService {

    private static final int GATEWAY_LIMIT   = 20;   // contractual, from the provider
    private static final long ACQUIRE_TIMEOUT_MS = 100;   // 40% of the p99 budget

    private final Semaphore permits = new Semaphore(GATEWAY_LIMIT);
    private final PaymentGateway gateway;
    private final Counter shed;
    private final Timer waitTime;

    PaymentService(PaymentGateway gateway, MeterRegistry meters) {
        this.gateway = gateway;
        this.shed = Counter.builder("orderflow.gateway.shed")
                .description("charges rejected because the bulkhead was full")
                .register(meters);
        this.waitTime = Timer.builder("orderflow.gateway.permit.wait")
                .description("time spent waiting for a bulkhead permit")
                .register(meters);
        // Topic 118: an un-gauged bulkhead is an invisible bulkhead.
        Gauge.builder("orderflow.gateway.permits.available", permits,
                        Semaphore::availablePermits).register(meters);
        Gauge.builder("orderflow.gateway.permits.waiting", permits,
                        Semaphore::getQueueLength).register(meters);
    }

    public PaymentResult charge(Order order) throws InterruptedException {
        long t0 = System.nanoTime();
        boolean acquired = permits.tryAcquire(ACQUIRE_TIMEOUT_MS, TimeUnit.MILLISECONDS);
        waitTime.record(System.nanoTime() - t0, TimeUnit.NANOSECONDS);

        if (!acquired) {
            shed.increment();
            throw new DownstreamAtCapacityException(
                    "payment gateway at capacity; retry shortly");   // -> HTTP 503 + Retry-After
        }
        try {
            return gateway.charge(order);
        } finally {
            permits.release();          // the ONLY release. In finally. Always.
        }
    }
}
```

**Why each part is load-bearing:**

| Element | What it buys |
|---|---|
| `Semaphore(20)` | The contractual limit expressed as itself, in one place, with a comment naming its source |
| `tryAcquire(100ms)` | Callers wait a bounded time. `TIMED_WAITING`, never `WAITING`. No permanent hang is possible. |
| `throw` on failure | **Load shedding.** The request fails fast with a retryable status instead of consuming a request thread for two seconds. |
| `release()` in `finally` | The only correct place. Every exception path returns the permit. |
| `permits.available` gauge | You can see the bulkhead working — or leaking (Trap 3), which is otherwise invisible. |
| `permits.waiting` gauge (`getQueueLength`) | Distinguishes "at capacity, callers queueing" from "leaked, nobody can get in". |
| `shed` counter | The alert. A non-zero shed rate means the downstream is the constraint. |
| `waitTime` timer | Tells you whether the 100 ms timeout is right. If the p99 wait is 95 ms, you are close to the edge. |

**What this deliberately does not do:** it does not retry, it does not queue, and it does
not fall back. Those are Topic 111's decisions, layered on top. The bulkhead's single job
is to guarantee that no more than 20 calls are in flight and that everyone else finds out
quickly.

### Fix 2 — the startup gate, where a latch is exactly right

The catalogue cache must be warm before the service accepts traffic. That is genuinely
one-shot, so a latch is correct:

```java
@Component
public class CatalogueWarmup implements HealthIndicator {

    private final CountDownLatch warm = new CountDownLatch(3);   // products, inventory, pricing
    private volatile Throwable failure;

    @PostConstruct
    void start() {
        warmAsync("products",  cache::warmProducts);
        warmAsync("inventory", cache::warmInventory);
        warmAsync("pricing",   cache::warmPricing);
    }

    private void warmAsync(String name, Runnable task) {
        Thread.ofVirtual().name("warmup-" + name).start(() -> {
            try { task.run(); }
            catch (Throwable t) { failure = t;                 // a latch carries no error
                                  log.error("warmup {} failed", name, t); }
            finally { warm.countDown(); }                       // ALWAYS
        });
    }

    /** Topic 121: readiness, not liveness. Do not restart a pod that is merely warming. */
    @Override public Health health() {
        if (warm.getCount() > 0)
            return Health.down().withDetail("warmup.remaining", warm.getCount()).build();
        return failure == null ? Health.up().build()
                               : Health.down().withException(failure).build();
    }

    public void awaitWarm() throws InterruptedException {
        if (!warm.await(30, TimeUnit.SECONDS))
            throw new IllegalStateException("catalogue warm-up did not complete in 30s");
    }
}
```

**Three things this gets right that the naive version does not:**

1. **`countDown()` in `finally`.** If `warmProducts` throws, the count still drops. Without
   it, readiness never turns green, Kubernetes kills the pod when the startup probe's budget
   expires, and the actual warm-up error never surfaces because the pod dies before it can
   be reported.
2. **The error is captured separately.** A latch carries no values and no exceptions:
   `await()` returning tells you the tasks *finished*, not that they *worked*. That gap is
   exactly what Topic 102's `StructuredTaskScope` closes, and if this warm-up produced
   results rather than side effects, a structured scope would be the better tool.
3. **`getCount()` is exposed for diagnostics only.** It tells an operator how many warm-ups
   are outstanding; it never makes a control-flow decision, because a check-then-act on
   `getCount()` races exactly like Topic 92's `containsKey`-then-`put`.


### Fix 3 — the reusable case: a `CyclicBarrier` for the reconciliation batch

For the trace's nightly batch, the barrier version in outline:

```java
private final CyclicBarrier roundBarrier =
        new CyclicBarrier(SHARDS, applier::applyCurrentRound);   // action runs on the LAST arriver

void runShard(int shard) {
    for (int round = 1; round <= ROUNDS && !failed; round++) {
        try {
            reconcile(round, shard);
            roundBarrier.await(5, TimeUnit.MINUTES);        // bounded. Always.
        } catch (BrokenBarrierException e) {                // another party died
            failed = true; log.error("shard {} abandoning round {}", shard, round, e); return;
        } catch (TimeoutException e) {
            failed = true; roundBarrier.reset();            // free the others deliberately
            log.error("shard {} timed out in round {}", shard, round, e); return;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); return;     // also breaks it for the others
        }
    }
}
```

**The two properties that make this correct and the latch version wrong:** it **resets**, so
round 2's `await()` genuinely blocks; and it **fails loudly** — one party dying gives every
other party a `BrokenBarrierException`, so there is no interleaving in which some shards
proceed and others do not.

The barrier action is worth noticing: it runs on the last-arriving thread with every other
party still parked, which is a safe place to mutate shared round state without a lock.

**And the honest limitation:** a `CyclicBarrier` needs its party count at construction. If
shards were added or removed mid-run you would need a `Phaser` — `register()` /
`arriveAndDeregister()` — and that is the one production shape where `Phaser` earns its
complexity.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a `CountDownLatch` reused for a second round

**Wrong approach.** One latch, hoisted out of a loop, awaited each round. The Concurrency
trace above.

**Exact symptom.** **There is no symptom.** No exception, no log line, no metric, no
`BLOCKED` thread, no `WAITING` thread — the second `await()` returns immediately. The
program runs *faster* than the correct version, which is often how it is noticed, and it is
noticed as good news.

The damage appears in the data: results computed from stale inputs, work applied out of
order, and concurrent rounds racing on shared state. In `orderflow`'s reconciliation that
was £40,000 of wallet drift discovered six weeks later by finance.

The only pre-production signal is a test that runs **more than one round** — which is
precisely the test nobody writes, because the single-round test passes and looks
sufficient.

**Root cause.** `CountDownLatch` is one-shot **by design**. Its AQS `state` counts down to
zero and shared-mode acquires succeed unconditionally thereafter. There is no `reset()`,
and `countDown()` past zero is a documented no-op. The class is behaving exactly as
specified; the design chose a one-shot primitive for a repeating problem.

**Fix.** Choose by lifecycle, and let a compile-time property enforce it:

```java
// (a) Reusable rounds -> CyclicBarrier. It resets itself. This is the fix.
CyclicBarrier round = new CyclicBarrier(4, this::applyCorrections);

// (b) If a latch is genuinely right, make reuse IMPOSSIBLE: construct it inside the loop.
for (int round = 1; round <= 3; round++) {
    CountDownLatch done = new CountDownLatch(4);   // final, effectively scoped to one round
    // ... submit 4 tasks that countDown() in finally ...
    if (!done.await(5, TimeUnit.MINUTES)) throw new IllegalStateException("round " + round);
}
```

**(b) is the underrated fix.** A latch declared inside the loop body cannot be reused,
because it does not outlive the round. Allocation is not the concern people imagine; one
object per round is nothing. **Scoping the object to its lifecycle removes the entire class
of bug**, and it removes it at compile time rather than at review time.

**And the review rule:** a `CountDownLatch` field on a class is a smell. A latch's natural
home is a local variable. If it is a field, ask what happens on the second call.

### Trap 2 — `await()` and `acquire()` without a timeout

**Wrong approach.**

```java
done.await();              // no timeout
permits.acquire();         // no timeout
barrier.await();           // no timeout
```

**Exact symptom.** A thread parked forever in `WAITING (parking)`. In a batch job the
process never exits and CI hangs until a job timeout kills it, with no output explaining
why. In a service, request threads accumulate in `WAITING` until the Tomcat pool is
exhausted and every endpoint 503s — including ones that touch nothing related.

The thread dump shows the truth: `WAITING (parking)`, `parking to wait for` a
`Semaphore$NonfairSync`, under a `Semaphore.acquire` frame, on a thread named
`http-nio-8080-exec-*`. Proof 2 shows that section's exact shape.

**Root cause. An un-timed `await`/`acquire` is a promise that the awaited condition will
eventually occur. Nothing enforces that promise. A task that throws before its
`countDown()`, a permit that is never released, a barrier party that never arrives — each
makes the wait permanent. `WAITING` has no deadline; only `TIMED_WAITING` does.

**Fix.** A timeout on every wait in production code, and a decision about what the timeout
*means*:

```java
if (!done.await(30, TimeUnit.SECONDS))
    throw new IllegalStateException("warmup did not complete in 30s");

if (!permits.tryAcquire(100, TimeUnit.MILLISECONDS)) {
    shed.increment();
    throw new DownstreamAtCapacityException("gateway at capacity");
}

barrier.await(5, TimeUnit.MINUTES);   // throws TimeoutException, and BREAKS the barrier
```

**Two things people get wrong even after adding the timeout:**

- **Ignoring the `boolean`.** `latch.await(30, SECONDS)` that discards its return value is
  worse than no timeout, because the code now proceeds as if the wait succeeded. Treat an
  ignored return here as a build error.
- **Forgetting that `CyclicBarrier.await(t, u)` breaks the barrier on timeout.** That is
  correct — a rendezvous missing a party cannot complete — but it means the *other* parties
  get `BrokenBarrierException`, and you must handle that too.

### Trap 3 — `Semaphore.acquire()` without a matching `release()` in `finally`

**Wrong approach.**

```java
permits.acquire();
PaymentResult r = gateway.charge(order);   // throws on a 5xx from the provider
permits.release();                          // never reached on the exception path
return r;
```

**Exact symptom.** A slow, monotonic strangulation, and this is the shape to memorise:

1. Everything is normal. Occasionally the provider returns a 5xx.
2. Each failure permanently destroys one permit. Nothing reports this.
3. Throughput of the payment path drifts down over hours or days. Nobody connects it to
   anything.
4. `availablePermits()` reaches zero and **stays** there.
5. Every subsequent caller parks in `acquire()` forever. Request threads accumulate.
6. The service 503s on **every** endpoint, and the payment gateway is completely healthy.
7. A restart "fixes" it, and it recurs on the same schedule. The incident is closed as
   "transient".

There is **no deadlock** — `ThreadMXBean.findDeadlockedThreads()` returns null, because a
semaphore is not a monitor and has no owner to build a cycle from. The threads are simply
waiting for permits that no longer exist.

**Root cause.** A semaphore has no owner and no automatic release. Unlike `synchronized`,
whose monitor is released when the frame unwinds for any reason, a permit is returned only
by an explicit `release()`. Any path that skips it — an exception, an early `return`, a
`break`, a `continue` — leaks a permit permanently.

**Fix.** The identical discipline as `ReentrantLock` in Topic 94:

```java
if (!permits.tryAcquire(100, TimeUnit.MILLISECONDS)) { shed.increment(); throw ...; }
try {
    return gateway.charge(order);
} finally {
    permits.release();          // the ONLY release. Nothing between tryAcquire and try.
}
```

Then make the leak detectable, because the fix above is only as good as its review:

```java
Gauge.builder("orderflow.gateway.permits.available", permits, Semaphore::availablePermits)
     .register(meters);
```

**Alert on `available == 0` sustained for more than a minute, and on the gauge's daily
maximum falling below the configured limit.** That second alert is the one that catches a
slow leak: if the maximum available permits ever observed today is 18 when the limit is 20,
you have leaked two, and you know it before the count reaches zero.

**And the structural fix that makes the whole trap impossible** — wrap the discipline once:

```java
public <T> T withPermit(Callable<T> work) throws Exception {
    if (!permits.tryAcquire(timeoutMs, MILLISECONDS)) throw new DownstreamAtCapacityException();
    try { return work.call(); } finally { permits.release(); }
}
```

Now there is exactly one `acquire`/`release` pair in the codebase and it is correct. Every
caller uses `withPermit(...)`. This is the same argument as Topic 94's ordered-lock helper:
**do not rely on every future author remembering; remove the opportunity.**

### Trap 4 — releasing more permits than you acquired

**Wrong approach.** A refactor moves `release()` inside a retry loop:

```java
for (int attempt = 0; attempt < 3; attempt++) {
    try { return gateway.charge(order); }
    finally { permits.release(); }        // releases once PER ATTEMPT, acquired once
}
```

**Exact symptom.** The inverse of Trap 3, and harder to see. The bulkhead silently **grows**:
`availablePermits()` reports 23 when the semaphore was constructed with 20. No exception —
`Semaphore` explicitly allows this. Downstream, the provider starts rejecting connections
because you are exceeding the contractual limit, and the investigation looks at the
provider, the network and the connection pool, never at a bulkhead everyone knows is
"configured to 20".

**Root cause.** `release()` increments `state` unconditionally: no upper bound, no owner
check, no exception. That is deliberate — it is what lets a semaphore be used as a
producer-consumer signal, where one thread releases what another acquired. The cost of that
flexibility is that a double release is undetectable by the class.

**Fix.** Structurally, the `withPermit(...)` wrapper above makes acquire and release a
matched pair by construction. Detectively, **alert on `availablePermits() > configuredLimit`** —
it should be impossible, so if it fires you have a bug and want to know within a minute. And
if you need a genuinely hard cap, `Semaphore` will not give you one: use a bounded
`BlockingQueue` of permit tokens (Topic 93), where `offer()` returning `false` *is* the cap.


### Trap 5 — using a thread pool where you needed a semaphore

**Wrong approach.**

```java
// "Only 20 concurrent gateway calls" implemented as a pool of 20 threads.
private final ExecutorService gatewayPool = Executors.newFixedThreadPool(20);

public PaymentResult charge(Order o) throws Exception {
    return gatewayPool.submit(() -> gateway.charge(o)).get();   // and .get() blocks the caller!
}
```

**Exact symptom.** Three, arriving in order as the system evolves:

1. **Immediately:** you have not limited anything from the caller's point of view. The
   request thread calls `.get()` and blocks, so you now consume *two* threads per in-flight
   charge — one Tomcat thread and one pool thread. The pool bounds the gateway calls;
   nothing bounds the callers.
2. **With `Executors.newFixedThreadPool`:** the queue is an **unbounded**
   `LinkedBlockingQueue` (Topic 93, Trap 1). Under a provider slowdown, work piles up in
   heap with no limit and no rejection, and the "limit of 20" silently became "20
   concurrent, unbounded queued".
3. **On migrating to virtual threads (Topic 101):** the whole construct collapses. The
   right change is `Executors.newVirtualThreadPerTaskExecutor()`, which has **no
   concurrency limit at all** — so the migration silently removes the bulkhead entirely and
   you discover it by exceeding the provider's contractual limit in production.

**Root cause.** A thread pool bounds **threads**. The requirement bounds **concurrent calls
to a downstream**. Those two numbers coincide on platform threads and diverge everywhere
else. Encoding a downstream's capacity as your own thread count is a category error that
happens to work until the threading model changes.

**Fix.** Express the constraint as what it is:

```java
private final Semaphore gatewayPermits = new Semaphore(20);   // the provider's limit
```

No extra threads, no hidden queue, no hand-off, correct on platform *and* virtual threads,
and the number is annotated with where it came from. **This is why Topic 111's bulkhead is
a permit count and not a pool size**, and it is the single most useful transferable idea in
this document.

---

## Hands-on proof

No JVM ran here. These are the commands; the outputs are yours.

### Setup

```bash
java -version && mkdir -p /tmp/coord && cd /tmp/coord
```

### Proof 1 — a reused latch does not block

```java
// Reuse.java
import java.util.concurrent.*;
public class Reuse {
    public static void main(String[] a) throws Exception {
        CountDownLatch latch = new CountDownLatch(2);
        latch.countDown(); latch.countDown();
        System.out.println("round 1 count = " + latch.getCount());

        long t0 = System.nanoTime();
        latch.await();                              // should this block?
        System.out.println("round 2 await returned after " + (System.nanoTime()-t0)/1000 + " us");

        latch.countDown(); latch.countDown(); latch.countDown();
        System.out.println("after 3 more countDowns, count = " + latch.getCount());

        CyclicBarrier b = new CyclicBarrier(1);
        System.out.println("barrier round 1 index = " + b.await());
        System.out.println("barrier round 2 index = " + b.await());   // reusable
    }
}
```

```bash
java Reuse.java
```

| What you see | What it means |
|---|---|
| `round 2 await returned after` a handful of microseconds | **Trap 1, demonstrated.** The second `await` did not block. It never will again. |
| `after 3 more countDowns, count = 0` | `countDown()` past zero is a silent no-op. The latch cannot be re-armed. |
| Both barrier `await` calls return | `CyclicBarrier` is reusable by construction. Note the arrival index it returns. |
| The barrier's second call hangs | You constructed it with more than 1 party — with one party each `await` completes immediately. |

**Do this before reading further.** Twenty seconds of output makes the one-shot/reusable
distinction permanent in a way that reading cannot.

### Proof 2 — see each primitive in a thread dump

```java
// Dump.java
import java.util.concurrent.*;
public class Dump {
    public static void main(String[] a) throws Exception {
        System.out.println("pid " + ProcessHandle.current().pid());
        CountDownLatch latch = new CountDownLatch(1);
        Semaphore sem = new Semaphore(0);
        CyclicBarrier barrier = new CyclicBarrier(2);
        Phaser phaser = new Phaser(2);

        new Thread(() -> { try { latch.await(); }   catch (Exception e) {} }, "on-latch").start();
        new Thread(() -> { try { sem.acquire(); }   catch (Exception e) {} }, "on-semaphore").start();
        new Thread(() -> { try { barrier.await(); } catch (Exception e) {} }, "on-barrier").start();
        new Thread(() -> phaser.arriveAndAwaitAdvance(),                      "on-phaser").start();
        new Thread(() -> { try { latch.await(5, TimeUnit.MINUTES); } catch (Exception e) {} },
                   "on-latch-timed").start();
        Thread.currentThread().join();
    }
}
```

```bash
java Dump.java &
jcmd <pid> Thread.print | grep -A 8 -E '"on-(latch|semaphore|barrier|phaser|latch-timed)"'
```

**Structure of what you will find** — *illustration of the format, not captured output*:

```
"on-semaphore" #<n> prio=5 tid=0x... nid=0x... waiting on condition [0x...]
   java.lang.Thread.State: WAITING (parking)
        at jdk.internal.misc.Unsafe.park(java.base@<ver>/Native Method)
        - parking to wait for  <0x...> (a java.util.concurrent.Semaphore$NonfairSync)
        at java.util.concurrent.locks.LockSupport.park(...)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(...)
        at java.util.concurrent.Semaphore.acquire(...)
        at Dump.lambda$main$1(Dump.java:<n>)
```

| What you see | What it means |
|---|---|
| `on-latch`: `WAITING (parking)` on `CountDownLatch$Sync` | The class name in the `parking to wait for` line identifies the primitive directly. |
| `on-semaphore`: `Semaphore$NonfairSync` | Same. Change to `new Semaphore(0, true)` and watch it become `$FairSync`. |
| `on-barrier`: `AbstractQueuedSynchronizer$ConditionObject` | **The barrier does not name itself.** You must read the `CyclicBarrier.dowait` frame above. |
| `on-phaser`: `Phaser$QNode`, or briefly `RUNNABLE` | `Phaser` spins before parking. Take two dumps if you catch it spinning. |
| `on-latch-timed`: **`TIMED_WAITING (parking)`** | The state difference a timeout makes — and the difference between a bounded wait and a permanent hang. |
| Zero `BLOCKED` threads anywhere | Expected. None of these use intrinsic monitors. |

**Build this reference once and keep the output.** At 3am you will be matching a production
dump against it.

### Proof 3 — a semaphore permit leak, seen in `availablePermits`

```java
// Leak.java
import java.util.concurrent.*;
public class Leak {
    static final Semaphore SEM = new Semaphore(3);
    public static void main(String[] a) throws Exception {
        for (int i = 0; i < 5; i++) {
            try {
                SEM.acquire();
                if (i % 2 == 0) throw new RuntimeException("downstream 5xx");
                SEM.release();                 // WRONG: not in finally
            } catch (RuntimeException e) {
                System.out.println("attempt " + i + " failed; permits now = "
                                   + SEM.availablePermits());
            }
        }
        System.out.println("final permits = " + SEM.availablePermits()
                           + "  (constructed with 3)");
        System.out.println("queue length  = " + SEM.getQueueLength());
    }
}
```

```bash
java Leak.java
```

| What you see | What it means |
|---|---|
| permits decreasing on each failure and never recovering | **Trap 3, demonstrated.** Each exception path destroyed a permit permanently. |
| `final permits = 0` with a limit of 3 | The bulkhead is now closed. The next `acquire()` parks forever. |
| Moving `release()` into a `finally` makes the final count 3 | The fix, proved rather than asserted. |

Then invert it: add a second `SEM.release()` and watch `availablePermits()` exceed 3.
**No exception.** That is Trap 4, and it is why the "greater than the limit" alert exists.

## Failure drill

**The drill:** reproduce the reused-latch bug on the `orderflow` reconciliation batch,
observe that it produces **no** diagnostic signal at all, then fix it — and separately
reproduce the semaphore permit leak under the Topic 65 load and watch it strangle the
service with a completely healthy downstream.

Two parts, because they teach opposite lessons: **one failure is invisible in every tool,
and the other is invisible only until you add one gauge.**

### Part A — the silent latch

Build a three-round batch with one hoisted latch, as in the Concurrency trace. Give each
round a deliberately observable side effect: round N writes `round=N` to a table row along
with a timestamp, and `applyCorrections(N)` reads back what round N wrote.

```bash
# Run it. Then, while it runs:
jcmd <pid> Thread.print > /tmp/latch-dump.txt
grep -c "CountDownLatch" /tmp/latch-dump.txt
grep "java.lang.Thread.State" /tmp/latch-dump.txt | sort | uniq -c
```

**WHAT TO LOOK FOR — the point of this part is that you will find nothing:**

| Observation | What it means |
|---|---|
| Zero threads parked on `CountDownLatch$Sync` after round 1 | The latch is open. Nothing is waiting. **The dump is clean.** |
| No `BLOCKED`, no exceptions, no ERROR log lines | There is nothing to find in the usual places. |
| The batch completes **faster** than the correct version | The bug's only performance signature is that it looks like an improvement. |
| `applyCorrections(2)` read round **1**'s row | **The only evidence, and it is in the data.** |

**Write this down before fixing anything:** which tool would have caught this? A test that
runs more than one round and asserts on the round number in the data — not a dump, not a
profiler, not a metric. That is the lesson.

### Part B — fix it two ways and compare

```java
// Fix 1 — CyclicBarrier: reusable by construction, fails loudly.
CyclicBarrier round = new CyclicBarrier(SHARDS, applier::applyCurrentRound);

// Fix 2 — a fresh latch per round: reuse is impossible because it does not outlive the round.
for (int r = 1; r <= ROUNDS; r++) {
    CountDownLatch done = new CountDownLatch(SHARDS);
    /* ... submit; countDown() in finally ... */
    if (!done.await(5, TimeUnit.MINUTES)) throw new IllegalStateException("round " + r);
}
```

Run both. Then **deliberately kill one shard mid-round** (throw from `reconcile`) and
compare:

| Variant | What happens when a shard dies |
|---|---|
| Original (hoisted latch) | nothing; the batch completes and reports success with wrong data |
| Fix 1 (`CyclicBarrier`) | every other party gets **`BrokenBarrierException`**; the batch fails visibly |
| Fix 2 (per-round latch, `countDown` in `finally`) | the count still reaches zero; the round completes with a **missing shard** unless you also track failures |

**That third row is the drill's most valuable finding.** A per-round latch fixes the *reuse*
bug and not the *failure-propagation* bug, because a latch carries no errors. "All shards
succeeded" needs the barrier's breaking semantics, an explicit failure flag, or Topic 102's
`StructuredTaskScope`. **Write down which you would ship and why.**

### Part C — the permit leak under load

Deploy the bulkhead of Fix 1 from Example 2 with the release moved out of the `finally`,
and make the gateway stub return a 5xx for 1% of calls. Run the Topic 65 k6 profile for
twenty minutes.

**Instrument first, or this teaches nothing:**

```java
Gauge.builder("orderflow.gateway.permits.available", permits, Semaphore::availablePermits).register(meters);
Gauge.builder("orderflow.gateway.permits.waiting",   permits, Semaphore::getQueueLength).register(meters);
```

**Capture:**

```bash
# every 30s:
curl -s localhost:8080/actuator/metrics/orderflow.gateway.permits.available
curl -s localhost:8080/actuator/metrics/orderflow.gateway.permits.waiting

# when latency starts climbing:
jcmd <pid> Thread.print -l > /tmp/sem-dump.txt
grep -c "Semaphore.acquire" /tmp/sem-dump.txt
grep -c "http-nio" /tmp/sem-dump.txt

# and prove it is NOT a deadlock:
jcmd <pid> Thread.print | grep -c "Found one Java-level deadlock"
```

**WHAT TO LOOK FOR:**

| Observation | What it means |
|---|---|
| `permits.available` stepping down, once per gateway 5xx, never recovering | **The leak, made visible.** One gauge turns an invisible failure into an obvious one. |
| `permits.waiting` rising as available approaches zero | Callers queueing. This is the signal that separates "at capacity" from "leaked". |
| Eventually available = 0, waiting = large, and it stays there | Total strangulation. The bulkhead has become a wall. |
| Threads parked in `Semaphore.acquire` in the dump, named `http-nio-*` | Request threads consumed by a downstream that is **completely healthy**. |
| `Found one Java-level deadlock` count = **0** | **Critical.** This is not a deadlock and the detector will never find it. A semaphore has no owner, so there is no cycle to detect. Topic 98's decision table. |
| The provider's own dashboards showing normal latency and no errors | The downstream is fine. The bug is entirely ours. |

### Part D — fix, and prove the fix

Move `release()` into the `finally`, re-run, and confirm `available` returns to its full
value between bursts and never drifts. Then add the three alerts: `available == 0` sustained
for one minute (page); the **daily maximum** of `available` below the configured limit (the
slow-leak detector); and `available > configured limit` (the Trap 4 detector, which should
be impossible).

### What the drill proves

- A reused latch has **no** diagnostic signature — not in dumps, metrics or logs. The only
  defences are the right primitive and a test with more than one round.
- A permit leak has **exactly one**, and you must have added it in advance. Without the gauge
  it is indistinguishable from "the downstream is slow".
- Neither is a deadlock, and the detector correctly reports nothing for both — which is why
  Topic 98 needs four diagnoses rather than one.
- `finally` is not a style preference here. It is the difference between a bulkhead and an
  outage.

---

## Measurement

### The instrument for each claim

| Claim you want to make | The right instrument | The wrong instrument |
|---|---|---|
| "the bulkhead is at capacity" | Micrometer gauge on `availablePermits()` **and** `getQueueLength()` | latency alone — it looks the same as a slow downstream |
| "we are shedding gateway calls" | a `Counter` in the `tryAcquire`-returned-false branch | absence of errors; shedding is silent by default |
| "permits are leaking" | the **daily maximum** of the available gauge falling below the limit | the current value, which looks fine until it is zero |
| "threads are stuck on a latch" | `jcmd Thread.print`, grep `CountDownLatch$Sync` | a deadlock detector — it will find nothing |
| "a latch was reused" | **a test that runs more than one round**, asserting on data | any runtime instrument; there is no signal |
| "the acquire timeout is right" | a `Timer` on permit wait time; compare its p99 to the timeout | the timeout value looking reasonable |
| "coordination is costing us latency" | JFR `jdk.ThreadPark` durations, aggregated by parked-on class | `System.nanoTime()` around the call — Topic 77 |

### Micrometer — the four instruments a `Semaphore` always gets

Two gauges (`availablePermits` and `getQueueLength`), a `Counter` for shed calls, and a
`Timer` on permit wait time — the registration code is in Example 2's Fix 1. The pairing is
what matters:

**Why both gauges.** `available == 0` alone is ambiguous — it is what a correctly saturated
bulkhead looks like *and* what a fully leaked one looks like. Pair it with `waiting`:

| available | waiting | Diagnosis |
|---|---|---|
| ≈ limit | 0 | idle · healthy |
| 0 | small and fluctuating | **at capacity, working as designed** · watch the shed counter |
| 0 | large and growing | saturation *or* a leak — check whether the downstream is actually slow |
| 0 | large, and the **downstream is healthy** | **a permit leak.** Trap 3. |
| > limit | — | **a double release.** Trap 4. Should be impossible; alert immediately. |

For a `CountDownLatch`, gauge `getCount()` **for diagnostics only**. Never branch on it: a
check-then-act on `getCount()` races exactly like Topic 92's `containsKey`-then-`put`.
Topic 118 has the full treatment, plus the standing rule: **never tag these with an order id.**

### `jcmd Thread.print` — the commands you will type

```bash
# three dumps, ten seconds apart: state, then whether state is CHANGING
for i in 1 2 3; do jcmd <pid> Thread.print -l > /tmp/d$i.txt; sleep 10; done

# state histogram first, always:
grep "java.lang.Thread.State" /tmp/d1.txt | sort | uniq -c | sort -rn

# which primitive is anything waiting on?
grep -oE 'a java\.util\.concurrent\.(CountDownLatch|Semaphore|Phaser)[A-Za-z$]*' /tmp/d1.txt \
  | sort | uniq -c

# barriers do not name themselves; find them by frame:
grep -c "CyclicBarrier.dowait" /tmp/d1.txt

# are REQUEST threads the ones waiting? (the outage signature)
grep -B 6 "Semaphore.acquire" /tmp/d1.txt | grep '"http-nio'

# rule deadlock in or out explicitly:
jcmd <pid> Thread.print | grep -A 30 "Found one Java-level deadlock"
```

| What you see | What it means |
|---|---|
| Many threads on `Semaphore$NonfairSync`, `available` gauge at 0, downstream healthy | Permit leak. Trap 3. |
| Many threads on `Semaphore$NonfairSync`, `available` at 0, downstream **slow** | Correct backpressure. Look downstream, not here. |
| One thread on `CountDownLatch$Sync` for minutes across all three dumps | A task died before its `countDown()`. Find the task, not the latch. |
| Threads on `CyclicBarrier.dowait` in all three dumps | A party never arrived, or the barrier broke and nobody handled it. |
| `Found one Java-level deadlock` prints nothing | Expected for **every** failure in this topic. These are not monitor cycles. |

### JFR — the durations a dump cannot give you

```bash
java -XX:StartFlightRecording=duration=120s,filename=/tmp/coord.jfr,settings=profile -jar orderflow.jar

jfr summary /tmp/coord.jfr
jfr print --events jdk.ThreadPark /tmp/coord.jfr | head -60
```

**WHAT TO LOOK FOR:** `jdk.ThreadPark` carries the class parked on and the duration. A dump
says a thread is waiting *now*; JFR says how long threads wait *in aggregate*. Aggregate by
parked-on class:

| What you see | What it means |
|---|---|
| Large total park time on `Semaphore$NonfairSync` | The bulkhead is your bottleneck — which may be correct. Compare with the downstream's latency. |
| Semaphore park time **growing** while downstream latency is flat | A leak, quantified. |
| Park time on `CountDownLatch$Sync` outside startup | A latch on your request path — almost always a design error. |

### The standing rule: a naive `System.nanoTime()` loop is wrong

```java
// WRONG. Produces a number; the number is false.
long t0 = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) { sem.acquire(); sem.release(); }
System.out.println(System.nanoTime() - t0);
```

No warm-up, so most of it runs interpreted; C2 may eliminate an uncontended acquire/release
pair entirely (Topic 75); and **single-threaded, so it measures the uncontended fast path,
which is not what any of these classes are for**. Use JMH with `@Threads` — the
contended path is the only interesting one. Topic 77.

### `perf` — and the honest note about macOS

```bash
# Linux only:
perf stat -e context-switches,cpu-migrations -p <pid> -- sleep 30
```

Each park/unpark that reaches the OS is a context switch. On Linux, a context-switch count
that dwarfs your request rate points at excessive handoff — most often a **fair** semaphore
or barrier where non-fair would do.

**On macOS this is unavailable.** `perf` is a Linux kernel facility that does not exist on
Darwin, and I will not invent an equivalent. Use JFR's `jdk.ThreadPark` counts as the
portable proxy — it counts parks directly, which is the quantity you wanted — or run in a
Linux container.

---

## Practice exercises

### 1 — Easy: build the thread-dump reference card

Run Proof 2's `Dump.java`, dump, and extract the five stack sections into one reference
file. For each record the `Thread.State`, the class in the `parking to wait for` line, and
the two frames that identify the primitive. Then modify it: make the semaphore fair, add a
`poll(t,u)` on a `BlockingQueue`, and add a thread blocked on `synchronized`. Re-dump.

**Deliverable:** one page mapping *what you see* to *which primitive*, from your own JVM.
**You will use this at 3am.** The `BLOCKED` row is there to make the contrast concrete — it
is the only one in the set.

### 2 — Medium: a correct, instrumented bulkhead (combines 39, 87, 90, 92, 93, 94, 95)

Build `GatewayBulkhead` for `orderflow`:

- A `Semaphore` whose permit count comes from `@ConfigurationProperties` (Topic 43), with a
  comment naming the contractual source of the number.
- A single `withPermit(Callable<T>)` that is the **only** place `acquire` and `release`
  appear in the codebase — enforced by an ArchUnit test.
- `tryAcquire` with a timeout derived from the p99 budget; a `DownstreamAtCapacityException`
  mapped to HTTP 503 with `Retry-After` (Topic 46).
- The three Micrometer instruments plus the three alerts from Part D.
- A `LongAdder` (Topic 95) counting permitted calls, with a comment saying why not
  `AtomicLong` — referencing Topic 96.
- A JUnit test using a start latch and a done latch (Example 1) that launches 200 threads
  against a limit of 20 and asserts the **maximum observed concurrency inside the critical
  section never exceeded 20** (an `AtomicInteger` high-water mark).
- A second test that throws on every third call and asserts `availablePermits()` returns to
  the full limit. **This is the test that catches Trap 3**, and nobody writes it.

### 3 — Hard: production simulation on the `orderflow` baseline

1. Run Failure drill Parts A–D end to end under the k6 profile, capturing the permit gauges
   throughout. Produce a chart of `available` and `waiting` over time for the leaking and
   fixed versions. **The chart is the deliverable.**
2. Compare three bulkhead implementations under the same load: a `Semaphore`, a
   `ThreadPoolExecutor` with 20 threads and a bounded queue (Topic 93), and Resilience4j's
   `Bulkhead` (Topic 111). Report on shed behaviour, request-thread consumption, and what
   happens to each when you flip `spring.threads.virtual.enabled=true` (Topic 101).
   **Predict the virtual-thread result before you run it**, and write your prediction down.
3. Replace the batch's `CyclicBarrier` with a `Phaser` and change the shard count mid-run
   (`register()` / `arriveAndDeregister()`). State what the `Phaser` made possible and —
   honestly — whether it was worth the lost readability here.
4. Write a one-page note in the shape Topic 131 will ask for: the constraint and its source,
   the primitive chosen and why, the timeout and its derivation, the shed behaviour, the
   three alerts, and the failure mode you deliberately accepted.

---

## Interview questions

### Q1 — "`CountDownLatch` versus `CyclicBarrier`. When do you use each?"

**MID-LEVEL ANSWER.** "A `CountDownLatch` waits for a count to reach zero and a
`CyclicBarrier` waits for a number of threads to reach a point. The barrier can be reused
and the latch can't. I'd use a latch to wait for startup tasks."

**SENIOR ANSWER.** "The mechanical difference is direction and lifecycle: a latch counts
**down** to zero, once, and stays open forever; a barrier counts **up** to a party count,
releases everyone, and **resets** for the next round.

But the difference that decides the design is who waits. With a latch, the counters and the
waiters are **different sets of threads** — workers call `countDown()`, a coordinator calls
`await()`. With a barrier, every party calls the same `await()`, which both signals arrival
and blocks. So a barrier is for peers advancing in lockstep, and a latch is for an event
some other thread is waiting on.

I'd pick by lifecycle first. One-shot — service startup, a test's starting gun, waiting for
a fan-out — latch. Repeating rounds — a multi-phase batch, a simulation stepping in
lockstep — barrier, because **a latch reused for a second round does not throw; it simply
stops blocking**, and your round two proceeds against round one's data. That failure is
silent and shows up as wrong numbers weeks later, which makes it one of the worst bugs in
this whole area.

Two more things I'd bring up. A barrier has an **action** that runs on the last-arriving
thread with everyone else still parked — a safe place to mutate shared state, which is
genuinely useful. And a barrier **breaks**: if one party is interrupted or times out,
everybody else gets `BrokenBarrierException`. That is more code to write, and it is a
feature, because a latch would have let the round silently complete short a participant.

If the party count needs to change at runtime, neither works and you want a `Phaser`."

**What separates them.** The mid answer states the API difference. The senior answer names
the property that decides between them (who waits versus who signals), the **failure mode of
the wrong choice** and its business consequence, the barrier action and breaking semantics,
and where the boundary to `Phaser` is.

**Follow-up:** *"You need one-shot semantics but also need to know whether the tasks
succeeded."* — Then a latch is the wrong tool, because it carries no values and no
exceptions: `await()` returning tells you they *finished*, not that they *worked*. I'd use
`CompletableFuture.allOf` for the results, or better, Topic 102's `StructuredTaskScope`
with a shutdown-on-failure policy, which gives me cancellation and error propagation that a
latch fundamentally cannot.

### Q2 — "How would you limit concurrent calls to a flaky downstream to 20?"

**MID-LEVEL ANSWER.** "Use a thread pool with 20 threads for those calls. That way only 20
can run at once."

**SENIOR ANSWER.** "A `Semaphore` with 20 permits, `tryAcquire` with a timeout, `release`
in a `finally`, and instrumentation on the available permits.

I'd argue against the thread pool specifically. Three reasons. First, a pool bounds
**threads**, and my requirement bounds **concurrent calls to them** — those numbers only
coincide on platform threads, and they stop coinciding the moment we enable virtual
threads, at which point the pool becomes `newVirtualThreadPerTaskExecutor` with no limit at
all and the bulkhead silently disappears. Second, submitting and then calling `.get()`
blocks the caller too, so I've consumed two threads per in-flight call rather than one.
Third, `Executors.newFixedThreadPool` has an unbounded queue, so under a downstream
slowdown I've accidentally built '20 concurrent, unlimited queued', which is Topic 93's
OOM.

The semaphore expresses the constraint as what it actually is: a fact about *their*
capacity, in one number, with a comment naming the contract it came from. It costs no extra
threads, it behaves identically on platform and virtual threads, and there is no hidden
queue.

The details that make it production code: `tryAcquire(100, MILLISECONDS)` rather than
`acquire()`, so a caller waits a bounded time and I get `TIMED_WAITING` instead of
`WAITING`; the failure path increments a shed counter and returns 503 with `Retry-After`
rather than blocking; `release()` in a `finally` with nothing between `tryAcquire` and the
`try`; and one wrapper method that is the only place acquire and release appear, enforced
by an ArchUnit test — because a missing release is a permanent capacity leak that no
deadlock detector will ever find."

**What separates them.** The senior answer identifies the **category error** — bounding
threads versus bounding downstream concurrency — knows the virtual-thread migration breaks
it, names the unbounded-queue trap, and treats the leak as a structural risk to design out
rather than a discipline to remember.

**Follow-up:** *"How would you know if a permit leaked?"* — A gauge on
`availablePermits()`, and specifically an alert on the **daily maximum** falling below the
configured limit. Current value alone reaches zero only after the last permit is gone; the
maximum catches the first one. And an alert on the value *exceeding* the limit, which
catches a double release.

### Q3 — "The service is hung. Walk me through diagnosing it."

**MID-LEVEL ANSWER.** "Check the logs and CPU, then take a thread dump and look for
deadlock. If there's no deadlock, restart it and see if it recovers."

**SENIOR ANSWER.** "First: do not restart. That destroys the evidence and guarantees a
repeat. Pull the pod out of the load balancer if I can — Topic 121's readiness-versus-
liveness distinction exists for this — and leave it running.

Three dumps, ten seconds apart, with `jcmd <pid> Thread.print -l`. One dump gives state;
three tell me whether the state is *changing*, which is the difference between stuck and
slow. Then a state histogram, and I branch:

`BLOCKED` threads in a cycle plus a 'Found one Java-level deadlock' section — deadlock, and
the section names the threads and the monitors. `RUNNABLE` at high CPU with identical
stacks across all three dumps — livelock. Threads only in `WAITING (parking)` — and this is
where this topic lives, because **all four coordination primitives park, none of them
produce `BLOCKED`, and the deadlock detector will report nothing for any of them**.

So for `WAITING` I read the `parking to wait for` line. `CountDownLatch$Sync` means a task
died before its `countDown()` — I go and find that task; the latch is the symptom.
`Semaphore$NonfairSync` is ambiguous and I need a second signal: if the available-permits
gauge is zero and the downstream is genuinely slow, that is correct backpressure and I look
downstream; if the gauge is zero and the downstream is *healthy*, that is a permit leak and
the bug is mine. `ConditionObject` with `CyclicBarrier.dowait` above it means a party never
arrived.

And the thing I'd say explicitly: **`findDeadlockedThreads()` returning null does not mean
nothing is stuck.** It only knows about monitors and `Lock` owners. A latch that never
opens and a semaphore with no permits are permanent hangs that it cannot see."

**What separates them.** The mid answer treats "hung" as one diagnosis with one tool. The
senior answer has a branching procedure, knows this topic's failures are invisible to the
deadlock detector, and — the strongest signal — knows which evidence is **ambiguous** and
names the second signal that resolves it.

**Follow-up:** *"Your permits gauge shows zero and the downstream is healthy. Now what?"* —
That is a leak, so I look for an `acquire` whose `release` is not in a `finally`. Then I
restructure so there is one wrapper with the only pair in it, and add the maximum-permits
alert so the next one is caught in minutes rather than days.

### Q4 — "Is a `Semaphore` with one permit the same as a lock?"

**MID-LEVEL ANSWER.** "Pretty much — both allow one thread in at a time. The semaphore is a
bit more flexible because you can have more than one permit."

**SENIOR ANSWER.** "It provides mutual exclusion, and it is not a lock. Three differences,
all of which have bitten someone.

**No reentrancy.** A thread that acquires a binary semaphore and then acquires it again
**deadlocks against itself**, permanently, because there is no hold count. `ReentrantLock`
and `synchronized` both let the owner re-enter. In a codebase where a method might be
called from inside another method that already took the same guard, this is a real hazard.

**No owner.** A `Semaphore` does not record which thread holds a permit. Thread A may
acquire and thread B may release — which is a *feature* if you are building a resource pool
or a producer-consumer signal, and a hazard because nothing detects a double release, and
because `ThreadMXBean.findDeadlockedThreads()` and the JVM's deadlock reporter cannot see
it. A cycle through a semaphore is invisible to the detector.

**No `Condition`, and no automatic release.** `synchronized` releases its monitor when the
frame unwinds for any reason. A permit is returned only by an explicit `release()`, so
every exception path is a potential permanent leak.

So: use `ReentrantLock` or `synchronized` when you mean *mutual exclusion on data*, and a
`Semaphore` when you mean *capacity* — how many of something may be in flight. The permit
count being greater than one is the giveaway that you are in the second case; a permit count
of exactly one usually means someone reached for the wrong abstraction."

**What separates them.** The senior answer knows self-deadlock, knows ownerlessness cuts
both ways, and knows that ownerlessness is *why the deadlock detector cannot see semaphore
cycles* — a fact you only learn by having diagnosed one.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `CountDownLatch` uses AQS's **shared** mode and `ReentrantLock` uses **exclusive** mode.
   Explain why that choice makes a latch permanently open once it reaches zero — and why
   adding a `reset()` method would be incoherent rather than merely inconvenient.

2. `CyclicBarrier` is not an AQS subclass; it is a `ReentrantLock` plus a `Condition` plus a
   `Generation` object. Name the two features that need the `Generation`, and explain what
   would break if a waiter woke up and did not check which generation it belonged to.

3. `Semaphore.release()` can raise the permit count above the constructed value, with no
   exception. Argue that this is correct design. Then make the strongest case that it should
   throw, and say which real bug each choice makes easier to write.

4. A latch carries no values and no exceptions; a `Promise.all` carries both. Given that,
   explain what a `CountDownLatch` is *for* in a world that has `CompletableFuture.allOf`
   and `StructuredTaskScope` — and name the two cases where it is still the right tool.

5. A `Semaphore` bounds concurrent operations; a thread pool bounds threads. Describe a
   concrete `orderflow` situation where those two numbers differ by a factor of a thousand,
   and say which one the payment provider cares about.

6. Every failure in this topic is `WAITING (parking)` and invisible to
   `findDeadlockedThreads()`. Explain what the detector knows about monitors and `Lock`
   owners that it cannot know about a semaphore, then design the smallest change to
   `Semaphore` that would make it detectable and say what that change would cost.

7. A non-fair `Semaphore` lets an arriving thread barge past queued waiters, and it is the
   faster default. Explain how unfairness produces higher throughput, then name the
   `orderflow` workload where that trade is unacceptable.

---

## Quick reference card

### The four, at a glance

| | `CountDownLatch` | `CyclicBarrier` | `Semaphore` | `Phaser` |
|---|---|---|---|---|
| Counts | **down**, to zero | **up**, to a party count | permits, up and down | arrivals per phase |
| Lifecycle | **one-shot** | **reusable** (auto-resets) | continuous | reusable, multi-phase |
| Party count | fixed at construction | fixed at construction | n/a | **dynamic** at runtime |
| Who waits | a *different* thread from the counters | **every party** (`await` = arrive + wait) | any acquirer | every registered party |
| Reset | **impossible** | automatic, plus `reset()` | n/a | automatic |
| Action on release | none | **barrier action**, on the last arriver | none | `onAdvance` hook |
| Failure propagation | **none** | `BrokenBarrierException` to all | none | phase termination |
| Built on | AQS shared mode | `ReentrantLock` + `Condition` + `Generation` | AQS shared mode | own 64-bit state word |
| Thread state when waiting | `WAITING (parking)` | `WAITING (parking)` | `WAITING (parking)` | spin, then `WAITING` |
| Named in the dump | `CountDownLatch$Sync` | `ConditionObject` (**unhelpful**) | `Semaphore$NonfairSync` | `Phaser$QNode` |

### Choose by asking two questions

```
1. One-time event, or repeating rendezvous?
      one-time  -> CountDownLatch  (or StructuredTaskScope if you need results/errors)
      repeating -> CyclicBarrier   (or Phaser if the party count changes)
2. Am I counting events/participants, or capacity?
      capacity  -> Semaphore. Always. Never a thread pool.
```

### The idioms — the only correct shapes

```java
// Latch: fresh per round, countDown in finally, await with a timeout, check the boolean.
CountDownLatch done = new CountDownLatch(n);
pool.execute(() -> { try { work(); } finally { done.countDown(); } });
if (!done.await(30, TimeUnit.SECONDS)) throw new IllegalStateException("timed out");

// Semaphore: tryAcquire with a budget, nothing between it and try, release in finally.
if (!permits.tryAcquire(100, TimeUnit.MILLISECONDS)) { shed.increment(); throw ...; }
try { return downstream.call(); } finally { permits.release(); }

// Barrier: bounded await, and handle BOTH exception types.
try { barrier.await(5, TimeUnit.MINUTES); }
catch (BrokenBarrierException e) { /* a party died: abandon the round */ }
catch (TimeoutException e)       { barrier.reset(); /* free the others deliberately */ }
```

### Thread-dump triage

| `parking to wait for` names | Frame above | Diagnosis |
|---|---|---|
| `CountDownLatch$Sync` | `CountDownLatch.await` | a task died before `countDown()` — find the task |
| `Semaphore$NonfairSync` | `Semaphore.acquire` | at capacity **or** leaked — check the permits gauge |
| `...$ConditionObject` | `CyclicBarrier.dowait` | a party never arrived |
| `Phaser$QNode` | `Phaser.internalAwaitAdvance` | a registered party never arrived |
| **nothing; state is `BLOCKED`** | `synchronized` | **not this topic** — Topic 85 |

### Gotchas checklist

- [ ] No `CountDownLatch` as a field; construct it inside the scope that uses it.
- [ ] Every `countDown()` is in a `finally`.
- [ ] Every `await()` and `acquire()` has a timeout, and the return value is checked.
- [ ] Every `release()` is in a `finally`, with nothing between `tryAcquire` and `try`.
- [ ] Exactly one acquire/release pair in the codebase, in a wrapper, enforced by a test.
- [ ] `availablePermits` and `getQueueLength` are gauged; shed calls are counted.
- [ ] Alerts on: permits at 0 sustained; daily max below the limit; value above the limit.
- [ ] Bulkheads are semaphores, never pool sizes — they must survive the virtual-thread flip.
- [ ] Barrier code handles `BrokenBarrierException` **and** `TimeoutException`.
- [ ] A test exists that runs **more than one round**.

---

## When would I use this at work?

**1. Adding a limit to a fragile downstream, correctly, the first time.**

The provider's contract says 20 concurrent connections. Someone proposes a 20-thread pool.
You point out that the constraint is a fact about *them* and the pool is a fact about *us*,
that the two stop matching the day we enable virtual threads, and that
`newFixedThreadPool` hides an unbounded queue. You ship a `Semaphore(20)` behind one
`withPermit` wrapper with three gauges and three alerts. **Six months later the team enables
virtual threads and the bulkhead keeps working** — nobody notices, which is what correct
infrastructure looks like.

**2. Reviewing a batch job.**

A PR adds a multi-round batch with one `CountDownLatch` hoisted out of the loop "to avoid
allocating per round". You ask one question: *"what does `await()` do on round two?"* That
question — not a style comment — catches the silent-wrong-data bug before it ships. The fix
is either a `CyclicBarrier` or moving the latch inside the loop, and the review conversation
teaches the author a distinction they will keep.

**3. The 3am hang with a healthy downstream.**

Payment latency alert fires; the provider's status page is green and their dashboards agree.
Three dumps: dozens of `http-nio` threads `WAITING (parking)` on `Semaphore$NonfairSync`, no
deadlock section, no `BLOCKED` threads. The permits gauge reads zero and its daily maximum
has drifted down all week. **That is a permit leak, and you knew it in ninety seconds** —
because you added the gauge months earlier, and because you knew the deadlock detector's
silence was expected rather than reassuring. The fix is a `finally`. The value was the
instrumentation and the vocabulary.

---

## Connected topics

**Prerequisites:**

- **94 — explicit locks and AQS.** The machine underneath all of this: one `volatile int
  state` plus a CLH queue. Every class here is that machine with a different meaning for
  `state`, and the BLOCKED/WAITING/TIMED_WAITING table you built there is what makes this
  topic's thread dumps readable.
- **89 — `wait`/`notify`.** `CyclicBarrier` is a `ReentrantLock` plus a `Condition` — the
  same guarded-block pattern, written correctly, with a `Generation` to make it reusable.
- **90 — executors and pool sizing.** The direct contrast in Trap 5: a pool bounds threads,
  a semaphore bounds concurrency, and knowing which your constraint is *is* the design.
- **93 — blocking queues.** The other way to express a bound: a queue bounds *buffered*
  work, a semaphore bounds *in-flight* work. Both convert a hang into a rejection once you
  give them a timeout, and both need somewhere for what they shed to go.
- **91 — `CompletableFuture`.** `allOf` is the value-carrying, error-propagating
  alternative to a fan-out latch. Knowing when a latch suffices — signals, not results — is
  the choice this topic makes.
- **87 — `volatile` and the JMM.** Why `getCount()` and `availablePermits()` are honest but
  instantly stale reads, and why branching on them is Topic 92's check-then-act race.

**This unlocks:**

- **98 — the concurrency bug taxonomy.** The diagnostic capstone, and this topic supplies
  two of its hardest cases: a permit leak is a hang that is **not** a deadlock and that
  `findDeadlockedThreads()` cannot see, and a fair-versus-non-fair semaphore is where
  starvation comes from. Topic 98's decision table keeps those apart from the two that
  genuinely are deadlock and livelock.
- **101 — virtual threads.** Where the semaphore-versus-pool distinction stops being
  academic: a million virtual threads may wait on 20 permits and consume no carriers, so the
  bulkhead survives the migration and a pool-based limit does not.
- **102 — structured concurrency.** The modern replacement for most application uses of
  `CountDownLatch`: `StructuredTaskScope` adds cancellation, error propagation and no leaked
  tasks — the three things a latch fundamentally cannot carry.
- **109 — the HikariCP pool deadlock.** A connection pool is a semaphore over connections
  and `connectionTimeout` is this document's `tryAcquire` timeout. Ten threads each holding
  one connection and waiting for a second is a permit cycle that, exactly like Trap 3,
  produces no deadlock report.
- **111 — resilience patterns.** This topic's `Semaphore` **is** Resilience4j's `Bulkhead`;
  knowing the primitive is what lets you judge the library rather than trust it.
- **118 — Micrometer metrics.** The three instruments and three alerts, done properly —
  including why the *daily maximum* of a gauge is what catches a slow leak.
- **121 — readiness versus liveness probes.** Fix 2's warm-up latch is a readiness question.
  A warming pod must not be restarted, and a pod whose permits have leaked must be pulled
  from the load balancer and **dumped**, not killed.

---

*Java baseline 21, running on JDK 25. One thing here is deliberately hedged rather than
asserted: the exact internal structure of `Phaser` and `CyclicBarrier` on your build — both
have been rewritten, neither is a public contract, and `javap` plus `src.zip` settle it in a
minute. Everything else — that all four park rather than block, that a `CountDownLatch`
cannot be reset, that `Semaphore.release()` has no upper bound and no owner, that a broken
barrier notifies every party, and that none of these failures will ever appear in a
"Found one Java-level deadlock" section — is stable API contract, and it is what stands
between you and a permit leak that looks exactly like a slow downstream.*
