# 85 — `synchronized`, Intrinsic Monitors, and Lock Inflation

## Phase: 9 — Concurrency
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21, 25
## Project spine: adds the first real mutual exclusion to `orderflow` — a `synchronized` critical section around the hot-SKU inventory decrement and the wallet debit. It also produces the first *measurable* contention number for the service, which becomes the baseline that Topics 92, 94, 95 and 101 are compared against.

---

## Mechanical statement

> **Every Java object has a mark word in its header that can encode lock state.**
>
> An **uncontended** lock is a compare-and-swap on that word. No syscall, no kernel,
> a handful of nanoseconds.
>
> **Contention inflates it.** The JVM allocates a native `ObjectMonitor`, points the
> mark word at it, and from then on waiting threads are parked and unparked through
> the operating system. That is real `park`/`unpark` work with kernel involvement,
> measured in microseconds.
>
> So `synchronized` is not one cost. It is two costs, separated by whether a second
> thread showed up. **"Is `synchronized` slow?" is not a question with an answer. "What
> is my contention rate?" is, and JFR answers it directly.**

And the second half, which people forget:

> `synchronized` gives you **mutual exclusion** *and* a **happens-before edge**. Two
> guarantees, not one. Most people can recite the first. The bugs come from the second.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE — and the reason is worth stating precisely

JavaScript has no locks. Not because the designers forgot, and not because it is a
simpler language. **Because it does not need them.**

The event loop gives you run-to-completion. Between the first and last line of your
callback, nothing else runs. That *is* a critical section. It is a critical section
covering every callback you ever write, granted for free, enforced by the runtime, and
impossible to get wrong.

Java's `synchronized` block is you, manually, drawing a much smaller version of the
boundary JavaScript draws around your entire callback. That is the whole idea:

> **`synchronized` is how you buy back a small piece of the run-to-completion guarantee
> you used to get for the whole callback.**

That framing is worth holding onto, because it explains both the value and the cost. In
Node, the guarantee is total and free — and the price is that nothing else can run
either, which is why one slow callback stalls the entire process. In Java, the
guarantee is narrow and paid for — and the price is contention, which is what happens
when the boundary you drew is too wide.

Notice that the failure modes rhyme. **A `synchronized` block held across a network
call is Node's blocking-the-event-loop bug, scoped to one lock instead of the whole
process.** You already have the instinct for that bug. This is the one piece of your
Node experience that transfers cleanly into this topic, and Trap 1 is exactly it.

### What about `Atomics.wait` / `Atomics.notify`?

Once you have a `SharedArrayBuffer`, JavaScript does give you `Atomics.wait` and
`Atomics.notify`, and you can build a mutex out of them. That is a genuine analogue —
of Topic 89's `wait`/`notify` and Topic 95's CAS, more than of `synchronized`. If you
have used it, the shape of `ObjectMonitor`'s wait set will look familiar. If you have
not, do not detour; you will meet all of it here.

### What does not transfer at all

| You know | Java | Verdict |
|---|---|---|
| A callback is atomic with respect to other callbacks | Nothing is atomic unless you make it so | **NO ANALOGUE** |
| `await` yields control at a marked point you can see | Preemption happens at unmarked points you cannot see | **NO ANALOGUE** |
| A mutex library in Node exists only to serialise *async* sections | `synchronized` serialises *actual parallel execution* on multiple cores | **NO ANALOGUE** |
| Deadlock is essentially impossible in single-threaded JS | Two locks acquired in two orders is a hang, today | **NO ANALOGUE** — Topic 94 |

---

## What is this?

### The two forms

```java
// Form 1: synchronized method. Locks 'this'.
public synchronized boolean tryReserve(String sku) { ... }

// Form 2: synchronized block. Locks whatever object you name.
private final Object inventoryLock = new Object();

public boolean tryReserve(String sku) {
    synchronized (inventoryLock) {
        ...
    }
}

// Form 3: static synchronized method. Locks the CLASS object, InventoryCache.class.
public static synchronized void resetAll() { ... }
```

Three facts follow immediately, and each is an interview question:

1. **A `synchronized` method locks `this`.** So anyone holding a reference to your bean
   can lock it too — including code you have never seen. That is why Form 2 with a
   `private final` lock object is the better default.
2. **`static synchronized` locks the `Class` object**, which is a *different* monitor
   from `this`. A static synchronized method and an instance synchronized method on the
   same class do **not** exclude each other. That is Trap 3.
3. **The lock is on an object, not on a block of code.** Two threads running two
   different `synchronized` blocks that name the same object exclude each other. Two
   threads running the *same* block on two different objects do not.

### What it guarantees — both halves

**Half one: mutual exclusion.** At most one thread at a time holds a given object's
monitor. Everyone else waits.

**Half two: a happens-before edge.** From the JLS:

> An unlock of a monitor happens-before every subsequent lock of that same monitor.

In practice that means: when thread B acquires the monitor that thread A released,
**everything A did before releasing is guaranteed visible to B.** Not just the fields
you were thinking about. Everything.

This second half is why `synchronized` fixes *visibility* bugs and not just
*interleaving* bugs, and it is why removing a `synchronized` block "because it was
uncontended anyway" can break a program in a way that has nothing to do with races on
the field you were looking at.

Say it once, out loud:

> **`synchronized` is not only a lock. It is also a memory barrier.** Topics 86 and 87
> are the full account of the second half.

### Reentrancy

```java
public synchronized void placeOrder(Order o) {
    validate(o);                      // also synchronized on 'this'
}
public synchronized void validate(Order o) { ... }
```

This does not deadlock. Java monitors are **reentrant**: the monitor records an owner
and a recursion count. If the owning thread enters again, the count increments. It
releases only when the count returns to zero.

Without reentrancy, every `synchronized` method calling another `synchronized` method
on the same object would self-deadlock, which would make the feature nearly unusable
with inheritance.

The cost of reentrancy is that it hides scope creep: a `synchronized` method three
calls deep does not announce itself, and you can end up holding a lock across far more
work than you intended.

### What it does *not* give you

| It does not | Consequence |
|---|---|
| Guarantee fairness | A thread can be starved indefinitely. The monitor is barging-friendly by design because that is faster. Topic 94's `ReentrantLock(true)` offers fairness, at a throughput cost |
| Support a timeout | There is no `synchronized (x, 100ms)`. You wait forever. `ReentrantLock.tryLock(timeout)` exists precisely for this |
| Respond to interruption | A thread `BLOCKED` on a monitor ignores `interrupt()`. `lockInterruptibly()` exists for this |
| Support multiple wait conditions | One monitor, one wait set. `Condition` objects (Topic 94) give you several |
| Allow non-block-structured locking | You cannot acquire in one method and release in another. This is a *feature* — it makes leaks impossible — until you need a hand-off |
| Protect anything you did not lock | If one code path takes the lock and another does not, you have no mutual exclusion at all. Trap 3 |
| Compose | Two atomic operations in sequence are not one atomic operation. Topic 92 |

### The bytecode: two shapes, one mechanism

A `synchronized` **block** compiles to explicit instructions:

```
monitorenter
   ... body ...
monitorexit
   ... plus a synthetic exception handler that also does monitorexit ...
athrow
```

`javac` generates **two** `monitorexit` instructions: one for the normal path and one
in an exception handler, so the monitor is released even if the body throws. This is
what makes `synchronized` leak-proof and it is why you cannot forget to unlock.

A `synchronized` **method** compiles to *no* instructions at all. Instead the method
gets the `ACC_SYNCHRONIZED` access flag in its `method_info`, and the JVM acquires the
monitor as part of method entry and releases it on any exit. Same semantics, different
mechanism.

You will confirm both with `javap -c -p -v` in the Hands-on proof. Knowing that a
`synchronized` method has *no* `monitorenter` in its body is a good way to catch
yourself reading bytecode too literally.

---

## Why does it matter?

**1. It is the only correctness tool most Java codebases use.**

Walk into any enterprise Java service and `synchronized` is the concurrency control.
Not `StampedLock`, not `VarHandle`, not lock-free queues. If you cannot read
`synchronized`, reason about what it protects, and measure what it costs, you cannot
review or debug the majority of Java that exists.

**2. The advice you will read online is a decade out of date.**

Search results about `synchronized` performance are dominated by material written
between 2008 and 2016, when **biased locking** existed. Biased locking optimised the
case where one thread repeatedly locks the same object, by writing that thread's
identity into the mark word and skipping the CAS entirely.

> **Biased locking was disabled by default in JDK 15 (JEP 374) and removed in JDK 18.**

It is gone. Do not repeat advice that depends on it. Do not tune
`-XX:BiasedLockingStartupDelay`; the flag does not exist on your JVM. Anything you read
that says "the first `synchronized` is expensive because of bias revocation" is
describing a JVM you are not running.

**Why it was removed** is a genuinely good interview answer: it cost a great deal of
complexity across the runtime, its revocation path required a safepoint operation, and
the workloads it was designed for (single-threaded use of thread-safe collections
written in the 1990s — `Vector`, `Hashtable`, `StringBuffer`) had largely been replaced
by unsynchronised equivalents. The cost of maintaining it exceeded the value.

**What replaced it:** nothing. Uncontended locking is now always a CAS on the mark word,
which is cheap enough that the extra machinery was not worth carrying.

**3. `orderflow` has exactly two contended writes, and you have already found them.**

From Topic 52: inventory decrement and wallet debit. Under Topic 65's load, with a
small hot set of products, those two are where every thread in the service converges.
This topic is where you learn to measure that convergence rather than guess at it.

**4. It interacts with virtual threads in a way that changes deployment decisions.**

On JDK 21, a virtual thread blocked inside a `synchronized` block **pins** its carrier
thread — it cannot unmount, so a carrier is consumed while doing nothing. With a
default scheduler sized to the core count, a handful of pinned virtual threads can
starve the whole JVM. JDK 24 (JEP 491) removed most of that pinning.

Which means the sentence "use `synchronized`, it is fine" has a version dependency
now. Verify on your JDK rather than trusting either answer — Topic 101 gives you the
drill and `-Djdk.tracePinnedThreads=full` gives you the evidence.

---

## Machine-level reality

### The mark word

Every Java object header contains a **mark word** — one machine word of mutable
metadata. Topic 69 measured it. Here you use it.

On a classic 64-bit HotSpot layout (no compact headers), the mark word is 64 bits, and
its **low two bits are the lock bits** that select how the rest is interpreted:

```
 63                                                       2   1 0
+---------------------------------------------------------+---+--+
| unused:25 | identity_hashcode:31 | unused:1 | age:4 | 0  | 0 1 |   NEUTRAL (unlocked)
+---------------------------------------------------------+-----+
| pointer to lock record on the owning thread's stack :62        | 0 0 |   STACK-LOCKED (thin)
+----------------------------------------------------------------+
| pointer to the native ObjectMonitor :62                        | 1 0 |   INFLATED (fat)
+----------------------------------------------------------------+
| forwarding pointer used by GC :62                              | 1 1 |   MARKED FOR GC
+----------------------------------------------------------------+
```

*Illustration of the classic layout, not captured output. Verify yours with JOL — see
the Hands-on proof.*

Three things to notice, all of them load-bearing:

1. **The identity hash code lives in the same word as the lock state.** So calling
   `Object.hashCode()` on an object forces the JVM to find somewhere permanent to keep
   the hash — which historically interacted badly with biased locking, and today means
   a hashed object cannot use the same encoding tricks as an unhashed one. Once an
   object has an identity hash and is then locked, the JVM must inflate to a monitor,
   because there is nowhere else to put the hash.
2. **The GC age lives there too**, which is why the mark word is not just "a lock
   field" — it is shared real estate, and that sharing is why the design is constrained.
3. **The bit that used to say "biased" is now unused.** That is the removal, visible in
   the layout.

`[JAVA 25]` **Compact object headers** (`-XX:+UseCompactObjectHeaders`, JEP 519,
production in JDK 25) compress the header from 96 to 64 bits by folding the class
pointer into the mark word, which shrinks the space available for the identity hash and
changes this diagram. The *lock states* remain; the field widths do not. Check before
quoting:

```bash
java -XX:+PrintFlagsFinal -version | grep -i CompactObjectHeaders
```

### The three lock states, and the path between them

**State 1 — neutral.** Nobody holds the lock. Mark word is `...01`.

**State 2 — stack-locked (also called "thin").** Thread A wants the lock:

1. A allocates a **lock record** on its own stack frame — a couple of words.
2. A copies the object's current mark word into that lock record. This copy is called
   the **displaced mark word**.
3. A performs a **CAS** on the object's mark word, replacing it with a pointer to the
   lock record, with the low bits `00`.
