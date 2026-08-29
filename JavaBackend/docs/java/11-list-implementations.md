# 11 — List Implementations: ArrayList, LinkedList, ArrayDeque, and Cache Reality

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Same warehouse, two ways to store 10,000 boxes.

**Warehouse A — the numbered aisle.**
One long shelf. Bins numbered 0 to 9,999, side by side. You want bin 4,471? You walk
straight to it. You do not look at bins 0 through 4,470 on the way; you just go.

To insert a box in the middle, you must shove everything after it along by one. That
sounds terrible. It is not, and this is the surprising part: **shoving 5,000 boxes
along a shelf is one smooth continuous motion.** The forklift is built for exactly
this. It is one operation the machine is optimised to do.

**Warehouse B — the treasure hunt.**
Each box contains a note saying where the next box is. The boxes are scattered across
the whole building — one on floor 3, the next in the basement, the next by the loading
bay.

To insert in the middle, you rewrite two notes. That is genuinely two small edits.
**But first you must find the middle**, and finding it means reading 5,000 notes and
walking 5,000 times to somewhere unpredictable.

Now the fact that decides everything:

> **Walking to an unpredictable place takes about 100× longer than reading the next
> thing on the shelf in front of you.**

Big-O counts the boxes. It says both warehouses do "O(n) work" to insert in the
middle. It is right, and it is useless, because it treats "shove along a shelf" and
"walk across the building" as the same unit.

The wall clock counts the walking.

Carry this through: **contiguous and predictable beats scattered and pointer-chasing**,
almost always, at almost every size you will meet in `orderflow`.

---

## The bridge from what you know

### What transfers

JavaScript's `Array` is your `ArrayList`. Not conceptually — genuinely. V8 stores a
packed `JSArray` as a contiguous backing store and grows it geometrically, exactly as
`ArrayList` does. `push` is amortised O(1) in both. `shift`/`unshift` are O(n) in both
(V8 has some fast paths, but the shape is the same).

```ts
const lines: OrderLine[] = [];
lines.push(line);          // amortised O(1), may reallocate
lines[4471];               // O(1)
lines.splice(5000, 0, x);  // O(n) — memmove
lines.shift();             // O(n) — everything moves down
```

```java
List<OrderLine> lines = new ArrayList<>();
lines.add(line);           // amortised O(1), may reallocate
lines.get(4471);           // O(1)
lines.add(5000, x);        // O(n) — System.arraycopy
lines.remove(0);           // O(n) — everything moves down
```

**Verdict: HONEST ANALOGUE for `ArrayList`.** Your instincts about JS arrays transfer
almost perfectly.

### What has no counterpart

> **NO TYPESCRIPT ANALOGUE for `LinkedList` and `ArrayDeque`.**
>
> JavaScript has exactly one built-in sequence type. There is no linked list, no
> deque, no `Stack`, no `PriorityQueue` in the standard library. Any queue you have
> written in Node was an array with `push`/`shift`, or a library. So you have no habit
> to transfer here and no intuition to correct — but also no bad habit. Many Java
> developers reach for `LinkedList` reflexively because a data-structures course told
> them "O(1) insertion". You do not have that reflex. Do not acquire it.

One more thing you have never had to think about: **V8's garbage collector and object
layout are opaque.** You cannot ask how many bytes a JS object occupies or where it
sits in memory. In Java you can measure both, and this topic is where that starts to
matter.

### Mapping table

| You know | Java | Verdict |
|---|---|---|
| `Array` (packed) | `ArrayList` | **HONEST ANALOGUE** |
| `Array` (holey / dictionary mode) | — | Java arrays are never sparse |
| `push` / `pop` | `add` / `remove(size-1)` | **HONEST ANALOGUE** |
| `shift` / `unshift` | `ArrayDeque.pollFirst` / `addFirst` | **PARTIAL** — Java has a proper O(1) type for it |
| — | `LinkedList` | **NO ANALOGUE** |
| — | `ArrayDeque`, `PriorityQueue` | **NO ANALOGUE** |
| `new Array(n)` presizing | `new ArrayList<>(n)` | **HONEST ANALOGUE** — same reason, same benefit |
| `arr.slice(a,b)` (copy) | `list.subList(a,b)` (**view**) | **PARTIAL — and this one bites**; see Trap 5 |
| Opaque memory layout | JOL, heap dumps, `-Xlog:gc` | **NO ANALOGUE** — you gain measurement and the duty to use it |

---

## What is this?

Three implementations of `List`/`Deque` that you will actually meet, and one legacy
pair you must recognise in old code.

### `ArrayList` — an array plus a size

```java
// conceptually, inside ArrayList:
transient Object[] elementData;   // the backing array — capacity
private   int      size;          // how many slots are actually used
```

- `size` is what you have. `elementData.length` is what you have room for. They are
  different numbers, and `size()` reports the first.
- Created empty, the backing array is a shared zero-length constant. The first `add`
  allocates the default capacity of **10**. So `new ArrayList<>()` allocates almost
  nothing until you use it.
- When full, it grows by **1.5×**: the JDK computes roughly
  `newCapacity = oldCapacity + (oldCapacity >> 1)`, allocates a new array, and copies.
  Amortised over many adds, `add` is O(1).
- `get(i)` is one array index. That is it.
- `add(i, x)` and `remove(i)` call `System.arraycopy` to shift the tail. That is an
  **intrinsic** — the JIT compiles it to a hardware-optimised block move, not a Java
  loop.
- `new ArrayList<>(expectedSize)` skips all the growth reallocation. Do it whenever
  you know the size.

### `LinkedList` — a doubly-linked chain

```java
// conceptually, inside LinkedList:
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
transient Node<E> first, last;
transient int size;
```

- Every element costs a separate heap object holding three references plus the element
  reference. On a 64-bit JVM with compressed references that is a 12-byte header plus
  3 × 4 bytes = **roughly 24 bytes per element of pure overhead**, before the element
  itself. Verify this with JOL rather than trusting the arithmetic — it is Proof 3.
- Those node objects are allocated at different times and may end up anywhere in the
  heap. Following `next` is a **dependent load**: the CPU cannot start fetching the
  next node until the current one has arrived, so it cannot overlap the latency.
- `get(i)` is O(n). The implementation is slightly clever — it walks from whichever
  end is nearer (`if (index < (size >> 1))` forward, else backward) — so it is O(n/2)
  at worst, which is still O(n).
- It implements **both** `List` and `Deque`. As a deque it is respectable; as a `List`
  it is almost never the right choice.
- Where it genuinely wins: you already hold a `ListIterator` positioned where you want
  to work, and you do many `add`/`remove` there. `iterator.add(x)` really is O(1) with
  no shifting. That is a narrow window.

### `ArrayDeque` — a circular buffer

```java
// conceptually:
transient Object[] elements;
transient int head;    // index of the first element
transient int tail;    // index one past the last
```

- A resizable array used as a ring. `addFirst` decrements `head` (wrapping);
  `addLast` writes at `tail` and increments (wrapping). Both are O(1) with **no
  shifting at all**.
- Contiguous memory, like `ArrayList`. One object for the whole structure, not one per
  element.
- **Rejects `null`.** It must: `null` is the sentinel that marks an empty slot in the
  ring. `addLast(null)` throws `NullPointerException`.
- It is the correct choice for both a **queue** and a **stack**. The Javadoc says so
  outright: faster than `Stack` when used as a stack, faster than `LinkedList` when
  used as a queue.
- It is a `Deque`, not a `List` — no `get(i)`. If you need indexed access, you wanted
  a `List`.

