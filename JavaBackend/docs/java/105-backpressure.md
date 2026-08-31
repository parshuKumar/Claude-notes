# 105 — Reactor II: Backpressure, `request(n)`, and Overflow Strategies

## Phase: 10 — Reactive & Async at Scale
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21. Reactor and Netty versions come from the Spring Boot BOM — do not pin them by hand and do not quote a version number from this document.
## Project spine: `orderflow`'s payment-callback ingestion as a `Flux` with an explicit, bounded overflow strategy — and a written statement of what happens to a callback that overflows, signed off by someone who owns the money.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **`request(n)` is a signal that travels UPSTREAM, from the consumer to the producer. It
> says: "I can accept n more items." A compliant producer emits at most n and then stops
> until it is asked again.**
>
> **Every operator in a chain maintains its own demand accounting. `map` passes demand
> through unchanged. `filter` re-requests to replace what it dropped. `flatMap` translates
> outer demand into a bounded number of concurrent inner subscriptions. `buffer(k)`
> requests k upstream for every 1 requested downstream.**
>
> **Backpressure exists only along the path where that demand actually propagates. The
> moment you bridge a source that pushes on its own schedule — a Kafka listener, an HTTP
> handler, an SSE feed, a `Sink` — without translating its pushes into demand, you have an
> unbounded buffer. And an unbounded buffer is not backpressure. It is an
> `OutOfMemoryError` with a delay.**

The whole topic is one question, asked at every boundary in your system: **where does the
demand signal stop?** Wherever it stops is where your buffer is, and that buffer is either
bounded by you or unbounded by accident.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE — and this is the one thing reactive uniquely provides

This section matters more than it looks. Almost everything else in Phase 10 has a decent
Node analogue: Netty's event loops are libuv (Topic 103), `Flux` is `Observable`
(Topic 104). **This does not.** And because it does not, you will be tempted to model it
with something you already have, and every one of those models is wrong in a way that
produces an OOM.

**Promises have no consumer-demand signal. None. At all.**

```typescript
const rows = await db.query('SELECT * FROM order_lines');   // 5,000,000 rows
```

There is no point in that expression where the consumer says "give me a thousand and
wait". The promise resolves with everything, or it rejects. The producer decides how much
to produce and when; the consumer's only options are to accept it or to have already
crashed. A promise is a **single future value**, and a single value cannot be
flow-controlled — the concept does not apply.

**RxJS largely does not have it either, and this is the trap for you specifically.**

RxJS 5 removed the backpressure machinery that RxJS 4 and RxJava had experimented with.
Modern RxJS `Observable` is a **push** model: the producer calls `next()` whenever it
likes, and the subscriber's `next` handler runs. There is no `request(n)`. There is no
credit. The operators RxJS offers for this problem —

```typescript
source$.pipe(
  throttleTime(100),      // DROP items during a window
  auditTime(100),         // DROP all but the last in a window
  debounceTime(100),      // DROP until quiet
  bufferTime(100),        // ACCUMULATE into an array -- unbounded by default
  sampleTime(100),        // DROP all but the most recent
)
```

— are all **lossy or accumulating operations applied at the consumer**. Every one of them
is a decision the *consumer* makes about data the *producer* has already produced and
already handed over. The producer was never slowed down. It never learned anything. If it
is reading from a socket as fast as the kernel will give it bytes, it keeps doing that,
and `bufferTime` accumulates them in an array that has no maximum size.

**That distinction — "the producer was told" versus "the consumer coped" — is the entire
content of this topic.** RxJS gives you the second. Reactive Streams gives you the first.

### The one thing in Node that genuinely rhymes — and exactly how far it goes

There is one place Node does real flow control, and it is worth naming precisely because
your intuition from it is half right.

```typescript
const ok = writable.write(chunk);
if (!ok) {
  readable.pause();                       // stop reading
  writable.once('drain', () => readable.resume());
}
// or, which does all of the above correctly:
await pipeline(readable, transform, writable);
```

Node streams have a `highWaterMark`, `write()` returns `false` when the internal buffer is
over it, and `pipe`/`pipeline` wire the pause/resume for you. That **is** backpressure, and
if you have ever debugged a memory blow-up from ignoring `write()`'s return value, you have
the right instinct.

Now the differences, and they are not cosmetic:

| | Node stream backpressure | Reactive Streams `request(n)` |
|---|---|---|
| Signal type | **Boolean.** Stop / go. | **Numeric credit.** "Send me exactly n more." |
| Direction | Consumer pauses the source it is piped from | Demand travels upstream **through every operator**, each one translating it |
| Composition | Works along a `pipe` chain of streams | Works through `map`, `filter`, `flatMap`, `concatMap`, `groupBy`, merges, and any operator you write |
| Granularity | Whatever the chunk size happens to be | Exact item counts, accounted per operator |
| Opt-out | Not really | `request(Long.MAX_VALUE)` — and that is the most common bug (Trap 4) |
| Applies to | Byte and object streams | Any `Publisher`, including one item |
| Bound is set by | `highWaterMark` | Whatever your slowest consumer requests, plus explicit prefetch |

**The sentence to carry:** *Node backpressure is a boolean between two adjacent streams.
Reactive Streams backpressure is a numeric credit that composes through an arbitrary
operator graph. Same idea; one of them survives being put through six transformations.*

### Why Java needed a specification for this and Node did not

Because in Node there is one runtime, one stream implementation, and one event loop, so
"how do two streams flow-control each other" is an implementation detail of one library.

In Java there are many publishers (Reactor, RxJava, Akka Streams, the JDK's own
`Flow`/`SubmissionPublisher`, R2DBC drivers, Kafka clients) and they must interoperate.
**Reactive Streams is a four-interface specification precisely so a Reactor `Flux` can
consume an RxJava `Flowable` and have the demand signal survive the boundary:**

```java
public interface Publisher<T>     { void subscribe(Subscriber<? super T> s); }
public interface Subscriber<T>    { void onSubscribe(Subscription s);
                                    void onNext(T t);
                                    void onError(Throwable t);
                                    void onComplete(); }
public interface Subscription     { void request(long n); void cancel(); }
public interface Processor<T,R> extends Subscriber<T>, Publisher<R> { }
```

These are in the JDK as `java.util.concurrent.Flow.*` since Java 9. Reactor implements the
`org.reactivestreams` versions, and adapters exist both ways. **The specification is a
contract with rules** — `request` must be additive and non-blocking, `onNext` must not be
called more than the requested amount, signals must be serialised — and those rules are
what make the composition work.

**Verdict: NO TYPESCRIPT ANALOGUE for the mechanism. A partial, boolean-shaped analogue in
Node streams for the intuition. This is the one capability reactive has that neither
Promises, nor RxJS, nor virtual threads (Topic 101) provide — and it is therefore the
entire technical basis of Topic 107's recommendation.**

---

## What is this?

Backpressure is **flow control for an asynchronous stream**: a way for a slow consumer to
limit a fast producer without either blocking a thread or losing data by accident.

### The demand handshake, in full

```java
// What subscribe() actually sets in motion:
publisher.subscribe(subscriber);
   -> subscriber.onSubscribe(subscription)     // the ONLY chance to request
        -> subscription.request(32)            // "send me 32"
   <- subscriber.onNext(item)   x 32           // at most 32
        -> subscription.request(32)            // "32 more"
   <- subscriber.onNext(item)   x 32
   ...
   <- subscriber.onComplete()  OR  onError(t)  // terminal, once
```

Three rules from the specification that you should be able to state:

1. **`onNext` may be called at most as many times as have been requested.** A publisher
   that emits more is non-compliant. That is a bug in the publisher, not a load problem.
2. **`request(n)` is additive.** Outstanding demand accumulates. Reactor saturates at
   `Long.MAX_VALUE` rather than overflowing (`Operators.addCap`).
3. **`request(Long.MAX_VALUE)` means "unbounded".** It is the documented way to say "I do
   not want backpressure", and publishers take a fast path when they see it. **It is also
   what several convenient-looking APIs do on your behalf.** See Trap 4.

### The three kinds of source, and only one of them is safe by default

This taxonomy is the practical core of the topic. Every source you will ever wire up is
one of these three.

**1. A pull source — backpressure is free.**

```java
Flux.range(1, 5_000_000)
Flux.fromIterable(orderLines)
Flux.generate(...)                     // one item per request, by construction
```

The source *is* a function of demand. It produces exactly what was asked for. Nothing can
overflow because nothing is produced without a request.

**2. A pull source with an inherent buffer — backpressure works, with a prefetch.**

```java
r2dbcClient.sql("SELECT ...").fetch().all()      // driver requests rows in batches
kafkaReceiver.receive()                          // Reactor Kafka pauses partitions
```

The underlying protocol has its own windowing, so the driver translates demand into
protocol-level flow control. This is the good case and it is why R2DBC and Reactor Kafka
exist as separate things from JDBC and the plain Kafka client.

**3. A push source — backpressure DOES NOT EXIST until you create it.**

```java
Flux.create(sink -> webhookServer.onCallback(sink::next))   // arrives when it arrives
Sinks.many().multicast().onBackpressureBuffer()             // you emit; nobody asked
kafkaListener -> sink.tryEmitNext(record)                   // a bridge you wrote
sseConnection.onMessage(sink::next)
```

**The producer has its own clock.** A payment provider posts a callback when a payment
settles. It does not care what your consumer is doing and there is no protocol by which
you could tell it. **Every OOM in this document starts here.**

The question for a push source is not "does it have backpressure" — it does not — but
**"what do I do with the item I cannot handle right now?"** That is a product question with
four possible answers, and you must pick one on purpose.

### The four answers, and what each one means to the business

| Operator | Behaviour | Correct when | Wrong when |
|---|---|---|---|
| `onBackpressureBuffer(n, DROP_OLDEST)` | Keep at most n; discard the oldest to make room | The newest data is the most valuable — live prices, positions, telemetry | Every item must be processed |
| `onBackpressureBuffer(n, DROP_LATEST)` | Keep at most n; reject new arrivals | The oldest data must be processed in order and losing new work is acceptable | Rarely the right answer; usually a sign you wanted ERROR |
| `onBackpressureBuffer(n, ERROR)` | Fail the stream on overflow | You would rather fail loudly and restart than lose data silently | A brief burst should not kill the pipeline |
| `onBackpressureDrop(consumer)` | Discard, unbounded, with a callback | Sampling telemetry, metrics, presence pings | Anything financial |
| `onBackpressureLatest()` | Keep only the most recent | "Current value" semantics — a dashboard gauge, a cursor position | Event streams where each item is distinct |
| `onBackpressureBuffer()` — **no arguments** | Buffer **without limit** | **Never in a service.** | Always |

