# 25 — Parallel Streams: When They Help and When They Actively Hurt

## Phase: 2 — Modern Java
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## Mechanical statement

No analogy. Here is literally what happens when you write `.parallel()`.

**1. `.parallel()` sets a boolean flag on the pipeline.** It is not a position in the
chain. Writing `.parallel()` first, last, or in the middle is identical. The last call
to `parallel()` or `sequential()` anywhere in the chain wins, and it applies to the
whole pipeline.

**2. The terminal operation submits the work to `ForkJoinPool.commonPool()`.** Not a
new pool. Not your pool. **The** pool — one JVM-wide static singleton, created lazily,
shared by every parallel stream in the process, by `CompletableFuture`'s default
`*Async` methods, by `Arrays.parallelSort`, and by any library in your dependency tree
that reaches for a default executor.

**3. That pool has `Runtime.getRuntime().availableProcessors() - 1` worker threads.**
The thread that called the terminal operation also participates, so total concurrency is
`availableProcessors()`. On a machine or container reporting **1** processor,
parallelism is **0**: there are no workers, and everything runs on the calling thread.
`.parallel()` then does nothing at all, silently, with no warning and no error.

**4. The source is divided by its `Spliterator`.** `trySplit()` is called repeatedly to
carve the source into chunks. How well that works is a property of the source, not of
your code:

| Source | `trySplit` behaviour |
|---|---|
| `ArrayList`, arrays, `IntStream.range` | Perfect. Halve the index range. O(1), exactly balanced. |
| `HashMap`/`HashSet` | Good. Splits the bucket table. Roughly balanced. |
| `TreeMap`/`TreeSet` | Reasonable. Splits by subtree. |
| `LinkedList` | Terrible. Must walk the list to find a midpoint, and reports no exact size. |
| `Stream.iterate` / `Stream.generate` | Effectively none. Unknown size, inherently sequential. |
| `BufferedReader.lines` / `Files.lines` | Poor. Splits by buffered chunk, unbalanced. |

**5. Each split gets its own accumulation container** (the collector's `supplier`, from
Topic 24) and each is confined to one thread while it accumulates. Then the
**combiner** merges them pairwise. **The combiner only ever runs here.** A combiner that
is wrong, non-associative, or drops one side is invisible in every sequential test and
produces silently wrong answers only under parallelism.

**6. Nothing about this is asynchronous.** The terminal operation blocks the calling
thread until all splits complete. `.parallel()` is not a way to avoid waiting. It is a
way to use more CPUs while waiting.

**7. A blocking operation inside a parallel stream occupies a common-pool worker for
the whole block.** A parked worker cannot steal work. With `cores − 1` workers, a
handful of blocked tasks stalls every other parallel stream in the entire JVM —
including ones in code you did not write. That is the failure drill.

---

## The bridge from what you know

### `Promise.all` is not the analogue. Say it out loud.

```ts
await Promise.all(orderIds.map(id => fetchOrder(id)));
```

That is **concurrency**: one thread, many in-flight I/O operations, the event loop
interleaving continuations while sockets are waiting. It gives you throughput on
**waiting**. Zero CPUs were added.

```java
orders.parallelStream().map(this::computePricing).toList();
```

That is **parallelism**: many threads, many CPU cores executing simultaneously. It gives
you throughput on **computing**. Nothing about it helps with waiting.

The two are solutions to opposite problems, and conflating them produces the single
worst decision in this topic: putting HTTP or JDBC calls inside a parallel stream
because "that's how you do concurrent I/O in Java". It is not, and it damages the whole
JVM.

**Verdict: NO ANALOGUE. Do not transfer anything from `Promise.all`.**

### `worker_threads` is closer, and still breaks

```ts
// Node
const worker = new Worker('./price.js');
worker.postMessage(orders);
```

Genuinely parallel — separate OS threads, separate CPUs. Closest thing you have.

Where it breaks: **each Node worker has its own isolated V8 heap.** Data crosses by
copying (structured clone) or through a `SharedArrayBuffer`. You cannot accidentally
share a mutable object, because there is nothing to share.

Java threads share **one heap**. Every object your lambda touches is reachable and
mutable from every worker. The isolation Node gives you for free is something you must
construct by discipline in Java. That one difference is why "no shared mutable state" is
a *precondition* here and a *tautology* in Node.

**Verdict: PARTIAL ANALOGUE.** Transfer "this is real parallelism". Unlearn "the
runtime protects me from sharing".

### The summary table

| You know | Java parallel streams | Verdict |
|---|---|---|
| `Promise.all` | `.parallel()` | **NO ANALOGUE** — concurrency vs parallelism |
| `worker_threads` | ForkJoinPool workers | **PARTIAL** — isolated heaps vs one shared heap |
| `SharedArrayBuffer` + `Atomics` | Shared heap + JMM | **HONEST ANALOGUE** — the one place JS has a real memory model |
| `cluster`, one process per core | One JVM, many threads | **NO ANALOGUE** — sharing is the default, not the exception |
| Node has no global thread pool | `ForkJoinPool.commonPool()` is a JVM-wide singleton | **NO ANALOGUE** — you can starve code you never wrote |
| Array methods have no parallel switch | `.parallel()` is one word | **NO ANALOGUE** — and the ease of typing it is the danger |

---

## What is this?

`.parallel()` (or `collection.parallelStream()`) runs a stream pipeline across multiple
threads by splitting the source, processing each split independently, and merging the
partial results.

```java
long revenuePence = orders.parallelStream()
        .mapToLong(Order::totalPence)
        .sum();
```

Same API as Topic 23. Same collectors as Topic 24. One word changed.

### The three preconditions

This is the mastery line from the master plan. All three must hold, or do not do it.

**1. A splittable source.** `ArrayList` and arrays split in O(1) into exactly balanced
halves. `LinkedList` must be walked. `Stream.iterate` cannot split at all. If the source
does not split, you get all the coordination overhead and none of the parallelism.

**2. Sufficient `N × Q`.** `N` is the number of elements. `Q` is the CPU cost per
element. The overhead of splitting, scheduling, and merging is roughly fixed, so the
work has to be big enough to pay for it. The widely-cited heuristic is
**`N × Q` above roughly 10,000 "units"** — where a unit is something like a handful of
simple operations. It is a starting hypothesis to be tested, not a rule.

The important shape: **it is the product that matters, not either factor.** Ten million
elements with `Q = 1` (adding an int) is often a loss. Two hundred elements with
`Q =` a cryptographic hash is often a win.

**3. No shared mutable state, and an associative reducer.** Every lambda in the
pipeline runs on multiple threads simultaneously over one shared heap. Any mutation of
anything outside the pipeline is a data race. And the reduction must be associative —
`(a ⊕ b) ⊕ c == a ⊕ (b ⊕ c)` — because the framework decides the grouping and it varies
between runs.

Subtraction is not associative. Neither is "take the first". Neither is a running
average computed naively. Sum, product, min, max, string concatenation in order, and set
union are.

### When it is a clear win

- CPU-bound work over a large array or `ArrayList`.
- Numeric aggregation with `IntStream`/`LongStream`/`DoubleStream` over a range.
- `Arrays.parallelSort` on a large primitive array (this one is genuinely excellent).
- Batch/offline processing where the JVM is doing nothing else.

### When it is a clear loss, or worse

- Any I/O: HTTP, JDBC, file reads, anything that blocks.
- Inside a request-handling thread in a web service — you are competing with every
  other request for the same `cores − 1` workers.
- Small `N`, or cheap `Q`, or both.
- `LinkedList`, `Stream.iterate`, or an I/O-backed source.
- Anything with a side effect, ordering requirement, or shared accumulator.
- In a container with a CPU limit at or near 1 core.

---

## Why does it matter?

**1. It is one word, and the word looks free.** No other performance decision in Java is
this easy to type and this easy to get wrong. There is no compiler warning, no lint
default, no runtime signal. A junior engineer adds `.parallel()` to a slow report,
observes it is faster on their 10-core laptop, and ships something that is slower and
occasionally wrong in a 1-core production container.

**2. The blast radius is the whole JVM.** Almost every other performance mistake is
local. This one is not: the common pool is shared, so blocking in it degrades unrelated
code — other endpoints, background jobs, library internals. That is a property you
cannot discover by reading the file the bug is in, and it is the reason this topic is a
DIFFERENTIATOR.

**3. The correctness bugs are load-dependent and non-reproducible.** A non-associative
reducer or a broken combiner (Topic 24) gives right answers on your laptop and
occasionally wrong ones in production, with different wrongness each run. You cannot
bisect it. You cannot write a failing test unless you already know what you are looking
for.

**4. It is the gateway to Phase 9.** The common pool, work stealing, blocking-worker
starvation, and `ManagedBlocker` are all Topic 100. `CompletableFuture`'s default
executor being this same pool is Topic 91's drill. Getting the mechanical model right
here means those topics are recognition rather than learning.

---

## Machine-level reality

### The work-stealing deque