> **Version note, and I am flagging genuine uncertainty:** `ArrayDeque`'s internal
> capacity policy changed in the Java 9 era. The older implementation required a
> power-of-two capacity so that wrap-around could be a bitmask. The rewritten version
> grows by roughly `oldCapacity + (oldCapacity < 64 ? oldCapacity + 2 : oldCapacity >> 1)`
> and no longer maintains a power of two. The behaviour you rely on (O(1) ends,
> contiguous storage, no nulls) is unchanged either way. Settle what *your* JDK does
> with the source-reading command in Proof 5 rather than trusting any blog post,
> including this one.

### `[LEGACY — still asked]` `Vector` and `Stack`

```java
Vector<Order> orders = new Vector<>();   // do not write this
Stack<Order>  stack  = new Stack<>();    // nor this
```

- `Vector` is `ArrayList` with every method `synchronized`. That gives you method-level
  atomicity, which is almost never the atomicity you need — `if (!v.contains(x))
  v.add(x)` is still a race (Topic 92). You pay for a lock and get no useful guarantee.
- `Stack extends Vector`, which means a stack that supports `get(i)` and `add(i, x)` —
  a broken abstraction. It also iterates **bottom-to-top**, which is the opposite of
  what almost everyone expects from a stack.
- Both date to Java 1.0 and predate the collections framework. They are retained for
  compatibility.
- **Use `ArrayDeque` for stacks and queues, `ArrayList` for lists, and
  `Collections.synchronizedList` or a concurrent collection when you genuinely need
  thread safety.** Interviewers still ask about `Vector` vs `ArrayList`, so know the
  answer; never write it.

### Cache reality — the part Big-O leaves out

Memory is not flat. A rough, order-of-magnitude picture on a modern server CPU:

| Where the data is | Roughly how long to get it |
|---|---|
| a register | 0 cycles |
| L1 cache | ~4 cycles |
| L2 cache | ~12 cycles |
| L3 cache | ~40 cycles |
| main memory (DRAM) | ~200+ cycles |

Two hardware behaviours follow, and they are what decide the wall clock:

**1. Cache lines.** Memory moves between DRAM and cache in fixed-size blocks — 64
bytes on x86-64, 128 bytes on Apple Silicon. Reading one 4-byte reference pulls in its
whole line. So a linear scan of an array gets 16 references (on a 64-byte line) for
the price of one memory fetch.

**2. The prefetcher.** The CPU watches your access pattern. If you walk forward
through memory at a constant stride, it fetches the *next* lines before you ask for
them. A sequential array scan therefore runs at close to memory bandwidth. A pointer
chase defeats the prefetcher completely — the address of the next node is only known
after the current node has arrived.

Now apply it:

| Operation | `ArrayList` | `LinkedList` |
|---|---|---|
| `get(n/2)` | one indexed load | n/2 dependent loads, most likely cache misses |
| insert at `n/2` | one `arraycopy` of n/2 references — streaming, prefetched, intrinsic | n/2 dependent loads to *find* the position, then 2 pointer writes |
| iterate all | sequential scan, prefetcher fully engaged | n dependent loads |
| memory for n elements | one array, ~4 bytes/element plus up to 50% slack | n node objects, ~24 bytes/element overhead |

**This is why `LinkedList` loses even at its supposed best case.** Both are O(n) for a
middle insertion. But `ArrayList` pays that O(n) in the one operation the hardware is
best at, and `LinkedList` pays it in the one operation the hardware is worst at.

Big-O ranks algorithms. The memory hierarchy decides the wall clock. You need both,
and when they disagree at realistic sizes, measure.

---

## Why does it matter?

**1. The classic advice is wrong, and it is taught everywhere.** "LinkedList is O(1)
for insertion" is the most durable piece of misinformation in Java. Acting on it makes
code slower, allocates a node per element, and is defended with a complexity argument
that sounds authoritative. Being able to explain *why* it is wrong — in terms of
dependent loads and cache lines, not vibes — is a genuine mid-to-senior marker.

**2. Quadratic behaviour hides until the data grows.** `remove(0)` in a loop and
`List.contains` in a loop are both invisible in a 500-row test fixture and both turn a
four-second job into a seven-minute one at 200,000 rows. The signature is 10× the
input, 100× the time, and recognising that shape saves the first hour of every
investigation.

**3. Memory is a capacity question with a real answer.** "Will a million in-flight
tasks fit in a 2 GB container?" differs by a factor of several between `ArrayList` and
`LinkedList`. That is the difference between "yes" and "we need Redis" — an
architecture decision that you can settle with a measurement instead of an argument.

**4. It is where you learn that measurement has rules.** Every performance claim in
this doc is falsifiable, and a naive timing loop will falsify it *wrongly*. Learning
here that a wall-clock loop lies — and what a real harness does about it — is what
makes every performance conversation in Phases 8 and 9 productive rather than
anecdotal.

---

## Syntax breakdown

New constructs only. Topics 05–07 and 10 covered generics, wildcards and the
collection interfaces.

### Pre-sizing

```java
List<OrderLine> lines = new ArrayList<>(expectedLineCount);
```

| Bit | What it means |
|---|---|
| The `int` argument | **Initial capacity**, not size. `lines.size()` is still 0. |
| What it saves | Every growth step allocates a new array and copies. Going from 10 to 100,000 elements costs about 20 allocate-and-copy rounds; pre-sizing costs zero. |
| Getting it wrong high | Wasted memory, no correctness issue. |
| `new ArrayList<>(someCollection)` | A different constructor — copies the contents, sized exactly. |

### `ArrayDeque` as a queue and as a stack

```java
Deque<Long> retryQueue = new ArrayDeque<>();

// FIFO — a queue
retryQueue.addLast(orderId);
Long next = retryQueue.pollFirst();      // null if empty

// LIFO — a stack
retryQueue.push(orderId);                // == addFirst
Long top = retryQueue.pop();             // == removeFirst, throws if empty
```

| Method pair | Behaviour on empty |
|---|---|
| `pollFirst` / `pollLast` | returns `null` |
| `removeFirst` / `removeLast` | throws `NoSuchElementException` |
| `peekFirst` / `peekLast` | returns `null`, does not remove |
| `getFirst` / `getLast` | throws, does not remove |

Pick `poll`/`peek` when empty is expected, `remove`/`get` when empty is a bug. That
choice is Topic 09's decision test applied to a data structure.

### Sequenced collections `[JAVA 21]`

Java 21 added `SequencedCollection`, which finally gives `List` and `Deque` a uniform
first/last API:

```java
List<OrderLine> lines = new ArrayList<>(...);
OrderLine first = lines.getFirst();       // Java 21+
OrderLine last  = lines.getLast();        // Java 21+
List<OrderLine> reversed = lines.reversed();   // a VIEW, not a copy
```

The pre-21 equivalents, which you will still see everywhere:
```java
OrderLine first = lines.get(0);
OrderLine last  = lines.get(lines.size() - 1);
List<OrderLine> reversed = new ArrayList<>(lines);
Collections.reverse(reversed);
```

Note `reversed()` returns a **view**. Mutating it mutates the original.

### `subList` — a view, not a slice

```java
List<OrderLine> page = allLines.subList(0, 50);
```

| Bit | What it means |
|---|---|
| Returns | A **window onto the same backing array**, not a copy. `page.set(0, x)` changes `allLines`. |
| If `allLines` is structurally modified | `page` throws `ConcurrentModificationException` on next use. |
| To get a copy | `List.copyOf(allLines.subList(0, 50))` or `new ArrayList<>(...)`. |
| Retention hazard | The view holds a reference to the whole parent list. Returning a 50-element `subList` of a 1,000,000-element list keeps all million alive. See Trap 5. |

### `System.arraycopy` — what is actually happening under `add(i, x)`

You will not usually call it directly, but you should recognise it:

```java
System.arraycopy(src, srcPos, dest, destPos, length);
```

It is a JVM **intrinsic**: the JIT replaces the call with an optimised block move
(vectorised where possible), not a Java loop. That is why "shifting 500,000 elements"
is far cheaper than the element count suggests, and it is the mechanical reason
`ArrayList` middle insertion beats `LinkedList` traversal.

---

## Example 1 — minimal

The point is to see the *shape* of the difference, not to get a number.

```java
import java.util.*;

public class ShapeOfIt {

    static long middleInsert(List<Integer> list, int n) {
        for (int i = 0; i < n; i++) list.add(i);
        long t = System.nanoTime();
        for (int i = 0; i < 10_000; i++) list.add(list.size() / 2, i);
        return System.nanoTime() - t;
    }

    static long middleRead(List<Integer> list, int n) {
        for (int i = 0; i < n; i++) list.add(i);
        long t = System.nanoTime();
        long sink = 0;
        for (int i = 0; i < 10_000; i++) sink += list.get(list.size() / 2);
        long elapsed = System.nanoTime() - t;
        if (sink == Long.MIN_VALUE) System.out.print("");   // keep the loop alive
        return elapsed;
    }

    public static void main(String[] args) {
        for (int round = 0; round < 3; round++) {
            System.out.printf("round %d%n", round);
            System.out.printf("  insert  ArrayList  %,12d ns%n", middleInsert(new ArrayList<>(), 100_000));
            System.out.printf("  insert  LinkedList %,12d ns%n", middleInsert(new LinkedList<>(), 100_000));
            System.out.printf("  read    ArrayList  %,12d ns%n", middleRead(new ArrayList<>(), 100_000));
            System.out.printf("  read    LinkedList %,12d ns%n", middleRead(new LinkedList<>(), 100_000));
        }
    }
}
```

> **Read this before you read the numbers.** This program is **not a benchmark** and
> its numbers are not trustworthy. There is no warm-up discipline, the JIT may
> eliminate work it can prove unobservable, garbage collection can land anywhere, and
> the two lists are not measured in the same JIT state. It is good for exactly one
> thing: seeing whether a difference is *orders of magnitude* or *within noise*. The
> read case should be dramatic enough to survive all that. The insert case may not be,
> and if it is close, **you have learned nothing and must not conclude anything**.
> Topic 77 (JMH) is where you learn to do this properly, and the Hands-on Proof section
> below gives you the real harness.

---

## Example 2 — production scenario

`orderflow` has a payment-retry worker. Failed payments go into a queue; a worker
drains it, retries each one, and puts still-failing ones back with a longer backoff.
Meanwhile an admin endpoint pages through the queue's contents.

Three access patterns in one component, and the wrong collection for any of them shows
up as latency.

```java
package com.orderflow.payments;

import java.util.*;
import java.util.concurrent.*;

public final class PaymentRetryWorker {

    public record RetryTask(long paymentId, int attempt, long notBeforeEpochMs) { }

    /**
     * FIFO of tasks ready to attempt.
     *
     * ArrayDeque, not LinkedList:
     *   - O(1) at both ends with no per-element node allocation
     *   - one contiguous array, so draining it is a prefetched sequential scan
     *   - rejects null, which is a free correctness check here
     * ArrayDeque, not ArrayList:
     *   - removing from the front of an ArrayList is O(n); this queue drains
     *     front-first thousands of times per minute
     */
    private final Deque<RetryTask> ready = new ArrayDeque<>(1024);

    /**
     * Tasks whose backoff has not elapsed, ordered by when they become due.
     * PriorityQueue because the question is always "what is due next?",
     * never "what is at position 7?".
     */
    private final PriorityQueue<RetryTask> waiting =
            new PriorityQueue<>(Comparator.comparingLong(RetryTask::notBeforeEpochMs));

    /**
     * Payment IDs currently anywhere in this worker, for idempotency.
     * A Set because the question is membership (Topic 10).
     */
    private final Set<Long> inFlight = new HashSet<>();

    private final PaymentGateway gateway;
    private final int maxAttempts;

    public PaymentRetryWorker(PaymentGateway gateway, int maxAttempts) {
        this.gateway = gateway;
        this.maxAttempts = maxAttempts;
    }

    public boolean submit(long paymentId) {
        if (!inFlight.add(paymentId)) return false;      // already queued
        ready.addLast(new RetryTask(paymentId, 0, 0L));
        return true;
    }

    /** Move anything now due from waiting -> ready. O(k log n) for k promoted. */
    private void promoteDue(long nowMs) {
        while (!waiting.isEmpty() && waiting.peek().notBeforeEpochMs() <= nowMs) {
            ready.addLast(waiting.poll());
        }
    }

    /** Drains everything currently ready. Returns how many were finally given up on. */
    public int drain(long nowMs) {
        promoteDue(nowMs);

        int abandoned = 0;
        int batch = ready.size();                 // snapshot: do not re-drain requeues

        for (int i = 0; i < batch; i++) {
            RetryTask task = ready.pollFirst();
            if (task == null) break;

            if (gateway.retry(task.paymentId())) {
                inFlight.remove(task.paymentId());
                continue;
            }
            int nextAttempt = task.attempt() + 1;
            if (nextAttempt >= maxAttempts) {
                inFlight.remove(task.paymentId());
                abandoned++;
                continue;
            }
            long backoffMs = 1_000L << Math.min(nextAttempt, 10);   // capped exponential
            waiting.add(new RetryTask(task.paymentId(), nextAttempt, nowMs + backoffMs));
        }
        return abandoned;
    }

    /**
     * Admin endpoint: page through what is waiting.
     *
     * NOTE: PriorityQueue iteration is NOT in priority order (Topic 10). We must
     * copy and sort. We return an immutable copy, never a subList view, because
     * a subList would retain the whole backing array and would throw CME the
     * moment the worker thread modified the queue.
     */
    public List<RetryTask> waitingPage(int offset, int limit) {
        List<RetryTask> snapshot = new ArrayList<>(waiting);          // pre-sized by the copy ctor
        snapshot.sort(Comparator.comparingLong(RetryTask::notBeforeEpochMs));
        int from = Math.min(offset, snapshot.size());
        int to   = Math.min(offset + limit, snapshot.size());
        return List.copyOf(snapshot.subList(from, to));               // copy, not view
    }

    public interface PaymentGateway { boolean retry(long paymentId); }
}
```

What each choice bought, and what it would cost to get it wrong:

| Choice | Alternative | Cost of the alternative |
|---|---|---|
| `ArrayDeque` for `ready` | `LinkedList` | one node object per task; at 50k tasks/minute that is 50k short-lived 24-byte objects per minute, plus a pointer chase on every drain |
| `ArrayDeque` for `ready` | `ArrayList` + `remove(0)` | every poll shifts the entire remaining array: draining n tasks becomes O(n²). At n=10,000 that is 50 million element moves per drain cycle |
| `new ArrayDeque<>(1024)` | default | ~7 grow-and-copy rounds on the first busy minute, then stable. Small, free to avoid |
| `PriorityQueue` for `waiting` | sorted `ArrayList` with `add` + `sort` | O(n log n) per insertion instead of O(log n) |
| `HashSet` for `inFlight` | `List.contains` | O(n) per submit; at 10,000 in flight, every submit scans 10,000 entries |
| snapshot `batch = ready.size()` | `while (!ready.isEmpty())` | requeued tasks get retried again in the same cycle, so a permanently failing payment spins the loop forever and the backoff never applies |
| `List.copyOf(...subList(...))` | returning the `subList` | a `ConcurrentModificationException` in the admin endpoint the moment the worker touches the queue, plus retention of the whole snapshot |