**Read the last row again.** The no-argument overload compiles, reads as if it is doing
something responsible, and is an `OutOfMemoryError` on a timer. It is Trap 1 and it is the
failure drill.

### And the one that is not on the list

For `orderflow`'s payment callbacks, none of the lossy options is acceptable — dropping a
payment callback means a customer paid and the order never shipped. The correct answer is
**not a Reactor operator at all**:

> **Persist first, process later.** Accept the callback, write it to durable storage
> (a table, a Kafka topic with sufficient retention), return `202 Accepted`, and process
> from that durable buffer at your own rate. Then the "buffer" is a disk, its bound is a
> disk-space alert, and overflow is a paging problem instead of a data-loss problem.

Knowing that the best answer to a backpressure question is sometimes "make the buffer
durable" — rather than picking a Reactor operator — is the difference between using the
API and understanding the problem. Topic 115's outbox pattern is the same insight pointed
the other way.

---

## Why does it matter?

**1. It is the only capability that survives Topic 107's argument.** Virtual threads give
you cheap concurrency and readable code, and they take most of reactive's historical
justification away. They do not give you demand propagation. When you write the Topic 107
recommendation, this is the paragraph that decides where the boundary goes, so understand
it mechanically rather than as a slogan.

**2. The failure is an OOM, and the OOM names the wrong culprit.** A heap dump from an
unbounded backpressure buffer shows an enormous queue of domain objects. The dominator
tree (Topic 79) points at a Reactor internal queue class. Nothing in the dump says "your
producer is faster than your consumer" — you have to know to read it that way.

**3. "We use reactive so it handles load" is a real sentence that real teams say.** It is
true only along the path where demand actually propagates. One `Sinks.many()` in the middle
of an otherwise perfect chain and the guarantee is gone from that point upstream. Being
able to point at the exact line where the demand signal dies is a genuinely valuable skill
in a code review.

**4. The overflow strategy is a business decision wearing an API's clothing.** Dropping
price ticks is fine. Dropping payment callbacks is a financial incident. The person who
chooses `DROP_OLDEST` in a payments pipeline has made a product decision, probably without
realising it, and probably without telling anyone. **Part of your job here is to make that
decision visible to someone who can approve it.**

**5. `orderflow` has exactly one genuine push source.** The payment callbacks. Everything
else — catalogue reads, order reads, order placement — is request/response and is
backpressured by the fact that the caller is waiting. That single push source is the whole
reason Phase 10 exists in this curriculum, and it is the spine for Topics 105, 106 and 108.

---

## Machine-level reality

### Per-operator demand accounting