`ForkJoinPool` does not have one shared task queue. Each worker thread owns a
**double-ended queue** (deque) of tasks.

```
Worker 1 deque:   [ T1a | T1b | T1c ]
                    ^                ^
                    |                |
              own end (LIFO)    steal end (FIFO)

Worker 2 deque:   [ ]   <- idle, will steal from another worker's steal end
```

- A worker **pushes and pops its own tasks at the head, LIFO**. LIFO because the most
  recently created subtask is the one whose data is still in L1/L2 cache. This is a
  cache-locality decision, not a fairness one.
- An **idle worker steals from the tail of another worker's deque, FIFO**. FIFO because
  the oldest task in a divide-and-conquer decomposition is the *largest* remaining
  chunk — steal once, get a lot of work, minimise future stealing.
- The two ends being different is what makes stealing mostly contention-free: the owner
  and the thief touch opposite ends of the array.

**Consequence that matters:** a worker that is **blocked** — parked on a lock, waiting
on a socket, sleeping — is not running and **cannot steal**. It is a dead thread
occupying one of your `cores − 1` slots. Three blocked tasks on a 4-core machine leave
zero workers.

That single sentence is the entire failure drill.

### Splitting cost, concretely

`trySplit()` is called recursively until chunks are small enough (the pool targets
roughly `4 × parallelism` tasks, so there is work to steal). For an `ArrayList` of
1,000,000:

```
[0, 1000000)
  → [0, 500000)  [500000, 1000000)
      → [0, 250000) [250000, 500000) ...
```

Each split is: allocate a new `Spliterator` object, record two indices. Two arithmetic
operations. Effectively free, and the chunks are exactly equal.

For a `LinkedList` of 1,000,000, `trySplit` must **traverse** to find a midpoint,
following a pointer chain that is scattered across the heap (Topic 11). The default
`Spliterators.IteratorSpliterator` buffers elements into an array in increasing batch
sizes instead. So you pay:

- a full traversal with a cache miss per node, plus
- array allocation and copying for the batches, plus
- unbalanced chunks, so one worker finishes early and steals, adding coordination,

...before any of your actual work happens. This is why the master plan says
"`LinkedList` splits terribly while `ArrayList`/arrays split perfectly", and it is a
memory-hierarchy fact, not a Big-O one.

### Why `N × Q` is the real threshold

Let:
- `S` = fixed cost of splitting, scheduling, and merging (task objects, deque
  operations, cache-line traffic on the steal ends, combiner invocations).
- `P` = effective parallelism, at most `availableProcessors()`.

Sequential time ≈ `N × Q`.
Parallel time ≈ `(N × Q) / P + S`.

Parallel wins when `(N × Q)(1 − 1/P) > S`. `S` is roughly constant in the tens of
microseconds. So the whole decision reduces to: **is `N × Q` large enough to dwarf a
fixed startup cost, divided across `P − 1` extra cores?**

Two consequences people get wrong:

- **`N` alone tells you nothing.** Ten million elements at `Q = 2ns` is 20 ms of total
  work. Splitting overhead is a meaningful fraction of that, and memory bandwidth — not
  CPU — is usually the binding constraint anyway, so extra cores do not help.
- **`Q` alone tells you nothing.** Four elements at `Q = 100ms` is 400 ms of work, but
  with four elements you get at most four-way splitting and the tail latency is one
  element's cost.

And then there is Amdahl: if the terminal operation is `sorted()` or a collector with an
expensive combiner, the merge phase is serial and caps your speedup regardless of core
count.

### Container reality

Since JDK 10, `Runtime.availableProcessors()` respects cgroup CPU limits
(`UseContainerSupport`, on by default). So:

| Container config | `availableProcessors()` | Common pool parallelism | `.parallel()` does |
|---|---|---|---|
| `--cpus=8` | 8 | 7 | real parallelism |
| `--cpus=2` | 2 | 1 | one worker plus the caller |
| `--cpus=1` | 1 | **0** | **nothing — all on the calling thread** |
| `--cpus=0.5` | 1 | **0** | **nothing** |
| K8s `limits.cpu: 500m` | 1 | **0** | **nothing** |