The `batch = ready.size()` line is the one that causes a real incident, and it is not a
data-structure choice at all — it is a consequence of choosing a structure you can
requeue into. Worth noticing that the collection decision changed the control flow.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `LinkedList` chosen from Big-O

**Wrong:**
```java
// "We insert in the middle a lot, so LinkedList is O(1)."
List<OrderLine> lines = new LinkedList<>();
lines.add(lines.size() / 2, newLine);
```

**Exact symptom:** p99 latency on the order-edit endpoint rises with basket size in a
way that p50 does not. A CPU profiler (Topic 78) shows the hot frame is
`java.util.LinkedList.node(int)` — a method you never call. On Linux,
`perf stat -e cache-misses,cycles` on the process shows a cache-miss rate several times
higher than the `ArrayList` version at the same throughput. Allocation profiling shows
one `LinkedList$Node` per element.

**Root cause:** `add(index, element)` on a `LinkedList` is O(1) **only after** you have
walked to `index`, which is O(n) — and every hop of that walk is a dependent load to
an unpredictable address. `ArrayList` also pays O(n), but it pays it inside
`System.arraycopy`, a JIT intrinsic that moves a contiguous block at close to memory
bandwidth. Same complexity class, wildly different constant.

**Fix:** `ArrayList`, pre-sized. If the profile still shows insertion cost dominating,
the question is whether you need a list at all — a `TreeMap` keyed by position, or an
append-only log with a separate index, may be the real answer.

**How to be sure rather than to believe me:** the JMH harness in Proof 2. Do not accept
this trap on authority — including mine.

---

### Trap 2 — `remove(0)` in a loop

**Wrong:**
```java
List<RetryTask> queue = new ArrayList<>(pendingTasks);
while (!queue.isEmpty()) {
    RetryTask task = queue.remove(0);      // <-- the defect
    process(task);
}
```

**Exact symptom:** the job's runtime is **superlinear** in the input size, and this is
the observable signature. Concretely: 20,000 tasks in about 4 seconds, 200,000 tasks in
about 400 seconds — 10× the input, 100× the time. A profiler shows time in
`System.arraycopy` called from `ArrayList.remove`. There is no exception and no error
metric; the only signal is that the nightly job started missing its window.

**Root cause:** `remove(0)` shifts every remaining element down one slot. Doing that n
times moves n²/2 elements in total. `arraycopy` is fast per element, which is exactly
why this hides at small n and appears only when the input grows.

**Fix:**
```java
Deque<RetryTask> queue = new ArrayDeque<>(pendingTasks);
RetryTask task;
while ((task = queue.pollFirst()) != null) {
    process(task);
}
```
`pollFirst` moves a `head` index. Nothing shifts. The job becomes linear.

**Two cheaper fixes when you cannot change the type:** iterate forward with an index
and never remove, or iterate the list in reverse and `remove(size-1)`, which shifts
nothing.

---

### Trap 3 — `contains` on a `List` inside a loop

**Wrong:**
```java
List<String> processedSkus = new ArrayList<>();
for (OrderLine line : incoming) {              // 50,000 lines
    if (!processedSkus.contains(line.sku())) { // O(n) scan, every time
        processedSkus.add(line.sku());
        priceService.fetch(line.sku());
    }
}
```

**Exact symptom:** the import is CPU-bound with almost no I/O wait and no allocation
pressure. A flame graph is a single wide tower ending in `ArrayList.indexOf` and
`String.equals`. Runtime scales as the square of the batch size: fine in a 500-line
test fixture, 20 minutes on a 50,000-line supplier feed.

**Root cause:** `List.contains` is a linear scan calling `equals` on every element. In
a loop over n items with up to n distinct SKUs, that is n²/2 string comparisons — about
1.25 billion at n = 50,000.

**Fix:**
```java
Set<String> processedSkus = new HashSet<>();
for (OrderLine line : incoming) {
    if (processedSkus.add(line.sku())) {   // add() returns false if already present
        priceService.fetch(line.sku());
    }
}
```
Two changes: `Set` for O(1) membership (Topic 10), and `add`'s return value instead of
a separate `contains` (one hash lookup instead of two).

---

### Trap 4 — no pre-sizing on a known-size list

**Wrong:**
```java
List<OrderLine> lines = new ArrayList<>();     // capacity 10 on first add
for (ResultSet row : rows) {                   // 500,000 rows
    lines.add(map(row));
}
```

**Exact symptom:** modest but real. `-Xlog:gc` shows more young collections than the
equivalent pre-sized run, and an allocation profile (Topic 78) attributes bytes to
`java.util.Arrays.copyOf` from `ArrayList.grow`. On a 500,000-element list you pay
about 25 grow rounds, and the largest few copies dominate — the final growth alone
copies ~330,000 references and allocates a ~2 MB array that is immediately garbage.
None of this is catastrophic. It shows up as a modestly higher allocation rate, which
matters when you are already close to a GC budget.

**Root cause:** geometric growth is efficient *asymptotically* but every step allocates
a new array and abandons the old one.

**Fix:**
```java
List<OrderLine> lines = new ArrayList<>(expectedRowCount);
```
If the size is unknown but bounded, pass the bound. If truly unknown, leave it alone —
this is not worth guessing about.

**Honest caveat:** this is the smallest trap in this doc, and it is the one people
over-apply. Do not go through a codebase adding capacity hints. Do it where the size
is known and the list is large, and let a profiler tell you about the rest.

---

### Trap 5 — `subList` retains the whole parent

**Wrong:**
```java
public List<OrderLine> firstPage(long orderId) {
    List<OrderLine> all = repository.loadAllLines(orderId);   // 200,000 lines
    return all.subList(0, 50);                                // looks like a small result
}
```

**Exact symptom:** two of them, and they appear at different times.

*Immediately:* if anything structurally modifies `all` (or `all` is reused and
modified), any use of the returned list throws
`java.util.ConcurrentModificationException` from `ArrayList$SubList.checkForComodification`.

*Later, and worse:* if the 50-element result is cached — put in a `Map`, held by a
session, stored in a field — it keeps the entire 200,000-element backing array alive.
A heap dump (Topic 79) shows the dominator tree rooted at your cache holding megabytes
per entry, with a retained size wildly out of proportion to the 50 objects you can
see. The service OOMs after several hours, and the cache looks innocent because it has
few entries.

**Root cause:** `subList` returns a **view** holding a reference to the parent list. It
is a window, not a slice. (Historical note for context: `String.substring` had exactly
this shape before Java 7 and was changed for exactly this reason.)

**Fix:**
```java
return List.copyOf(all.subList(0, 50));
```
Or, better, do not load 200,000 rows to return 50 — push the paging into the query.
That is Topic 47's territory and is the real fix; the copy is the safety net.

---

### Trap 6 — `null` in an `ArrayDeque`

**Wrong:** migrating a queue from `LinkedList` to `ArrayDeque` where `null` was used
as a sentinel.
```java
Deque<RetryTask> queue = new ArrayDeque<>();
queue.addLast(null);        // used to mean "end of batch"
```

**Exact symptom:**
```
java.lang.NullPointerException
	at java.base/java.util.ArrayDeque.addLast(ArrayDeque.java:308)
```
appearing immediately after a refactor that changed only the collection type, on a
line that worked for years.

**Root cause:** `ArrayDeque` uses `null` internally to mark empty slots in its ring
buffer, so a stored `null` would be indistinguishable from an empty slot. It rejects
nulls by contract. `LinkedList` allows them.

**Fix:** stop using `null` as a sentinel — that was always the real bug. Use a marker
record, or restructure so the batch boundary is explicit:
```java
sealed interface QueueItem permits Task, BatchEnd { }
```
This is Topic 28 again: a sentinel value is a discriminated union in disguise, and Java
21 lets you say so.