4. If the CAS succeeds, A owns the lock. Total cost: one atomic instruction. On x86
   that is a `lock cmpxchg`; on aarch64 a `ldaxr`/`stlxr` pair or an LSE `casal`.

Unlocking is the reverse CAS: swap the displaced mark word back. Also one atomic
instruction.

**This is the fast path, and it is the common case.** Uncontended `synchronized` is
one CAS to enter and one to exit. Single-digit nanoseconds on modern hardware. There
is no kernel, no allocation on the heap, no queue.

If the CAS in step 3 *fails* — because another thread already replaced the mark word —
then two things are possible. Either the current thread already owns it (reentrancy:
the pointer already points into this thread's own stack, so it just pushes another
lock record with a null displaced header and increments its recursion depth), or
another thread owns it, which is **contention**.

**State 3 — inflated (also called "fat").** Contention forces inflation:

1. The JVM allocates a native `ObjectMonitor` — a C++ object in the JVM's own memory,
   not on the Java heap.
2. It copies the displaced mark word into the monitor and points the object's mark word
   at the monitor, with the low bits `10`.
3. From now on, entering the lock goes through the monitor's real queueing machinery.

**Inflation is one-way for the lifetime of the contention.** The object stays inflated
until the JVM decides to deflate it, which since JDK 15 happens asynchronously in a
background thread rather than at a safepoint.

### Inside `ObjectMonitor`

The fields you should be able to name, because they explain every symptom:

| Field | What it holds |
|---|---|
| `_owner` | The thread currently holding the monitor, or null |
| `_recursions` | How many times the owner has re-entered |
| `_cxq` | The **contention queue** — a lock-free LIFO stack of arriving threads |
| `_EntryList` | The queue of threads eligible to be handed the lock next |
| `_WaitSet` | Threads that called `Object.wait()` on this monitor — Topic 89 |
| `_succ` | The designated successor, to avoid waking everyone at once |

The flow when thread B finds the lock held:

1. **B spins briefly.** HotSpot does adaptive spinning: if the lock has recently been
   held for a short time, spinning is cheaper than parking. The spin uses the CPU's
   pause instruction (`pause` on x86, `yield`/`isb` on aarch64 — this is what
   `Thread.onSpinWait()` exposes to you at the Java level).
2. **If spinning fails, B is pushed onto `_cxq` and parked.** `LockSupport.park()` →
   HotSpot `Parker` → on Linux a **futex** (`FUTEX_WAIT`), on macOS a pthread mutex plus
   condition variable. B is now off the CPU entirely; the scheduler will not run it
   again until someone unparks it.
3. **When A exits, it picks a successor** from `_EntryList` (refilling from `_cxq` if
   needed) and **unparks** it: `FUTEX_WAKE` on Linux, `pthread_cond_signal` on macOS.
4. **B wakes, is rescheduled, and re-attempts the CAS.** It may still lose to a thread
   that barged in without queueing — HotSpot monitors are deliberately unfair, because
   barging avoids a full park/unpark round trip and raises throughput.

### What that costs, concretely

Do not memorise numbers; memorise the *ratios* and the reasons.

| Operation | Order of magnitude | Why |
|---|---|---|
| Uncontended lock/unlock | Nanoseconds | One CAS each way, no kernel |
| Contended, resolved by spinning | Tens to hundreds of nanoseconds | Burns CPU but stays in userspace |
| Contended, park + unpark | Microseconds | Two context switches, kernel involvement, plus a cold cache for the woken thread |
| The *indirect* cost of a context switch | Frequently larger than the direct cost | The woken thread's L1/L2 working set is gone; it re-faults it in |

**The two-order-of-magnitude gap between the fast path and the parked path is the
entire answer to "is `synchronized` slow".** It is not slow. Contention is slow. Your
job is to measure which one you have.

### Cache coherence — the cost you cannot see in a profiler

Even the fast path is not free when threads on different cores touch the same object.

The CAS operates on a 64-byte **cache line**. Under the MESI-family coherence protocol
your CPU implements, a core must own that line in **Modified** or **Exclusive** state to
write it. When core 0 CASes the mark word, it invalidates core 1's copy. When core 1
CASes, it invalidates core 0's. The line ping-pongs across the interconnect.

That means: **a lock with very short critical sections and high frequency can be
limited by cache-line ownership transfer rather than by the critical section itself.**
The lock looks cheap in isolation and scales terribly. Topics 95 and 96 develop this;
it is why `LongAdder` exists and why `@Contended` exists.

### Lock elision and lock coarsening — the JIT's contribution

From Topic 75, and directly relevant here:

- **Lock elision.** If escape analysis proves the locked object cannot be reached by
  any other thread, C2 removes the locking entirely. The classic case is a
  `StringBuffer` created and discarded inside one method — all its `synchronized`
  methods, and the locks vanish.
- **Lock coarsening.** If C2 sees the same lock acquired and released repeatedly in a
  tight sequence, it may merge them into one larger critical section. Fewer atomics, a
  longer hold.

Both mean **the locking you wrote is not necessarily the locking that executes**, which
is one more reason a naive `nanoTime` microbenchmark of `synchronized` is fiction: the
JIT may have deleted the thing you are timing. Topic 77.

### x86-TSO versus aarch64, in this topic

The CAS is an atomic instruction on both architectures; atomicity is not the difference.
What differs is the **barrier** the monitor operations must also provide.

- On **x86-64**, a `lock`-prefixed instruction is already a full barrier including
  `StoreLoad`. So the CAS that acquires the lock also supplies the ordering, for free.
- On **aarch64**, the JVM must additionally emit acquire and release semantics —
  typically via `ldaxr`/`stlxr` or the LSE atomics with acquire/release variants, plus
  `dmb ish` where needed — because the hardware does not provide it implicitly.

**Which way this cuts for you on Apple Silicon:** the *lock* is correct on both. What
differs is that on aarch64, code that *forgot* the lock is more likely to expose a
reordering that x86 would have hidden. That matters enormously in Topics 87 and 88 and
hardly at all here, because here you took the lock. Do not expect this topic's drill to
behave differently between architectures.

---

## Concurrency trace

**Read this before any correct code.** The trace is the bug; everything after it is
commentary.

### The scenario

`orderflow` debits a customer's wallet during order placement. Customer 8812 has a
balance of **£50**. They have two orders in flight at the same instant — a genuine
double-submit from a mobile client that retried on a slow response — each for **£40**.

The code under trace, with no synchronisation:

```java
// WalletService — a Spring singleton, shared by all 200 request threads.
private final Map<Long, Long> balancePenceByCustomer = new HashMap<>();

public boolean debit(long customerId, long amountPence) {
    Long balance = balancePenceByCustomer.get(customerId);      // READ
    if (balance == null || balance < amountPence) {
        return false;                                           // insufficient funds
    }
    balancePenceByCustomer.put(customerId, balance - amountPence);  // WRITE
    return true;
}
```

### The interleaving

Balances are in pence: £50 = 5000, £40 = 4000.