A Reactor chain is not a pipeline of functions. **It is a chain of `Subscriber`s, each of
which is also a `Subscription` for the one below it.** When you call `subscribe()`, the
chain is walked from the bottom up, each operator subscribing to its upstream, and the
`Subscription` objects are wired downward (Topic 104's two-phase walk). Then demand flows
back up that same chain of `Subscription`s.

Each operator translates demand according to its own semantics:

| Operator | Downstream requests n | It requests upstream |
|---|---|---|
| `map(fn)` | n | **n** — one in, one out |
| `filter(p)` | n | **n**, then **1 more for each item dropped** — otherwise a filtered stream would stall |
| `take(k)` | n | `min(n, k)`, then cancels |
| `buffer(k)` | n | **n × k** — it needs k items to produce one output |
| `flatMap(fn, c)` | n | **c** upstream (the concurrency), and n distributed across the c inner subscriptions |
| `concatMap(fn)` | n | **1** at a time — that is what makes it ordered |
| `publishOn(s)` | n | **prefetch**, replenished as its queue drains |
| `window`, `groupBy` | n | Complicated; each has its own prefetch, and each is a place demand can be lost |

**Two consequences worth holding.**

First, `filter`'s replenishment. If `filter` did not request a replacement for each dropped
item, a stream where 99% of items are filtered out would deliver almost nothing. So a
filtered chain issues far more upstream requests than downstream ones, which is correct and
occasionally surprising when you read a `.log()` trace.

Second, **`flatMap` is where most people wrongly believe they have backpressure.**
`flatMap(fn, 8)` bounds the number of *concurrent inner publishers* to 8. It does not bound
how many items each inner publisher emits, and it maintains an internal queue for inner
results whose size is the operator's prefetch. It is a concurrency limiter with a buffer,
which is useful and is not the same thing as end-to-end demand propagation. Trap 5.

### The drain loop

Reactive Streams requires that signals to a `Subscriber` be **serialised** — no two
`onNext` calls concurrently, no `onNext` racing `onComplete`. But `request(n)` can arrive
from a different thread than the one delivering `onNext`. Reactor resolves this with the
standard **work-in-progress drain loop**:

```java
// The shape of essentially every Reactor operator. Not literal Reactor source.
void drain() {
    if (WIP.getAndIncrement(this) != 0) {
        return;                 // someone else is already draining; they will see our work
    }
    int missed = 1;
    for (;;) {
        long r = requested;     // current demand
        long e = 0;             // emitted this pass
        while (e != r) {
            T item = queue.poll();
            if (item == null) break;
            actual.onNext(item);
            e++;
        }
        if (e != 0) Operators.produced(REQUESTED, this, e);   // decrement demand
        missed = WIP.addAndGet(this, -missed);
        if (missed == 0) return;
    }
}
```

Three things follow that you can reason about:

- **A thread calling `request(n)` may end up doing the emission work itself**, if no other
  thread is currently draining. Demand is not a message to another thread; it is a CAS on a
  counter plus possibly running the loop.
- **`requested` is an `AtomicLong` updated with a saturating add.** `Operators.addCap`
  caps at `Long.MAX_VALUE` rather than overflowing to a negative — which would be a
  catastrophic bug, so the specification requires the cap.
- **`Long.MAX_VALUE` demand is treated as a special "unbounded" mode**, and operators take
  a faster path when they see it (no per-item decrement). This is why the difference
  between "requested a lot" and "requested unbounded" is a real difference and not just a
  large number.

### The prefetch, and where the default buffers actually are

Any operator that crosses a thread boundary must buffer, because the producing thread and
the consuming thread run at different speeds. `publishOn` is the obvious one:

```java
flux.publishOn(Schedulers.parallel())            // uses the default prefetch
flux.publishOn(Schedulers.parallel(), 32)        // explicit, and usually better
```

`publishOn` requests **prefetch** items upstream, puts them in a bounded queue, and hands
them to the downstream on the new scheduler, replenishing upstream as the queue drains.

> **Do not quote a default from memory — including mine.** The default prefetch comes from
> Reactor's small-buffer size, configurable with the `reactor.bufferSize.small` system
> property, and replenishment happens at a fraction of it rather than when it empties.
> **Print it:**
>
> ```java
> System.out.println("small buffer = " + reactor.util.concurrent.Queues.SMALL_BUFFER_SIZE);
> System.out.println("xs buffer    = " + reactor.util.concurrent.Queues.XS_BUFFER_SIZE);
> ```
>
> That is one line and it tells you the truth for your BOM's Reactor version.

**The operational point:** those prefetch queues are bounded, per-operator, per-subscriber.
Multiply them. A chain with three `publishOn`s and 10,000 concurrent subscriptions has
30,000 bounded queues, and their combined size is a real number that belongs in your
capacity model. On a per-connection SSE endpoint this is exactly how "bounded" becomes
"gigabytes".

### Fusion, and why the operator count is not the queue count

Reactor's operator fusion (Topic 104) lets adjacent operators share a queue instead of each
allocating one. A `map` after a `filter` after a `range` can be fused into a single
synchronous path with no intermediate queue at all. This is why a long chain does not cost
what a naive reading suggests, and it is also why reasoning about "how many buffers do I
have" from the source code alone is unreliable. **Measure the heap; do not count operators.**

### Where the demand signal dies

This is the diagnostic checklist. Demand propagation stops at:

1. **`Flux.create(sink -> ...)`** — unless the sink's overflow strategy bounds it. The
   external callback has no idea what a `Subscription` is.
2. **Any `Sinks.many()`** — you call `tryEmitNext` on your own schedule.
3. **`onBackpressureBuffer()` with no bound**, and every operator that buffers without a
   limit. Downstream demand is satisfied from the buffer, so upstream never learns.
4. **`subscribe()` with a lambda** — the default subscriber requests `Long.MAX_VALUE`.
5. **`.toStream()`, `.toIterable()`, `.collectList()`, `.block()`** — all materialising or
   unbounded-demand operations.
6. **Any bridge you write from a callback API.**

Everything upstream of the first item on this list is unbackpressured, no matter how
elegant the operators are. **Read your chain from the subscriber upward and find the first
of these. That is where your buffer is.**

---

## Example 1 — minimal

### 1a — see `request(n)` with your own eyes

```java
package com.orderflow.lab;

import reactor.core.publisher.BaseSubscriber;
import reactor.core.publisher.Flux;

public class DemandProbe {

    public static void main(String[] args) throws InterruptedException {

        Flux<Integer> source = Flux.range(1, 20)
                .doOnRequest(n -> System.out.println("  SOURCE saw request(" + n + ")"));

        source.subscribe(new BaseSubscriber<Integer>() {

            @Override
            protected void hookOnSubscribe(reactor.core.publisher.
                                           BaseSubscriber.SubscriptionWrapper unusedShape) {
                // NOTE: the real hook signature takes org.reactivestreams.Subscription.
                // Shown here only to make the point that you MUST request in this hook.
            }

            @Override
            protected void hookOnSubscribe(org.reactivestreams.Subscription subscription) {
                System.out.println("onSubscribe -> requesting 3");
                request(3);                       // <-- the ONLY reason anything happens
            }

            @Override
            protected void hookOnNext(Integer value) {
                System.out.println("onNext " + value);
                if (value % 3 == 0) {
                    System.out.println("  consumed a batch -> requesting 3 more");
                    request(3);
                }
            }

            @Override
            protected void hookOnComplete() {
                System.out.println("onComplete");
            }
        });

        Thread.sleep(500);
    }
}
```

(Delete the first `hookOnSubscribe` stub before compiling — it is there to make the point
that the hook you need is the one taking a `Subscription`, and that requesting inside it is
not optional. A `BaseSubscriber` that never requests receives nothing at all.)

**What to look for:** the interleaving of `SOURCE saw request(n)` with `onNext`.

| What you see | What it means |
|---|---|
| `request(3)` then exactly three `onNext`, then another `request(3)` | **The handshake.** The source produced three items because three were asked for, and stopped. This is backpressure, and there is no thread blocked anywhere. |
| Nothing at all happens if you delete `request(3)` from `hookOnSubscribe` | Demand is the *only* thing that causes emission. Zero demand, zero items. Not an error — just silence. |
| `Flux.range` reports several small requests rather than one large one | Correct: each is additive, and the source tracks outstanding demand. |
| You change `hookOnSubscribe` to `requestUnbounded()` and see one `request(9223372036854775807)` | **That is `Long.MAX_VALUE`, and it is what a plain `subscribe(System.out::println)` does for you.** You have just turned backpressure off. |

**Now do the comparison that makes the point.** Replace the whole `BaseSubscriber` with:

```java
source.subscribe(System.out::println);
```

**What to look for:** a single `SOURCE saw request(9223372036854775807)`.

That is the default subscriber, and it says "send me everything as fast as you can". Every
convenience API in Reactor that takes a lambda does this. **With `Flux.range(1, 20)` it is
harmless. With a source that can produce faster than you consume, it is the bug.**

### 1b — build the OOM in twelve lines

```java
package com.orderflow.lab;

import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;
import java.time.Duration;

public class UnboundedBridge {
    public static void main(String[] args) throws InterruptedException {

        Flux.create(sink -> {
                // A push source with its own clock. It does not know what a request is.
                new Thread(() -> {
                    long i = 0;
                    while (true) sink.next(new byte[1024]);   // 1 KB per item, no pause
                }).start();
            })
            .publishOn(Schedulers.single())
            .subscribe(item -> {
                try { Thread.sleep(1); } catch (InterruptedException ignored) { }  // slow
            });

        Thread.sleep(Duration.ofMinutes(5).toMillis());
    }
}
```

```bash
java -Xmx256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp \
     -Xlog:gc UnboundedBridge.java
```

**What to look for:** the GC log, and how long it takes to die.

| What you see | What it means |
|---|---|
| GC pauses becoming more frequent, then continuous, then `OutOfMemoryError: Java heap space` | **The whole topic in one program.** The producer emits ~1000x faster than the consumer, and the difference accumulates in a buffer nobody bounded. |
| It survives longer with a larger `-Xmx` | Of course. **Heap size does not fix a rate mismatch; it changes the time to failure.** Say that sentence in an interview. |
| The heap dump's dominator tree points at a Reactor internal queue | Topic 79's tool, applied here. The retaining path names the operator that owns the buffer. |
| No OOM, and you see `OverflowException` instead | You are on a Reactor version whose `Flux.create` default overflow strategy is not unbounded buffering, or you passed a strategy. **Check `Flux.create`'s default in your BOM's javadoc — do not assume it from this document.** An error here is the *good* outcome and it is what the fix produces deliberately. |

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` ingests payment callbacks. Real requirements:

1. The payment provider POSTs to `/internal/payments/callbacks` when a payment settles. It
   retries with backoff if we do not return 2xx, and it gives up after a number of attempts.
2. **A lost callback means a customer paid and the order was never fulfilled.** This is a
   money-losing, support-ticket-generating, regulator-interesting failure.
3. Volume is spiky: normal traffic is modest, but a batch settlement at the provider can
   deliver tens of thousands of callbacks in a couple of minutes.
4. Processing a callback means: look up the order, update payment status, decrement
   inventory, publish an event. Several hundred milliseconds, mostly waiting on Postgres.
5. The Topic 65 baseline is recorded, and the callback endpoint must not degrade the
   catalogue and order endpoints when a burst arrives.
6. There is more than one `orderflow` pod, and any of them can restart at any time.

Requirements 2 and 6 together rule out every in-memory buffering strategy on their own.
**Say that before writing any Reactor code**, because the most common failure here is
reaching for an operator when the answer is an architecture.

### The version that ships and OOMs

```java
// DO NOT SHIP THIS. It is Trap 1 and it is the failure drill.
@Configuration
public class CallbackIngestion {

    private final Sinks.Many<PaymentCallback> sink =
            Sinks.many().multicast().onBackpressureBuffer();     // <-- UNBOUNDED

    @Bean
    ApplicationRunner startProcessing(PaymentCallbackProcessor processor) {
        return args -> sink.asFlux()
                .flatMap(processor::process)                     // slow: hits Postgres
                .subscribe();                                    // request(Long.MAX_VALUE)
    }

    @PostMapping("/internal/payments/callbacks")
    public ResponseEntity<Void> receive(@RequestBody PaymentCallback callback) {
        sink.tryEmitNext(callback);                              // fire and forget
        return ResponseEntity.accepted().build();                // 202, always
    }
}
```

Everything about this looks responsible. It returns 202 immediately so the provider is not
kept waiting. It processes asynchronously. It uses reactive types. **It has three
independent fatal defects and they compound.**

### What actually happens under a settlement burst

| Time | What is happening |
|---|---|
| t=0 | Burst begins. 500 callbacks/second arrive. The processor handles maybe 50/second — it is bounded by Postgres and the connection pool (Topic 109). |
| t=0 to t=60s | The sink's buffer grows by ~450 items/second. Each `PaymentCallback` plus its parsed JSON is a few KB. Heap climbs steadily. |
| t=60s | ~27,000 buffered callbacks. Old-generation occupancy climbing. G1 starts doing more mixed collections (Topic 71). |
| t=120s | GC time climbing. **The catalogue endpoint's p99 degrades** — it shares a heap and a GC with the ingestion path, and it has nothing to do with payments. |
| t=180s | GC thrashing. Throughput on every endpoint collapses. |
| t=200s | `OutOfMemoryError: Java heap space`. **The pod dies with ~40,000 callbacks in memory. All of them are lost.** |
| t=201s | Kubernetes restarts the pod. The provider is still sending. The remaining pods absorb the burst and start the same climb. |
| t=+minutes | The provider retries the callbacks it did not get a 2xx for. **But we returned 202 for every one of them**, including the 40,000 we then dropped. **The provider will never retry those.** |

**That last row is the real disaster and it has nothing to do with memory.** Returning
`202 Accepted` is a promise that you have taken responsibility for the item. We returned
202 and then lost the data. The provider's retry mechanism — the thing that would have
saved us — was disabled by our own response code.

Three defects, then, in increasing order of severity:

1. **Unbounded buffer** → OOM. Visible, dramatic, and the easiest to fix.
2. **`subscribe()` with no subscriber** → `request(Long.MAX_VALUE)`, so no demand
   propagates and the flatMap has no reason to slow the sink. Also: **no error handler**,
   so the first `onError` terminates the subscription permanently and ingestion silently
   stops while the endpoint keeps returning 202.
3. **202 before durability** → the acknowledgement is a lie. This one is not a Reactor bug
   at all, and it is the one that costs money.

### The version that ships

The shape is: **make the buffer durable, keep the in-memory buffer small and bounded, and
only acknowledge what you have actually persisted.**

```java
package com.orderflow.payments.ingestion;

@RestController
public class PaymentCallbackController {

    private final PaymentCallbackInbox inbox;      // durable: a Postgres table
    private final Counter accepted, rejected;

    /**
     * The ONLY thing this endpoint does is persist and acknowledge. It is deliberately
     * boring. Processing happens elsewhere, at its own rate, from durable storage.
     *
     * 202 is now TRUE: we have taken responsibility, because it is on disk.
     */
    @PostMapping("/internal/payments/callbacks")
    public ResponseEntity<Void> receive(@RequestBody @Valid PaymentCallback callback) {
        try {
            inbox.insertIfAbsent(callback);        // unique on (provider, callbackId)
            accepted.increment();
            return ResponseEntity.accepted().build();
        } catch (InboxFullException | DataAccessException e) {
            rejected.increment();
            // 503 -> the provider RETRIES. This is the whole point.
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                                 .header(HttpHeaders.RETRY_AFTER, "30")
                                 .build();
        }
    }
}
```

```java
package com.orderflow.payments.ingestion;

/**
 * Reads from the durable inbox and processes at a rate the database can sustain.
 * THIS is where the reactive chain belongs -- over a PULL source, where backpressure
 * works by construction.
 */
@Component
public class PaymentCallbackPump {

    private static final int BATCH = 64;
    private static final int CONCURRENCY = 8;      // matched to the Hikari pool (T109)

    private final PaymentCallbackInbox inbox;
    private final PaymentCallbackProcessor processor;

    @Bean
    ApplicationRunner pump() {
        return args ->
            Flux.defer(() -> Flux.fromIterable(inbox.claimBatch(BATCH)))
                // A PULL source: claimBatch is called only when demand asks for it.
                .repeatWhen(completed -> completed.delayElements(Duration.ofMillis(200)))
                .flatMap(processor::process, CONCURRENCY)
                .doOnNext(result -> inbox.markProcessed(result.callbackId()))
                .onErrorContinue((error, item) -> {
                    // One poisoned callback must not stop ingestion. Park it.
                    inbox.markFailed(item, error);
                    poisonCounter.increment();
                })
                .subscribe(
                    result -> { },
                    error   -> log.error("PUMP TERMINATED -- ingestion has STOPPED", error),
                    ()      -> log.warn("pump completed unexpectedly")
                );
    }
}
```

**And now the bounded in-memory buffer, for the case where you genuinely have a push source
you cannot persist synchronously.** `orderflow` has one: an internal fan-out of processed
payment events to any connected admin dashboards over SSE. Nobody loses money if a
dashboard misses an update.

```java
/**
 * Live payment events for connected dashboards. This IS a legitimate place for a
 * lossy strategy, and the reason is written in the code so nobody has to guess.
 *
 * OVERFLOW POLICY: keep the newest 256 events per connected dashboard; drop the
 * oldest. Rationale: a dashboard showing slightly incomplete history is acceptable;
 * a dashboard that OOMs the payment service is not. Approved by <name>, <date>.
 * The authoritative record is the payments table; this stream is a convenience.
 */
@GetMapping(value = "/admin/payments/live", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<PaymentEvent> live() {
    return paymentEvents.asFlux()
            .onBackpressureBuffer(256,
                    dropped -> droppedEvents.increment(),        // COUNT every loss
                    BufferOverflowStrategy.DROP_OLDEST)
            .doOnCancel(() -> connectedDashboards.decrementAndGet());
}
```

**Read the comment block, not just the code.** The overflow policy is documented, the
rationale is stated, the person who approved it is named, and the loss is counted as a
metric. That comment is the deliverable of this topic. An `onBackpressureBuffer(256,
DROP_OLDEST)` with no explanation is an undocumented product decision, and in six months
nobody will know whether dropping was ever acceptable.

### The trade-off, stated explicitly

| | Unbounded buffer | `DROP_OLDEST` at 256 | Durable inbox |
|---|---|---|---|
| Data loss | None, until the OOM loses **everything** | Bounded, counted, and predictable | None |
| Memory | Unbounded | Bounded and small | Bounded and small |
| Failure mode | Pod death, all in-flight lost, catalogue degraded first | Oldest events missing from a dashboard | 503s and provider retries |
| Latency under burst | Grows without limit | Bounded by the buffer | Grows — items wait on disk |
| Recovery after restart | Nothing survives | Nothing survives (acceptable here) | **Everything survives** |
| Correct for | Nothing | Live dashboards, telemetry, price ticks | **Payment callbacks** |
| Costs you | Everything | Completeness | Latency, a table, and 503-handling |

**The middle column is not "safer" than the first — it is a different failure.** You have
traded an unpredictable total loss for a predictable partial loss. That is almost always
the right trade, and it is still a trade, and someone other than you should agree to it.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — unbounded `onBackpressureBuffer`, and its several disguises

**Wrong approach.** All of these are the same bug:

```java
Sinks.many().multicast().onBackpressureBuffer()       // no bound
Sinks.many().unicast().onBackpressureBuffer()         // no bound
flux.onBackpressureBuffer()                           // no bound
Flux.create(sink -> ...)                              // default strategy -- CHECK YOURS
flux.collectList()                                    // materialises everything
flux.cache()                                          // retains everything, forever
```

**Exact symptom.** Old-generation heap growing linearly with time under load and never
recovering after a young collection. GC frequency rising, then GC time dominating, then
`OutOfMemoryError: Java heap space`. **Before the OOM, unrelated endpoints degrade**,
because they share the heap and the collector. The heap dump's dominator tree points at a
Reactor internal queue holding hundreds of thousands of your domain objects.

**Root cause.** Downstream demand is satisfied out of the buffer, so the upstream is never
slowed. The demand signal terminates at the buffer. **A buffer is where backpressure goes
to die** — a bounded one converts it into a decision, an unbounded one converts it into an
OOM.

**Fix.**

```java
Sinks.many().multicast().onBackpressureBuffer(1024);              // bounded
flux.onBackpressureBuffer(1024, this::countDropped, DROP_OLDEST); // bounded + counted
Flux.create(sink -> ..., FluxSink.OverflowStrategy.ERROR);        // explicit
```

And the review rule that catches all the disguises: **every buffering operator must have a
numeric argument, and every drop must increment a counter.** If you cannot say what the
number should be, you do not yet understand the rate mismatch, and that is the thing to go
and measure.

**Observable:** old-gen occupancy under sustained load, from `-Xlog:gc*`; the dominator
tree in a heap dump; and the dropped-item counter, which should exist even when it reads
zero.

### Trap 2 — a blocking call on an event-loop thread

**This is the most damaging mistake in the whole reactive phase.** It appears in
Topics 103, 105, 106 and 108 because it is the one that turns a working service into an
outage.

**Wrong approach.**

```java
@PostMapping("/internal/payments/callbacks")
public Mono<Void> receive(@RequestBody PaymentCallback callback) {
    return Mono.fromRunnable(() -> {
        jdbcTemplate.update("INSERT INTO payment_callbacks ...", ...);   // BLOCKING JDBC
    });                                                       // and no subscribeOn
}

// Or, in a Netty channel handler:
channel.pipeline().addLast(new SimpleChannelInboundHandler<Callback>() {
    protected void channelRead0(ChannelHandlerContext ctx, Callback msg) {
        repository.save(msg);        // blocking. On the event loop.
    }
});

// Or the subtle one, deep inside an operator:
flux.map(callback -> orderRepository.findById(callback.orderId()))   // blocking JDBC
```

**Exact symptom.** Throughput collapses to roughly `ioWorkerCount / blocking-call-duration`
requests per second. **Every connection assigned to that event loop stalls** — including
connections to endpoints that do no database work at all. CPU is near idle. Latency is
bimodal: requests that land on an unblocked loop are fast, requests on a blocked loop wait
for every other request queued on it. As load rises, all loops get blocked and everything
stops.

This is Topic 101's carrier starvation and Topic 100's blocked FJP worker, arriving by a
third route. **Same signature every time: collapsed throughput, idle CPU.**

**Root cause.** There are `reactor.netty.ioWorkerCount` event-loop threads — a small number,
derived from `availableProcessors()` — and each one serves many connections by
multiplexing. A blocked loop serves none of them. This is exactly the Node rule you already
have installed, with the blast radius divided into N loops instead of one.

**Fix.**

```java
// 1. Use a non-blocking driver. R2DBC for Postgres. This is the real answer and it
//    is a real commitment -- see Topic 106.

// 2. If you must call blocking code, get it OFF the loop, onto a bounded scheduler:
Mono.fromCallable(() -> jdbcTemplate.update(...))
    .subscribeOn(Schedulers.boundedElastic())        // or newBoundedElastic(n, q, name)
    .then();

// 3. Prove it, in CI:
BlockHound.install();
```

**Observable — and this one you can automate.** BlockHound instruments the JDK so that a
blocking call from a thread it considers non-blocking throws immediately:

```xml
<dependency>
  <groupId>io.projectreactor.tools</groupId>
  <artifactId>blockhound</artifactId>
  <scope>test</scope>
</dependency>
```

```java
@BeforeAll static void installBlockHound() { BlockHound.install(); }
```

| What you see | What it means |
|---|---|
| `BlockingOperationError` naming a JDBC or socket method | **You found it.** Read the thread name in the stack: `reactor-http-nio-*` means an event loop. |
| Test passes | No *instrumented* blocking call ran on a non-blocking thread on this path. Not proof of absence — BlockHound knows JDK blocking primitives, not your CPU-heavy loop. |
| BlockHound fails to install | Java agent / module access. **Flagged uncertainty: BlockHound support for the newest JDKs sometimes lags — verify it installs on your JDK before relying on it in CI.** |

Run BlockHound in tests and staging, not production — it instruments the JDK and has a
cost. Topic 103 Proof 4 has the fuller setup.

### Trap 3 — `block()` on an event-loop thread

**Wrong approach.**

```java
public Mono<OrderDetail> detail(OrderId id) {
    Product product = productClient.get(id).block();     // <-- on whatever thread we are on
    return Mono.just(new OrderDetail(product));
}
```

**Exact symptom.** In recent Reactor versions, an immediate exception:

```
java.lang.IllegalStateException: block()/blockFirst()/blockLast() are blocking,
which is not supported in thread reactor-http-nio-3
```

*(Illustration of the format, not captured output.)*

On an older version, or on a thread Reactor does not recognise, you get no exception and
the far worse outcome: a silently stalled event loop, exactly as in Trap 2.

**Root cause.** `block()` subscribes and then parks the calling thread on a latch until the
`Mono` terminates. If the calling thread is the one that would have delivered the
completion signal — the event loop — you have deadlocked that loop against itself. Even
when it does not self-deadlock, you have converted a non-blocking chain into a blocking
call on a thread that must never block.

**Fix.** Do not unwrap. Compose:

```java
public Mono<OrderDetail> detail(OrderId id) {
    return productClient.get(id).map(OrderDetail::new);
}
```

**`block()` is legitimate in exactly three places:** `main()`, a test, and a genuinely
imperative boundary on a thread you own and that is not an event loop. Everywhere else it
is a bug, and the fact that it *sometimes* throws is a safety net, not a design.

**Observable:** the exception, plus BlockHound, plus a grep for `.block()` in your
`src/main` reactive packages that should return zero results.

### Trap 4 — `subscribe(lambda)` silently requesting unbounded

**Wrong approach.**

```java
someFlux.subscribe(item -> process(item));       // requests Long.MAX_VALUE
someFlux.subscribe();                            // same, and discards errors too
```

**Exact symptom.** You added `onBackpressureBuffer(1024, DROP_OLDEST)` to a chain, deployed
it, and **the drop counter never increments while the heap still grows**. Or: you carefully
built a demand-aware pipeline and it behaves exactly as it did before.

**Root cause.** The lambda-taking `subscribe` overloads install a `LambdaSubscriber`, which
requests `Long.MAX_VALUE` in `onSubscribe`. Unbounded demand downstream means every
buffering operator upstream is satisfied instantly and never applies pressure. **You built
the backpressure and then turned it off at the last line.**

The same applies to several convenience paths: `toStream()`, `toIterable()`, and — subtly —
returning a `Flux` from a controller, where the framework subscribes on your behalf with a
policy you did not choose.

**Fix.** Where demand matters, subscribe with something that requests deliberately:

```java
someFlux.subscribe(new BaseSubscriber<Item>() {
    @Override protected void hookOnSubscribe(Subscription s) { request(32); }
    @Override protected void hookOnNext(Item item) { process(item); request(1); }
    @Override protected void hookOnError(Throwable t) { log.error("stream died", t); }
});
```

Or — usually better — put the bound where it belongs: `flatMap(fn, concurrency)`,
`limitRate(n)`, or an explicit prefetch on `publishOn`. `limitRate(n)` is the cheapest fix
and is often exactly right:

```java
someFlux.limitRate(64).subscribe(this::process);   // caps demand regardless of subscriber
```

**Also fix the missing error handler.** `subscribe(lambda)` with one argument discards
errors — the stream terminates, ingestion stops, and nothing is logged. **A production
`subscribe` always has an error handler.**

**Observable:** `.log()` on the chain and read the first `request(...)` line. If it says
`unbounded` or `9223372036854775807`, you have this bug.

### Trap 5 — believing `flatMap` gives you backpressure

**Wrong approach.**

```java
callbacks.flatMap(this::processCallback)        // default concurrency
         .subscribe();
```

**Exact symptom.** The downstream service or the connection pool sees far more concurrent
work than you expected. `hikaricp.connections.pending` climbs. Under a burst, memory grows
because `flatMap` holds an internal queue of results from inner publishers that have
completed while the downstream is not consuming.

**Root cause.** `flatMap` bounds **concurrency** — how many inner publishers are subscribed
at once — with a default that is a Reactor constant, not 1. It does not bound how much each
inner emits, and it maintains its own prefetch queue. It is a concurrency limiter with a
buffer. That is useful and it is not end-to-end demand propagation.

**Fix.** Set the concurrency explicitly, from a real constraint:

```java
callbacks.flatMap(this::processCallback, 8)     // 8 = the Hikari pool size, not a guess
```

And where order matters, `concatMap` requests one at a time and is the genuinely
demand-driven option — at the cost of no concurrency at all.

**Print the default rather than trusting anyone's memory:**

```java
System.out.println(reactor.util.concurrent.Queues.SMALL_BUFFER_SIZE);
```

**Observable:** concurrent downstream calls (Topic 118), `hikaricp.connections.pending`
(Topic 109), and a `.log()` trace showing the request counts `flatMap` issues upstream.

### Trap 6 — choosing the overflow strategy without asking anyone

**Wrong approach.**

```java
paymentCallbacks.onBackpressureDrop().subscribe(this::process);
```

**Exact symptom.** Nothing. The service is healthy, memory is flat, latency is good, no
alerts fire. Three weeks later, support has 40 tickets from customers who were charged and
never received their order. The correlation to a settlement burst is found by someone
scrolling through a dashboard.

**Root cause.** `onBackpressureDrop()` with no consumer discards items silently. It is the
correct operator for price ticks and telemetry, and it is a financial incident for payment
callbacks. **The API cannot tell the difference, and neither can the code review, unless
the intent is written down.**

**Fix.** Three parts, all mandatory:

1. **Count every drop.** `onBackpressureDrop(dropped -> droppedCounter.increment())`.
   A loss you cannot see is indistinguishable from no loss.
2. **Write the rationale in the code**, including who approved it and when. See Example 2.
3. **For anything that must not be lost, do not choose a lossy strategy at all** — make the
   buffer durable and let the sender retry. That is not a Reactor operator; it is an
   architecture, and recognising when the API is the wrong layer to solve the problem is
   the actual senior skill here.

**Observable:** the dropped-item counter on a dashboard with a non-zero alert threshold —
where "non-zero" is a deliberate choice too. For dashboards, alert at a high rate. For
payments, alert at 1.

---

## Hands-on proof

### Setup

```bash
# Confirm what you actually have. The BOM decides your Reactor version -- do not pin it.
./mvnw dependency:tree | grep -E 'reactor|reactive-streams|netty'
```

```java
// Print the buffer constants for YOUR version. One line, and it beats any documentation.
System.out.println("SMALL_BUFFER_SIZE = " + reactor.util.concurrent.Queues.SMALL_BUFFER_SIZE);
System.out.println("XS_BUFFER_SIZE    = " + reactor.util.concurrent.Queues.XS_BUFFER_SIZE);
```

### Proof 1 — read the demand signals with `.log()`

`.log()` is the primary instrument for this topic. It prints every Reactive Streams signal
including `request`.

```java
Flux.range(1, 100)
    .log("source")
    .filter(i -> i % 10 == 0)
    .log("after-filter")
    .map(i -> i * 2)
    .log("after-map")
    .subscribe(new BaseSubscriber<Integer>() {
        @Override protected void hookOnSubscribe(Subscription s) { request(2); }
        @Override protected void hookOnNext(Integer v) { request(1); }
    });
```

**What to look for:** the `request(n)` lines at each stage, and how the numbers differ.

| What you see | What it means |
|---|---|
| `after-map` requests 2, `after-filter` requests 2, `source` requests 2 | `map` and `filter` pass initial demand through unchanged. |
| `source` receives many more requests than `after-filter` does | **`filter` is replenishing.** For every item it drops it requests a replacement, otherwise a 90%-filtered stream would deliver almost nothing. |
| The first request line says `unbounded` | You are using a lambda subscriber somewhere. Trap 4. |
| Requests interleaved with `onNext` rather than batched | Normal: demand is a running counter, not a batch protocol. |

Then change the subscriber to `subscribe(System.out::println)` and re-read the log.

| What you see | What it means |
|---|---|
| A single `request(unbounded)` at every stage, then all 100 items | **The default subscriber turns backpressure off.** This one comparison teaches the trap better than any explanation. |

### Proof 2 — find where the demand signal dies in a real chain

Take the `orderflow` callback chain and add `.log()` at three points:

```java
sink.asFlux()
    .log("1-after-sink")
    .onBackpressureBuffer(1024, this::countDropped, DROP_OLDEST)
    .log("2-after-buffer")
    .flatMap(processor::process, 8)
    .log("3-after-flatmap")
    .subscribe(...);
```

**What to look for:** the `request(n)` numbers at points 1 and 2.

| What you see | What it means |
|---|---|
| Point 2 shows small, repeated requests; point 1 shows `unbounded` | **Correct and expected.** The buffer requests unbounded from the sink because the sink is a push source that cannot be slowed — and the buffer is where the pressure is absorbed. Demand propagation ends at the buffer, deliberately and with a bound. |
| Point 1 shows `unbounded` and there is **no bounded buffer** anywhere below it | **The bug.** Unbounded demand into a push source with nothing to absorb it. |
| Point 3 shows unbounded | Trap 4 at the subscriber. Fix that before reading anything else. |

**The rule this proof teaches:** an `unbounded` request is not automatically wrong. It is
wrong when there is no bounded buffer between it and a source that can outrun you. **Read
your chain upward from `subscribe()` and find the first bound.**

### Proof 3 — measure the actual rate mismatch

Before choosing a buffer size, measure the two rates. Guessing a buffer size is how you get
a number nobody can defend.

```java
// On the producer side:
Counter produced = registry.counter("orderflow.callbacks.received");

// On the consumer side:
Counter consumed = registry.counter("orderflow.callbacks.processed");
Timer   duration = registry.timer("orderflow.callbacks.processing.duration");
```

```
# In your dashboard:
rate(orderflow_callbacks_received_total)   # arrivals per second
rate(orderflow_callbacks_processed_total)  # departures per second
```

**What to look for:** the peak of `received - processed`, integrated over the burst
duration. That integral **is** your required buffer size.

| What you see | What it means |
|---|---|
| Rates equal at steady state, with a spike in the difference during a burst | Normal and healthy. The buffer's job is to absorb the integral of that spike. Size it from the measured worst case plus headroom. |
| `received` persistently above `processed` | **No buffer size will save you.** You have a sustained capacity shortfall, and buffering only delays the failure. Fix the consumer, add consumers, or shed load at the door. |
| `processed` above `received` | Retries, duplicates, or a broken counter. Investigate — this often reveals an idempotency bug. |

**The sentence to remember:** *a buffer absorbs a burst; it does not add capacity.* If the
average arrival rate exceeds the average service rate, the queue grows without bound no
matter how large it is. That is a queueing-theory fact, not a tuning problem, and it is
worth being able to say in exactly those words.

### Proof 4 — watch the buffer fill, in the heap

```bash
java -Xmx512m -Xlog:gc*:file=gc.log:time,uptime \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps \
     -jar orderflow.jar
```

While a burst runs:

```bash
jcmd <pid> GC.class_histogram | head -30
jcmd <pid> GC.heap_info
```

**What to look for:** the growth of your callback class and of Reactor queue classes.

| What you see | What it means |
|---|---|
| `PaymentCallback` instance count climbing with no plateau | The buffer is filling and not draining. Confirms the rate mismatch. |
| Old-gen occupancy climbing across young collections and never recovering | The buffered items are being promoted (Topic 68). This is what turns a rate mismatch into a GC problem. |
| A Reactor internal queue class high in the histogram | The buffer, named. Open a heap dump and use the dominator tree (Topic 79) to find which operator owns it. |
| Everything flat with a healthy drop counter | The bounded strategy is working. **Check the drop counter is actually non-zero during the burst** — a bounded buffer that never drops was never tested. |

### Proof 5 — BlockHound, for the consumer

The consumer of a `Flux` runs on a Reactor scheduler. If it blocks on the wrong one, you
have Trap 2. Prove it does not:

```java
@SpringBootTest
class CallbackIngestionDoesNotBlockTest {

    @BeforeAll
    static void installBlockHound() {
        BlockHound.builder()
                  // Whitelist known-safe blocking, with a comment saying WHY:
                  .allowBlockingCallsInside("com.orderflow.payments.LegacyGateway", "charge")
                  .install();
    }

    @Test
    void ingestion_path_does_not_block_a_non_blocking_thread() {
        webTestClient.post().uri("/internal/payments/callbacks")
                     .bodyValue(sampleCallback())
                     .exchange()
                     .expectStatus().isAccepted();
    }
}
```

**What to look for:** a pass, or a `BlockingOperationError` naming the method and thread.
See Trap 2's table for how to read it.

---

## Failure drill

**Mandatory.** Do not read the analysis until you have produced the OOM yourself and
written down what you saw.

### The scenario

Bridge an unbounded push source into a `Flux` with a slow consumer. Watch the heap grow
until the JVM dies. Then apply `onBackpressureBuffer(n, DROP_OLDEST)` and show the
trade-off explicitly — because the fix does not make the problem go away, it converts it
into a different problem that you chose.

### Setup

`src/main/java/com/orderflow/lab/BackpressureDrill.java`:

```java
package com.orderflow.lab;

import reactor.core.publisher.BufferOverflowStrategy;
import reactor.core.publisher.Flux;
import reactor.core.publisher.FluxSink;
import reactor.core.scheduler.Schedulers;

import java.time.Duration;
import java.util.concurrent.atomic.AtomicLong;

public class BackpressureDrill {

    /** Stands in for a settlement-burst payment callback. ~2 KB each. */
    record Callback(long id, byte[] payload) {
        static Callback of(long id) { return new Callback(id, new byte[2048]); }
    }

    static final AtomicLong produced  = new AtomicLong();
    static final AtomicLong consumed  = new AtomicLong();
    static final AtomicLong dropped   = new AtomicLong();

    /** mode: "unbounded" | "bounded" | "error" */
    public static void main(String[] args) throws Exception {
        String mode = args.length > 0 ? args[0] : "unbounded";
        System.out.printf("pid=%d mode=%s%n", ProcessHandle.current().pid(), mode);

        Flux<Callback> source = Flux.create(sink -> {
            Thread.ofPlatform().daemon().name("provider").start(() -> {
                long id = 0;
                while (!Thread.currentThread().isInterrupted()) {
                    sink.next(Callback.of(id++));       // NO pause. The provider's clock.
                    produced.incrementAndGet();
                }
            });
        }, FluxSink.OverflowStrategy.BUFFER);           // <-- unbounded at the SINK

        Flux<Callback> pipeline = switch (mode) {
            case "bounded" -> source.onBackpressureBuffer(
                    10_000,
                    c -> dropped.incrementAndGet(),
                    BufferOverflowStrategy.DROP_OLDEST);
            case "error"   -> source.onBackpressureBuffer(
                    10_000,
                    c -> dropped.incrementAndGet(),
                    BufferOverflowStrategy.ERROR);
            default        -> source;                   // the defect
        };

        // Sampler: the two rates and the heap, every second.
        Thread.ofPlatform().daemon().start(() -> {
            long t0 = System.currentTimeMillis();
            while (true) {
                Runtime r = Runtime.getRuntime();
                System.out.printf(
                    "t=%4ds produced=%9d consumed=%8d dropped=%9d inFlight=%9d heapMB=%5d%n",
                    (System.currentTimeMillis() - t0) / 1000,
                    produced.get(), consumed.get(), dropped.get(),
                    produced.get() - consumed.get() - dropped.get(),
                    (r.totalMemory() - r.freeMemory()) / (1024 * 1024));
                try { Thread.sleep(1000); } catch (InterruptedException e) { return; }
            }
        });

        pipeline
            .publishOn(Schedulers.single(), 32)
            .subscribe(
                callback -> {
                    try { Thread.sleep(1); } catch (InterruptedException ignored) { }
                    consumed.incrementAndGet();
                },
                error -> System.out.println("STREAM TERMINATED: " + error),
                ()    -> System.out.println("STREAM COMPLETED"));

        Thread.sleep(Duration.ofMinutes(10).toMillis());
    }
}
```

### Commands

```bash
# Run 1 -- the defect. Small heap so you do not wait long.
java -Xmx256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp \
     -Xlog:gc*:file=/tmp/gc-unbounded.log:time,uptime \
     BackpressureDrill.java unbounded 2>&1 | tee run-unbounded.log

# Run 2 -- bounded, lossy.
java -Xmx256m -Xlog:gc*:file=/tmp/gc-bounded.log:time,uptime \
     BackpressureDrill.java bounded 2>&1 | tee run-bounded.log

# Run 3 -- bounded, fail-fast.
java -Xmx256m BackpressureDrill.java error 2>&1 | tee run-error.log

# During run 1, in another terminal:
jcmd <pid> GC.class_histogram | head -20
jcmd <pid> GC.heap_info
```

### What to capture, before reading on

For each run:

1. Time to `OutOfMemoryError`, if it happens.
2. The `heapMB` and `inFlight` curves over time.
3. `produced`, `consumed`, `dropped` at 30 seconds and at the end.
4. The ratio `consumed / produced`.
5. From run 1's heap dump: the top three classes by retained size, and the dominator.
6. From the GC logs: when old-gen occupancy stopped recovering after young collections.

**Now write down, before reading on: how much data did run 1 lose, and how much did run 2
lose?** Most people get this backwards on the first attempt.

### How to read it

| What you see | What it means |
|---|---|
| **Run 1:** `heapMB` climbing steadily, `inFlight` climbing in lockstep, then `OutOfMemoryError` | **The drill has fired.** The producer outran the consumer by roughly 1000x and every excess item was retained. |
| **Run 1:** the GC log shows old-gen occupancy rising across young collections and never falling | The buffered callbacks survived enough young collections to be promoted (Topic 68). This is exactly how a rate mismatch becomes a GC death spiral. |
| **Run 1:** the heap dump's dominator tree names a Reactor queue holding `Callback[]` | Topic 79's tool, applied here. **The dump does not say "your producer is too fast" — you have to read it that way.** |
| **Run 1:** `consumed / produced` is tiny, and at the OOM everything in flight is lost | **The answer to the question above: run 1 lost EVERYTHING.** Every buffered callback died with the process. |
| **Run 2:** `heapMB` rises to a plateau and stays there; `dropped` climbs steadily | **The fix, working.** Memory is bounded. Loss is bounded, counted, and visible. |
| **Run 2:** `consumed` is the same rate as in run 1 | **The critical observation, and the point of the whole drill.** The buffer did not make the consumer faster. It never could. All it changed was *what happens to the excess* — from "retained until we die" to "discarded on purpose, and counted". |
| **Run 2:** `dropped` is roughly `produced - consumed` | Correct arithmetic, and confirmation that the counter is wired up. |
| **Run 3:** the stream terminates almost immediately with an overflow error | **`ERROR` is not a gentler `DROP`.** It fails the whole stream on the first overflow. Ingestion stops. If you choose this, you need a supervision strategy that resubscribes — otherwise you have converted a memory problem into a silent total outage. |
| Run 2 never drops anything | Your consumer is faster than your producer, or the buffer is larger than the mismatch over the run. Slow the consumer or lengthen the run. **A bounded buffer that never drops has not been tested.** |

### The trade-off, made explicit

This table is the deliverable of the drill. Fill it in with **your** numbers.

| | Run 1: unbounded | Run 2: `DROP_OLDEST` at 10k | Run 3: `ERROR` at 10k |
|---|---|---|---|
| Time to failure | | Does not fail | |
| Peak heap | | | |
| Items processed in 60 s | | | |
| Items lost in 60 s | | | |
| **Items lost at failure** | **everything in flight** | 0 (no failure) | everything in flight |
| Is the loss visible? | No | **Yes — counted** | Yes — the error |
| Recovers on its own? | No (pod restart) | Yes | **No — the stream is dead** |
| Correct for payment callbacks | No | No | No |
| Correct for a live dashboard | No | **Yes** | No |

**The two sentences to carry out of this drill:**

1. *An unbounded buffer does not prevent data loss. It defers it, and then loses
   everything at once, at the least convenient moment, along with the process.*
2. *A bounded buffer does not add capacity. It converts an unpredictable total loss into a
   predictable partial loss, which you can measure, alert on, and get someone to approve.*

### Part B — on `orderflow`, under the Topic 65 baseline

1. Wire the unbounded sink version of the callback endpoint into `orderflow` (a branch).
2. Run the Topic 65 mix at baseline arrival rates, **plus** a callback burst: 500/second
   for five minutes against `/internal/payments/callbacks`.
3. Record: heap, GC time, and the **catalogue read p99** — an endpoint that touches nothing
   in the payment path.
4. Then the bounded version. Then the durable-inbox version from Example 2.

**What to look for:** the catalogue p99 during the burst.

**How to read it:** if catalogue p99 degrades under the unbounded version, you have shown
that an ingestion buffer degrades an unrelated endpoint through the shared heap and
collector. That is the fourth distinct route to the same blast-radius lesson in this
curriculum, after Topics 100, 101 and 102, and by now you should recognise the shape before
you see the graph.

For the durable-inbox version, look at something different: the **503 rate** and whether
the provider retried. That is the version where overload becomes a conversation with the
sender instead of a memory problem, and the 503s are the conversation.

### What the drill proves

1. **Backpressure is not a feature you enable; it is a property of a path.** The chain in
   run 1 used Reactor types throughout and had no backpressure at all, because the source
   was a push source and nothing translated pushes into demand.
2. **Every buffer is a decision.** Bounded or unbounded, chosen or defaulted, you have one.
   The only question is whether you picked the number.
3. **The overflow strategy is a product decision.** `DROP_OLDEST` is right for a dashboard
   and wrong for money, and the code cannot tell — only a comment naming the approver can.

---

## Measurement

### The standing rule

A naive `System.nanoTime()` loop is the **wrong** way to measure JVM performance. It
measures JIT warm-up, dead-code elimination, on-stack replacement and ambient noise.
**Topic 77 (JMH)** is where you learn to do it properly.

The specific version for this topic: **do not microbenchmark backpressure.** The behaviour
you care about only appears at sustained mismatched rates over minutes, with a real heap
and a real collector. The instruments are the Topic 65 harness, GC logs and counters — not
a benchmark harness.

The counters in the drill (`produced`, `consumed`, `dropped`) are counters, not timings, so
reading them with plain wall-clock sampling is fine.

### The instrument for each claim

| Claim | Instrument | What invalidates it |
|---|---|---|
| "This chain has backpressure" | `.log()` at the source; read the first `request(n)` | An `unbounded` request with no bounded buffer below it |
| "The buffer is bounded" | Read the operator's numeric argument; grep for zero-arg buffering overloads | `collectList`, `cache`, `Sinks...onBackpressureBuffer()` with no size |
| "We are not losing data" | The drop counter — **which must exist even when it reads zero** | No counter, so loss is invisible |
| "The buffer is big enough" | Integral of `received - processed` over the worst measured burst | Guessing; sizing from a healthy period |
| "We can sustain this rate" | `rate(received)` vs `rate(processed)` at steady state | Measuring during a burst only |
| "Nothing blocks the event loop" | BlockHound in CI, plus thread-name logging in operators | Assuming; BlockHound not installing on your JDK |
| "Memory is stable" | Old-gen occupancy across young collections, `-Xlog:gc*` | Watching total heap, which is noisy |
| "The unrelated endpoints are fine" | Catalogue p99 during a callback burst, vs Topic 65 baseline | Only measuring the endpoint under test |

### The dashboard

```java
@Configuration
class IngestionMetrics {

    @Bean
    MeterBinder callbackIngestionMetrics(PaymentCallbackInbox inbox,
                                         AtomicLong bufferOccupancy) {
        return registry -> {
            Gauge.builder("orderflow.callbacks.buffer.occupancy",
                    bufferOccupancy, AtomicLong::get).register(registry);
            Gauge.builder("orderflow.callbacks.inbox.depth",
                    inbox, PaymentCallbackInbox::pendingCount).register(registry);
        };
    }
}

// Counters at the call sites:
registry.counter("orderflow.callbacks.received").increment();
registry.counter("orderflow.callbacks.processed").increment();
registry.counter("orderflow.callbacks.dropped", "reason", "overflow").increment();
registry.counter("orderflow.callbacks.rejected", "reason", "inbox-full").increment();
```

**The four alerts worth having:**

1. **`dropped` above zero**, for any stream where loss is not intended. Threshold 1.
2. **Buffer occupancy above 80% of its bound** for more than a minute. You are about to
   start dropping, and this is the warning.
3. **`rate(received) > rate(processed)` sustained for five minutes.** A capacity shortfall,
   not a burst. No buffer size fixes this.
4. **Inbox depth growing monotonically.** The durable buffer is filling, which is the
   graceful version of the same problem and still needs someone to look.

### Against the Topic 65 baseline — the rules

1. Same dataset, same JVM flags, same container limits.
2. **Open-model arrival rate**, and the callback burst must be a *separate, independently
   controlled* arrival rate. A closed-loop generator cannot produce the overrun you need,
   because it throttles itself when the service slows — which is exactly the effect you are
   trying to create.
3. **Record the unaffected endpoints.** Catalogue p99 during a callback burst is the blast-
   radius measurement.
4. Run long enough to see promotion, not just allocation. Minutes, not seconds.
5. Three configurations: unbounded, bounded-lossy, durable. Report all three even though
   only one ships — the comparison is the argument.

---

## Practice exercises

### 1 — easy: find every place demand dies in `orderflow`

Produce a table with one row per reactive chain in the codebase. Columns: file and line,
source type (pull / pull-with-prefetch / push), where the first bound is, what the overflow
strategy is, whether drops are counted, and what the business consequence of a drop would
be.

Then answer three questions with evidence:

1. Which chain has the largest unbounded buffer, and how do you know?
2. Which chain's `subscribe()` requests unbounded, and does that matter there?
3. Print `Queues.SMALL_BUFFER_SIZE` and `Queues.XS_BUFFER_SIZE` for your BOM's Reactor.

**Every row must cite the line of code that justifies it.** A row you inferred rather than
read does not count.

### 2 — medium: the audit (combines Topics 01–104)

Audit `orderflow`'s ingestion path end to end and write it up. For each finding: file, line,
the topic it comes from, the symptom under load, the fix, and the risk of the fix.

Find at least:

1. Every zero-argument buffering operator and every `Sinks` builder without a size
   (Trap 1).
2. Every `subscribe(lambda)` on a path where demand matters (Trap 4), **and** every
   `subscribe` with no error handler — the second is often the more urgent bug.
3. Every blocking call reachable from an event-loop thread (Trap 2, Topic 103). Prove it
   with BlockHound rather than by reading.
4. Every `block()` in `src/main` (Trap 3).
5. Every `flatMap` with a default concurrency, and what the right number would be, derived
   from the Hikari pool size or the downstream's documented limit (Trap 5, Topic 109).
6. Every place a `Mono` is built and never subscribed (Topic 104, Trap 1) — a silent no-op.
7. Every lossy strategy without a counter or a rationale comment (Trap 6).
8. The interaction with Topic 90: your bounded queue and rejection policy on the MVC side
   is *the same idea* as an overflow strategy. Write the sentence that connects them.

**Deliverable:** the audit table, plus one chain fixed properly with a before/after
measurement plan.

### 3 — hard: size the payment-callback pipeline from data

Design, implement and defend the callback-ingestion pipeline for `orderflow`, with every
number derived from a measurement rather than chosen.

**Method:**

1. **Measure the service rate.** How many callbacks per second can one pod process,
   end to end, at the Topic 65 baseline with the catalogue and order traffic also running?
   This is bounded by Postgres and the Hikari pool, not by Reactor. Use the Topic 65
   harness, not a microbenchmark.
2. **Measure or obtain the arrival rate**, including the worst settlement burst you can
   justify — from provider documentation, from production data, or from an explicit
   assumption you write down and label as an assumption.
3. **Compute the required buffer** as the integral of `arrival - service` over the burst
   duration, times the size of a callback. Show the arithmetic.
4. **Decide where that buffer lives.** Heap? A Postgres inbox? A Kafka topic (Topics
   113–114)? Justify it against requirements 2 and 6 from Example 2 — no loss, and pods
   restart.
5. **Implement it**, with the durable inbox and the bounded in-memory pump.
6. **Choose and document the overflow behaviour at the door.** What HTTP status does the
   endpoint return when the inbox is full, and what does the provider do with it? Read the
   provider's retry documentation; if you do not have it, write down what you assumed.
7. **Prove it.** Run the burst. Show: bounded memory, zero data loss, the 503 rate, the
   provider retry behaviour, and no degradation of the catalogue endpoint.
8. **Then break it deliberately** and show the pipeline recovers: kill the pod mid-burst
   and demonstrate that no callback was lost.

**Deliverable — at most two pages:**

- The capacity arithmetic, with each input labelled measured or assumed.
- The chosen design and the two you rejected, with the reason.
- The overflow policy in the words you would use with the payments product owner —
  **specifically: which callbacks, if any, may be lost, and what happens to that customer.**
- The measurement table proving bounded memory and zero loss.
- Named risks and a rollback trigger.

**The grading criterion is whether a payments product owner could read the overflow policy
and either approve it or object to something specific.** If they cannot, the decision is
still hiding inside an operator argument, and this topic has not landed.

---

## Interview questions

### Q1 — "What is backpressure, and why does Reactor have it when RxJS doesn't?"

**MID-LEVEL.** "It's when the consumer can't keep up with the producer, so the producer
slows down. Reactor handles it automatically with operators like `onBackpressureBuffer`."

**SENIOR.** "Backpressure in Reactive Streams is a specific mechanism, not a general idea:
`request(n)` is a demand signal that travels **upstream** from the subscriber to the
publisher. The subscriber says 'I can take n more'; a compliant publisher emits at most n
and stops. Every operator does its own demand accounting — `map` passes n through, `filter`
re-requests to replace what it dropped, `concatMap` requests one at a time, `buffer(k)`
requests n times k.

RxJS doesn't have it because RxJS 5 dropped it. Its `Observable` is a pure push model with
no `request`. What it offers instead — `throttleTime`, `debounceTime`, `bufferTime`,
`sampleTime` — are all lossy or accumulating operations at the *consumer*. The producer is
never told anything. That's the distinction that matters: 'the producer was slowed down'
versus 'the consumer coped'.

Java needed a specification because there are many publisher implementations that must
interoperate — Reactor, RxJava, R2DBC drivers, Kafka clients, the JDK's own `Flow` — and
demand has to survive crossing between them.

The practical consequence I care about is that backpressure only exists along a path where
demand actually propagates. Bridge a push source — a webhook, a Kafka listener, a `Sink` —
without translating pushes into demand and you have an unbounded buffer, which is an OOM
with a delay."

**What separates them.** Naming `request(n)` and its direction; naming per-operator demand
translation; knowing precisely why RxJS's operators are not backpressure; and ending on the
push-source caveat, which is where the real bugs are.

**Follow-up:** *"Where does demand die in a typical service?"* → `Flux.create`, any `Sinks`,
any unbounded buffering operator, and `subscribe(lambda)` — which requests `Long.MAX_VALUE`
and quietly disables everything above it. I read the chain upward from `subscribe()` and
find the first bound.

### Q2 — "Your service OOMs under load. It's fully reactive. Where do you look?"

**MID-LEVEL.** "Take a heap dump and see what's using the memory. Maybe increase the heap
or add `onBackpressureBuffer`."

**SENIOR.** "First, the shape: is old-gen occupancy climbing across young collections and
never recovering? That's retention, not allocation rate, and it points at a buffer rather
than at garbage.

Then the dump. `-XX:+HeapDumpOnOutOfMemoryError`, open the dominator tree in MAT, find the
largest retained set. If it's a Reactor internal queue holding domain objects, I have my
answer, and the dominator's incoming reference names the operator.

Then the source classification, which is the actual diagnosis. Is the source a pull source,
a pull source with protocol-level flow control, or a push source? Only push sources produce
this failure, and there's usually exactly one in a service — a webhook endpoint, a Kafka
bridge, an SSE feed.

Then I'd check `subscribe()` — a lambda subscriber requests unbounded and disables every
bound above it, which produces exactly this symptom in a chain that looks correct.

Two things I would *not* do first. I wouldn't raise the heap: heap size doesn't fix a rate
mismatch, it changes the time to failure. And I wouldn't add `onBackpressureBuffer` without
a size, which is the same bug with a more reassuring name.

The fix depends on what the data is worth. Lossy strategy with a counter for telemetry;
a durable buffer and a 503 with `Retry-After` for anything that must not be lost."

**What separates them.** Distinguishing retention from allocation before touching a tool;
classifying the source; knowing that the fix depends on the data's value rather than being
an operator choice; and explicitly rejecting the two reflexive non-fixes.

**Follow-up:** *"How would you size the buffer?"* → Measure both rates, integrate the
difference over the worst observed burst, multiply by item size, add headroom. And check
the average rates first — if arrivals exceed service on average, no buffer size works, and
that's a capacity problem, not a tuning problem.

### Q3 — "A payment callback endpoint returns 202 and buffers in memory. What do you think?"

**MID-LEVEL.** "You should bound the buffer so it doesn't run out of memory —
`onBackpressureBuffer` with a size and a drop strategy."

**SENIOR.** "Bounding the buffer is necessary and it isn't the main problem.

The main problem is that 202 is a promise. It tells the provider we've taken
responsibility, which means the provider stops retrying. If we then lose the item — by
dropping it on overflow, or by the pod restarting, or by OOMing — that payment callback is
gone permanently, and the customer paid for an order that never ships. We disabled the
provider's retry mechanism with our own response code.

So I'd invert it: persist first, acknowledge second. Write the callback to a Postgres inbox
table with a unique constraint on the provider's callback id, then return 202. Process from
that table at whatever rate the database sustains. Now the buffer is a disk, the bound is a
disk-space alert, and overflow is a paging problem instead of a data-loss problem.

And when the inbox genuinely can't accept — 503 with `Retry-After`, not 202. That turns
overload into a conversation with the sender, which is what backpressure looks like when it
crosses a network boundary and you can't send a `request(n)`.

The bounded in-memory buffer with `DROP_OLDEST` is still right in this system — for the
admin dashboard's live event stream, where losing an event costs nothing. I'd want the
rationale and the approver written in a comment above it, because otherwise someone reuses
the pattern where it costs money."

**What separates them.** Recognising that the HTTP status code is the real defect;
proposing durability rather than a bigger buffer; knowing that 503 plus `Retry-After` *is*
backpressure across a network; and separating where lossy is acceptable from where it is
not.

**Follow-up:** *"What if the provider doesn't retry on 503?"* → Then read their
documentation and find what they do retry on; if the answer is nothing, the inbox must
never be full, which turns it into a capacity commitment with an alert rather than a
strategy choice. And I'd raise it with them, because a webhook API with no retry is a
liability for both sides.

### Q4 — "Does `flatMap(fn, 8)` give you backpressure?"

**MID-LEVEL.** "Yes, it limits it to 8 at a time, so the upstream can't overwhelm you."

**SENIOR.** "It limits **concurrency** — at most 8 inner publishers subscribed at once —
which is genuinely useful and is not the same thing.

It doesn't bound how much each inner publisher emits, and it keeps its own prefetch queue
for results from inners that completed while the downstream isn't consuming. So it's a
concurrency limiter with a buffer, not end-to-end demand propagation.

Where it matters practically: if the upstream is a push source, `flatMap` won't slow it
down. The pressure has to be absorbed somewhere between the source and the flatMap, and if
there's no bounded buffer there, the mismatch accumulates.

Two things I'd do. Set the concurrency from a real constraint — the Hikari pool size, or a
documented downstream limit — rather than leaving the default. And if order matters, use
`concatMap`, which requests one at a time and *is* genuinely demand-driven, at the cost of
no concurrency at all.

I'd also print `Queues.SMALL_BUFFER_SIZE` rather than quoting the default prefetch from
memory — those constants come from the BOM's Reactor version."

**What separates them.** The concurrency-versus-demand distinction, which is the single
most common misconception in reactive code; sourcing the concurrency number from a real
constraint; and refusing to quote a version-dependent default.

**Follow-up:** *"What if the inner publishers are slow and unbounded in number?"* → Then
`flatMap`'s queue grows while the 8 in-flight ones finish, and you need `limitRate` or a
bounded buffer above it. Also worth asking whether unbounded fan-out was ever the intent —
usually the source should be chunked.

### Q5 — "Virtual threads give you cheap blocking concurrency. Do you still need backpressure?"

**MID-LEVEL.** "Not really — with virtual threads you can just have a thread per item and
they're cheap, so nothing queues up."

**SENIOR.** "You need it more, and it's a different problem entirely.

Virtual threads solve thread *cost*. They let you have fifty thousand concurrent blocked
operations for the price of fifty thousand heap continuations. What they don't do is tell a
producer to slow down. If a webhook provider sends fifty thousand callbacks a second and
your database handles five hundred, virtual threads let you create fifty thousand threads
that all queue on the connection pool. You've moved the queue from a buffer into a thread
pile, and you've made it *more* expensive, because each waiter now carries a continuation
instead of being a row in a queue.

So it's the same failure with a different shape: unbounded work in flight against a bounded
resource. The fixes are the same too — a `Semaphore` bulkhead, a bounded queue with a
rejection policy, a 503 at the door, or a durable buffer.

This is precisely the residual that keeps reactive relevant. Loom gives you thread economy
and takes back the imperative model. It does not give you demand propagation, and demand
propagation is what reactive uniquely provides. So my line is: high-concurrency
request/response goes on virtual threads; a genuinely streaming path with a producer that
can outrun the consumer and cannot be told to stop is where reactive still earns its
complexity."

**What separates them.** Recognising it as a resource-bound problem rather than a
thread-count problem; naming the "you moved the queue" shape; and landing the Topic 107
boundary precisely rather than as a slogan.

**Follow-up:** *"So where exactly would you put the boundary in `orderflow`?"* → Virtual
threads for the MVC core — catalogue, orders, placement — because those are request/response
and the caller waiting is itself the backpressure. Reactive, or a durable inbox, for
payment-callback ingestion, because that is the only genuine push source in the system.
And I'd benchmark both before committing, which is Topic 107's deliverable.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `filter` requests a replacement item for every one it drops. Construct a source and a
   predicate where this behaviour is a problem, and say what you would do about it.

2. `request(Long.MAX_VALUE)` means unbounded. Argue that the specification should have made
   unbounded impossible — every subscriber must name a number. Then argue against it. What
   breaks in each world?

3. An unbounded buffer loses everything at the OOM; a bounded lossy buffer loses a
   predictable fraction continuously. Construct a system where the first is genuinely
   preferable, and say what property makes it so.

4. A bounded buffer does not add capacity. Explain that in queueing terms, precisely enough
   that someone could act on it. What is the one measurement that decides whether a buffer
   will ever be enough?

5. Node streams' backpressure is a boolean; Reactive Streams' is a numeric credit. Design
   a system where the boolean version is strictly better, and say why.

6. A durable inbox turns an overflow into a 503. That is backpressure across a network
   boundary. Name two other places in `orderflow` where a status code or a protocol feature
   is doing the same job, and one place where it should be and is not.

7. You have a chain: push source, `onBackpressureBuffer(1000, DROP_OLDEST)`, `flatMap(fn,
   8)`, `subscribe(lambda)`. Trace the demand from the subscriber upward and say exactly
   what each operator requests. Then say what changes if you replace the lambda subscriber
   with one that requests 10 at a time.

---

## Quick reference card

### The mechanism in five lines

```
request(n) travels UPSTREAM: consumer -> producer. "Send me n more."
Every operator translates demand: map n->n, filter n->n+replacements,
   buffer(k) n->n*k, concatMap n->1, flatMap(c) n->c inner subscriptions.
Backpressure exists only where demand PROPAGATES.
A push source has no demand. Bridging one without a bound = unbounded buffer = OOM.
request(Long.MAX_VALUE) = unbounded = backpressure OFF. subscribe(lambda) does this.
```

### Source taxonomy — classify every source you wire up

```
PULL           Flux.range, fromIterable, generate        -> backpressure free
PULL+PREFETCH  R2DBC, Reactor Kafka                      -> works, has a prefetch
PUSH           Flux.create, Sinks, callbacks, SSE, bridges -> NO backpressure. You must
                                                              choose an overflow policy.
```

### Overflow strategies

```java
onBackpressureBuffer(n, onDrop, DROP_OLDEST)  // newest data most valuable
onBackpressureBuffer(n, onDrop, DROP_LATEST)  // oldest must be processed in order
onBackpressureBuffer(n, onDrop, ERROR)        // fail loudly; needs a resubscribe strategy
onBackpressureDrop(onDrop)                    // sampling, telemetry, presence
onBackpressureLatest()                        // "current value" semantics only
onBackpressureBuffer()                        // <-- NEVER. Unbounded. This is the bug.
limitRate(n)                                  // cap demand regardless of the subscriber
```

### Where demand dies — the diagnostic checklist

```
Flux.create(...)                  -- unless you pass a bounding strategy
Sinks.many()...                   -- you emit on your own schedule
onBackpressureBuffer()            -- zero-arg overloads, any operator
collectList(), cache(), toStream(), toIterable(), block()
subscribe(lambda)                 -- requests Long.MAX_VALUE
any bridge you wrote from a callback API
```

### Print, do not quote

```java
reactor.util.concurrent.Queues.SMALL_BUFFER_SIZE   // default prefetch source
reactor.util.concurrent.Queues.XS_BUFFER_SIZE
// -Dreactor.bufferSize.small=NNN to change it
```

### Diagnostics

```java
flux.log("stage-name")                  // prints request(n) -- THE instrument
flux.doOnRequest(n -> ...)              // demand at one point
flux.doOnNext(x -> log.info("thread={}", Thread.currentThread().getName()))
```

```bash
-Xlog:gc*:file=gc.log:time,uptime       # old-gen climbing = retention, not garbage
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
jcmd <pid> GC.class_histogram | head -30
# then MAT dominator tree (Topic 79)
```

### The two sentences

```
An unbounded buffer defers data loss and then loses EVERYTHING at once.
A bounded buffer adds no capacity; it converts unpredictable total loss into
  predictable partial loss that you can measure, alert on, and get approved.
```

### Gotchas checklist

- [ ] Every buffering operator has a numeric argument. No exceptions.
- [ ] Every drop increments a counter, even where the counter should read zero.
- [ ] Every lossy strategy has a comment naming the rationale and the approver.
- [ ] Every production `subscribe` has an error handler.
- [ ] `flatMap` concurrency comes from a real constraint, not the default.
- [ ] No `block()` in `src/main` reactive code.
- [ ] BlockHound in CI for anything touching an event loop.
- [ ] Buffer size derived from a measured rate mismatch, with the arithmetic written down.
- [ ] Anything that must not be lost gets a durable buffer, not a Reactor operator.
- [ ] 202 means "I have taken responsibility". Do not say it before you have.

---

## When would I use this at work?

**1. Reviewing any PR that bridges an external push source.**

A Kafka listener writing into a `Sink`, an SSE endpoint, a webhook handler, a WebSocket
fan-out. Three questions, all mechanical: is the buffer bounded, is the drop counted, and
is losing an item acceptable to whoever owns this data? Those three comments take a minute
and prevent the most expensive bug in this document. The third one is the one nobody else
asks.

**2. Diagnosing a slow memory climb in a reactive service.**

Old-gen occupancy rising across young collections and never recovering is retention, not
allocation. In a reactive service that is a buffer until proven otherwise. Heap dump,
dominator tree, find the operator, classify the source. Twenty minutes, and it is a
diagnosis rather than a guess — as opposed to raising `-Xmx` and buying a week.

**3. Turning an overflow policy into a decision someone actually approves.**

This is the underrated one. Somewhere in every event-driven system there is an operator
argument that silently decides which data is allowed to disappear. Finding it, writing
down what it means in business terms — "under a settlement burst above N/second we would
drop the oldest callbacks, meaning a customer is charged and not fulfilled" — and taking
that sentence to the person who owns the money is a senior contribution that has nothing to
do with Reactor's API and everything to do with understanding it.

---

## Connected topics

**Prerequisites:**

- **68 — Heap, generations, TLAB:** where the buffer lives, and why promotion turns a rate
  mismatch into a GC death spiral.
- **79 — Memory leaks and heap dumps:** the dominator tree is how you name the buffer from
  a dump. An unbounded backpressure buffer is a retention problem, and the tooling is
  identical.
- **90 — Executors and pool sizing:** a bounded queue plus a rejection policy **is**
  backpressure, in the imperative world. `CallerRunsPolicy` is the imperative
  `onBackpressureBuffer(n, ERROR)`. Same idea, different vocabulary.
- **93 — Bounded queues:** backpressure by another name, and the direct ancestor of this
  topic.
- **103 — NIO and Netty's event loop:** the threads underneath. Trap 2 is the same bug this
  topic and that one both warn about, and BlockHound is the shared instrument.
- **104 — `Mono`/`Flux`:** assembly versus subscription, cold versus hot, and the
  two-phase subscription walk that carries demand upward.

**Also relevant:**

- **25 — Parallel streams:** no backpressure at all, and the same shape of failure.
- **65 — The load baseline:** where you measure the arrival and service rates that size
  every buffer in this document.
- **71 / 72 — G1 and ZGC:** which collector you are on changes how a filling buffer
  presents — as pause time or as allocation stalls.
- **97 — Coordination primitives:** `Semaphore` as an admission-control alternative to a
  buffer.
- **101 — Virtual threads:** the thing that does **not** solve this problem, and why.
- **102 — Structured concurrency:** also does not solve it. Both are about lifetime and
  cost, not about rate.

**This unlocks:**

- **106 — WebFlux vs MVC:** where the ingestion path lives, and whether it is worth a
  separate module.
- **107 — The Loom-vs-reactive decision:** **this topic is the technical core of that
  recommendation.** Backpressure is the one capability reactive uniquely provides, and
  everything else in the comparison is a trade you could go either way on.
- **108 — Reactive debugging:** when the overflow fires at 3am, the stack trace shows the
  subscription stack and not your assembly point. That is the next document.
- **113–114 — Kafka:** the canonical push source, and the canonical durable buffer. Reactor
  Kafka translates demand into partition pausing, which is the good case from the source
  taxonomy.
- **115 — The outbox pattern:** "persist first, process later" pointed the other way. Same
  insight, opposite direction.
- **118 — Metrics:** the received/processed/dropped triple and the alerts built on it.

---

*Java baseline 21. The Reactive Streams specification (`org.reactivestreams`, and
`java.util.concurrent.Flow` in the JDK since Java 9) is stable and is not going to change.
Reactor's operator names in this document are stable API, but **default prefetch sizes and
the default overflow strategy of `Flux.create` are version-dependent** — print
`Queues.SMALL_BUFFER_SIZE` and read your BOM's javadoc rather than quoting a number from
here. Reactor and Netty versions come from the Spring Boot BOM; do not pin them by hand and
do not trust a version number quoted in any document, including this one.*