---

## Hands-on proof

Commands **you** run. I have no JVM and will not print numbers as if they were real.

### The warning that comes first

**A naive timing loop lies.** Every one of these will corrupt your measurement:

- **No warm-up.** Java starts interpreted. The JIT compiles a method only after it has
  run enough times, and may recompile it several times. Your first thousand
  iterations measure the interpreter.
- **Dead-code elimination.** If the JIT can prove a result is never used, it deletes
  the computation. You then time an empty loop and conclude your code is infinitely
  fast.
- **On-stack replacement.** A long-running loop gets compiled *while it is running*, so
  early and late iterations execute different machine code.
- **Garbage collection.** A young collection landing inside your timed region adds
  milliseconds with no relationship to what you are measuring.
- **Setup inside the timed region.** Building the 100,000-element list inside the
  measurement times allocation, not the operation.
- **Measurement order.** Whichever you run first warms shared code paths for whichever
  runs second.

Example 1 in this doc is deliberately naive so you can see the shape of a large
difference. **Do not quote its numbers anywhere.** Topic 77 covers JMH properly. What
follows is enough JMH to make *this* claim honestly.

### Setup

```bash
mkdir -p ~/java-lab/11 && cd ~/java-lab/11
java --version
mvn --version      # you need Maven for the JMH harness
```

### Proof 1 — generate a real JMH project

```bash
mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jmh \
  -DarchetypeArtifactId=jmh-java-benchmark-archetype \
  -DgroupId=com.orderflow -DartifactId=list-bench -Dversion=1.0

cd list-bench
```

**What to look for:** a `pom.xml` with a `jmh-core` and `jmh-generator-annprocess`
dependency, and `src/main/java/com/orderflow/MyBenchmark.java`.

| What you see | What it means |
|---|---|
| The project generates | Good. The archetype pins a JMH version; check it in `pom.xml` and bump `<jmh.version>` if it is old. |
| `archetype not found` | Your Maven cannot reach Maven Central, or the archetype coordinates changed. Add `-DarchetypeVersion=1.37` (or the current version from `search.maven.org`) and retry. |
| It builds but `benchmarks.jar` is missing after `verify` | The shade plugin did not run. Confirm `mvn clean verify` and not `mvn package`. |

### Proof 2 — the ArrayList-vs-LinkedList measurement, done honestly

Replace `src/main/java/com/orderflow/MyBenchmark.java` with:

```java
package com.orderflow;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.util.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Benchmark)
@Fork(value = 3)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
public class ListBench {

    @Param({"1000", "10000", "100000"})
    public int size;

    private List<Integer> arrayList;
    private List<Integer> linkedList;

    @Setup(Level.Invocation)     // rebuild per invocation: mutation changes the state
    public void setup() {
        arrayList  = new ArrayList<>(size);
        linkedList = new LinkedList<>();
        for (int i = 0; i < size; i++) { arrayList.add(i); linkedList.add(i); }
    }

    @Benchmark public void middleInsertArrayList(Blackhole bh) {
        for (int i = 0; i < 100; i++) arrayList.add(arrayList.size() / 2, i);
        bh.consume(arrayList);
    }

    @Benchmark public void middleInsertLinkedList(Blackhole bh) {
        for (int i = 0; i < 100; i++) linkedList.add(linkedList.size() / 2, i);
        bh.consume(linkedList);
    }

    @Benchmark public void iterateArrayList(Blackhole bh) {
        long sum = 0;
        for (int v : arrayList) sum += v;
        bh.consume(sum);
    }

    @Benchmark public void iterateLinkedList(Blackhole bh) {
        long sum = 0;
        for (int v : linkedList) sum += v;
        bh.consume(sum);
    }

    @Benchmark public void queueDrainArrayDeque(Blackhole bh) {
        Deque<Integer> d = new ArrayDeque<>(arrayList);
        Integer x; long sum = 0;
        while ((x = d.pollFirst()) != null) sum += x;
        bh.consume(sum);
    }

    @Benchmark public void queueDrainArrayListRemoveZero(Blackhole bh) {
        List<Integer> l = new ArrayList<>(arrayList);
        long sum = 0;
        while (!l.isEmpty()) sum += l.remove(0);
        bh.consume(sum);
    }
}
```

```bash
mvn clean verify
java -jar target/benchmarks.jar ListBench -prof gc
```

**What each annotation is doing — this is why the number is trustworthy:**

| Annotation | What it prevents |
|---|---|
| `@Fork(3)` | Runs three separate JVMs. Protects against one unlucky JIT profile. |
| `@Warmup(5)` | Discards the interpreted and partly-compiled iterations. |
| `@Measurement(10)` | Enough samples for a meaningful error bar. |
| `Blackhole.consume` | Stops dead-code elimination. Without it the JIT may delete the loop. |
| `@Setup(Level.Invocation)` | Rebuilds the mutated lists so each invocation starts from the same state. It has overhead — accept it here, because a shared mutated list would be worse. |
| `@Param` | Runs each benchmark at three sizes, so you see the *shape*, not one point. |
| `-prof gc` | Adds allocation-per-operation, which is where `LinkedList`'s node cost appears. |

**What to look for and how to read it:**

| Column / observation | How to read it |
|---|---|
| `Score` and `Error` (the `±` value) | If two scores differ by less than the sum of their error bars, **you have no result**. Say "no measurable difference", not "about the same". |
| `Units` | `us/op` here. Check it — misreading ns for us is the classic 1000× error. |
| `iterate*` at every size | Expect `ArrayList` to win clearly. This is the pure cache-locality case with no algorithmic difference at all: both iterate n elements. Any gap is memory hierarchy, nothing else. |
| `middleInsert*` across the three sizes | Watch how the *ratio* changes with `size`. If `LinkedList` closes the gap or wins at 1,000 and loses badly at 100,000, that is traversal cost growing. If it never wins, say so plainly. |
| `queueDrainArrayListRemoveZero` vs `queueDrainArrayDeque` | Expect a very large, size-dependent gap: this is Trap 2's O(n²) against O(n). If this one is *not* dramatic, something is wrong with your harness — check `Blackhole` usage. |
| `gc.alloc.rate.norm` (bytes/op) | `LinkedList` should allocate substantially more per operation. This is the ~24 bytes per node, measured rather than asserted. |
| Results that contradict this document | **Trust your measurement.** Post the full output. Different CPU, different heap size, different JDK — all legitimately change the answer, and being able to say "on my hardware, at this size, the answer is X" is the actual skill. |

### Proof 3 — measure the memory, do not estimate it

```bash
curl -o jol-cli.jar \
  https://repo1.maven.org/maven2/org/openjdk/jol/jol-cli/0.17/jol-cli-0.17-full.jar
```

`Footprint.java`:
```java
import org.openjdk.jol.info.GraphLayout;
import java.util.*;

public class Footprint {
    public static void main(String[] args) {
        int n = 1_000_000;
        List<Integer> al = new ArrayList<>(n);
        List<Integer> ll = new LinkedList<>();
        Deque<Integer> ad = new ArrayDeque<>(n);
        for (int i = 0; i < n; i++) { al.add(i); ll.add(i); ad.addLast(i); }

        System.out.println("ArrayList   " + GraphLayout.parseInstance(al).totalSize());
        System.out.println("LinkedList  " + GraphLayout.parseInstance(ll).totalSize());
        System.out.println("ArrayDeque  " + GraphLayout.parseInstance(ad).totalSize());
        System.out.println();
        System.out.println(GraphLayout.parseInstance(ll).toFootprint());
    }
}
```