| Step | Thread A — `http-nio-8080-exec-11` (order #90210, £40) | Thread B — `http-nio-8080-exec-19` (order #90211, £40) | Shared state / what is visible |
|---|---|---|---|
| 1 | Enters `debit(8812, 4000)` | — | `balance[8812] = 5000` |
| 2 | Map `get` → reads **5000** into a local on A's private stack | — | `balance[8812] = 5000` |
| 3 | Evaluates `5000 < 4000` → false. Sufficient funds. Proceeds | — | `balance[8812] = 5000` |
| 4 | **Preempted here.** A's local `balance = 5000` is frozen on A's stack. Nothing about A's intent exists in shared memory | — | `balance[8812] = 5000`. To any observer, no debit has begun |
| 5 | — | Enters `debit(8812, 4000)` | `balance[8812] = 5000` |
| 6 | — | Map `get` → reads **5000**. This read is *not wrong* — the map genuinely still says 5000 | `balance[8812] = 5000` |
| 7 | — | Evaluates `5000 < 4000` → false. Sufficient funds. Proceeds | `balance[8812] = 5000` |
| 8 | — | `put(8812, 5000 - 4000)` → writes **1000** | `balance[8812] = 1000` |
| 9 | — | Returns `true`. Order #90211 proceeds: payment row written, inventory decremented, order confirmed to the customer | `balance[8812] = 1000`, one order committed |
| 10 | **A resumes.** It does not re-read the map. It still holds `balance = 5000` from step 2 | — | `balance[8812] = 1000` |
| 11 | `put(8812, 5000 - 4000)` → writes **1000** over B's 1000 | — | `balance[8812] = 1000`. Two debits happened; the balance moved once |
| 12 | Returns `true`. Order #90210 proceeds: payment row written, inventory decremented, order confirmed | — | `balance[8812] = 1000`, **two** orders committed |
| **13** | **OUTCOME: the customer had £50 and received £80 of goods. £30 has been given away. The wallet balance reads £10 — a completely plausible number that reconciles with exactly one of the two orders. Finance discovers it in a month-end reconciliation break, by which time the goods have shipped and the account may be empty.** | | |

### The second bug in the same code, which the trace above does not show

Suppose the two threads were *not* interleaved at all. A runs completely, then B starts.
Is B guaranteed to see A's write of 1000?

**No.** There is no happens-before edge between A's write and B's read. Without one:

- A's write may sit in A's core's **store buffer**, not yet visible to B's core.
- The JIT may have kept `balancePenceByCustomer` state in registers.
- On aarch64, the hardware itself may reorder or delay the store's visibility.

So B can read **5000** even though A has already, in wall-clock time, written 1000. The
trace above is a *timing* bug. This second one is a *visibility* bug, and it does not
require any unlucky interleaving at all. Topics 86 and 87 are entirely about it.

**`synchronized` fixes both**, because it provides both mutual exclusion and the
happens-before edge. That is why it is the right first tool and why "I'll just make the
field `volatile`" is not a substitute — `volatile` fixes the second problem and not the
first.

### The third bug, for completeness

`balancePenceByCustomer` is a `HashMap` mutated by 200 threads. Concurrent structural
modification during a resize is undefined behaviour: lost entries, entries under the
wrong key, `get` returning something that was never put. A lost entry for a customer
means their balance reads `null`, which this code treats as "insufficient funds" — so
a paying customer is told their wallet is empty while the database says otherwise.

Three independent defects in seven lines. The fix for all three, applied correctly, is
below.

---

## Example 1 — minimal

### The unguarded version, then the guarded one

```java
public class MonitorBasics {

    static class Wallet {
        private long balancePence;

        Wallet(long balancePence) { this.balancePence = balancePence; }

        /** UNGUARDED. Reproduces the trace above. */
        boolean debitUnsafe(long amount) {
            if (balancePence < amount) return false;
            balancePence -= amount;
            return true;
        }

        /** GUARDED. Mutual exclusion + happens-before, on 'this'. */
        synchronized boolean debitSafe(long amount) {
            if (balancePence < amount) return false;
            balancePence -= amount;
            return true;
        }

        synchronized long balance() { return balancePence; }
        //         ^^^^^^^^^^^^ this matters — see below
    }

    public static void main(String[] args) throws Exception {
        run(true);
        run(false);
    }

    static void run(boolean safe) throws Exception {
        final long startBalance = 1_000_000;
        final long debitAmount  = 1;
        final int  threads      = 8;
        final int  perThread    = 100_000;   // 8 * 100_000 == startBalance exactly

        Wallet wallet = new Wallet(startBalance);
        java.util.concurrent.atomic.AtomicLong succeeded =
                new java.util.concurrent.atomic.AtomicLong();

        Thread[] ts = new Thread[threads];
        for (int i = 0; i < threads; i++) {
            ts[i] = Thread.ofPlatform().name("debiter-" + i).unstarted(() -> {
                for (int n = 0; n < perThread; n++) {
                    boolean ok = safe ? wallet.debitSafe(debitAmount)
                                      : wallet.debitUnsafe(debitAmount);
                    if (ok) succeeded.incrementAndGet();
                }
            });
        }
        for (Thread t : ts) t.start();
        for (Thread t : ts) t.join();

        long finalBalance = wallet.balance();
        System.out.printf("%-8s debitsAccepted=%,d  finalBalance=%,d  accepted+balance=%,d (want %,d)%n",
                safe ? "SAFE" : "UNSAFE",
                succeeded.get(), finalBalance,
                succeeded.get() + finalBalance, startBalance);
    }
}
```

```bash
java MonitorBasics.java
```

**What to look for:** the last column. `debitsAccepted + finalBalance` must equal
`startBalance`. Every pound is either still in the wallet or accounted for by an
accepted debit. That is the invariant, and it is the only number that matters.

| What you see | What it means |
|---|---|
| `SAFE` line: the sum equals 1,000,000 exactly | The monitor did its job. Note that this must be **exact**, every run |
| `UNSAFE` line: the sum is larger than 1,000,000 | **Money created from nothing.** More debits were accepted than the wallet could fund. This is the trace, reproduced |
| `UNSAFE` line: the sum is smaller than 1,000,000 | Also wrong, in the other direction — lost writes meant the balance did not fall as far as the accepted debits imply. Both directions are corruption |
| `UNSAFE` line happens to be exactly right on this run | Possible, especially at low thread counts on a lightly loaded machine. Raise `threads` to 32 and `perThread` to 1,000,000 and re-run. **A clean run is not evidence of safety.** Topic 99 exists because of this |
| Both lines identical and correct, always | Check that `debitUnsafe` is really the one being called. Also try `-Xint`, which changes optimisation and often changes the outcome |

### Why `balance()` is also `synchronized`

That is the part people leave out, and it is the more interesting half.

`balance()` does not mutate anything. It reads one `long`. Why lock it?

**Because the happens-before edge is what makes the read see the writes.** Without the
monitor, the reader has no edge with any writer, so it may read a stale value — or on a
32-bit VM, a torn one. Locking the reader on the *same monitor* the writers use is what
puts it into the ordering.

> **A rule you should adopt permanently: if a field is guarded by a lock, EVERY access
> to it — read and write — must hold that lock.** A `synchronized` setter with an
> unsynchronised getter is a bug that looks like an optimisation.

The alternative for a single field is `volatile`, which gives the visibility without
the exclusion. That is Topic 87, and it works here *only* because `balance()` reads a
single field and makes no decision based on it. The moment a read participates in a
check-then-act, it needs the lock.

---

## Example 2 — production scenario (on the project spine)

### The constraints

From Topic 65's recorded baseline:

- `orderflow` containerised: app + Postgres + Redis, under `docker compose`.
- 100k products, 1M orders, 5M order lines. A small hot set of SKUs.
- k6, open-model arrival rate, 70% catalogue read / 20% order read / 10% placement.
- `server.tomcat.threads.max=200`.
- Recorded p50/p95/p99 per endpoint, committed under `/docs/java/baselines/`.

### Attempt 1 — the fix that works and destroys throughput

The obvious response to the trace is to lock the service:

```java
package com.orderflow.inventory;

import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.Map;

@Service
public class InventoryCache {

    private final Map<String, Integer> availableBySku = new HashMap<>();

    /** Correct. Also a global bottleneck. */
    public synchronized boolean tryReserve(String sku, int quantity) {
        Integer available = availableBySku.get(sku);
        if (available == null || available < quantity) {
            return false;
        }
        availableBySku.put(sku, available - quantity);
        return true;
    }

    /** Also synchronized — same monitor, so reads are consistent. */
    public synchronized Integer available(String sku) {
        return availableBySku.get(sku);
    }

    /** ALSO synchronized, and this is the disaster. */
    public synchronized Map<String, Integer> snapshotForAdminDashboard() {
        return new HashMap<>(availableBySku);       // copies 100,000 entries
    }
}
```

Every one of those methods locks the same monitor: `this`.

**What that means under Topic 65's load:**

1. **Every reservation for every SKU serialises through one monitor.** Two orders for
   two completely unrelated products cannot proceed concurrently. You have taken a
   service with 200 request threads and given it a section that exactly one thread at a
   time may execute.
2. **The admin dashboard endpoint holds the same lock while copying 100,000 map
   entries.** One admin page refresh stops order placement for the whole service for the
   duration of that copy. This is the shape of an outage caused by an internal tool.
3. **Contention inflates the monitor immediately**, so every waiting thread is parked
   and unparked through the OS. At high arrival rates you are paying two context
   switches per reservation.
4. **Throughput stops scaling with cores**, and may *fall* as you add threads, because
   the additional threads only add park/unpark overhead to an already serialised
   section.

**What you would observe** — and you will, in the Failure drill:

- `jcmd <pid> Thread.print` shows many `http-nio-8080-exec-*` threads `BLOCKED (on
  object monitor)` all `waiting to lock` the **same** address.
- JFR emits `jdk.JavaMonitorEnter` events naming `InventoryCache` as the `monitorClass`.
- p99 for `POST /orders` climbs sharply while CPU utilisation stays low. **Low CPU with
  high latency is the signature of a lock, not of a slow computation.**

### Attempt 2 — narrow the lock, and separate the concerns

Three changes, each with a distinct justification.

```java
package com.orderflow.inventory;

import org.springframework.stereotype.Service;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.LongAdder;

@Service
public class InventoryCache {

    /**
     * (1) A ConcurrentHashMap, so the map itself is safe and reads never block.
     *     (2) Per-SKU atomicity via compute(), so unrelated SKUs do not contend.
     */
    private final ConcurrentHashMap<String, Integer> availableBySku = new ConcurrentHashMap<>();

    /** (3) Contention-free counters. Topic 95. */
    private final LongAdder attempts   = new LongAdder();
    private final LongAdder rejections = new LongAdder();

    public boolean tryReserve(String sku, int quantity) {
        attempts.increment();

        boolean[] reserved = { false };
        availableBySku.compute(sku, (k, available) -> {
            if (available == null || available < quantity) {
                return available;
            }
            reserved[0] = true;
            return available - quantity;
        });

        if (!reserved[0]) rejections.increment();
        return reserved[0];
    }

    /** Lock-free read. Sees a consistent value for this key. */
    public Integer available(String sku) {
        return availableBySku.get(sku);
    }

    /** No longer blocks reservations. The snapshot is weakly consistent — say so. */
    public Map<String, Integer> snapshotForAdminDashboard() {
        return Map.copyOf(availableBySku);
    }

    public long attempts()   { return attempts.sum(); }
    public long rejections() { return rejections.sum(); }
}
```

What changed and why:

| Change | Reason |
|---|---|
| `HashMap` → `ConcurrentHashMap` | The map is now safe to mutate concurrently, and reads are lock-free volatile reads (Topic 92) |
| `synchronized` method → `compute` | The atomic unit is now **one key's bin**, not the whole service. Two SKUs no longer contend. `compute` still uses a `synchronized` block internally — on the bin's first node — so this is not "removing the lock", it is **making the lock finer** |
| Counters → `LongAdder` | Striped, so counter updates do not become the new bottleneck (Topic 95) |
| Admin snapshot no longer shares the reservation lock | An internal tool can no longer stop order placement. The snapshot is now *weakly consistent* — it may reflect a mix of before-and-after states — which is the correct trade for a dashboard, and must be documented as such |

**The honest caveat you must state whenever you make this change:** `Map.copyOf` over a
`ConcurrentHashMap` does not give you a point-in-time snapshot. The iterator is weakly
consistent. For a dashboard that is exactly right. For a financial report it is exactly
wrong, and you would need the database.

### Attempt 3 — the design fix, which is not a lock at all

Everything above makes the in-memory guard correct. None of it makes the in-memory
guard *authoritative*. A cache and a database row are two sources of truth, and they
drift across restarts, replicas and deploys.

The invariant belongs where it can be enforced — Topic 52's atomic conditional UPDATE:

```sql
UPDATE inventory
   SET available = available - :qty
 WHERE sku = :sku
   AND available >= :qty
```

Zero rows updated means insufficient stock, in one round trip, with the database's own
concurrency control doing the work you were trying to do in Java.

The in-memory cache then becomes a **load shedder** — a hint that rejects hopeless
requests early. It is allowed to be wrong in the safe direction (letting through an
order the database then refuses) and must never be wrong in the unsafe direction.

> **This is the senior move, and it is worth naming explicitly: the best concurrency
> fix is often to relocate the invariant to something that already enforces it, rather
> than to build enforcement yourself.** You still need to know how to build it — that is
> the rest of this phase — but reaching for a lock first is a mid-level reflex.

### Where `synchronized` *is* still the right answer in `orderflow`

Do not read the above as "never use it". Concretely, in this service:

- **Lazily building an expensive singleton** — a pricing rules engine loaded once at
  first use. `synchronized` inside a holder class, or better, an eagerly-initialised
  static (Topic 88).
- **A short critical section around a genuinely composite in-memory operation** that no
  concurrent collection expresses — for example, updating two related fields of a
  metrics object that must be read together.
- **Guarding a small, non-thread-safe collaborator** you did not write, where the calls
  are infrequent.

The rule: **`synchronized` around work measured in nanoseconds, never around work
measured in milliseconds.** Everything in the traps below is a violation of that one
line.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — holding a monitor across a network call

**Wrong:**

```java
public synchronized PaymentResult charge(Order order) {
    PaymentResult result = paymentGateway.charge(order);   // HTTP. 200ms. Sometimes 5s.
    recordPayment(order, result);
    return result;
}
```

**Exact symptom:** the whole service degrades whenever the payment gateway is slow —
*including endpoints that have nothing to do with payments*, if they touch the same
bean. Concretely:

- p99 on `POST /orders` tracks the gateway's latency exactly, multiplied by the queue
  depth.
- `jcmd <pid> Thread.print` shows nearly every request thread `BLOCKED (on object
  monitor)` waiting on one address, and exactly one thread `RUNNABLE` inside
  `SocketInputStream.read` holding it.
- **CPU utilisation is near zero while latency is enormous.** That combination is the
  fingerprint of a lock.
- Under Topic 65's open-model load, the arrival rate does not fall to match, so the
  queue grows without bound and you eventually exhaust Tomcat's thread pool. Then every
  endpoint returns 503, including `/health`.

**Root cause:** a monitor is held for the duration of the block, and the block contains
a call whose duration you do not control. You have made a third party's latency into
your own service's mutual exclusion window. Throughput through that section is now
capped at `1 / gateway_latency` — about 5 requests per second at 200 ms.

This is precisely the "blocking the event loop" bug you already know from Node, scoped
to one lock instead of the whole process. Your instinct is correct; the lock just made
its blast radius configurable.

**Fix — in order of preference:**

```java
// 1. Do not hold the lock across the call. Lock only the state mutation.
public PaymentResult charge(Order order) {
    PaymentResult result = paymentGateway.charge(order);   // NO lock held
    synchronized (this) {
        recordPayment(order, result);                      // nanoseconds
    }
    return result;
}

// 2. If the call must be serialised per customer, lock per customer, not globally.
//    (Beware: this map must be bounded, or it is Topic 79's leak.)
private final ConcurrentHashMap<Long, Object> perCustomerLocks = new ConcurrentHashMap<>();

public PaymentResult charge(Order order) {
    Object lock = perCustomerLocks.computeIfAbsent(order.customerId(), k -> new Object());
    synchronized (lock) { ... }
}

// 3. If it must be serialised globally, use a Semaphore as an explicit concurrency
//    limit with a timeout, so you shed load instead of queueing forever. Topic 97.
```

And add the timeout at the source: any HTTP client on a request path needs both a
connect timeout and a read timeout, or option 1 alone will not save you.

**On virtual threads specifically:** on JDK 21 this trap is worse than described,
because a virtual thread blocked inside `synchronized` pins its carrier. A handful of
these can starve the entire scheduler. JDK 24's JEP 491 removed most of that pinning —
verify on your JDK. Topic 101 drills exactly this code.

---

### Trap 2 — locking on the wrong object

**Wrong, four ways:**

```java
// (a) Locking on a String literal. Interned — shared across the ENTIRE JVM.
synchronized ("inventory") { ... }

// (b) Locking on a boxed Integer. Values -128..127 are cached and shared. (Topic 01)
private Integer lock = 1;
synchronized (lock) { ... }

// (c) Locking on a non-final field that gets reassigned.
private Object lock = new Object();
public void reconfigure() { lock = new Object(); }     // now two threads hold "the" lock
synchronized (lock) { ... }

// (d) Locking on 'this' in a public class, so callers can lock you too.
public synchronized void tryReserve(...) { ... }
```

**Exact symptoms, which differ per case:**

- **(a)** Unrelated classes in unrelated libraries that happen to use the same literal
  block each other. You see `BLOCKED` threads whose stacks have nothing in common. The
  monitor address in the dump belongs to a `java.lang.String`, which is the tell.
- **(b)** The same, but the monitor class is `java.lang.Integer`. Worse: if the value
  ever goes above 127 the boxing produces a *new* object and the lock silently stops
  excluding anything at all. Intermittent, value-dependent correctness.
- **(c)** Two threads execute the "same" critical section at the same time, because
  they read different values of `lock`. Silent data corruption with no `BLOCKED`
  threads anywhere. Undetectable from a thread dump.
- **(d)** A caller does `synchronized (inventoryCache) { ... }` for their own reasons
  and now serialises your service, or deadlocks against it. You will not find this by
  reading your own class.

**Root cause:** the monitor identity is the *object*, and any object that can be
reached by other code can be locked by other code. Interned strings, cached boxes and
`Class` objects are all globally reachable.

**Fix — one line, and adopt it as a habit:**

```java
private final Object inventoryLock = new Object();
```

`private`, so nobody else can reach it. `final`, so it cannot be reassigned. A plain
`Object`, so it has no other purpose and no other meaning. Name it after what it
guards.

**And document what it guards.** Java has no way to express "this lock protects those
fields" in the type system, so it goes in a comment — or, if your build uses JSR-305 /
Error Prone, in a `@GuardedBy("inventoryLock")` annotation that a static analyser can
actually check. That annotation is worth adding to `orderflow`; it turns a convention
into a build failure.

---

### Trap 3 — inconsistent locking: guarding some accesses but not all

**Wrong:**

```java
public class InventoryCache {
    private int available;

    public synchronized void reserve(int qty) { available -= qty; }

    public int available() { return available; }        // NOT synchronized
}
```

Or the subtler static/instance variant:

```java
public synchronized void reserve(int qty)      { available -= qty; }
public static synchronized void resetAll()     { /* touches the same state */ }
```

**Exact symptom for the first form:** the read returns stale or nonsensical values, and
does so *intermittently and more often under load*. Because it is a single `int`, it
never tears, so you get a plausible-looking wrong number rather than an exception.
Anything computed from it — a rejection rate, a dashboard, a decision to reorder stock —
is wrong. There are no `BLOCKED` threads and no error in any log.

**Exact symptom for the second form:** genuine simultaneous execution of two critical
sections that both mutate the same state. `reserve` and `resetAll` can run at the same
instant on the same data. You will find it as corrupted state after an admin action.

**Root cause:**

- Form 1: the unsynchronised read has no happens-before edge with the synchronised
  write. The lock provides ordering only to code that *takes* it. **A lock is not a
  property of a field; it is an agreement among all the code that touches the field.**
- Form 2: `synchronized` on an instance method locks `this`. `static synchronized`
  locks the `Class` object. **Two different monitors.** They do not exclude each other,
  at all, ever.

**Fix:**

```java
public class InventoryCache {
    private final Object lock = new Object();
    private int available;                     // @GuardedBy("lock")

    public void reserve(int qty)  { synchronized (lock) { available -= qty; } }
    public int  available()       { synchronized (lock) { return available; } }
    public void resetAll()        { synchronized (lock) { available = 0; } }
}
```

One monitor, named, private, final, taken by every access including the reads and
including anything static.

**The review heuristic:** for any field, list every method that touches it. If they do
not all take the same lock, you have a bug — even if the odd one out is "only a read".

---

### Trap 4 — double-checked locking without `volatile`

**Wrong:**

```java
public class PricingEngineHolder {
    private static PricingEngine instance;              // NOT volatile

    public static PricingEngine get() {
        if (instance == null) {                          // check 1, unlocked
            synchronized (PricingEngineHolder.class) {
                if (instance == null) {                  // check 2, locked
                    instance = new PricingEngine();      // <-- the bug
                }
            }
        }
        return instance;
    }
}
```

**Exact symptom:** intermittently, some thread receives a `PricingEngine` whose fields
are still `null` or `0`. The resulting `NullPointerException` comes from *inside* the
engine, on a field the constructor demonstrably sets. Every reviewer reads the
constructor, sees the field being set, and concludes the NPE is impossible. It happens
once per few thousand cold starts, so it is dismissed as flaky.

Under Topic 65's load the failure is concentrated at the very start of a run, when many
threads race to initialise for the first time — which is exactly when nobody is
watching because "it's just warming up".

**Root cause — and this is the mechanism you must be able to state:**

`instance = new PricingEngine()` is not one operation. It is three:

1. allocate memory for the object,
2. run the constructor, initialising its fields,
3. write the object's address into `instance`.

**The JIT and the CPU are permitted to reorder 2 and 3**, because within a single
thread the reordering is unobservable — and the JMM only constrains what is observable
within a thread unless you establish an edge. So another thread, executing check 1 with
no lock held, can observe a non-null `instance` pointing at an object whose constructor
has not finished.

**Fix — three options, in order of preference:**

```java
// 1. BEST for a static singleton: the holder idiom. Class initialisation is
//    guaranteed thread-safe and lazy by the JVM itself. No locking in your code.
public class PricingEngineHolder {
    private static class Holder {
        static final PricingEngine INSTANCE = new PricingEngine();
    }
    public static PricingEngine get() { return Holder.INSTANCE; }
}

// 2. If it must be an instance field: volatile. This is the ONLY thing that makes
//    double-checked locking correct, and it has been since Java 5.
private volatile PricingEngine instance;

// 3. In Spring: just declare it a @Bean and let the container do it once, eagerly.
```

Option 1 is the one to reach for. The JVM specification guarantees that a class is
initialised exactly once, with proper synchronisation, on first *active use* — so
`Holder.INSTANCE` is lazy, thread-safe, lock-free on every call after the first, and
requires you to write no concurrency code at all.

**Topic 88 is the full account of why this happens and why `final` fields change the
answer.** Note it now: if every field of `PricingEngine` were `final`, the freeze at
constructor end would prevent the partially-constructed observation — which is a
remarkable and non-obvious property, and it is that topic's whole payoff.

---

### Trap 5 — "`synchronized` is slow, use `ReentrantLock`"

**Wrong:** replacing `synchronized` with `ReentrantLock` throughout a codebase on the
basis of a benchmark from 2009.

**Exact symptom:** no measurable performance change, plus a new class of bug. Somebody
eventually writes:

```java
lock.lock();
doWork();                 // throws
lock.unlock();            // never reached. The lock is now held forever.
```

and the service hangs permanently, requiring a restart. `jcmd Thread.print` shows every
thread `WAITING` (parked, not `BLOCKED`) with `Locked ownable synchronizers` naming the
`ReentrantLock` — and the thread that owns it is off doing something else entirely,
having forgotten it holds a lock.

**Root cause:** two of them.

1. **The performance premise is dated.** In the Java 5 era, `ReentrantLock`
   outperformed `synchronized` under contention. HotSpot's monitor implementation then
   improved substantially, and the gap closed. Today the two perform comparably in the
   uncontended case (both a CAS) and comparably under contention (both park). Anyone
   quoting a large difference is quoting a decade-old measurement.
2. **`synchronized` cannot leak; `ReentrantLock` can.** `synchronized` is
   block-structured — `javac` emits the exception-path `monitorexit` for you. An
   explicit lock requires a `try`/`finally` that a human must remember.

**Fix:**

```java
lock.lock();
try {
    doWork();
} finally {
    lock.unlock();        // ALWAYS. No exceptions. Never anywhere else.
}
```

**And the actual decision rule.** Reach for `ReentrantLock` when you need something
`synchronized` genuinely cannot do:

| Need | Why `synchronized` cannot |
|---|---|
| `tryLock()` or `tryLock(timeout)` | No timeout exists. This is how you *avoid* deadlock rather than diagnose it |
| Interruptible acquisition | A `BLOCKED` thread ignores `interrupt()` |
| Fairness | Monitors are deliberately barging |
| Multiple wait conditions | One monitor has one wait set; `newCondition()` gives you many |
| Non-block-structured locking (hand-off, lock coupling) | The block structure forbids it |
| **Virtual threads on JDK 21–23** | A virtual thread pins its carrier inside `synchronized`; on a `ReentrantLock` it unmounts cleanly. Verify against your JDK — JEP 491 in JDK 24 changed this |

That last row is the one that actually changes real decisions in 2026, and it is
Topic 94 and Topic 101.

---

### Trap 6 — assuming `synchronized` composes

**Wrong:**

```java
// Every individual method here is synchronized and correct.
if (!inventory.isAvailable(sku, qty)) {      // atomic
    throw new OutOfStockException(sku);
}
inventory.reserve(sku, qty);                  // atomic
```

**Exact symptom:** oversell, exactly as in Topic 84's trace, despite every method
having the `synchronized` keyword on it. Code review passes because every line looks
guarded.

**Root cause:** each *call* is atomic. The *sequence* is not. The monitor is released
between them, and another thread can reserve the last unit in that gap. This is
check-then-act, and adding `synchronized` to the individual operations does nothing to
close it.

**Fix:** make the compound operation the atomic unit.

```java
// The decision and the mutation happen inside ONE critical section.
public boolean tryReserve(String sku, int qty) {
    synchronized (inventoryLock) {
        int available = availableBySku.getOrDefault(sku, 0);
        if (available < qty) return false;
        availableBySku.put(sku, available - qty);
        return true;
    }
}
```

**The general principle, which Topic 92 restates for `ConcurrentHashMap`:**

> **Thread-safe operations do not compose into thread-safe sequences.** The unit of
> atomicity must match the unit of your invariant. If your invariant is "never reserve
> more than is available", then check-and-reserve is one operation, and it must be
> written as one.

---

## Hands-on proof

Every command below is one you run. I have no JVM and will not print output and call it
real. You get the exact invocation, what to look for, and how to read what comes back.

### Setup

```bash
mkdir -p ~/java-lab/85 && cd ~/java-lab/85
java --version
uname -m
```

### Proof 1 — `monitorenter` / `monitorexit`, and the method that has neither

`Locks.java`:

```java
public class Locks {
    private int available = 10;
    private final Object lock = new Object();

    // Form A: a block.
    void reserveWithBlock() {
        synchronized (lock) {
            available--;
        }
    }

    // Form B: a synchronized method.
    synchronized void reserveWithMethod() {
        available--;
    }

    // Form C: static synchronized.
    static synchronized void resetAll() { }
}
```

```bash
javac Locks.java
javap -c -p Locks.class          # bodies
javap -v -p Locks.class | grep -A3 "reserveWithMethod\|resetAll\|flags"
```

**What to look for:**

| What you see | What it means |
|---|---|
| In `reserveWithBlock`: one `monitorenter` and **two** `monitorexit` | Correct. The second is in the synthetic exception handler, so the monitor is released even if the body throws. **This is why `synchronized` cannot leak a lock** |
| An `Exception table` entry in `reserveWithBlock` covering the body, target `any` | The compiler-generated unlock-on-throw path. Confirm it exists |
| In `reserveWithMethod`: **no** `monitorenter` at all | Correct, and surprising the first time. Look at the method's flags instead |
| `flags: (0x0020) ACC_SYNCHRONIZED` on `reserveWithMethod` | There it is. The JVM acquires the monitor at method entry from this flag; there is no bytecode for it |
| `flags: (0x0028) ACC_STATIC, ACC_SYNCHRONIZED` on `resetAll` | Static and synchronized — this one locks the `Class` object, a **different** monitor from `this`. Trap 3 |

Topic 76 is the full reading guide if any of the operand syntax is unfamiliar.

### Proof 2 — watch the mark word change under a lock

This is the most direct evidence available that lock state lives in the object header.

```bash
curl -O https://repo1.maven.org/maven2/org/openjdk/jol/jol-cli/0.17/jol-cli-0.17-full.jar
```

`MarkWord.java`:

```java
import org.openjdk.jol.info.ClassLayout;

public class MarkWord {
    public static void main(String[] args) throws Exception {
        Object o = new Object();

        System.out.println("=== 1. fresh, never locked, never hashed ===");
        System.out.println(ClassLayout.parseInstance(o).toPrintable());

        System.out.println("=== 2. inside a synchronized block (uncontended) ===");
        synchronized (o) {
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }

        System.out.println("=== 3. after the block, released ===");
        System.out.println(ClassLayout.parseInstance(o).toPrintable());

        System.out.println("=== 4. after identityHashCode ===");
        System.out.println("hash = " + Integer.toHexString(System.identityHashCode(o)));
        System.out.println(ClassLayout.parseInstance(o).toPrintable());

        System.out.println("=== 5. hashed AND locked ===");
        synchronized (o) {
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
    }
}
```

```bash
javac -cp jol-cli-0.17-full.jar MarkWord.java
java  -cp .:jol-cli-0.17-full.jar MarkWord
```

**What to look for:** the first 8 bytes of each dump — the mark word — and specifically
its **low two bits**.

| What you see | What it means |
|---|---|
| Section 1: mark word low bits `01`, most of the word zero | Neutral. Unlocked, unhashed |
| Section 2: low bits `00`, and the rest of the word is a large value that looks like an address | **Stack-locked.** That value is a pointer to a lock record on `main`'s stack. This is the fast path, and you are looking directly at it |
| Section 3: back to `01` and identical to section 1 | The displaced mark word was restored on unlock. Confirms the swap-and-restore mechanism |
| Section 4: low bits `01`, but now the word contains your hash value | The identity hash is stored *in the mark word*. Compare it to the printed `hash` |
| Section 5: low bits `10`, or a value that no longer contains your hash | **Inflated.** The hash had nowhere to live once the word was needed for locking, so the JVM allocated an `ObjectMonitor` and moved it there. You have just observed inflation caused by hashing rather than by contention |
| Section 5 still shows `00` | Also possible depending on JVM version and flags — HotSpot's exact policy has changed across releases. Note what you got; do not assume mine |
| Everything is 8 bytes shorter than you expected | You are on JDK 25 with compact headers enabled. Confirm with `-XX:+PrintFlagsFinal -version \| grep -i CompactObjectHeaders`, and re-run with `-XX:-UseCompactObjectHeaders` to compare |

Section 5 is the interesting one and it is worth sitting with. **The lock state and the
identity hash compete for the same bits.** That is not an implementation curiosity — it
is why the mark word design is as constrained as it is, and it is a genuinely good
answer to "tell me something about Java object headers."

### Proof 3 — biased locking is gone

```bash
java -XX:+PrintFlagsFinal -version | grep -i bias
java -XX:+UseBiasedLocking -version
```

| What you see | What it means |
|---|---|
| The `grep` returns nothing | Correct on JDK 18+. The flag no longer exists. This is your evidence that any tuning advice mentioning it is obsolete |
| `Unrecognized VM option 'UseBiasedLocking'` and the JVM refuses to start | Confirmed removed. Note the exact wording — recognising it saves you time when you inherit an old startup script |
| The flag exists and is `false` | You are on JDK 15–17: disabled by default (JEP 374) but not yet removed |
| The flag exists and is `true` | You are on JDK 14 or older. Nothing else in this document changes, but the fast path you measure will differ |

**Why to run this:** you will inherit a `JAVA_OPTS` from someone's 2016 Confluence page
that sets `-XX:BiasedLockingStartupDelay=0`, and the container will refuse to start with
a message nobody recognises. Now you will.

### Proof 4 — produce a `BLOCKED` thread and read it in a dump

`Blocker.java`:

```java
public class Blocker {
    static final Object INVENTORY_LOCK = new Object();

    public static void main(String[] args) throws Exception {
        Thread holder = Thread.ofPlatform().name("holder").start(() -> {
            synchronized (INVENTORY_LOCK) {
                try { Thread.sleep(600_000); } catch (InterruptedException e) { }
            }
        });

        Thread.sleep(200);

        for (int i = 0; i < 5; i++) {
            Thread.ofPlatform().name("waiter-" + i).start(() -> {
                synchronized (INVENTORY_LOCK) {
                    System.out.println("got it");
                }
            });
        }

        System.out.println("pid = " + ProcessHandle.current().pid());
        Thread.sleep(600_000);
    }
}
```

```bash
java Blocker.java &
sleep 2
jcmd <pid> Thread.print > blocked.txt

# Now read it:
grep 'java.lang.Thread.State' blocked.txt | sort | uniq -c
grep -n 'waiting to lock' blocked.txt
grep -n '\- locked' blocked.txt
```

**What to look for:** the monitor address. It appears once with `- locked` (the owner)
and five times with `- waiting to lock` (the waiters).

| What you see | What it means |
|---|---|
| 5 threads `BLOCKED (on object monitor)`, 1 thread `TIMED_WAITING` | Exactly right. The `TIMED_WAITING` one is the holder, asleep *while holding the lock* — which is Trap 1 in miniature |
| The same `<0x...>` address on one `- locked` line and five `- waiting to lock` lines | **This is the technique.** Grep one address to find the owner and count the waiters. It is how you diagnose contention from a dump in ten seconds |
| The monitor class shown in parentheses is `java.lang.Object` | Because that is what `INVENTORY_LOCK` is. In a real service this is where you would see `com.orderflow.inventory.InventoryCache` — and that class name is your entire diagnosis |
| `Locked ownable synchronizers: - <none>` on all of them | Correct. That section is for `ReentrantLock` and friends. Intrinsic monitors never appear there. **That is how you tell at a glance which locking mechanism a service uses** |

Kill it when done: `kill %1`.

### Proof 5 — the uncontended fast path really is cheap

Do this with JMH, not with `nanoTime`. See the Measurement section for why. The
benchmark shape:

```java
@Benchmark @Threads(1)
public int uncontendedSynchronized() {
    synchronized (lock) { return ++counter; }
}

@Benchmark @Threads(1)
public int noLockAtAll() {
    return ++counter;
}
```

**What to look for:** the ratio between the two at `-t 1`.

| What you see | What it means |
|---|---|
| `uncontendedSynchronized` is within a small multiple of `noLockAtAll` | Expected. One CAS each way. This is the number that makes "`synchronized` is slow" false |
| They are nearly identical | Suspect **lock elision** — if C2 proved `lock` does not escape, it deleted the locking entirely. Make `lock` a static field reachable from elsewhere so it cannot be elided, and re-run. Topic 75 |
| `uncontendedSynchronized` is dramatically slower even at one thread | Check `-t`. If more than one thread is running you are measuring contention, not the fast path — which is a different and also valid measurement, just not this one |

---

## Failure drill

**Mandatory.** This is Topic 85's assigned drill from Section G of the master plan:

> **85** — `synchronized` inventory decrement under load → capture JFR monitor-blocked
> events → *contention is measurable, not guessable*.

Do not read the "How to read it" table until you have your own numbers written down.

### The scenario

You apply the *correct* fix from Example 2, Attempt 1 — a `synchronized` method on the
inventory decrement — and run Topic 65's load. Correctness is now perfect. Throughput
is not. Your job is to **quantify the contention** rather than describe it.

This drill is the mirror image of Topic 84's. There, you produced an invisible
correctness bug that no thread dump could show you. Here, you produce a visible
performance cost that JFR measures precisely. **That is the trade `synchronized` makes,
and this drill is where you put a number on it.**

### Setup

`src/main/java/com/orderflow/lab/ContentionDrillController.java`:

```java
package com.orderflow.lab;

import org.springframework.context.annotation.Profile;
import org.springframework.web.bind.annotation.*;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@Profile("drill")
@RestController
@RequestMapping("/drill/contention")
public class ContentionDrillController {

    /** MODE A: one global monitor. Correct, maximally contended. */
    private final Object globalLock = new Object();
    private final Map<String, Integer> guardedBySku = new HashMap<>();

    /** MODE B: per-key atomicity. Correct, striped. */
    private final ConcurrentHashMap<String, Integer> chmBySku = new ConcurrentHashMap<>();

    private final AtomicLong accepted = new AtomicLong();

    @PostMapping("/reset")
    public Map<String, Object> reset(@RequestParam int stock,
                                     @RequestParam(defaultValue = "1") int skuCount) {
        synchronized (globalLock) {
            guardedBySku.clear();
            chmBySku.clear();
            for (int i = 0; i < skuCount; i++) {
                guardedBySku.put("SKU-" + i, stock);
                chmBySku.put("SKU-" + i, stock);
            }
        }
        accepted.set(0);
        return Map.of("skuCount", skuCount, "stockEach", stock);
    }

    /** MODE A. Every SKU contends on ONE monitor. */
    @PostMapping("/reserve/global")
    public Map<String, Object> reserveGlobal(@RequestParam String sku) {
        boolean ok;
        synchronized (globalLock) {
            Integer available = guardedBySku.get(sku);
            ok = available != null && available >= 1;
            if (ok) guardedBySku.put(sku, available - 1);
        }
        if (ok) accepted.incrementAndGet();
        return Map.of("reserved", ok);
    }

    /** MODE B. Only the same SKU contends. */
    @PostMapping("/reserve/striped")
    public Map<String, Object> reserveStriped(@RequestParam String sku) {
        boolean[] ok = { false };
        chmBySku.compute(sku, (k, available) -> {
            if (available == null || available < 1) return available;
            ok[0] = true;
            return available - 1;
        });
        if (ok[0]) accepted.incrementAndGet();
        return Map.of("reserved", ok[0]);
    }

    @GetMapping("/state")
    public Map<String, Object> state(@RequestParam String sku) {
        synchronized (globalLock) {
            return Map.of(
                    "accepted", accepted.get(),
                    "globalRemaining", guardedBySku.get(sku),
                    "stripedRemaining", chmBySku.get(sku));
        }
    }
}
```

`k6/contention.js`:

```javascript
import http from 'k6/http';
import { check } from 'k6';

const MODE     = __ENV.MODE     || 'global';     // 'global' or 'striped'
const SKUCOUNT = parseInt(__ENV.SKUCOUNT || '1');
const RATE     = parseInt(__ENV.RATE     || '3000');

export const options = {
  scenarios: {
    burst: {
      executor: 'constant-arrival-rate',
      rate: RATE,
      timeUnit: '1s',
      duration: '30s',
      preAllocatedVUs: 400,
      maxVUs: 800,
    },
  },
  thresholds: { http_req_failed: ['rate<0.05'] },
};

export default function () {
  const sku = 'SKU-' + Math.floor(Math.random() * SKUCOUNT);
  const res = http.post(`http://localhost:8080/drill/contention/reserve/${MODE}?sku=${sku}`);
  check(res, { 'status 200': (r) => r.status === 200 });
}
```

### Commands

```bash
# 1. Launch with JFR, and LOWER the monitor-enter threshold so short blocks are
#    recorded. The default profile threshold is on the order of 10-20ms, which is
#    far too coarse for lock contention on an in-memory operation.
./mvnw spring-boot:run \
  -Dspring-boot.run.profiles=load,drill \
  -Dspring-boot.run.jvmArguments="\
-XX:StartFlightRecording=name=contention,filename=contention-global.jfr,settings=profile,dumponexit=true,jdk.JavaMonitorEnter#threshold=1ms \
-XX:+UnlockDiagnosticVMOptions"