That bottom half of the table is the common case in production. A great many Kubernetes
services run with a CPU limit under two cores, which means every `.parallel()` in that
codebase is dead code that nobody has noticed — and the one place it *does* run
(someone's laptop, a load-test box) is where the concurrency bugs hide.

You can override with `-Djava.util.concurrent.ForkJoinPool.common.parallelism=N`, but
setting it above your CPU quota just means the kernel throttles you: more threads,
same CPU-seconds, more context switching, worse latency.

---

## Example 1 — minimal

Reveal the pool, the threads, and the parallelism, with no benchmarking claims.

```java
import java.util.*;
import java.util.concurrent.ForkJoinPool;
import java.util.stream.*;

public class WhoRunsThis {
    public static void main(String[] args) {

        System.out.println("availableProcessors      = "
                + Runtime.getRuntime().availableProcessors());
        System.out.println("commonPool parallelism   = "
                + ForkJoinPool.getCommonPoolParallelism());
        System.out.println("commonPool poolSize      = "
                + ForkJoinPool.commonPool().getPoolSize());

        System.out.println("\n--- SEQUENTIAL ---");
        threadsUsed(IntStream.range(0, 64).boxed().toList().stream());

        System.out.println("\n--- PARALLEL ---");
        threadsUsed(IntStream.range(0, 64).boxed().toList().parallelStream());
    }

    static void threadsUsed(Stream<Integer> stream) {
        Set<String> names = stream
                .map(i -> Thread.currentThread().getName())
                .collect(Collectors.toCollection(java.util.concurrent.ConcurrentSkipListSet::new));
        names.forEach(n -> System.out.println("  " + n));
    }
}
```

Two details that are deliberate:

- The collector is a **thread-safe** `ConcurrentSkipListSet`. Collecting into a plain
  `HashSet` from a parallel stream is exactly the bug this topic is about, and we are
  not going to demonstrate the mechanism using the bug.
- `getPoolSize()` may read 0 before any work has been submitted — the pool creates
  workers lazily. Run the parallel section first if you want to see it non-zero.

---

## Example 2 — production scenario

`orderflow` has a nightly risk-scoring job: for every order placed in the last 24 hours,
compute a fraud score. The scoring function is pure CPU — a few hundred arithmetic
operations plus a small model evaluation — around 80 µs per order. There are roughly
900,000 orders.

### The version someone writes at 4pm on a Friday

```java
@Scheduled(cron = "0 30 2 * * *")
public void scoreOvernight() {

    List<Order> orders = orderRepository.findPlacedSince(Instant.now().minus(1, DAYS));

    Map<OrderId, RiskScore> scores = new HashMap<>();          // shared mutable state

    orders.parallelStream().forEach(order -> {
        RiskScore score = riskModel.score(order);              // 80 us of CPU  — fine
        PaymentHistory history = paymentClient.historyFor(order.customerId());  // 60 ms HTTP
        scores.put(order.id(), score.adjustedFor(history));    // unsynchronised HashMap
    });

    riskRepository.saveAll(scores);
}
```

Four separate defects, in increasing order of nastiness:

1. **`scores` is a plain `HashMap` written from many threads.** Data race. Symptoms
   below.
2. **A 60 ms blocking HTTP call inside a parallel stream.** Every common-pool worker
   spends 99.9% of its time parked. With `cores − 1` workers, this job monopolises the
   entire pool.
3. **The blocking is not just slow for this job — it stalls the JVM.** Any other
   parallel stream, any `CompletableFuture.supplyAsync` without an explicit executor,
   any `Arrays.parallelSort`, anywhere in the process, now queues behind blocked
   workers.
4. **In a 1-CPU container, none of this parallelises at all** — so the whole thing is
   sequential, 900,000 × 60 ms of HTTP = 15 hours, and the "fix" of adding `.parallel()`
   did nothing.

Symptom 1 in the wild, on a shared `HashMap`:
```
java.lang.NullPointerException
	at java.base/java.util.HashMap.putVal(HashMap.java:637)
```
or an infinite loop inside `HashMap.getNode` (pre-Java-8 resize cycling; on Java 8+
usually a lost entry instead), or — most often — no exception at all and
`scores.size()` quietly less than `orders.size()`, differing on every run.

### The version that separates the two problems

The key insight: **there are two different bottlenecks here and they need two different
tools.** The HTTP calls need concurrency. The scoring needs parallelism. Do not use one
mechanism for both.

```java
@Scheduled(cron = "0 30 2 * * *")
public void scoreOvernight() {

    List<Order> orders = orderRepository.findPlacedSince(Instant.now().minus(1, DAYS));

    // ---- Stage 1: the I/O. Batch it. This is not a parallel-stream problem. ----
    Set<CustomerId> customers = orders.stream()
            .map(Order::customerId)
            .collect(Collectors.toSet());

    Map<CustomerId, PaymentHistory> histories =
            paymentClient.historyForAll(customers);          // ONE bulk call, or paged

    // ---- Stage 2: the CPU. This is what parallel streams are for. ----
    Map<OrderId, RiskScore> scores = orders.parallelStream()
            .collect(Collectors.toConcurrentMap(
                    Order::id,
                    order -> riskModel.score(order)
                                      .adjustedFor(histories.get(order.customerId()))));

    riskRepository.saveAll(scores);
}
```

What each change did:

| Change | Why |
|---|---|
| The HTTP call moved **out** of the stream | No blocking in the common pool. Nothing else in the JVM is affected. |
| One bulk call replaces 900,000 | The actual fix. This was an N+1 (Topic 50) wearing a parallelism costume. |
| `toConcurrentMap` instead of `forEach` + shared map | No shared mutable state that you manage. The collector owns thread safety. |
| `parallelStream` retained for scoring only | `N × Q` = 900,000 × 80 µs = 72 seconds of pure CPU. That is comfortably above the threshold, over an `ArrayList` that splits perfectly. |

`toConcurrentMap` is the right collector here rather than `toMap` because it declares
`CONCURRENT` + `UNORDERED` — every thread accumulates into one shared
`ConcurrentHashMap` rather than each building a private `HashMap` and merging. For a
map with ~900,000 distinct keys, skipping the merge is a real saving. If the result
needed encounter order, `toMap` would be correct and the merge would be the price.

### What is still wrong, honestly

- **It still runs on the common pool.** For a scheduled batch job in a service that also
  serves traffic, that is a shared-resource decision you should make explicitly. See the
  dedicated-pool idiom in the Failure drill.
- **`histories` for 900,000 orders may not fit in memory.** The real version pages, or
  processes per-customer.
- **Nothing here is measured.** Everything above is a *hypothesis* that parallelism
  helps. The Measurement section is where you find out.
- **`saveAll` of 900,000 rows is its own problem** — Topic 53.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `.parallel()` silently does nothing in a 1-CPU container

**Wrong:**
```java
long total = orders.parallelStream().mapToLong(Order::totalPence).sum();
```
...deployed with:
```yaml
resources:
  limits:
    cpu: "500m"
```

**Exact symptom:** none. No error, no warning, no log line. The correct answer is
returned. The p99 latency is exactly what it was before the change. The engineer who
added `.parallel()` reports it "didn't help much" and moves on. Meanwhile
`ForkJoinPool.getCommonPoolParallelism()` returns `0` and every parallel stream in the
service has been running sequentially for two years.

The worse version of this symptom: it *did* work on the load-test box (8 cores), where
a latent race in a shared accumulator manifested once in ten thousand runs and was
dismissed as flakiness.

**Root cause:** common-pool parallelism is `availableProcessors() - 1`. Since JDK 10,
`availableProcessors()` honours the cgroup CPU quota. `500m` rounds up to 1 processor,
so parallelism is 0, so there are no worker threads and the caller does all the work.

**Fix — in order of preference:**

1. **Verify before you assume.** Log it at startup:
   ```java
   log.info("cpus={} commonPoolParallelism={}",
            Runtime.getRuntime().availableProcessors(),
            ForkJoinPool.getCommonPoolParallelism());
   ```
   Put this in every service. It costs one line and answers a question you will
   otherwise ask under pressure.

2. **Give the container enough CPU, or remove the `.parallel()`.** Those are the two
   honest options. A `.parallel()` that cannot parallelise is a lie in the source code.

3. **Do not "fix" it with
   `-Djava.util.concurrent.ForkJoinPool.common.parallelism=8` on a 1-CPU container.**
   You get 8 threads sharing one CPU quota: same throughput, more context switches,
   worse p99, and CFS throttling. That is strictly worse than sequential.

---

### Trap 2 — a non-associative reducer

**Wrong:**
```java
// "net position: start from the opening balance and subtract each movement"
long net = movements.parallelStream()
        .reduce(openingBalance, (running, m) -> running - m.amountPence(), Long::sum);
```

**Exact symptom:** the answer is correct sequentially, wrong in parallel, and **wrong by
a different amount each run**. There is no exception. A finance reconciliation fails
with a variance that nobody can reproduce, and re-running the job "fixes" it about a
third of the time.

**Root cause:** two independent violations of `reduce`'s contract.

1. **Non-associative accumulator.** `(a − b) − c ≠ a − (b − c)`. The framework groups
   the elements however the splits landed, so the grouping — and therefore the answer —
   changes run to run.
2. **The identity is not an identity.** `reduce(identity, acc, combiner)` requires
   `acc(identity, x) == x`. Here `openingBalance − x ≠ x`. Under parallelism the identity
   is applied **once per split**, so `openingBalance` is subtracted from N times, not
   once.
3. **And the combiner disagrees with the accumulator.** The accumulator subtracts; the
   combiner (`Long::sum`) adds. `reduce` requires
   `combiner(a, acc(identity, b)) == acc(a, b)`. It does not hold.

Three bugs in one line, and every one of them is invisible sequentially, because
sequentially the identity is applied once and the combiner is never called.

**Fix — restate it as an associative operation:**
```java
long totalMovements = movements.parallelStream()
        .mapToLong(Movement::amountPence)
        .sum();                                   // sum IS associative
long net = openingBalance - totalMovements;       // apply the non-associative part once
```

**The general technique:** pull the non-associative part out of the reduction. Reduce
with something associative, then apply the rest once at the end. This works for almost
every case people get wrong — running averages become (sum, count) then divide;
"first match" becomes `findFirst` on an ordered stream, not a reduce.

**How to test for it:** compare parallel and sequential over a large random input, many
times.
```java
@RepeatedTest(50)
void parallelAgreesWithSequential() {
    List<Movement> input = randomMovements(100_000);
    assertThat(net(input.parallelStream())).isEqualTo(net(input.stream()));
}
```
This is the same test shape as Topic 24's combiner test, and for the same reason.

---

### Trap 3 — a shared mutable accumulator

**Wrong:**
```java
List<RiskScore> scores = new ArrayList<>();
orders.parallelStream().forEach(order -> scores.add(riskModel.score(order)));
```

**Exact symptom:** one of several, non-deterministically, on different runs of identical
input:

```
java.lang.ArrayIndexOutOfBoundsException: Index 47 out of bounds for length 47
	at java.base/java.util.ArrayList.add(ArrayList.java:487)
```
or
```
java.lang.NullPointerException
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
```
or — most commonly and most dangerously — **no exception at all**, with
`scores.size()` less than `orders.size()` by a varying amount. Nulls appear at random
indices where two threads wrote to the same slot.

**Root cause:** `ArrayList.add` is three unsynchronised steps: read `size`, write
`elementData[size]`, increment `size`. Two threads interleaving lose an element, write
to the same index, or resize concurrently and corrupt the backing array. This is a plain
data race on a shared heap — the thing `worker_threads` makes structurally impossible
and Java does not.

**Fix — in strict order of preference:**

```java
// 1. BEST: let the collector own thread-safety. Each split accumulates privately;
//    the combiner merges. No shared state at all.
List<RiskScore> scores = orders.parallelStream()
        .map(riskModel::score)
        .toList();

// 2. When you truly need a shared concurrent container:
List<RiskScore> scores = orders.parallelStream()
        .map(riskModel::score)
        .collect(Collectors.toCollection(
                () -> Collections.synchronizedList(new ArrayList<>())));
//    ...but note this serialises every add, which often eats the whole benefit.

// 3. NEVER: synchronized block around the add. Same cost as 2, more code, easier to
//    get wrong, and it makes the parallelism pointless.
```

Option 1 is right essentially always. **The rule: if a parallel stream's lambda mutates
anything outside itself, the pipeline is wrong.** Not "risky" — wrong. Rewrite it as a
collect.

> `Collectors.toList()` is safe under parallelism precisely because of Topic 24's
> mechanics: each split gets its own `ArrayList` from the supplier, thread-confined,
> and the combiner merges. Thread safety comes from confinement, not from locking.

---

### Trap 4 — blocking I/O inside a parallel stream

**Wrong:**
```java
List<Enriched> enriched = orders.parallelStream()
        .map(o -> new Enriched(o, paymentClient.statusOf(o.paymentId())))  // 60 ms HTTP
        .toList();
```

**Exact symptom:** this is the Failure drill below, so here is just the observable
shape.

- This job is barely faster than sequential — with `cores − 1` workers, you get at most
  `cores` concurrent HTTP calls, not the hundreds you would get from an async client.
- **Unrelated endpoints in the same JVM get slower.** A different endpoint's parallel
  stream, or a `CompletableFuture.supplyAsync` with no explicit executor, sits in the
  common pool's queue behind your blocked workers. Its p99 collapses.
- A thread dump (`jcmd <pid> Thread.print`) shows every
  `ForkJoinPool.commonPool-worker-N` parked in socket read.
- `ForkJoinPool.commonPool().getQueuedSubmissionCount()` climbs and does not fall.

**Root cause:** a blocked worker cannot steal. `ForkJoinPool` is designed for
CPU-bound, non-blocking, recursively-splitting tasks. Its sizing assumption is "each
worker is always runnable". Blocking violates the assumption, and the pool has no
mechanism to compensate — unless you use `ManagedBlocker`, which almost nobody does.

**Fix — in order of preference:**

1. **Remove the I/O from the loop.** A bulk endpoint, a join, a batch fetch. This is
   almost always the real fix, and it is usually a 10× improvement rather than a 4×
   one. See Example 2.
2. **Use a mechanism designed for concurrent I/O.** A bounded `ExecutorService` sized
   for your I/O concurrency (Topic 90), `CompletableFuture` with an **explicit**
   executor (Topic 91), virtual threads (Topic 101), or a reactive client (Topic 106).
   All of these are correct; a parallel stream is not.
3. **A dedicated `ForkJoinPool`,** if you insist on the stream shape. See the Failure
   drill — and read the honesty note attached to it.

---

### Trap 5 — a source that will not split

**Wrong:**
```java
List<Order> orders = new LinkedList<>(repository.findAll());   // 2,000,000
long total = orders.parallelStream().mapToLong(Order::totalPence).sum();
```

or:
```java
Stream.iterate(1L, i -> i + 1).limit(10_000_000).parallel().sum();
```

**Exact symptom:** the parallel version is **slower than the sequential one**, sometimes
by a large factor, with correct results. CPU usage across cores is uneven — one core
pinned, others intermittently busy. A profiler shows significant time in
`Spliterators$IteratorSpliterator.trySplit` and in array copying, not in your logic.

**Root cause:**
- `LinkedList` has no random access. `trySplit` falls back to buffering elements into
  arrays in growing batches, so you pay a full pointer-chasing traversal plus allocation
  and copying before doing any work. Chunks are unbalanced, so workers finish at
  different times and steal, adding coordination.
- `Stream.iterate` is inherently sequential: element `n+1` is a function of element `n`.
  It reports `UNKNOWN_SIZE` and cannot meaningfully split at all. The `limit` after it
  does not help.

**Fix:**
```java
// Use a source that splits.
List<Order> orders = new ArrayList<>(repository.findAll());
long total = orders.parallelStream().mapToLong(Order::totalPence).sum();

// For ranges, use the primitive range stream — SIZED, SUBSIZED, splits perfectly.
long sum = LongStream.rangeClosed(1, 10_000_000).parallel().sum();

// If the data is genuinely a LinkedList and you cannot change that, either copy to an
// array first (pay one traversal, then split perfectly) or stay sequential.
```

**How to check whether a source splits well** — query the `Spliterator`:
```java
Spliterator<Order> s = orders.spliterator();
System.out.println("estimateSize   = " + s.estimateSize());
System.out.println("SIZED          = " + s.hasCharacteristics(Spliterator.SIZED));
System.out.println("SUBSIZED       = " + s.hasCharacteristics(Spliterator.SUBSIZED));
System.out.println("ORDERED        = " + s.hasCharacteristics(Spliterator.ORDERED));
```
`SIZED` + `SUBSIZED` with an exact `estimateSize` is the signature of a source that
splits perfectly. `Long.MAX_VALUE` as the estimate means "unknown", which means bad
splitting.

---

## Hands-on proof

Commands you run. I have no JVM and will not print invented numbers. Everything below
is what to type, what to look for, and how to read each possible result.

### Setup

```bash
mkdir -p ~/java-lab/25 && cd ~/java-lab/25
java --version
nproc 2>/dev/null || sysctl -n hw.ncpu
```

### Proof 1 — reveal the pool and its size

Use `WhoRunsThis.java` from Example 1.

```bash
java WhoRunsThis.java
```

**What to look for:** the three numbers at the top, and the thread names in each
section.

| What you see | What it means |
|---|---|
| `commonPool parallelism` = `availableProcessors` − 1 | The documented sizing, confirmed on your machine. |
| Sequential section lists only `main` | Expected. No pool involvement at all. |
| Parallel section lists `main` **and** `ForkJoinPool.commonPool-worker-1..N` | The calling thread participates alongside the workers. This is why total concurrency is `availableProcessors()`, not `availableProcessors() − 1`. |
| Parallel section lists **only** `main` | Parallelism is 0 (one CPU) or the input was too small to split. Check the parallelism number you printed. |
| `poolSize` is 0 | Workers are created lazily. Print it again after the parallel section. |

### Proof 2 — change the parallelism and watch the thread set change

```bash
java -Djava.util.concurrent.ForkJoinPool.common.parallelism=1 WhoRunsThis.java
java -Djava.util.concurrent.ForkJoinPool.common.parallelism=3 WhoRunsThis.java
java -Djava.util.concurrent.ForkJoinPool.common.parallelism=0 WhoRunsThis.java
```

**What to look for:** the number of distinct thread names in the parallel section.

| What you see | What it means |
|---|---|
| `=3` gives up to 3 worker names plus `main` | The flag works and the pool respects it. |
| `=0` gives only `main` | **This is the 1-CPU container behaviour, reproduced on your laptop.** `.parallel()` compiled, ran, and did nothing. Remember this output. |
| `=1` gives one worker plus `main` | Two-way concurrency, which is what a 2-CPU container gets. |

### Proof 3 — simulate the container

If you have Docker:

```bash
cat > Dockerfile <<'EOF'
FROM eclipse-temurin:21
COPY WhoRunsThis.java /app/WhoRunsThis.java
WORKDIR /app
CMD ["java", "WhoRunsThis.java"]
EOF

docker build -t fjp-probe .
docker run --rm --cpus=8   fjp-probe
docker run --rm --cpus=2   fjp-probe
docker run --rm --cpus=1   fjp-probe
docker run --rm --cpus=0.5 fjp-probe
```

**What to look for:** `availableProcessors` and `commonPool parallelism` at each CPU
limit.

| What you see | What it means |
|---|---|
| `--cpus=1` → processors 1, parallelism 0, only `main` in the parallel section | Trap 1, reproduced exactly. This is what your Kubernetes deployment is doing right now if `limits.cpu` is at or below 1. |
| `--cpus=0.5` → processors 1, parallelism 0 | Fractional CPUs round up to 1 processor. Same outcome. |
| `--cpus=8` → processors 8, parallelism 7 | Container awareness working as documented. |
| `--cpus=1` → processors equal to the host's core count | Container support is off. Check for `-XX:-UseContainerSupport` in `JAVA_TOOL_OPTIONS`, or a very old JVM. This is Topic 82's drill. |

Then check your own cluster:
```bash
kubectl get deploy <name> -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

### Proof 4 — the combiner and the split boundaries, made visible

`SplitVisibility.java`:
```java
import java.util.*;
import java.util.stream.*;

public class SplitVisibility {
    public static void main(String[] args) {
        List<Integer> data = IntStream.range(0, 1_000).boxed().toList();

        Collector<Integer, ?, List<Integer>> noisy = Collector.of(
                () -> { System.out.println("supplier  " + Thread.currentThread().getName());
                        return new ArrayList<Integer>(); },
                List::add,
                (a, b) -> { System.out.println("COMBINER  " + Thread.currentThread().getName()
                                    + "  " + a.size() + " + " + b.size());
                            a.addAll(b); return a; });

        System.out.println("--- sequential ---");
        System.out.println("size " + data.stream().collect(noisy).size());

        System.out.println("--- parallel ---");
        System.out.println("size " + data.parallelStream().collect(noisy).size());
    }
}
```

```bash
java SplitVisibility.java
java -Djava.util.concurrent.ForkJoinPool.common.parallelism=4 SplitVisibility.java
```

**What to look for:** the count of `supplier` lines, the count and shape of `COMBINER`
lines, and the sizes being merged.

| What you see | What it means |
|---|---|
| Sequential: one `supplier`, zero `COMBINER` | The combiner is dead code sequentially. Topic 24's central claim, confirmed. |
| Parallel: several `supplier` lines, `COMBINER` lines showing sizes like `125 + 125` | The source split into equal chunks (it is a `List` from `toList()`, so `SIZED`+`SUBSIZED`). Equal sizes are the signature of a good source. |
| `COMBINER` lines with wildly unequal sizes | Poor splitting. If you swap the source for a `LinkedList`, you should expect this. Try it. |
| Different thread names on `supplier` and `COMBINER` for the same chunk | Normal. Work stealing moved the task. |

### Proof 5 — a source that splits badly

`SplitQuality.java`:
```java
import java.util.*;
import java.util.stream.*;

public class SplitQuality {
    static void report(String name, Spliterator<?> s) {
        System.out.printf("%-14s estimate=%-12s SIZED=%-6s SUBSIZED=%-6s ORDERED=%s%n",
                name,
                s.estimateSize() == Long.MAX_VALUE ? "UNKNOWN" : s.estimateSize(),
                s.hasCharacteristics(Spliterator.SIZED),
                s.hasCharacteristics(Spliterator.SUBSIZED),
                s.hasCharacteristics(Spliterator.ORDERED));
    }

    public static void main(String[] args) {
        List<Integer> arrayList  = new ArrayList<>(IntStream.range(0, 100_000).boxed().toList());
        List<Integer> linkedList = new LinkedList<>(arrayList);
        Set<Integer>  hashSet    = new HashSet<>(arrayList);

        report("ArrayList",   arrayList.spliterator());
        report("LinkedList",  linkedList.spliterator());
        report("HashSet",     hashSet.spliterator());
        report("IntStream",   IntStream.range(0, 100_000).spliterator());
        report("iterate",     Stream.iterate(1, i -> i + 1).spliterator());
        report("generate",    Stream.generate(Math::random).spliterator());
    }
}
```

```bash
java SplitQuality.java
```

**What to look for:** which sources report an exact size and both `SIZED` and
`SUBSIZED`.

| What you see | What it means |
|---|---|
| `ArrayList` and `IntStream`: exact estimate, `SIZED=true`, `SUBSIZED=true` | Splits perfectly. Safe candidates for `.parallel()`. |
| `LinkedList`: exact estimate, `SIZED=true`, `SUBSIZED=false` | It knows its size but sub-splits do not carry exact sizes. Splitting requires traversal. Poor candidate. |
| `iterate`/`generate`: `UNKNOWN`, `SIZED=false` | Cannot split meaningfully at all. Never worth parallelising. |
| `HashSet`: exact estimate, `SIZED=true` | Splits the bucket table. Acceptable, less even than an array. |

**How to read it:** this three-line check is the fastest possible answer to "should I
parallelise this source?", and it requires no benchmarking.

---

## Failure drill

> Master plan, Topic 25: *"Block inside `ForkJoinPool.commonPool()`. Capture: unrelated
> parallel-stream latency. The fix proves: the common pool is JVM-global shared state."*

The point of this drill is not that your blocking code is slow. It is that **code you
did not write, in a different part of the JVM, gets slower**.

### Setup

`PoolStarvation.java`:
```java
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.*;

public class PoolStarvation {

    /** The innocent bystander: a pure-CPU parallel reduction. */
    static long innocentWork() {
        return IntStream.range(0, 2_000_000)
                        .parallel()
                        .mapToLong(i -> (long) Math.sqrt(i) + (i % 7))
                        .sum();
    }

    static void measureInnocent(String label) {
        long worst = 0;
        for (int i = 0; i < 10; i++) {
            Instant t0 = Instant.now();
            innocentWork();
            long ms = Duration.between(t0, Instant.now()).toMillis();
            worst = Math.max(worst, ms);
            System.out.printf("  %s run %2d : %5d ms%n", label, i, ms);
        }
        System.out.printf("  %s WORST  : %5d ms%n", label, worst);
    }

    public static void main(String[] args) throws Exception {
        System.out.println("cpus=" + Runtime.getRuntime().availableProcessors()
                + " parallelism=" + ForkJoinPool.getCommonPoolParallelism());

        System.out.println("\n=== PHASE 1: baseline, pool is idle ===");
        measureInnocent("baseline");

        System.out.println("\n=== PHASE 2: flood the common pool with blocking tasks ===");
        int blockers = ForkJoinPool.getCommonPoolParallelism() + 2;
        System.out.println("  submitting " + blockers + " tasks that each sleep 30s");
        List<ForkJoinTask<?>> tasks = new ArrayList<>();
        for (int i = 0; i < blockers; i++) {
            tasks.add(ForkJoinPool.commonPool().submit(() -> {
                try { Thread.sleep(30_000); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }));
        }
        Thread.sleep(1_000);   // let them all get scheduled and park

        System.out.println("  queued submissions = "
                + ForkJoinPool.commonPool().getQueuedSubmissionCount());
        System.out.println("  active threads     = "
                + ForkJoinPool.commonPool().getActiveThreadCount());
        System.out.println("  running threads    = "
                + ForkJoinPool.commonPool().getRunningThreadCount());

        measureInnocent("starved ");

        System.out.println("\n=== PHASE 3: dedicated pool for the innocent work ===");
        ForkJoinPool dedicated = new ForkJoinPool(
                Math.max(2, Runtime.getRuntime().availableProcessors()));
        try {
            long worst = 0;
            for (int i = 0; i < 10; i++) {
                Instant t0 = Instant.now();
                dedicated.submit(PoolStarvation::innocentWork).get();
                long ms = Duration.between(t0, Instant.now()).toMillis();
                worst = Math.max(worst, ms);
                System.out.printf("  dedicated run %2d : %5d ms%n", i, ms);
            }
            System.out.printf("  dedicated WORST  : %5d ms%n", worst);
        } finally {
            dedicated.shutdown();
            dedicated.awaitTermination(10, TimeUnit.SECONDS);
        }

        tasks.forEach(t -> t.cancel(true));
        System.out.println("\ndone");
    }
}
```

### Run it

```bash
javac PoolStarvation.java

# Force a small pool so the effect is unmistakable even on a big machine.
java -Djava.util.concurrent.ForkJoinPool.common.parallelism=3 PoolStarvation
```

While Phase 2 is running, in a second terminal:

```bash
PID=$(jcmd -l | grep PoolStarvation | cut -d' ' -f1)
jcmd $PID Thread.print | grep -A 6 "commonPool-worker"
jcmd $PID Thread.print > starved-threads.txt
```

### What to capture

1. The `baseline WORST` figure from Phase 1.
2. The `starved WORST` figure from Phase 2.
3. `getActiveThreadCount()` and `getRunningThreadCount()` during Phase 2.
4. The thread dump section showing every `commonPool-worker-N`.
5. The `dedicated WORST` figure from Phase 3.

### How to read it

| What you see | What it means |
|---|---|
| `starved WORST` is many times `baseline WORST` | **The drill's point, demonstrated.** The innocent computation did not change. Only unrelated code elsewhere in the JVM changed. Its latency collapsed anyway. |
| Thread dump: every `commonPool-worker-N` in `TIMED_WAITING` at `java.lang.Thread.sleep` | The mechanism. Those workers are alive, occupying pool slots, and unable to steal. |
| `getRunningThreadCount()` near 0 while `getActiveThreadCount()` is high | Active means "has a task"; running means "not blocked". A large gap between them is the diagnostic signature of pool starvation. Learn this pair. |
| `starved WORST` ≈ `baseline WORST` | The pool spawned compensation threads (`ForkJoinPool` does this in some blocking situations, up to a large internal cap) or your parallelism was high enough to absorb it. Raise the blocker count to `parallelism * 3` and retry. Note honestly what you observed. |
| `dedicated WORST` ≈ `baseline WORST` even while the blockers are still parked | The fix works. The dedicated pool has its own workers and is unaffected by the common pool's state. |

### The fix, and an honest warning about it

```java
ForkJoinPool dedicated = new ForkJoinPool(8);
try {
    List<Result> results = dedicated
            .submit(() -> orders.parallelStream().map(this::score).toList())
            .get();
} finally {
    dedicated.shutdown();
}
```

**Why this works:** when a parallel stream's terminal operation is invoked from a thread
that is already a `ForkJoinPool` worker, the stream's tasks are forked into *that*
worker's pool rather than being submitted to the common pool.

**Now the honesty, because this matters more than the trick:**

1. **This is not a documented, supported API.** Nowhere does the `Stream` javadoc say
   "the pool of the calling `ForkJoinWorkerThread` is used". It is an implementation
   behaviour that has held across JDK versions and is widely relied upon — but it is not
   a contract, and a future JDK is not obliged to preserve it. Do not build something
   load-bearing on it without a comment saying exactly this.
2. **You must shut the pool down.** A `ForkJoinPool` per request is Topic 98's thread
   leak: `unable to create new native thread`. Create it once, as a managed singleton or
   a Spring bean with `@PreDestroy`.
3. **It does not make blocking acceptable.** It contains the damage to your own pool.
   The workers still block, still cannot steal, and you still get at most `n` concurrent
   I/O operations. If the work is I/O-bound, the right answers are a bounded
   `ExecutorService` (Topic 90), `CompletableFuture` with an explicit executor
   (Topic 91), virtual threads (Topic 101), or a reactive client (Topic 106).
4. **The real fix is usually upstream.** In Example 2, the dedicated pool would have
   been a workaround for an N+1. The bulk endpoint was the fix.

`ForkJoinPool.ManagedBlocker` is the JDK's sanctioned mechanism for blocking inside a
FJP: it tells the pool "I am about to block, spawn a compensation thread". It exists, it
works, and it is genuinely fiddly. Topic 100 covers it. It is not a licence to do I/O in
parallel streams.

### Extension: prove it crosses code boundaries

Add a completely unrelated piece of work that touches the common pool implicitly:

```java
CompletableFuture.supplyAsync(() -> expensiveComputation()).join();   // no executor!
```

Run it during Phase 2 and time it. `supplyAsync` without an explicit executor uses
`ForkJoinPool.commonPool()`. Its latency collapses too, and nothing in its source
mentions streams, parallelism, or pools. That is the point of the drill: **the blast
radius is not your file.** Topic 91's drill is this same fact from the other direction.

---

## Measurement

### The naive benchmark is wrong. Here is exactly how.

```java
// DO NOT DRAW CONCLUSIONS FROM THIS
long t0 = System.nanoTime();
long result = orders.stream().mapToLong(Order::totalPence).sum();
long elapsed = System.nanoTime() - t0;
System.out.println(elapsed);
```

Four independent things are wrong with it, and they do not cancel out:

**1. Dead-code elimination.** If `result` is not used in a way the JIT considers
observable, C2 can prove the computation has no effect and delete it. You measure an
empty loop. Numbers in the single-digit-nanoseconds range for real work are the tell.

**2. Constant folding.** If the input is a compile-time-visible constant or an
effectively-final field the JIT can prove immutable, the result may be computed once and
inlined. Your loop then measures a constant load.

**3. On-stack replacement (OSR).** A long-running loop in a method that has not been
called enough times to trigger normal compilation gets compiled *mid-loop* by OSR. OSR
code is compiled with less profile information and is often measurably worse than the
same loop compiled normally. So you measure a compilation mode that will never run in
production.

**4. Cold JIT / mixed compilation state.** The first thousands of iterations run
interpreted, then C1, then C2. Averaging across that gives a number that describes
neither. For parallel streams specifically there is a fifth: **the common pool creates
its worker threads lazily**, so the first parallel run pays thread-creation cost that no
subsequent run pays.

**5. And one that is unique to this topic:** whatever else is running on the machine —
your IDE, a build, Docker — is competing for the same cores. Sequential code is far less
sensitive to that than parallel code is, so a noisy machine systematically biases the
comparison *against* parallel.

Topic 77 is the full treatment. Here is the harness.

### A correct JMH harness sketch

`pom.xml` dependencies (JMH 1.37 at time of writing — check for current):
```xml
<dependency>
  <groupId>org.openjdk.jmh</groupId><artifactId>jmh-core</artifactId>
  <version>1.37</version>
</dependency>
<dependency>
  <groupId>org.openjdk.jmh</groupId><artifactId>jmh-generator-annprocess</artifactId>
  <version>1.37</version><scope>provided</scope>
</dependency>
```

`ParallelStreamBenchmark.java`:
```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.util.*;
import java.util.concurrent.TimeUnit;
import java.util.stream.*;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Warmup(iterations = 5, time = 2)          // let the JIT reach steady state
@Measurement(iterations = 10, time = 2)    // then measure
@Fork(3)                                   // 3 fresh JVMs — defeats profile pollution
@State(Scope.Benchmark)                    // shared setup, not re-created per invocation
public class ParallelStreamBenchmark {

    @Param({"1000", "100000", "10000000"})   // vary N
    private int size;

    @Param({"1", "50", "2000"})              // vary Q (work per element)
    private int workPerElement;

    private List<Long> data;                 // ArrayList — splits well
    private long[] primitives;

    @Setup(Level.Trial)
    public void setUp() {
        Random r = new Random(42);            // fixed seed: reproducible
        data = new ArrayList<>(size);
        primitives = new long[size];
        for (int i = 0; i < size; i++) {
            long v = r.nextInt(100_000);
            data.add(v);
            primitives[i] = v;
        }
    }

    /** Simulated per-element CPU cost. Blackhole.consumeCPU is JMH's calibrated burner. */
    private long score(long value) {
        Blackhole.consumeCPU(workPerElement);
        return value * 31 + 7;
    }

    @Benchmark
    public void sequential(Blackhole bh) {
        bh.consume(data.stream().mapToLong(this::score).sum());
    }

    @Benchmark
    public void parallel(Blackhole bh) {
        bh.consume(data.parallelStream().mapToLong(this::score).sum());
    }

    @Benchmark
    public void plainLoop(Blackhole bh) {          // the baseline you must include
        long total = 0;
        for (int i = 0; i < primitives.length; i++) {
            total += score(primitives[i]);
        }
        bh.consume(total);
    }
}
```

Every annotation is load-bearing:

| Element | What it defeats |
|---|---|
| `Blackhole` / `bh.consume(...)` | Dead-code elimination. The result provably escapes. |
| `@State(Scope.Benchmark)` + `@Setup` | Constant folding. The data is a field the JIT cannot fold, built at runtime. |
| `@Warmup(iterations = 5)` | Cold JIT and lazy common-pool thread creation. |
| `@Fork(3)` | Profile pollution and per-JVM luck. Three separate JVMs; JMH reports the spread. |
| `@Measurement(iterations = 10)` | Single-sample noise. You get a distribution. |
| `Blackhole.consumeCPU(n)` | Lets you vary `Q` independently of `N`, which is the whole experiment. |
| `plainLoop` baseline | Answers "is the stream itself the cost?" — often the real finding. |
| Fixed `Random` seed | Run-to-run input variation. |

Run it:
```bash
mvn clean package
java -jar target/benchmarks.jar ParallelStreamBenchmark -rf json -rff results.json

# then vary the pool and re-run
java -jar target/benchmarks.jar ParallelStreamBenchmark \
     -jvmArgs "-Djava.util.concurrent.ForkJoinPool.common.parallelism=1"
java -jar target/benchmarks.jar ParallelStreamBenchmark \
     -jvmArgs "-Djava.util.concurrent.ForkJoinPool.common.parallelism=8"

# add allocation profiling
java -jar target/benchmarks.jar ParallelStreamBenchmark -prof gc
```

### Reading the output

JMH prints score, error (99.9% confidence interval), and units per parameter
combination.

| What you see | What it means |
|---|---|
| Parallel beats sequential only at large `size × workPerElement` | The `N × Q` model, measured on your hardware rather than quoted from a blog post. Find your own crossover point and write it down. |
| Parallel loses at `size=1000` regardless of `workPerElement` | Fixed splitting and scheduling overhead dominates. Expected. |
| Error bars overlap between sequential and parallel | **There is no measured difference.** Report it that way. "Overlapping confidence intervals" is a complete and correct finding. |
| `plainLoop` beats both | Also a complete finding, and a common one for cheap `Q`. It means the stream machinery, not the parallelism, was the cost. |
| Wildly different scores across the 3 forks | Profile pollution (Topic 74), or an unstable machine. Increase forks, close everything else, disable turbo if you can. |
| `-prof gc` shows high `gc.alloc.rate.norm` on the boxed variant | Topic 01's boxing, quantified per operation rather than guessed at. |

**Two rules for reporting results:**
1. Never quote a parallel speedup without stating the core count, the container CPU
   limit, and the source type.
2. Never quote a speedup from a single fork.

---

## Practice exercises

### 1 — Easy: find your own crossover

**Part A.** Print `availableProcessors()` and `getCommonPoolParallelism()` on your
machine, then inside `--cpus=1`, `--cpus=2` and `--cpus=4` containers. Tabulate.

**Part B.** Using `SplitQuality.java` from Proof 5, add `ArrayDeque`, `TreeSet`,
`Arrays.asList(...)`, `String.chars()`, and `Files.lines(path)`. Rank all of them from
best-splitting to worst, and give the reason for each ranking — a structural reason, not
"it's a linked structure".

**Part C.** Write down, before measuring anything, your prediction for the smallest `N`
at which `.parallel()` wins for (i) summing an `int[]`, and (ii) computing a SHA-256 of
each element. You will check these in Exercise 3.

### 2 — Medium: the audit (combines Topics 01, 11, 13, 17, 21, 23, 24)

Find the **seven** defects. For each: the exact observable symptom (an exception with
its message, a wrong value, a latency change, or "nothing at all"), the root cause, and
the fix.

```java
public class SettlementReport {

    private final Map<CustomerId, Long> totals = new HashMap<>();
    private int processed = 0;

    public Report build(LinkedList<Payment> payments, BigDecimal openingBalance) {

        payments.parallelStream().forEach(p -> {
            totals.merge(p.customerId(), p.amountPence(), Long::sum);
            processed++;
        });

        BigDecimal net = payments.parallelStream()
                .map(Payment::amount)
                .reduce(openingBalance, BigDecimal::subtract);

        List<String> refs = new ArrayList<>();
        payments.parallelStream()
                .filter(p -> p.status() == SETTLED)
                .forEach(p -> refs.add(p.reference()));

        Map<Status, List<Payment>> byStatus = payments.parallelStream()
                .collect(Collectors.groupingBy(Payment::status));

        Optional<Payment> firstDisputed = payments.parallelStream()
                .filter(p -> p.status() == DISPUTED)
                .findAny();

        return new Report(totals, net, refs, byStatus, firstDisputed.orElse(null), processed);
    }
}
```

Hints, one per defect: shared `HashMap` across threads; `processed++` is not atomic;
`LinkedList` as a parallel source; `BigDecimal::subtract` and associativity, plus the
identity applied per split; `refs.add` from many threads; `groupingBy` versus
`groupingByConcurrent` and what map you got; `findAny` versus `findFirst` when the
report must be deterministic.

### 3 — Hard: production simulation on `orderflow`

**Part A — the drill.** Run the Failure drill exactly as written. Capture all five
artefacts listed under "What to capture". Write a three-paragraph incident-report-style
summary: what an on-call engineer would have seen first (a symptom, not a cause), what
they would have looked at second, and what the fix was.

**Part B — the crossover, measured properly.** Build the JMH harness from the
Measurement section. Sweep `size` ∈ {1e3, 1e4, 1e5, 1e6, 1e7} × `workPerElement` ∈
{1, 10, 100, 1000, 10000}. Plot the parallel/sequential ratio and find where it crosses
1.0. Compare against your Exercise 1 Part C predictions and explain any gap.

**Part C — the source matters.** Repeat one row of the sweep with the data in a
`LinkedList` instead of an `ArrayList`. Report the difference and explain it in terms
of Proof 5's `Spliterator` characteristics, not in terms of Big-O.

**Part D — the container.** Re-run the best-case row inside `--cpus=1`, `--cpus=2` and
`--cpus=4`. State the CPU limit at which `.parallel()` stops being worth the code it is
written in.

**Part E — the correctness test.** Write a `@RepeatedTest(100)` that asserts your
scoring pipeline gives identical results parallel and sequentially over 200,000 random
elements. Then deliberately introduce each of Trap 2's three violations in turn
(non-associative accumulator, non-identity identity, mismatched combiner) and record
which the test catches, how often, and how many repetitions it took.

**Part F — the judgement call.** You have the numbers. Write the PR description you
would attach to a change that adds `.parallel()` to `orderflow`'s risk-scoring job.
It must state: the measured speedup with error bars, the core count and container limit
it was measured at, the three preconditions and evidence each holds, the pool the work
runs on and why, and the specific condition under which you would revert it. If your
numbers do not support the change, write the PR description that says so — that is the
better outcome and the harder thing to write.

---

## Interview questions

### Q1 — "When would you use a parallel stream?"

**Mid-level answer:** "When you have a lot of data and want it to go faster. You just
add `.parallel()`."

**Senior answer:** "Rarely, and only with three preconditions met. First, a source that
splits well — `ArrayList`, arrays, `IntStream.range` split in O(1) into balanced halves;
`LinkedList` has to be traversed and `Stream.iterate` can't split at all. Second,
enough `N × Q` — number of elements times per-element CPU cost — to pay back the fixed
splitting and merging overhead. The product is what matters: ten million cheap
operations often loses because memory bandwidth is the constraint, while a few hundred
expensive ones can win. Third, no shared mutable state and an associative reducer,
because the framework decides the grouping and it varies run to run. Beyond that: it
runs on the common `ForkJoinPool`, which is a JVM-wide singleton sized `cores − 1`, so
inside a request-handling thread I'm competing with everything else in the process. And
I'd measure with JMH — three forks, warmup, `Blackhole` — before and after, because a
`nanoTime` loop measures dead-code elimination as much as it measures my code."

**What separates them:** the three named preconditions with mechanisms, the `N × Q`
product rather than "lots of data", awareness of the shared pool, and refusing to claim
a speedup without a measurement methodology.

**Interviewer's follow-up:** "What's `availableProcessors()` in your production
container?" This is the real question. A senior answer knows it is probably 1 or 2, and
therefore that `.parallel()` is doing nothing.

---

### Q2 — "Why is a blocking call inside a parallel stream a problem for the whole JVM?"

**Mid-level answer:** "It ties up threads, so the parallel stream is slow."

**Senior answer:** "Because the pool is shared and blocked workers can't steal.
`ForkJoinPool` gives each worker its own deque — it pushes and pops its own tasks LIFO
for cache locality, and steals from the tail of another worker's deque FIFO to grab the
biggest remaining chunk. The whole design assumes every worker is always runnable. A
worker parked on a socket read is occupying one of `cores − 1` slots and can't
participate. Since it's the *common* pool, the victims aren't just my code: any other
parallel stream, any `CompletableFuture.supplyAsync` without an explicit executor, and
`Arrays.parallelSort` anywhere in the process all queue behind my blocked workers. You
can see it in a thread dump — every `commonPool-worker-N` in `TIMED_WAITING` — and in
the gap between `getActiveThreadCount()` and `getRunningThreadCount()`. The fix isn't a
bigger pool; it's not doing blocking I/O there. Bulk the calls, or use a bounded
executor, virtual threads, or a reactive client. If I genuinely need the stream shape I
can submit it to a dedicated `ForkJoinPool` and the stream will use that pool — but
that's an undocumented implementation behaviour, not a supported API, and it contains
the damage rather than fixing it."

**What separates them:** the work-stealing mechanism explaining *why* blocking is
uniquely bad here, naming the unrelated victims, giving two concrete diagnostics, and
flagging the dedicated-pool trick as unsupported rather than presenting it as the
answer.

**Interviewer's follow-up:** "What about `ManagedBlocker`?" They want: it exists, it
tells the pool to spawn a compensation thread, it works, and it is fiddly enough that
restructuring the I/O is nearly always better.

---

### Q3 — "This reduce gives different answers each run under `.parallel()`. Why?"

```java
movements.parallelStream().reduce(openingBalance, (a, m) -> a - m.amount(), Long::sum);
```

**Mid-level answer:** "There's a race condition — the threads are interfering with each
other."

**Senior answer:** "No race — `reduce` has no shared state. There are three separate
contract violations. The accumulator isn't associative: `(a − b) − c ≠ a − (b − c)`, and
the framework groups elements by however the splits landed, which varies with input size
and core count. The identity isn't an identity: `reduce` requires
`accumulator(identity, x) == x`, and `openingBalance − x ≠ x` — worse, under parallelism
the identity is applied once *per split*, so the opening balance is subtracted N times
instead of once. And the combiner disagrees with the accumulator: the accumulator
subtracts, the combiner adds, and the contract requires
`combiner(a, accumulator(identity, b)) == accumulator(a, b)`. All three are invisible
sequentially, because sequentially the identity is applied once and the combiner is
never called at all. The fix is to pull the non-associative part out: sum the movements
with an associative reduction, then subtract from the opening balance once at the end.
And I'd add a repeated parallel-versus-sequential equivalence test over a large random
input, because that's the only kind of test that catches this class of bug."

**What separates them:** rejecting "race condition" — the reflex wrong answer — and
naming all three violations, particularly the per-split identity, which almost nobody
mentions.

**Interviewer's follow-up:** "Which common operations are not associative?" Subtraction,
division, "take the first", naive running averages, and any string concatenation whose
order matters.

---

### Q4 — "You added `.parallel()` and nothing changed. Debug it."

**Mid-level answer:** "Maybe the dataset was too small, or the JIT warmed up."

**Senior answer:** "I'd check four things in order, cheapest first. One:
`Runtime.getRuntime().availableProcessors()` and
`ForkJoinPool.getCommonPoolParallelism()`. Since JDK 10 `availableProcessors` honours
the cgroup CPU quota, so a container with `limits.cpu: 1` — or anything below 1 —
reports one processor, which makes common-pool parallelism zero. There are then no
workers at all and everything runs on the calling thread. `.parallel()` compiles, runs,
and silently does nothing. That's the most likely answer and a great many Kubernetes
services are in exactly that state. Two: the source's `Spliterator` — an exact
`estimateSize` with `SIZED` and `SUBSIZED` means it splits well; `Long.MAX_VALUE` and
`SIZED=false` means it can't split meaningfully. Three: `N × Q` — is there actually
enough work? Four: is the work I/O-bound, in which case more threads on the same cores
buys nothing. And before any of that I'd ask how the 'nothing changed' was measured,
because a `nanoTime` loop can't distinguish 'no improvement' from 'the JIT deleted both
versions'."

**What separates them:** the container/cgroup answer first, an ordered cheapest-first
debugging procedure, and questioning the measurement before accepting the premise.

**Interviewer's follow-up:** "How would you make sure this can't happen silently
again?" Log `availableProcessors` and common-pool parallelism at startup; add it to the
service's readiness output or a `/info` endpoint.

---

### Q5 — "Should `orderflow`'s order-listing endpoint use a parallel stream?"

**Mid-level answer:** "It could speed up the response if there are a lot of orders."

**Senior answer:** "Almost certainly not, for three reasons that compound. First,
sizing: a request handler is already one of many concurrent requests, and they'd all be
contending for the same `cores − 1` common-pool workers. Parallelising per request
doesn't add capacity — the server was already saturating its cores with concurrent
requests. It just moves contention from the thread pool to the FJ pool and adds
splitting overhead. Second, `N × Q`: a page of orders is tens to hundreds of elements
doing trivial mapping. That's far below any plausible crossover; it'll be slower.
Third, blast radius: if anything in that pipeline blocks — a lazy Hibernate association
triggering a query, which is exactly the shape of Topic 49's trap — I've put JDBC on
the common pool and every parallel operation in the JVM degrades. Parallel streams
belong in batch and offline work where the JVM isn't otherwise busy. If the endpoint is
slow, the fix is almost always in the query — a fetch join, a projection, an index — not
in the JVM. And I'd want the p99 number before and after, not a wall-clock number from
one run."

**What separates them:** recognising that a server is *already* parallel across
requests — which is the insight most candidates miss entirely — and escalating to "fix
the query" rather than optimising the wrong layer.

**Interviewer's follow-up:** "When *would* you use one in a service?" A scheduled batch
job, on a dedicated pool, over an in-memory `ArrayList`, with a measured `N × Q`, and
with the CPU limit checked.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The common pool is sized `cores − 1` and the submitting thread participates. Why
   `− 1` rather than `cores`? What breaks if you set the parallelism to `cores × 2`?

2. `ForkJoinPool` steals FIFO from the tail but pops LIFO from its own head. Both
   choices are deliberate. Give the reason for each, then describe a workload where each
   choice is exactly wrong.

3. `.parallel()` sets a flag on the whole pipeline rather than parallelising from that
   point onward. Would a position-sensitive design have been better? What would it cost
   to implement?

4. A broken combiner is invisible sequentially. So is a non-associative reducer. Both
   are *contract* requirements the compiler cannot check. Could the language have
   checked them? What would you have to give up?

5. Node's `worker_threads` have isolated heaps; Java threads share one. Java's choice
   makes parallel streams possible at all — you can hand a worker a slice of an
   `ArrayList` with no copying. Name the specific cost Java pays for that, in terms of
   what you must now verify about every lambda.

6. In a 1-CPU container, `.parallel()` does nothing. Should the JVM warn? Should it be
   a startup error? Argue for and against, then say what you would actually implement.

7. You measure a 3.2× speedup with JMH on your 8-core laptop. Production runs on
   `limits.cpu: 2`. Write the one-sentence claim you are entitled to make in the PR
   description, and the one you are not.

---

## Quick reference card

### The mechanical facts

```
.parallel()          -> a flag on the whole pipeline; last parallel()/sequential() wins
runs on              -> ForkJoinPool.commonPool()  — ONE JVM-wide static singleton
pool size            -> Runtime.availableProcessors() - 1  (caller participates: total = cores)
1-CPU container      -> parallelism 0 -> everything on the calling thread, silently
splitting            -> Spliterator.trySplit(), recursively, targeting ~4x parallelism tasks
combiner             -> runs ONLY here; never in a sequential pipeline
terminal op          -> BLOCKS the calling thread until all splits finish
```

### The three preconditions

1. **Splittable source** — `ArrayList`, arrays, `IntStream.range`. Not `LinkedList`,
   not `iterate`/`generate`, not `Files.lines`.
2. **Sufficient `N × Q`** — heuristic starting point ~10,000; measure your own crossover.
3. **No shared mutable state; associative reducer with a true identity.**

### Diagnostics

```java
Runtime.getRuntime().availableProcessors()
ForkJoinPool.getCommonPoolParallelism()
ForkJoinPool.commonPool().getPoolSize()
ForkJoinPool.commonPool().getActiveThreadCount()    // has a task
ForkJoinPool.commonPool().getRunningThreadCount()   // not blocked  <- big gap = starvation
ForkJoinPool.commonPool().getQueuedSubmissionCount()
list.spliterator().estimateSize()                   // Long.MAX_VALUE = unknown = bad
list.spliterator().hasCharacteristics(Spliterator.SUBSIZED)
```

```bash
-Djava.util.concurrent.ForkJoinPool.common.parallelism=N
jcmd <pid> Thread.print | grep -A 6 commonPool-worker
docker run --cpus=1 ...        # reproduce the container case
```

### Ordering under parallelism

| Operation | Parallel behaviour |
|---|---|
| `forEach` | **no order guarantee** |
| `forEachOrdered` | encounter order, at a synchronisation cost |
| `findFirst` | encounter order — more expensive in parallel |
| `findAny` | any match — cheaper, and non-deterministic |
| `sorted()` | correct, but the merge phase is largely serial |
| `.unordered()` | explicitly drops ordering; can speed up `distinct`, `limit`, `skip` |

### Collectors under parallelism

| Collector | Parallel behaviour |
|---|---|
| `toList()`, `toSet()`, `toMap()` | Safe. Per-split containers, merged by the combiner. |
| `groupingBy` | Safe. Per-split maps, merged. |
| `groupingByConcurrent`, `toConcurrentMap` | `CONCURRENT` + `UNORDERED` — one shared thread-safe container, no merge. Faster for many keys; loses encounter order. |
| A custom collector | Safe **only** if your combiner is correct and associative. Test it. |

### Gotchas checklist

- [ ] `.parallel()` in a container with `cpus <= 1` does nothing, silently.
- [ ] The common pool is JVM-wide. Blocking in it degrades unrelated code.
- [ ] `CompletableFuture.supplyAsync` with no executor uses the same pool.
- [ ] A blocked worker cannot steal.
- [ ] `LinkedList`, `Stream.iterate`, `Stream.generate`, `Files.lines` split badly or not at all.
- [ ] Reducer must be associative **and** the identity must be a true identity.
- [ ] The identity is applied once **per split**, not once per pipeline.
- [ ] Any mutation of external state from a parallel lambda is a bug.
- [ ] `forEach` gives no ordering; use `forEachOrdered` if you need it.
- [ ] Never benchmark this with `System.nanoTime()`. Use JMH, ≥3 forks.
- [ ] Never quote a speedup without core count, container limit, and source type.
- [ ] A dedicated `ForkJoinPool` works but is not a documented contract — and must be shut down.

---

## When would I use this at work?

**1. A nightly batch job that is genuinely CPU-bound.**
Risk scoring, report aggregation, image or document processing over an in-memory
`ArrayList`, on a pod sized for the job. You measure with JMH, you check the container's
CPU limit, you use a dedicated pool if the JVM also serves traffic, and you write the
numbers in the PR. This is the legitimate use, and it is rarer than people think.

**2. Reviewing a PR that adds `.parallel()`.**
This is the far more common way this topic pays. You ask four questions: what is
`availableProcessors()` in production; does the source split; what is `N × Q`; and does
anything in the pipeline block. Most PRs fail at question one, and you have saved a
sprint of confused benchmarking. Add: "is there a parallel-versus-sequential equivalence
test?"

**3. Diagnosing an unexplained latency collapse.**
An endpoint that touches no database and does no I/O suddenly has a terrible p99. You
take a thread dump, see every `commonPool-worker` parked, and go looking for whoever put
blocking work on the common pool — which will be in a completely different part of the
codebase, possibly in a library. Without this topic that dump is unreadable. With it,
it is a five-minute diagnosis.

---

## Connected topics

**Prerequisites:**
- **23 — Streams I**: the pipeline, laziness, and the `Spliterator` that this topic
  finally makes central.
- **24 — Collectors**: the combiner, and why it only runs here. This topic is the answer
  to "when does the combiner run".
- **21 — Lambdas**: why captured state must be effectively final, and why "no shared
  mutable state" is a precondition rather than advice.
- **01 — Primitives and boxing**: boxing costs multiply under parallelism because
  allocation pressure is per-thread.
- **11 — List implementations**: why `LinkedList` splits terribly, in cache-line terms.
- **17 — Immutability and safe publication**: the reason immutable elements make
  parallel pipelines safe by construction.

**This unlocks:**
- **100 — `ForkJoinPool` and work stealing**: the full treatment of deques, stealing,
  `ManagedBlocker`, and why the virtual-thread scheduler is a separate FJP.
- **91 — `CompletableFuture`**: the same common pool, the same starvation drill from the
  other direction, and why you always pass an explicit executor.
- **90 — `ExecutorService` sizing**: what to use instead when the work is I/O-bound.
- **77 — JMH**: the full version of this topic's Measurement section, including why
  every naive benchmark lies.
- **82 — Containers and the JVM**: the cgroup/`availableProcessors` story in depth, and
  its drill.
- **95 — Atomics and `LongAdder`**: what a correct shared counter looks like, and why
  lock-free is not the same as scalable.
- **101 — Virtual threads**: the right answer for high-concurrency blocking I/O, and the
  pinning trap.
- **106 — Reactive clients**: the other right answer, with backpressure.
- **74 — JIT and profile pollution**: why `@Fork(3)` is not optional.

---

*Java baseline 21. Parallel streams, `ForkJoinPool.commonPool()` and its `cores − 1`
sizing have been stable since Java 8; container-aware `availableProcessors()` since JDK
10 and on by default. Nothing here changed in Java 25. Virtual threads (21) do not
change any of it — they are a concurrency answer, not a parallelism one, and the common
pool is still the common pool.*