```bash
javac -cp jol-cli.jar Footprint.java
java -cp .:jol-cli.jar -Xmx4g Footprint
```

**What to look for:** the three totals, and the per-class breakdown in the footprint
table.

| What you see | What it means |
|---|---|
| `LinkedList` total is substantially larger than `ArrayList` | Expected. The footprint table names `java.util.LinkedList$Node` with a count of 1,000,000 and a total size — divide to get bytes per node. That is your measured overhead, not an estimate. |
| `ArrayList` and `ArrayDeque` are close | Both are one array of references plus the elements. Any difference is slack capacity. |
| A large `java.lang.Integer` line in the footprint, shared across all three | The `Integer` objects are counted in each graph, and above 127 they are distinct objects (Topic 01). If you want to isolate the *structure* cost, rerun with `-XX:AutoBoxCacheMax=1000000` and compare — the cached case shares instances. |
| Numbers differ from any figure quoted here | Trust yours. Compressed references are disabled above roughly a 32 GB heap, which changes every reference from 4 to 8 bytes. Check with `java -XX:+PrintFlagsFinal -version \| grep UseCompressedOops`. |

Object layout in full is Topic 69. Today you are collecting one measured fact.

### Proof 4 — cache misses (Linux only)

```bash
# Linux with perf available:
java -jar target/benchmarks.jar "ListBench.iterate" -prof perfnorm
```

**What to look for:** the `cache-misses` and `L1-dcache-load-misses` rows, normalised
per operation.

| What you see | What it means |
|---|---|
| `LinkedList` shows far more cache misses per op than `ArrayList` at the same element count | Direct evidence for the mechanism. Both iterate n elements; only the memory access pattern differs. |
| `-prof perfnorm` errors with "perf not available" | You are on macOS or in a container without `perf_event` access. There is no drop-in equivalent; use `-prof gc` for allocation evidence and accept that the cache-miss claim is unverified on your machine. **Say that** rather than repeating the claim as if you had measured it. |
| Counters look implausible (zero, or wildly variable) | Virtualised environments often expose broken or restricted PMU counters. Do not build an argument on them. |

### Proof 5 — read the source, settle the version questions

```bash
mkdir -p /tmp/jdksrc && cd /tmp/jdksrc
unzip -o "$JAVA_HOME/lib/src.zip" \
  'java.base/java/util/ArrayList.java' \
  'java.base/java/util/LinkedList.java' \
  'java.base/java/util/ArrayDeque.java' > /dev/null

grep -n "DEFAULT_CAPACITY" java.base/java/util/ArrayList.java
grep -n "oldCapacity >> 1" -B4 -A4 java.base/java/util/ArrayList.java
grep -n "Node<E> node(int index)" -A12 java.base/java/util/LinkedList.java
grep -n "private void grow" -A12 java.base/java/util/ArrayDeque.java
```

**What to look for:**

| Where | What to look for | What it means |
|---|---|---|
| `ArrayList` | `DEFAULT_CAPACITY = 10` and the growth expression using `>> 1` | Confirms 1.5× growth and the lazy default capacity for yourself. |
| `LinkedList.node(int)` | `if (index < (size >> 1))` then a forward walk, else a backward walk | The halving optimisation. Note it is still a loop of `x = x.next` — dependent loads. |
| `ArrayDeque.grow` | the `jump` computation and whether any power-of-two rounding remains | **This settles the version uncertainty I flagged earlier.** Whatever your JDK's source says is the truth for your JDK. |
| `src.zip` missing | some distributions omit it | Read the same files in the OpenJDK GitHub mirror for your exact release tag. |

---

## Practice exercises

### 1 — Easy: pick and justify

For each `orderflow` requirement, name the implementation and give the **one sentence
of access pattern** that decides it. Then name the cost of the obvious wrong answer.

1. The lines of an order, built once from a query result of known size, then iterated.
2. A worker's inbox: tasks added at one end, taken from the other, thousands per minute.
3. An undo stack for an admin editing a product.
4. The 20 most recent order events, oldest dropped, displayed newest-first.
5. Payments waiting on a backoff, always asking "which is due next?".
6. A read-mostly list of feature flags, written once at startup, read by every request.
7. A list you build by prepending, then iterate once in the order you prepended.

Then: for numbers 4 and 6, name a later topic that gives a better answer, and say why.

### 2 — Medium: combines Topics 01, 07, 08 and 10

Implement a bounded, generic ring buffer:

```java
public final class BoundedHistory<T> implements Iterable<T> {
    public BoundedHistory(int capacity) { ... }
    public void record(T item) { ... }             // evicts the oldest when full
    public List<T> newestFirst() { ... }
    public int size() { ... }
}
```

Requirements:
- Back it with an `Object[]` and head/tail indices. **Do not** use `ArrayDeque`
  internally — the point is to build one.
- `record` must be O(1) with no allocation after construction. Prove the no-allocation
  claim with `-prof gc` in a small JMH benchmark, or with `-Xlog:gc` and a large loop,
  and state honestly which evidence you actually obtained.
- `newestFirst()` returns an **immutable copy** (Topic 10, Trap 4). Explain in a
  comment why not a view.
- Implement `Iterable<T>` with a fail-fast iterator: keep a `modCount`, snapshot it,
  and throw `ConcurrentModificationException` from `next()`. Then write a comment
  explaining precisely why your fail-fast is *also* best-effort and name the case it
  misses (Topic 10).
- The constructor throws `IllegalArgumentException` for `capacity < 1`. Justify
  unchecked against Topic 09.
- Add a `record(T item)` overload accepting `Collection<? extends T>` and justify the
  wildcard against Topic 07.
- Decide whether `null` items are allowed. Whichever you choose, say what it cost you
  — if you allow them, how does your "empty slot" sentinel work?

Then compare your class's memory footprint against `ArrayDeque` and `LinkedList` at
100,000 elements using the JOL command from Proof 3.

### 3 — Hard: production simulation on `orderflow`

**Part A — reproduce Trap 2 as a curve.** Build a queue drainer that reads N pending
retry tasks and processes them with `ArrayList.remove(0)`. Run it at
N = 5,000 / 10,000 / 20,000 / 40,000 / 80,000 and record wall-clock time.

```bash
java -Xmx1g -Xlog:gc:file=drain-arraylist.log:time,uptime QueueDrain arraylist 80000
```

Plot time against N. State whether the curve is linear or quadratic and **derive it
from your own code**: name the line, count the element moves, and show the arithmetic.
Then repeat with `ArrayDeque.pollFirst` and plot both curves on one chart.

> This wall-clock measurement is legitimate for exactly one purpose: an O(n²) curve is
> visible through JIT and GC noise because it grows faster than the noise. Anything
> subtler than that needs JMH. Say so in your write-up.

**Part B — the honest JMH pass.** Take the *closest* comparison from Part A (probably
middle insertion, not the queue drain) and rebuild it as a JMH benchmark using the
harness from Proof 2. Report `Score ± Error` and `gc.alloc.rate.norm`. Then answer:
**at what size, on your hardware, does the answer change?** If it never changes, say
that, and say what that tells you about the "LinkedList is good for insertion" advice.

**Part C — memory.** Using JOL, measure the footprint of one million `RetryTask`
records held in an `ArrayList`, a `LinkedList` and an `ArrayDeque`. Then answer a
capacity question with a real number: **`orderflow` runs in a 2 GB container.
How many in-flight retry tasks fit in each structure, leaving 50% headroom?** Show
your working. Then say which of your three answers you would actually put in a design
document and why.

**Part D — the retention leak.** Modify `waitingPage` in Example 2 to return
`snapshot.subList(from, to)` directly instead of `List.copyOf(...)`. Cache 1,000 such
pages in a `HashMap`. Run under `-Xmx512m` with a load loop until it OOMs, then:

```bash
java -Xmx512m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./oom.hprof AdminPager
```

Open the dump in Eclipse MAT or VisualVM and find the dominator. Report the **retained
size** of your 1,000-entry cache and compare it to the shallow size of the 50,000
records you thought you were holding. Then fix it and re-run.

> Full heap-dump analysis is Topic 79. You are doing the shallow version here to see
> that "retained" and "shallow" are different numbers, and that a `subList` is why.

**Part E — argue against yourself.** You have now built a case that `ArrayList` and
`ArrayDeque` beat `LinkedList`. Find or construct one `orderflow` scenario where
`LinkedList` genuinely wins, measure it, and report the numbers. If you cannot
construct one, say so explicitly — that is a legitimate and useful finding, and it is
what the JDK's own maintainers have said publicly.

---

## Interview questions

### Q1 — "When would you use `LinkedList` over `ArrayList`?"

**Mid-level answer:** "`LinkedList` is O(1) for insertion and removal in the middle,
`ArrayList` is O(1) for random access. So if you insert a lot in the middle, use
`LinkedList`."

**Senior answer:** "Almost never, and the Big-O comparison is the reason people get
this wrong. `LinkedList`'s middle insertion is O(1) *after* you have paid O(n) to walk
there, and every hop of that walk is a dependent load to an unpredictable address — a
likely cache miss the prefetcher cannot help with. `ArrayList` also pays O(n), but it
pays it inside `System.arraycopy`, a JIT intrinsic that moves a contiguous block at
near memory bandwidth. Same complexity class, order-of-magnitude different constant.
On top of that `LinkedList` allocates a node object per element — around 24 bytes of
overhead each on a 64-bit JVM with compressed oops — which is real GC pressure and
worse locality on every subsequent iteration. The narrow case where it genuinely wins
is when you already hold a `ListIterator` positioned where you are working and do many
inserts there, so you never pay traversal. For queues and stacks I would use
`ArrayDeque`, which beats both. And I would confirm any of this with JMH before
changing code, because a wall-clock loop will give me a wrong answer."

**What separates them:** explaining the constant factor in terms of the memory
hierarchy, knowing the one narrow winning case, naming `ArrayDeque`, and refusing to
assert a performance claim without a measurement.

**Follow-up:** "How would you measure it?" They want JMH, forks, warm-up, `Blackhole`,
and ideally `-prof gc`. "I'd time it in a loop" is the wrong answer and they are
listening for it.

---

### Q2 — "This nightly job takes 4 seconds on 20,000 rows and 400 seconds on 200,000. What is your first hypothesis?"

**Mid-level answer:** "It's probably a slow database query, or it's running out of
memory and GC-ing."

**Senior answer:** "Ten times the input, a hundred times the time — that is a
quadratic signature, so my first hypothesis is an O(n²) operation in the code rather
than anything in the infrastructure. GC pressure and slow queries are usually linear
or worse-than-linear-but-not-cleanly-quadratic; a clean 10×-to-100× is a strong hint.
The two shapes I would grep for immediately are `remove(0)` or `remove(index)` inside
a loop over an `ArrayList`, and `List.contains` or `indexOf` inside a loop — the first
shifts the whole array n times, the second scans it n times. I would confirm with a
CPU profile: if the flame graph is one wide tower ending in `System.arraycopy` or
`ArrayList.indexOf` and `String.equals`, that is it. The fixes are `ArrayDeque` for a
drain queue and a `HashSet` for membership. If the profile does not show that, I'd look
at N+1 queries next, which have a similar shape."

**What separates them:** reading the *shape* of the numbers as evidence before
touching anything, having two specific greppable patterns, and naming what the profile
would have to show to confirm.

**Follow-up:** "The profile shows `arraycopy`, but the code has no `remove(0)`. Where
else does `arraycopy` come from?" `ArrayList.grow` (missing pre-size), `add(index, x)`,
`Arrays.copyOf` in `toArray`, and `subList` operations.

---

### Q3 — "Why does `ArrayDeque` reject `null` when `LinkedList` allows it?"

**Mid-level answer:** "It's just a restriction in the API — the Javadoc says nulls
aren't permitted."

**Senior answer:** "It falls out of the implementation, and it is the right call.
`ArrayDeque` is a circular buffer over an `Object[]` with `head` and `tail` indices,
and it uses `null` in the array to mean 'this slot is empty'. If you could store a
real `null`, the structure could not distinguish a stored value from an empty slot —
`pollFirst` returning `null` would be ambiguous between 'empty queue' and 'the next
element is null'. `LinkedList` has no such problem because each element lives in its
own node object, so absence and null are different things. Practically this bites when
you migrate a queue from `LinkedList` to `ArrayDeque` and something was using `null` as
a batch sentinel — you get an immediate NPE from `addLast` on a line that worked for
years. The real fix is that `null` as a sentinel was always a design smell; with Java
21 I would model it as a sealed interface with an explicit end-of-batch variant."

**What separates them:** deriving the rule from the data structure rather than quoting
the doc, and connecting it to the ambiguity in `poll`'s return value.

**Follow-up:** "What other collections reject null, and is there a pattern?"
`ConcurrentHashMap`, `PriorityQueue`, `TreeMap` keys, and all the `of()` factories. The
pattern: anywhere `null` is needed as an internal sentinel, or anywhere a null return
would be ambiguous, or where the designers simply decided nulls were a mistake and used
a new API as the chance to say so.

---

### Q4 — "What is the difference between an `ArrayList`'s size and its capacity, and when does it matter?"

**Mid-level answer:** "Size is how many elements are in it; capacity is how big the
internal array is. Capacity grows automatically."

**Senior answer:** "Right, and three consequences follow. First, growth is geometric —
about 1.5× in the JDK — so `add` is amortised O(1), but each growth allocates a new
array and copies the old one, and the last few growths on a large list dominate: going
to 500,000 elements, the final copy alone moves several hundred thousand references
and abandons a multi-megabyte array. Pre-sizing with `new ArrayList<>(n)` removes all
of it when you know n, which you usually do when you are mapping a result set. Second,
capacity never shrinks on `remove` — a list that peaked at a million elements and now
holds ten still holds a million-slot array until you call `trimToSize()`, which is a
real retention shape in a long-lived cache. Third, `new ArrayList<>()` allocates
essentially nothing until the first `add`, because the backing array starts as a shared
empty constant — so having many empty lists is cheap, which occasionally matters in a
map-of-lists. That said, I would not go through a codebase adding capacity hints; I do
it where the size is known and the list is large, and let a profiler tell me about the
rest."

**What separates them:** the non-shrinking retention shape, the lazy empty array, and
explicitly declining to over-apply the optimisation.

**Follow-up:** "How would you find a list that is retaining a huge backing array?" A
heap dump and a dominator-tree view — retained size versus shallow size. Topic 79.

---

### Q5 — "Big-O says these two are the same. Explain why one is ten times faster."

**Mid-level answer:** "Constant factors. Big-O ignores constants."

**Senior answer:** "Right, but the useful answer names *which* constant. Big-O counts
operations and treats every memory access as unit cost. That assumption was
approximately true in 1970 and is off by two orders of magnitude now: an L1 hit is
about 4 cycles and a DRAM access is 200-plus. So an algorithm that touches n
contiguous bytes and one that follows n scattered pointers both count as O(n), but the
first runs at memory bandwidth with the prefetcher fully engaged and the second
serialises on dependent loads that the prefetcher cannot predict. Concretely: reading
one 4-byte reference pulls in a whole 64-byte cache line, so a sequential array scan
gets sixteen references per memory fetch, while a linked-list walk gets one and cannot
even start the next fetch until the current node arrives. That is the whole
`ArrayList`-versus-`LinkedList` story. My working rule is: Big-O to rank algorithms and
rule out the quadratic ones, memory layout to predict the wall clock among survivors,
and a measurement before I claim a number."