# 2. Confirm the recording is running and the setting took effect.
jcmd $(jcmd -l | grep -i orderflow | cut -d' ' -f1) JFR.check verbose | grep -i monitor

# 3. Seed a single hot SKU. Maximum contention.
curl -s -XPOST 'http://localhost:8080/drill/contention/reset?stock=100000000&skuCount=1' | jq

# 4. MODE A: global monitor, one SKU.
MODE=global SKUCOUNT=1 RATE=3000 k6 run k6/contention.js

# 5. Snapshot a dump WHILE the load is running (third terminal).
jcmd <pid> Thread.print > during-global.txt

# 6. Dump and read the recording.
jcmd <pid> JFR.dump name=contention filename=contention-global.jfr
jfr summary contention-global.jfr
jfr print --events jdk.JavaMonitorEnter contention-global.jfr | head -80
```

Then repeat for the other three cells of the experiment:

```bash
# MODE A with 1000 SKUs — same lock, but the critical sections touch different keys.
curl -s -XPOST '.../reset?stock=100000&skuCount=1000'
MODE=global   SKUCOUNT=1000 RATE=3000 k6 run k6/contention.js

# MODE B with 1 SKU — striped locking, but only one stripe is used.
MODE=striped  SKUCOUNT=1    RATE=3000 k6 run k6/contention.js