**What separates them:** naming the specific hardware mechanisms — cache lines,
prefetching, dependent loads — and stating a rule for when to trust which model.

**Follow-up:** "How would you demonstrate the cache-miss claim rather than assert it?"
`perf stat -e cache-misses` or JMH's `-prof perfnorm` on Linux — and an honest "I can't
measure that on macOS, so I'd report it as unverified" is a strong answer, not a weak
one.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `ArrayList` grows by 1.5×, not 2×. Both give amortised O(1) appends. What does 1.5×
   buy that 2× does not? (Hint: think about what happens to the abandoned arrays and
   whether the allocator can reuse them.)

2. `LinkedList.node(int)` walks from the nearer end, halving the average traversal.
   Does that change its complexity class? Does it change the wall clock in a way you
   would notice? Are those the same question?

3. `ArrayDeque` is contiguous like `ArrayList` and O(1) at both ends, which `ArrayList`
   is not. What did it give up to get that, and why is it not simply a better `List`?

4. Suppose a future JVM allocated every `LinkedList$Node` contiguously as you appended.
   Which of this topic's arguments would survive, and which would collapse?

5. You measure and find `LinkedList` faster at n = 100 and slower at n = 100,000. What
   should you do with that finding? Is "use LinkedList for small lists" a sound
   engineering rule? Argue both sides.

6. Example 1 in this doc is deliberately a bad benchmark, and Proof 2 is a good one.
   Name three specific things the JMH version does that the naive version does not, and
   for each, describe the wrong conclusion you would reach without it.

7. Your JMH run contradicts a claim in this document. Walk through, in order, what you
   would check before concluding the document is wrong — and then say what would make
   you conclude it *is* wrong.

---

## Quick reference card

### Choosing

```
Indexed access, mostly append/read     -> ArrayList  (pre-size if n is known)
FIFO queue or LIFO stack               -> ArrayDeque
"What is due next?"                    -> PriorityQueue
Many inserts at a HELD iterator        -> LinkedList  (rare; measure first)
Read-mostly, shared across threads     -> CopyOnWriteArrayList (Topic 97)
Never changes after construction       -> List.of / List.copyOf
Legacy code you must recognise         -> Vector, Stack  [LEGACY — never write]
```

### Cost table

| Operation | `ArrayList` | `LinkedList` | `ArrayDeque` |
|---|---|---|---|
| `get(i)` | O(1) | O(n) | not supported |
| `add` at end | amortised O(1) | O(1) | amortised O(1) |
| `add`/`remove` at front | O(n) | O(1) | O(1) |
| `add`/`remove` in middle | O(n) — one `arraycopy` | O(n) to walk + O(1) to link | not supported |
| `contains` | O(n) | O(n) | O(n) |
| iterate all | sequential scan, prefetched | dependent loads | sequential scan |
| overhead per element | ~4 bytes ref + up to 50% slack | ~24 bytes node + ref | ~4 bytes ref + slack |
| allows `null` | yes | yes | **no** |

### Implementation facts worth remembering

- `ArrayList` default capacity **10**, allocated lazily on first `add`; growth ~**1.5×**.
- Capacity **never shrinks** on removal — `trimToSize()` if you need it back.
- `LinkedList` node: element ref + `next` + `prev`, in its own heap object.
- `LinkedList.node(int)` walks from the nearer end.
- `ArrayDeque` is a ring buffer; `null` is its empty-slot sentinel, hence no nulls.
- `Stack` extends `Vector` and iterates **bottom-to-top**. `ArrayDeque` instead.
- `subList` and `reversed()` are **views**. `List.copyOf` is a copy.
- Cache line: 64 bytes on x86-64, 128 on Apple Silicon.

### Benchmark discipline (Topic 77 in one box)

```bash
mvn archetype:generate -DarchetypeGroupId=org.openjdk.jmh \
    -DarchetypeArtifactId=jmh-java-benchmark-archetype -DinteractiveMode=false ...
mvn clean verify
java -jar target/benchmarks.jar MyBench -f 3 -wi 5 -i 10 -prof gc
```
- Always `@Fork`, `@Warmup`, `@Measurement`.
- Always consume results with `Blackhole`.
- Never build state inside the timed method.
- **If the error bars overlap, there is no result.**
- Never quote a `System.nanoTime()` loop in a design document.

---

## When would I use this at work?

**1. Reviewing a loop over a collection.**
Three patterns, all greppable in seconds: `remove(0)` inside a loop, `.contains(` on a
`List` inside a loop, and `new LinkedList<>()` anywhere. Each has a specific
superlinear consequence and a one-line fix. This is the single highest-yield review
habit in this phase.

**2. Diagnosing a job that got slow when the data grew.**
The 10×-input-100×-time signature points straight at a quadratic collection operation
before you touch the database or the infrastructure. Knowing to read the *shape* of
the timings first saves the first hour of every such investigation.

**3. Answering a capacity question with a number.**
"Can we hold a million in-flight retry tasks in a 2 GB container?" is `ArrayList`
versus `LinkedList` versus `ArrayDeque`, measured with JOL, and it is the difference
between "yes with headroom" and "we need Redis". You will do this properly in Topics
69 and 129; the instinct and the measuring tool start here.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and autoboxing**: `List<Integer>` boxes every element, which is
  the *other* memory cost sitting on top of everything measured here, and
  `remove(int)` vs `remove(Object)` is an overload trap.
- **05–07 — Generics and wildcards**: every signature here; `List.copyOf(Collection<?
  extends E>)` is PECS in the JDK.
- **10 — Collections Framework**: the interfaces, the contracts, `subList` as a view,
  and `ConcurrentModificationException` — all of which Trap 5 depends on.

**This unlocks:**
- **12 — HashMap internals**: the same "contiguous array plus indexing" idea, applied
  to hashing, with treeification as the collision escape hatch.
- **15 — LinkedHashMap and LRU**: what a linked structure is genuinely good for —
  ordering metadata layered over a hash table, not as the primary storage.
- **16 — EnumSet/EnumMap**: array-indexed and bitvector representations, the extreme
  end of "contiguous beats scattered".
- **23–25 — Streams**: `ArrayList` and arrays split perfectly for parallel streams;
  `LinkedList` splits terribly, which is one of the three preconditions in Topic 25.
- **68–69 — GC and object layout**: where the ~24 bytes per node comes from, verified
  rather than quoted, and why allocation rate matters.
- **77 — JMH**: the proper version of every measurement in this doc, and why Example 1
  was untrustworthy.
- **78 — Profiling**: flame graphs, and how `LinkedList.node(int)` appears in one.
- **79 — Heap dumps and leaks**: Trap 5's retention shape, and the difference between
  shallow and retained size.
- **96 — False sharing**: the other side of cache-line behaviour — when contiguity
  hurts instead of helping.
- **97 — Concurrent collections**: `CopyOnWriteArrayList`, and why "copy the whole
  array on every write" is sometimes the right trade.

---

*Java baseline 21. `ArrayList` and `LinkedList` have been stable for many releases.
`ArrayDeque`'s internal growth policy was rewritten around Java 9 and no longer
maintains a power-of-two capacity — the observable contract is unchanged, and Proof 5
settles what your JDK actually does. Sequenced collections (`getFirst`, `getLast`,
`reversed`) are new in Java 21 and are used above with the pre-21 fallback shown.
Project Valhalla's value types would eventually change the memory arithmetic here
substantially; it has not landed as of Java 25, so do not plan around it.*