# MODE B with 1000 SKUs — striped locking actually striping.
MODE=striped  SKUCOUNT=1000 RATE=3000 k6 run k6/contention.js
```

### Reading `jdk.JavaMonitorEnter`

The event fields you care about:

| Field | Meaning |
|---|---|
| `duration` | **How long this thread was blocked waiting to enter.** The number the drill is about |
| `monitorClass` | Which class's object was the monitor. Your diagnosis in one field |
| `previousOwner` | The thread that was holding it. Tells you *who* you are queueing behind |
| `address` | The monitor's identity, for correlating with a thread dump |
| `eventThread` | Who was blocked |
| `stackTrace` | Where in your code the blocking happened |

Useful aggregations:

```bash
# How many blocking events, and on what?
jfr summary contention-global.jfr | grep -i monitor

# Which class is the contended monitor?
jfr print --events jdk.JavaMonitorEnter contention-global.jfr \
  | grep 'monitorClass' | sort | uniq -c | sort -rn

# Which thread is most often the previous owner? (i.e. who holds it longest)
jfr print --events jdk.JavaMonitorEnter contention-global.jfr \
  | grep 'previousOwner' | sort | uniq -c | sort -rn | head

# Total blocked time — sum the durations. (jq if you export JSON.)
jfr print --json --events jdk.JavaMonitorEnter contention-global.jfr \
  | jq '[.recording.events[].values.duration] | length'
```

Java Mission Control (`jmc`) gives you the same data on its **Lock Instances** page,
with the durations already aggregated. Use the CLI to understand the shape; use JMC when
you want the answer fast.

### What to capture

For **each of the four cells**, write down:

1. k6 throughput (requests/sec actually completed) and p50 / p95 / p99.
2. `jdk.JavaMonitorEnter` **event count** from `jfr summary`.
3. The dominant `monitorClass`.
4. Total blocked time, and blocked time as a **percentage of wall-clock time × threads**.
   That percentage is your contention rate, and it is the number this drill exists to
   produce.
5. From `during-*.txt`: how many threads were `BLOCKED`, and on how many distinct
   monitor addresses.
6. The correctness check: `accepted + remaining == seeded`, exactly.

### How to read it

| What you see | What it means |
|---|---|
| MODE A / 1 SKU: many `jdk.JavaMonitorEnter` events, all with the same `monitorClass` | **The drill has fired.** You have measured contention rather than guessed at it. The `monitorClass` field names your bottleneck directly |
| MODE A / 1 SKU: p99 far above p50, while CPU utilisation stays low | The classic lock signature. **Low CPU plus high latency means queueing, not computing.** Write this pairing down; it is how you recognise a lock problem in a dashboard you have never seen before |
| MODE A / 1000 SKUs: roughly the same contention as 1 SKU | **The important result.** The lock does not care that the keys differ — one monitor serialises everything. This is the cost of lock *granularity*, isolated from the cost of *actual* data contention |
| MODE B / 1000 SKUs: far fewer monitor events, higher throughput | Striping works. `ConcurrentHashMap` locks the bin, so different keys proceed in parallel |
| MODE B / 1 SKU: contention comparable to MODE A | **Also the important result.** Striping cannot help when everyone wants the same key. If your hot set is one SKU, no amount of finer locking fixes it — you need a different algorithm (an atomic DB update, or per-thread reservations reconciled later) |
| Zero `jdk.JavaMonitorEnter` events despite obvious queueing | Your threshold is too high. Re-launch with `jdk.JavaMonitorEnter#threshold=1ms` or lower, and confirm with `JFR.check verbose`. Blocking that lasts under the threshold is simply not recorded |
| Very few events but throughput is still capped | The contention may be resolving in the **adaptive spin** phase, before any park happens — spinning is not a monitor-enter event of significant duration. Look at CPU utilisation: high CPU with flat throughput is spinning; low CPU with flat throughput is parking |
| `accepted + remaining != seeded` in any mode | Stop. Your fix is not correct, and no performance number from that run means anything. Correctness first, always |
| Throughput in MODE A **falls** as you raise `RATE` | Expected past the saturation point. Beyond the lock's capacity, extra arrivals add only park/unpark overhead. This is a **congestion collapse** curve and it is worth seeing once |

### Now quantify it properly

Write the following sentence with your own numbers filled in. This is the deliverable:

> "Under Topic 65's load at *R* requests/second against *N* hot SKUs, the global
> `synchronized` inventory decrement produced *E* monitor-enter events totalling *T* ms
> of blocked time across *W* threads — *P*% of available thread-time — capping
> throughput at *X* requests/second with p99 of *L* ms. Replacing the global monitor
> with per-key atomicity reduced *E* by *Y*% and raised throughput to *Z*, but only when
> the hot set exceeded one SKU; at a single hot SKU the two were within *D*% of each
> other, because the contention is on the data, not on the lock granularity."

If you can write that sentence from your own measurements, you can answer any
`synchronized` question in an interview, because you will be the only candidate with
numbers.

### Then fix it and re-measure

1. **Narrow the critical section.** Move anything that is not the actual state mutation
   outside the block. Re-measure.
2. **Stripe the lock.** `ConcurrentHashMap.compute`, or an array of lock objects indexed
   by `hash(sku) & (STRIPES-1)`. Re-measure.
3. **Remove the lock entirely** by moving the invariant to the database (Topic 52).
   Re-measure end-to-end p99, and note that the JVM-level contention disappears while
   the *system-level* contention moves to Postgres — where it is at least visible in
   `pg_stat_activity`.
4. **Write down what each fix did not solve.** Fix 3 does not help if the database is
   the bottleneck. Fix 2 does not help at one hot key. Fix 1 helps always and is nearly
   free — which is why it is first.

---

## Measurement

### The standing rule

> **A naive `System.nanoTime()` loop is not a measurement.**

For lock benchmarking specifically it is worse than usual, because the JIT may have
**elided the lock you are timing** (Topic 75). You would measure an unlocked loop and
publish it as the cost of locking.

```java
// WRONG on at least five counts.
long t0 = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    synchronized (lock) { counter++; }
}
System.out.println((System.nanoTime() - t0) / 10_000_000 + " ns/op");
```

1. Lock elision may delete the `synchronized` entirely if `lock` does not escape.
2. Lock coarsening may merge ten million acquisitions into a handful.
3. Dead-code elimination may delete `counter++`.
4. On-stack replacement blends interpreted, C1 and C2 execution into one average.
5. Single-threaded, so it measures only the fast path and says nothing about
   contention — which is the only thing anyone actually asks about.

Topic 77 is the full account. Never quote a lock cost from a `nanoTime` loop.

### JMH with `@Threads` — the shape that answers the real question

Contention is a function of thread count. **The measurement is a sweep, not a number.**

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;
import java.util.concurrent.locks.ReentrantLock;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)
@State(Scope.Benchmark)                  // ONE shared instance: this is the point
public class LockContentionBenchmark {

    /** How many distinct keys the threads spread across. 1 = maximum contention. */
    @Param({"1", "16", "256"})
    int keys;

    /** Static so escape analysis cannot elide the lock. */
    private static final Object MONITOR = new Object();
    private final ReentrantLock reentrant = new ReentrantLock();
    private final long[] counters = new long[1024];
    private final ConcurrentHashMap<Integer, Long> chm = new ConcurrentHashMap<>();
    private final LongAdder adder = new LongAdder();

    @Setup(Level.Iteration)
    public void seed() {
        for (int i = 0; i < keys; i++) chm.put(i, 0L);
    }

    @State(Scope.Thread)
    public static class PerThread {
        int cursor;
        int next(int keys) { return (cursor++) % keys; }
    }

    @Benchmark
    public long baselineNoSharing(PerThread t) {
        return t.cursor++;                          // control: no sharing at all
    }

    @Benchmark
    public long intrinsicMonitor(PerThread t) {
        int k = t.next(keys);
        synchronized (MONITOR) {                    // ONE monitor regardless of key
            return ++counters[k];
        }
    }

    @Benchmark
    public long reentrantLock(PerThread t) {
        int k = t.next(keys);
        reentrant.lock();
        try {
            return ++counters[k];
        } finally {
            reentrant.unlock();                     // ALWAYS in finally
        }
    }

    @Benchmark
    public long concurrentMapCompute(PerThread t) {
        int k = t.next(keys);
        return chm.compute(k, (key, v) -> v + 1);   // per-bin locking
    }

    @Benchmark
    public void striped(PerThread t) {
        adder.increment();                          // no lock at all
    }
}
```

```bash
mvn clean verify

# The sweep. This is the measurement.
java -jar target/benchmarks.jar LockContentionBenchmark \
     -t 1 -t 2 -t 4 -t 8 -t 16 -t 32 \
     -rf json -rff locks.json

# With the GC profiler, to check nothing here allocates unexpectedly
java -jar target/benchmarks.jar LockContentionBenchmark -prof gc

# On Linux with perf available, see the actual instructions
java -jar target/benchmarks.jar LockContentionBenchmark.intrinsicMonitor -prof perfasm
```

**What to look for:** the *shape* of each curve as `-t` rises.

| What you see | What it means |
|---|---|
| `baselineNoSharing` scales roughly linearly with `-t` | Your control is sound and your machine has that many usable cores. If this does not scale, nothing else you measure is interpretable |
| `intrinsicMonitor` flat across all thread counts, and identical at `keys=1` and `keys=256` | Correct and expected. A single monitor does not care about your keys. **This is the number that justifies finer locking** |
| `intrinsicMonitor` **falls** as threads rise | Past saturation: extra threads add park/unpark cost to a section that was already serialised. Congestion collapse |
| `reentrantLock` within noise of `intrinsicMonitor` | Expected on a modern JVM. This is the measurement that kills the "ReentrantLock is faster" folklore. Report it |
| `concurrentMapCompute` scales at `keys=256` but not at `keys=1` | Striping works when the load is spread and does nothing when it is not. **This is the single most useful result in the whole benchmark** |
| `striped` (LongAdder) scales best of all | Expected — no lock, and per-thread cells avoid cache-line ping-pong. Topics 95 and 96 |
| `intrinsicMonitor` at `-t 1` is close to `baselineNoSharing` | The uncontended fast path. One CAS. This is "`synchronized` is not slow", measured |
| Wild variance between forks | Look at per-fork numbers. On a laptop, thermal throttling and background processes dominate contention benchmarks. Close everything, plug in, re-run |
| A suspiciously fast result | Suspect elision or coarsening. Check with `-prof perfasm` if available, or make the lock object unmistakably escaping |

### JFR — the events for this topic

```bash
# Start with the monitor threshold lowered; the defaults are too coarse for locks.
java -XX:StartFlightRecording=settings=profile,filename=of.jfr,dumponexit=true,jdk.JavaMonitorEnter#threshold=1ms -jar app.jar

# Or on a live JVM
jcmd <pid> JFR.start name=locks settings=profile jdk.JavaMonitorEnter#threshold=1ms filename=locks.jfr
jcmd <pid> JFR.check verbose                       # CONFIRM the threshold took effect
jcmd <pid> JFR.dump name=locks filename=locks.jfr
```

| Event | What it answers |
|---|---|
| `jdk.JavaMonitorEnter` | **The core measurement.** Blocked-on-`synchronized` duration, monitor class, previous owner, stack trace |
| `jdk.JavaMonitorWait` | Time in `Object.wait()`. Different thing entirely — that is coordination, not contention. Topic 89 |
| `jdk.JavaMonitorInflate` | Monitor inflation events, where available on your JDK. Confirms the fast-path-to-fat transition directly |
| `jdk.ThreadPark` | `LockSupport.park` — so `ReentrantLock`, queues, `CompletableFuture`. **Not** `synchronized` |
| `jdk.ExecutionSample` | Where CPU goes. Contrast with the above: contention shows as *absence* from CPU |
| `jdk.VirtualThreadPinned` | Virtual thread pinned inside `synchronized`. Topic 101 |

The distinction between `jdk.JavaMonitorEnter` and `jdk.ThreadPark` is the fast way to
tell which locking mechanism a service is using without reading its source.

### Thread dumps as a sampling profiler for locks

You do not always have JFR. Repeated thread dumps are a poor man's lock profiler and
they work everywhere:

```bash
for i in $(seq 1 20); do
  jcmd <pid> Thread.print > dump-$i.txt
  sleep 0.5
done

# Which monitors show up most often as blocked-on?
cat dump-*.txt | grep 'waiting to lock' | sort | uniq -c | sort -rn | head

# What fraction of samples had threads blocked?
grep -l 'BLOCKED' dump-*.txt | wc -l
```

If a monitor address dominates that first list, you have found your bottleneck with no
tooling beyond `jcmd`. It is crude, it perturbs the JVM (each dump is a safepoint), and
it works on a locked-down production host at 3am when nothing else does.

### What to measure, in order

1. **Correctness.** `accepted + remaining == seeded`, exactly, every run. A fast wrong
   answer is not a result.
2. **Contention rate.** Blocked time as a fraction of available thread-time, from JFR.
3. **Throughput and p99**, end to end, from k6.
4. **Only then**, a microbenchmark, and only to compare two candidate fixes.

That ordering is not pedantry. Reversing it is how teams spend a sprint optimising a
lock that was not the bottleneck.

---

## Practice exercises

### 1 — Easy: prove the two monitors are different

Write one class with:

- a `synchronized` instance method that sleeps for 3 seconds,
- a `static synchronized` method that prints and returns immediately,
- a `main` that starts a thread running the instance method, waits 200 ms, and then
  calls the static method from `main`.

Predict, in writing, whether `main` blocks. Then run it.

Then do it again with two instance methods, and again with two static methods, and
build a 3×3 table of "does caller X block on holder Y". Include the case of two
instances of the same class.

Finally, answer: **why does `Collections.synchronizedList` return a wrapper that
documents "you must manually synchronize on the returned list when iterating"?** Connect
your answer to Trap 6.

### 2 — Medium: the audit (combines Topics 01–84)

The class below is from `orderflow`'s promotions module. It contains **eight** distinct
defects drawn from this topic and from Phases 1, 4, 5 and 8. Find them all. For each,
state the **exact symptom an on-call engineer observes**, then rewrite the class.

```java
package com.orderflow.promotions;

import org.springframework.stereotype.Service;
import java.util.*;

@Service
public class PromotionService {

    private static PromotionService instance;

    private Map<Long, Integer> discountByProduct = new HashMap<>();
    private Integer cacheVersion = 0;
    private long applicationsCount = 0;
    private boolean refreshing = false;

    public static PromotionService get() {
        if (instance == null) {
            synchronized (PromotionService.class) {
                if (instance == null) {
                    instance = new PromotionService();
                }
            }
        }
        return instance;
    }

    public synchronized void refreshFromRemote(PromotionsApiClient client) {
        refreshing = true;
        Map<Long, Integer> fresh = client.fetchAll();      // HTTP call, ~400ms
        discountByProduct = fresh;
        synchronized (cacheVersion) {
            cacheVersion = cacheVersion + 1;
        }
        refreshing = false;
    }

    public int discountFor(long productId) {
        while (refreshing) {
            Thread.onSpinWait();
        }
        Integer d = discountByProduct.get(productId);
        applicationsCount++;
        return d == null ? 0 : d;
    }

    public synchronized long applications() { return applicationsCount; }

    public static synchronized void clearAll() {
        instance.discountByProduct.clear();
    }
}
```

Hints, in no particular order:

- One defect is a lock that protects nothing, because the object being locked is
  reassigned inside the block it guards.
- One defect locks on an object whose identity changes with its value, and is shared
  JVM-wide for small values. Name the topic that explains why.
- One defect makes a 400 ms HTTP call the mutual-exclusion window for the entire
  service.
- One defect is a spin loop with no happens-before edge. Name the topic that proves it
  may never terminate.
- Two accesses to the same field use two different monitors.
- One counter loses increments under load.
- One defect can hand a caller an object whose fields are not yet initialised.
- One defect makes this class untestable and unmockable, and undoes Topic 39's entire
  argument for constructor injection.

Deliverable: the corrected class, a one-line justification per change, and a
`@GuardedBy` annotation (or comment) on every field stating which lock protects it.

### 3 — Hard: production simulation on `orderflow` under load

**Part A — the four-cell experiment.** Run the Failure drill's full matrix: {global
monitor, striped} × {1 hot SKU, 1000 SKUs}, at Topic 65's arrival rate. Record
throughput, p99, `jdk.JavaMonitorEnter` count, total blocked time, and the correctness
check for all four.

**Part B — find the knee.** For the global-monitor / 1-SKU cell, sweep the arrival rate
from 500 to 8000 requests/second in steps. Plot throughput against arrival rate. Find
the point where throughput stops rising, and the point where it starts *falling*. Name
both points and explain the mechanism at each: one is saturation of the critical
section, the other is park/unpark overhead exceeding the useful work.

**Part C — narrow the section.** Move everything except the state mutation out of the
`synchronized` block. Re-run Part B. How much did the knee move? Express the answer as
a ratio of critical-section duration, and check it against the model `max_throughput ≈
1 / critical_section_duration`. Where does your measurement disagree with that model,
and why?

**Part D — the deliberately bad version.** Add a 50 ms `Thread.sleep` inside the
`synchronized` block, simulating the payment-gateway call from Trap 1. Predict the
resulting throughput ceiling from the model above *before* running it. Then run it.
Report both numbers and explain any gap.

**Part E — measure the lock, not the system.** Build the JMH harness from the
Measurement section and run the thread sweep. Compare its picture of the same code
against k6's. Write two paragraphs on **why the two disagree** and what each one is
actually measuring. This is the most valuable part of the exercise.

**Part F — argue for the lock.** Given all your numbers, make the strongest case for
keeping the simple global `synchronized` in production. There is a real one, involving
the words "correct", "obvious", "auditable" and "not the bottleneck". Then state the
specific measurement that would flip your recommendation.

**Part G — the honest limit.** Your fixed version passed every run. Does that prove it
is correct? Write one sentence explaining why not, and name the topic that gives you a
better tool.

---

## Interview questions

### Q1 — "Is `synchronized` slow?"

**Mid-level answer:** "It has overhead, so you should minimise its use. `ReentrantLock`
or atomic variables are usually faster."

**Senior answer:** "That question doesn't have an answer as posed, because
`synchronized` has two completely different costs depending on whether anyone else
wants the lock.

Uncontended, it's a CAS on the object's mark word — a single atomic instruction each
way, single-digit nanoseconds, no kernel, no allocation. Contended, the monitor
inflates: the JVM allocates a native `ObjectMonitor` and waiting threads get parked and
unparked through the OS, which is microseconds plus a cold cache for the woken thread.
That's two orders of magnitude between the same keyword's two paths.

So the useful question is 'what is my contention rate', and it's directly measurable —
JFR's `jdk.JavaMonitorEnter` gives you blocked duration, the monitor class and the
previous owner. I'd lower the event threshold to a millisecond first, because the
defaults are too coarse for in-memory critical sections.

I'd also add two things people get wrong. First, biased locking is gone — disabled by
default in JDK 15 and removed in 18 — so most of the performance advice you'll find
online describes a JVM nobody runs. Second, `ReentrantLock` versus `synchronized` is not
a meaningful performance decision on a modern JVM; you pick `ReentrantLock` for
`tryLock` with a timeout, interruptibility, or multiple conditions. Though on JDK 21
through 23 there *is* a real performance reason, which is that a virtual thread pins its
carrier inside `synchronized` and doesn't on a `ReentrantLock` — JEP 491 in JDK 24
changed that, so it's a version-dependent answer and I'd check the JDK before advising."

**What separates them:** naming the two paths and the mechanism of each; converting the
question into a measurable one and naming the specific tool and its threshold caveat;
knowing biased locking is removed rather than repeating pre-2020 advice; and giving a
version-qualified answer on virtual threads instead of a confident wrong one.

**Follow-up:** "How would you reduce contention without changing the lock?" They want:
narrow the critical section, move I/O out of it, stripe by key, or eliminate the shared
state.

---

### Q2 — "What exactly does `synchronized` guarantee?"

**Mid-level answer:** "Only one thread can be in the block at a time."

**Senior answer:** "Two things, and the second is the one that causes bugs when people
forget it.

Mutual exclusion: at most one thread holds a given object's monitor. That's the half
everyone knows.

And a happens-before edge: an unlock of a monitor happens-before every subsequent lock
of *that same* monitor. So when I acquire a lock that you released, I'm guaranteed to
see everything you did before releasing it — not just the fields I was thinking about.
Everything.

That second half has a practical consequence people miss: if a field is guarded by a
lock, **every** access has to take that lock, including the reads. A synchronized setter
with an unsynchronised getter is broken — not because of a race on the write, but
because the reader has no edge with the writer and can see a stale value indefinitely.

And it's worth naming what `synchronized` doesn't give you: no fairness, no timeout, no
interruptibility, one wait set, and no composition. Two synchronized methods called in
sequence are not one atomic operation, which is where the check-then-act oversell bug
comes from."

**What separates them:** naming the memory-model half at all; deriving the "lock every
access including reads" rule from it; and pre-empting the composition trap.

**Follow-up:** "Give me a case where removing an *uncontended* `synchronized` block
breaks a program." The answer is the visibility half: the block was providing the edge,
not the exclusion. This is a very good question and most candidates have no answer.

---

### Q3 — "Walk me through what happens in the JVM when two threads contend for the same lock."

**Mid-level answer:** "One gets it and the other waits until it's released."

**Senior answer:** "In terms of the actual mechanism:

Thread A arrives first. It builds a lock record on its own stack, copies the object's
mark word into it — that copy is called the displaced header — and CASes a pointer to
that lock record into the object's mark word with the low bits set to stack-locked. One
atomic instruction, done.

Thread B arrives and its CAS fails, because the mark word no longer holds what B
expected. B first spins adaptively — HotSpot tunes this based on recent hold times, and
the spin uses the CPU's pause instruction, which is what `Thread.onSpinWait` exposes.
Spinning is cheaper than parking if the hold is short.

If spinning doesn't win, the object **inflates**: the JVM allocates a native
`ObjectMonitor` in its own memory, moves the displaced header into it, and repoints the
mark word at the monitor. B goes onto the monitor's contention queue and parks —
`LockSupport.park`, which is a futex on Linux and a pthread condition variable on macOS.
B is now off the CPU entirely.

When A exits, it picks a successor from the entry list and unparks it. B wakes, gets
rescheduled, and re-attempts the acquisition — and may still lose to a thread that
barged in without queueing, because HotSpot monitors are deliberately unfair. Barging
avoids a park/unpark round trip and raises throughput at the cost of fairness.

The reason that matters practically: the expensive part isn't the lock, it's the two
context switches and the cold cache for the woken thread. Which is why the fix for
contention is almost always to make the critical section shorter or narrower, not to
change the locking primitive."

**What separates them:** the displaced header, adaptive spinning before parking, the
inflation step being a real allocation, park mapping to a futex, deliberate unfairness
and why, and closing on the operational implication rather than trailing off.

**Follow-up:** "What happens to the object's identity hash code during all this?" It
lives in the same mark word. If the object was hashed before being locked, the JVM has
nowhere to put the hash on the fast path and must inflate. That is a genuinely obscure
detail and knowing it lands well.

---

### Q4 — "This method is `synchronized` and it made the whole service slow. Why?"

```java
public synchronized PaymentResult charge(Order order) {
    PaymentResult r = paymentGateway.charge(order);   // HTTP
    recordPayment(order, r);
    return r;
}
```

**Mid-level answer:** "Because it's synchronized, so requests queue up. You should
remove the synchronization or use a lock with a timeout."

**Senior answer:** "Because the monitor is held for the duration of an HTTP call, so
throughput through that section is capped at one over the gateway's latency. At 200 ms
that's about five requests per second, regardless of how many cores or threads you have.

The signature is distinctive and worth recognising: **latency climbs while CPU stays
near zero**. That combination means queueing, not computing. A thread dump would show
nearly every request thread `BLOCKED (on object monitor)` on one address, and exactly
one thread `RUNNABLE` inside a socket read while holding it. JFR's
`jdk.JavaMonitorEnter` would name the class in its `monitorClass` field.

There's a second-order failure too: under an open-model load generator the arrival rate
doesn't drop to match, so the queue grows without bound until Tomcat's thread pool is
exhausted — and then endpoints that have nothing to do with payments start returning
503. That's how a slow third party becomes a full outage.

The fix is to not hold the lock across the call: do the HTTP first, then take the lock
only for the state mutation, which is nanoseconds. If the call genuinely must be
serialised per customer, lock per customer rather than globally — with a bounded map,
or it becomes a leak. If it must be serialised globally, a `Semaphore` with a timeout is
better than a monitor, because you shed load instead of queueing forever. And
independently of all that, the HTTP client needs connect and read timeouts.

One more thing I'd flag: on JDK 21 through 23 this is worse than it looks, because a
virtual thread blocked inside `synchronized` pins its carrier and can starve the
scheduler."

**What separates them:** deriving the throughput ceiling as `1/latency` with a number;
naming the low-CPU-high-latency signature; the open-model second-order collapse; three
graded fixes with the bounded-map caveat; and the virtual-thread interaction.

**Follow-up:** "What if `recordPayment` must be atomic with the charge?" Then you have a
distributed-transaction problem, not a locking problem, and the answer is idempotency
keys plus an outbox — Topics 115 and 116. They are checking whether you can tell a
concurrency problem from a distributed-systems problem.

---

### Q5 — "Why was biased locking removed, and what replaced it?"

**Mid-level answer:** "It was an optimisation that got deprecated. I think locks are
just faster now."

**Senior answer:** "Biased locking optimised the case where one thread repeatedly locks
the same object, by writing that thread's identity into the mark word so subsequent
acquisitions skipped the CAS entirely. It was designed for a codebase full of
`Vector`, `Hashtable` and `StringBuffer` — thread-safe collections from the 1990s being
used single-threaded.

It was disabled by default in JDK 15 under JEP 374 and removed in JDK 18. Two reasons.
First, the workload it targeted had largely disappeared: modern code uses `ArrayList`
and `HashMap`, which aren't synchronized, so there was nothing to bias. Second, the cost
was enormous — revoking a bias when a second thread showed up required a safepoint
operation, and the machinery touched the interpreter, both JIT compilers and the runtime.
It was a large maintenance burden on a shrinking benefit.

Nothing replaced it. Uncontended locking is now always the stack-lock CAS, which is a
single atomic instruction and cheap enough that the extra tier wasn't worth it.

The practical consequence for me is that most `synchronized` tuning advice you find
online is describing a JVM nobody runs. If I inherit a startup script with
`-XX:BiasedLockingStartupDelay=0` in it, the container won't start on JDK 18+, and the
error is an unrecognised-option message that people waste an afternoon on."

**What separates them:** knowing what it actually did, both removal reasons (obsolete
workload *and* revocation cost), the correct JDK versions, that nothing replaced it, and
the operational consequence of an inherited flag.

**Follow-up:** "So what are the current lock states?" Neutral, stack-locked (thin), and
inflated (fat), selected by the low bits of the mark word — plus the GC-marked
encoding. And on JDK 25, compact object headers change the field widths without
changing those states.

---

## Mental model checkpoint

1. A `synchronized` block gives mutual exclusion *and* a happens-before edge. Construct
   a program that is broken by removing an **uncontended** `synchronized` block — one
   where no two threads ever actually compete for it. What does that tell you about
   what the block was really doing?

2. Inflation allocates a native `ObjectMonitor` and is not reversed immediately.
   Suppose the JVM deflated eagerly, on every unlock with no waiters. Name one workload
   that gets faster and one that gets much slower. Now argue for the policy HotSpot
   actually chose.

3. The identity hash code and the lock state share the mark word. Suppose Java had put
   the hash somewhere else from the start — say, a side table. What would that have
   cost, and what would it have bought? Would biased locking have survived?

4. Monitors are deliberately unfair: a barging thread can take the lock ahead of
   threads that have been queued longer. Make the case that this is the right default.
   Then describe the specific `orderflow` scenario in which it is the wrong one, and
   what you would use instead.

5. Trap 1 (a lock held across a network call) and Node's "don't block the event loop"
   are the same bug with different blast radii. Which one is *easier* to diagnose, and
   why? What does your answer imply about the value of the thread dump as a tool?

6. You measure a critical section at 100 nanoseconds and observe a throughput ceiling
   of about 10 million operations per second, which matches `1 / duration`. You then
   narrow the section to 50 nanoseconds and throughput rises to only 12 million rather
   than 20 million. Give three plausible explanations, and say which measurement would
   distinguish between them.

7. `synchronized` cannot leak a lock, because `javac` emits the exception-path
   `monitorexit`. `ReentrantLock` can. Yet `ReentrantLock` exists and is recommended in
   several situations. Argue that the block-structured constraint is a *feature* rather
   than a limitation, then name the case that defeats your own argument.

---

## Quick reference card

### Happens-before edges — the full list

| Edge | The rule |
|---|---|
| **Program order** | Within one thread, each action happens-before every later action in program order |
| **Monitor lock** | An unlock of a monitor happens-before every subsequent lock of **that same** monitor ← **this topic** |
| **Volatile** | A write to a volatile field happens-before every subsequent read of that same field — Topic 87 |
| **Thread start** | `Thread.start()` happens-before any action in the started thread |
| **Thread join** | Every action in a thread happens-before another thread returns from `join()` on it |
| **Thread termination** | Every action in a thread happens-before another thread detects it has terminated |
| **Interruption** | `interrupt()` happens-before the interrupted thread detects the interrupt |
| **Final fields** | Constructor end freezes final fields — Topic 88 |
| **Default values** | The default write (0/null/false) happens-before the first action in any thread |
| **Transitivity** | A hb B and B hb C implies A hb C |

Derived, and what you actually use: `ExecutorService.submit` happens-before the task
runs; task completion happens-before `Future.get()` returns; `CountDownLatch.countDown`
happens-before `await` returns; `BlockingQueue.put` happens-before the matching `take`;
`ConcurrentHashMap` write to a key happens-before a subsequent read of that key.

### The three lock states

| State | Mark word low bits | Rest of the word | Cost to acquire |
|---|---|---|---|
| Neutral (unlocked) | `01` | Identity hash, GC age | — |
| Stack-locked ("thin") | `00` | Pointer to a lock record on the owner's stack | One CAS |
| Inflated ("fat") | `10` | Pointer to a native `ObjectMonitor` | Spin, then park via futex/condvar |
| Marked for GC | `11` | Forwarding pointer | — |
| ~~Biased~~ | ~~`101`~~ | ~~Owner thread ID~~ | **Disabled JDK 15, removed JDK 18** |

### Flags

| Flag | What it does |
|---|---|
| `-XX:+PrintFlagsFinal -version` | Effective value of every flag. Run before quoting any default |
| `-XX:+UseBiasedLocking` | **Does not exist on JDK 18+.** Use it to prove removal |
| `-XX:+UseCompactObjectHeaders` | `[JAVA 25]` 64-bit headers; changes mark-word field widths, not the lock states |
| `-Xint` | Interpreter only. No lock elision, no coarsening — useful as a control |
| `-XX:-EliminateLocks` | Disable lock elision, to check whether the JIT deleted the lock you were measuring |
| `-XX:-DoEscapeAnalysis` | The bigger hammer for the same question. Topic 75 |
| `-XX:StartFlightRecording=...,jdk.JavaMonitorEnter#threshold=1ms` | Record short blocking events. **The defaults are too coarse for in-memory locks** |
| `-Djdk.tracePinnedThreads=full` | Report virtual threads pinned inside `synchronized`. Topic 101 |

### Diagnostic commands

| Command | What it answers |
|---|---|
| `javap -c -p C.class` | Find `monitorenter` / `monitorexit` and the synthetic exception handler |
| `javap -v -p C.class \| grep -B2 ACC_SYNCHRONIZED` | Which methods are synchronized without a `monitorenter` |
| `java -cp .:jol-cli.jar` + `ClassLayout.parseInstance(o)` | Read the mark word directly; watch it change under a lock |
| `jcmd <pid> Thread.print` | Full dump with `- locked` / `- waiting to lock` monitor addresses |
| `grep 'waiting to lock' dump.txt \| sort \| uniq -c \| sort -rn` | Which monitor is the bottleneck |
| `grep -B20 '<monitor-address>' dump.txt \| grep '\- locked'` | Who owns the contended monitor |
| `grep 'Locked ownable synchronizers' -A3 dump.txt` | `ReentrantLock` ownership — *not* intrinsic monitors |
| `jcmd <pid> JFR.start settings=profile jdk.JavaMonitorEnter#threshold=1ms filename=x.jfr` | Begin recording lock contention |
| `jcmd <pid> JFR.check verbose` | **Confirm your threshold override actually applied** |
| `jfr summary x.jfr \| grep -i monitor` | How many blocking events, at a glance |
| `jfr print --events jdk.JavaMonitorEnter x.jfr \| grep monitorClass \| sort \| uniq -c` | Contention by class |
| `java -jar benchmarks.jar X -t 1 -t 4 -t 16` | The thread sweep. The only honest lock benchmark |
| `java -jar benchmarks.jar X -prof perfasm` | See whether the lock survived compilation (Linux + perf) |

### Gotchas checklist

- [ ] Lock on a `private final Object`, never on `this`, a `String`, an `Integer`, or a
      `Class` you did not create.
- [ ] Every access to a guarded field takes the lock — **including reads**.
- [ ] `static synchronized` and instance `synchronized` are two different monitors.
- [ ] Never hold a monitor across I/O, a network call, or anything you did not time.
- [ ] Thread-safe operations do not compose. Match the lock to the invariant.
- [ ] Double-checked locking needs `volatile`, or use the holder idiom instead.
- [ ] `ReentrantLock.unlock()` goes in a `finally`. Always.
- [ ] Biased locking is removed. Delete it from inherited `JAVA_OPTS`.
- [ ] JFR's default monitor threshold is too coarse for in-memory locks. Lower it.
- [ ] `BLOCKED` in a dump means `synchronized`; `WAITING` means parked.
- [ ] Low CPU plus high latency means a lock, not slow code.
- [ ] Document what each lock guards — `@GuardedBy`, or a comment at minimum.

---

## When would I use this at work?

**1. Diagnosing "the service is slow" when CPU is flat.**

This is the highest-value application of the topic. A dashboard shows p99 climbing and
CPU utilisation at 15%. That pairing rules out slow computation immediately. You take a
thread dump, grep for `waiting to lock`, find one monitor address dominating, look up its
owner, and read the stack. Ten minutes from alert to named line of code. Without this
topic, the same investigation is a day of guessing and adding logging.

**2. Reviewing a PR that adds `synchronized` to fix a race.**

The author has correctly identified a race and reached for the correct tool. Your job in
review is the two questions they did not ask: *what object are you locking, and what
else is inside the block?* If they locked `this` on a public class, or if there is an
HTTP call, a database query or a large loop inside the section, you have prevented an
outage. This review takes ninety seconds and pays for itself the first time.

**3. Making a technology decision about virtual threads.**

Someone proposes flipping `spring.threads.virtual.enabled=true` on `orderflow`. The
honest answer depends on your JDK version and on where `synchronized` appears on your
request path. You can say: on JDK 21–23, any `synchronized` block spanning a blocking
call pins a carrier, and with carriers sized to the core count that is a starvation
risk — so we audit those sites first and convert them to `ReentrantLock`. On JDK 24+,
JEP 491 removed most of that pinning, so the audit is smaller but the measurement still
happens. Being able to give a version-qualified answer with a concrete audit plan, rather
than a yes or a no, is what makes the difference in a design review.

---

## Connected topics

**Prerequisites — what you are standing on:**

- **01 — The Integer cache.** Why `synchronized (someInteger)` is a real trap: values
  −128..127 are shared JVM-wide, and above that the identity changes with the value.
- **17 — Immutability and safe publication.** Immutable state needs no lock. The
  cheapest concurrency fix is to have nothing to guard.
- **52 — Hibernate locking.** The database's own concurrency control, and the atomic
  conditional UPDATE that makes the in-memory lock unnecessary.
- **66 — JVM architecture.** Shared heap versus per-thread stack — the stack is where
  the lock record lives.
- **69 — Object layout and the mark word.** You measured that word; here you use it.
- **73 — Safepoints.** Thread dumps are taken at safepoints, and biased-lock revocation
  used to require one — which was part of why it was removed.
- **75 — Escape analysis.** Lock elision and lock coarsening mean the locking you wrote
  may not be the locking that runs.
- **76 — Bytecode.** `monitorenter`, `monitorexit`, and `ACC_SYNCHRONIZED`.
- **77 — JMH.** Why the lock benchmark you were about to write is fiction.
- **78 — Profiling.** Contention shows up as *absence* from the CPU profile, which is
  exactly why wall-clock mode exists.
- **84 — Threads.** Preemption between bytecodes is the reason the critical section is
  needed at all.

**This unlocks:**

- **86 — The JMM I.** The happens-before half of `synchronized`, in full, and why
  "eventually" is not in the specification.
- **87 — The JMM II.** What the edge costs in CPU barriers, and why `volatile` is not a
  cheaper `synchronized`.
- **88 — Final fields and safe publication.** Why Trap 4's double-checked locking is
  broken, and the freeze action that fixes a whole class of publication bugs for free.
- **89 — `wait` / `notify`.** The `_WaitSet` in the `ObjectMonitor` you just learned
  about, exposed to Java.
- **92 — `ConcurrentHashMap`.** Per-bin `synchronized` — this exact mechanism, at a
  finer granularity, which is why Example 2's fix works.
- **94 — Explicit locks.** `tryLock` with a timeout, interruptibility, fairness,
  multiple conditions, and the AQS machinery underneath. Also the lock-ordering deadlock
  drill.
- **95 — CAS and atomics.** The instruction underneath the stack-lock fast path, used
  directly.
- **96 — False sharing.** Why the mark-word CAS can be limited by cache-line ownership
  rather than by the critical section.
- **98 — Bug taxonomy.** Deadlock, livelock, starvation — all of which `synchronized`
  can produce, and how to tell them apart from a dump.
- **99 — jcstress.** How to test a critical section rather than assert it is correct.
- **101 — Virtual threads.** Carrier pinning inside `synchronized`, the drill that
  demonstrates it, and the JDK-version dependency that makes this a live decision.

---

*Java baseline 21. **Biased locking was disabled by default in JDK 15 (JEP 374) and
removed in JDK 18** — treat thin-versus-fat as the current reality and biased locking as
history. `[JAVA 25]` compact object headers change the mark word's field widths without
changing the lock states; verify with `-XX:+PrintFlagsFinal -version | grep -i
CompactObjectHeaders`. Virtual-thread pinning inside `synchronized` was substantially
reduced by JEP 491 in JDK 24 — check your JDK rather than trusting any document,
including this one.*
