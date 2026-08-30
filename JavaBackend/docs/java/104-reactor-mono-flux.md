# 104 — Reactor I: `Mono`/`Flux`, Operators, Assembly vs Subscription Time

## Phase: 10 — Reactive & Async at Scale
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: the composition layer for `orderflow`'s payment-callback ingestion and for the order-enrichment read path. Topic 103 gave you the threads; this topic gives you what runs on them.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **A `Mono` or a `Flux` is a blueprint, not a running computation.**
>
> When you write `.map(...).filter(...).flatMap(...)`, you are at **ASSEMBLY TIME**.
> Each operator allocates a small object that holds a reference to its upstream. You
> have built a linked chain of `Publisher` objects. **Nothing has executed. No thread
> has been touched. No I/O has been issued.**
>
> When something calls `subscribe()`, you are at **SUBSCRIPTION TIME**. The
> subscription walks the chain from the bottom upward, and each operator creates a
> `Subscriber` that subscribes to its upstream. Only when the source receives that
> subscription — and only after the subscriber signals **demand** with `request(n)` —
> does anything actually run.
>
> **A `Mono` you build and never subscribe to is a silent no-op.** No exception. No
> warning. No log line. The side effect you wrote simply never happens.

That last paragraph is the bug you will write. Not "might write" — will. Everyone
arriving from Promises writes it at least once, and the reason is that your entire
prior model says otherwise.

---

## The bridge from what you know

### `Flux` ≈ RxJS `Observable`. Transfer everything.

```ts
// RxJS
from(orderIds).pipe(
  map(id => id * 2),
  filter(id => id > 10),
  mergeMap(id => this.http.get(`/orders/${id}`)),
  catchError(err => of(null)),
).subscribe(order => console.log(order));
```

```java
// Reactor
Flux.fromIterable(orderIds)
    .map(id -> id * 2)
    .filter(id -> id > 10)
    .flatMap(id -> webClient.get().uri("/orders/{id}", id)
                            .retrieve().bodyToMono(Order.class))
    .onErrorResume(e -> Mono.empty())
    .subscribe(order -> log.info("{}", order));
```

Same shape, same laziness, same operator vocabulary, same `subscribe()` at the end.

**Verdict: HONEST ANALOGUE.** This is the best-transferring API in the whole
curriculum. Your RxJS knowledge is directly usable; you mostly need a translation
table, which is in the Quick reference card.

| RxJS | Reactor | Note |
|---|---|---|
| `Observable<T>` | `Flux<T>` | 0..N elements |
| `Observable` of 0–1, `Promise` | `Mono<T>` | 0..1 elements |
| `of(x)` | `Mono.just(x)` / `Flux.just(x)` | both eager in their argument |
| `from(array)` | `Flux.fromIterable(list)` | |
| `defer(() => ...)` | `Mono.defer(() -> ...)` / `Flux.defer(...)` | the laziness escape hatch |
| `map` | `map` | synchronous 1:1 |
| `mergeMap` / `flatMap` | `flatMap` | **concurrent, interleaved** |
| `concatMap` | `concatMap` | **sequential, ordered** |
| `switchMap` | `switchMap` | cancel previous on new element |
| `exhaustMap` | `flatMap` with a guard, or `Flux.onBackpressure*` patterns | no exact 1:1; check current docs |
| `tap` | `doOnNext` / `doOnEach` | side effects only |
| `catchError` | `onErrorResume` / `onErrorReturn` / `onErrorMap` | three distinct intents |
| `retry` / `retryWhen` | `retry` / `retryWhen(Retry.backoff(...))` | Reactor's `Retry` builder is richer |
| `takeUntil` | `takeUntil` / `takeUntilOther` | |
| `combineLatest` | `Flux.combineLatest` | |
| `forkJoin` | `Mono.zip` | |
| `Subject` | `Sinks.many()` | `Processor` types are deprecated in favour of `Sinks` |
| `shareReplay` | `cache()` / `replay().refCount()` | |
| **(nothing)** | **`request(n)` backpressure** | **Topic 105. RxJS has no equivalent.** |

### The break that matters: `Mono` is COLD and LAZY; a `Promise` is EAGER

This is where your model must actually change, and it is worth writing out
side by side.

```ts
// TypeScript. The fetch has ALREADY STARTED by the time this line finishes.
const p = fetch('/payments/authorize', { method: 'POST' });
// Even if you never await p, the request went out.
// Even if you throw p away, the server was called.
```

```java
// Java. NOTHING has happened. This is a description of a request.
Mono<AuthResult> m = webClient.post().uri("/payments/authorize")
                              .retrieve().bodyToMono(AuthResult.class);
// If you never subscribe, no HTTP request is ever sent.
// The object m is a small blueprint on the heap. That is all it is.
```

A `Promise` is a **handle to work already in flight**. A `Mono` is a **recipe for work
not yet started**. Everything downstream of that difference:

| | `Promise<T>` | `Mono<T>` |
|---|---|---|
| When does the work start | at construction | at `subscribe()` |
| Effect of not consuming it | work still happened | **work never happened** |
| Effect of consuming it twice | one execution, two callbacks | **two executions, two sets of side effects** |
| Can you retry it | no — you must rebuild it | **yes — `retry()` just re-subscribes** |
| Can you cancel it | not really (`AbortController` is a side channel) | **yes — `Subscription.cancel()`** |
| Is it composable before it runs | no | **yes — that is the point** |

Two things fall out of that table that are genuinely better than Promises, and you
should be able to name both in an interview:

1. **`retry()` works.** Because a cold `Mono` is a recipe, retrying means
   re-subscribing, which re-runs the recipe. With Promises you cannot retry a
   `Promise`; you must retry the *function that creates* it. That difference is why
   RxJS retry logic is clean and Promise retry logic is a wrapper function.
2. **Cancellation is first-class.** `Subscription.cancel()` propagates upstream and
   the HTTP client can genuinely abort the request. A `Promise` has no cancel; the
   work continues and you ignore the result.

And one thing that is genuinely worse:

3. **You can silently do nothing.** A dropped `Promise` still ran. A dropped `Mono`
   did not.

**Verdict: PARTIAL — and this is the bug you will write.**

### The second break: `flatMap` interleaves, `concatMap` does not

Promises never forced you to think about this, because `Promise.all` is unordered
execution with ordered *results*, and `for await` is ordered execution. In Reactor
the choice is explicit and it is in the operator name.

```java
Flux.just("A", "B", "C")
    .flatMap(id -> callServiceTakingRandomTime(id))     // may emit C, A, B
    .subscribe(System.out::println);

Flux.just("A", "B", "C")
    .concatMap(id -> callServiceTakingRandomTime(id))   // always emits A, B, C
    .subscribe(System.out::println);
```

| Operator | Concurrency | Output order | Use when |
|---|---|---|---|
| `flatMap(f)` | up to 256 inner subscriptions by default | **arrival order — interleaved** | order does not matter and you want throughput |
| `flatMap(f, n)` | at most `n` at once | interleaved | order does not matter and you must bound fan-out |
| `flatMapSequential(f, n)` | up to `n` at once | **source order** (buffers to reorder) | you want concurrency *and* order, and can afford the buffer |
| `concatMap(f)` | **1 at a time** | source order | order matters and each step must finish before the next starts |
| `switchMap(f)` | 1, cancelling the previous | latest only | search-as-you-type; only the newest result matters |

The default concurrency of `flatMap` is 256 (`Queues.SMALL_BUFFER_SIZE`). That number
is not a suggestion: `flatMap` over a 5,000-element `Flux` against a service with a
10-connection pool will open up to 256 in-flight calls and drown it. Bound it.

> **Uncertainty, flagged:** the default concurrency and prefetch constants
> (`Queues.SMALL_BUFFER_SIZE`, `XS_BUFFER_SIZE`) are Reactor implementation details
> that have been stable for years but are not part of a compatibility promise. Confirm
> the value in the Reactor reference docs for the version your Boot BOM resolves,
> rather than hardcoding 256 into a design document.

### What has NO analogue at all

Backpressure. `request(n)`. A consumer telling a producer how much it can accept, with
that signal travelling **upstream**. RxJS does not have it. Promises do not have it.
There is nothing to transfer. That is Topic 105, and it is deliberately the next
document.

---

## What is this?

### The Reactive Streams specification — four interfaces, one contract

Reactor is an implementation of a small specification. The whole thing is four
interfaces:

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> s);
}

public interface Subscriber<T> {
    void onSubscribe(Subscription s);   // "here is your control handle"
    void onNext(T t);                   // "here is an element"
    void onError(Throwable t);          // terminal
    void onComplete();                  // terminal
}

public interface Subscription {
    void request(long n);               // "I can accept n more"  <- UPSTREAM
    void cancel();                      // "stop"                 <- UPSTREAM
}

public interface Processor<T, R> extends Subscriber<T>, Publisher<R> { }
```

The contract, which the whole ecosystem depends on:

```
onSubscribe  onNext*  (onError | onComplete)?
```

Exactly one `onSubscribe`, then zero or more `onNext`, then **at most one** terminal
signal. After a terminal signal, nothing more. And — this is the part people miss —
**a `Publisher` must not emit more `onNext` calls than have been requested.**

Java 9 adopted these same four shapes into the JDK as `java.util.concurrent.Flow`.
They are structurally identical; adapters exist in both directions
(`JdkFlowAdapter` in Reactor). In practice you will write against
`org.reactivestreams` because that is what Spring, R2DBC and the driver ecosystem
use.

### `Mono` and `Flux`

| | `Mono<T>` | `Flux<T>` |
|---|---|---|
| Cardinality | 0 or 1 element, then complete | 0..N elements, then complete (or never) |
| Analogue | `Promise<T \| undefined>` / RxJS `Observable` of ≤1 | RxJS `Observable` |
| Typical use | one HTTP response, one database row, one command result | a stream of callbacks, a result set, a server-sent event feed |
| Empty case | `Mono.empty()` — completes with no element | `Flux.empty()` |
| "Void" case | `Mono<Void>` — a completion signal only | — |

`Mono<Void>` deserves a note because it is everywhere in Spring's reactive APIs and
looks strange coming from TypeScript. It means "this operation has no result, but you
still need to know when it finished and whether it failed". It is `Promise<void>`, and
you must still subscribe to it or nothing happens — which makes it the single most
common instance of this topic's headline bug.

### Cold vs hot

- **Cold:** each subscriber triggers its own independent execution. `Mono` from a
  `WebClient` call, `Flux.fromIterable`, `Mono.fromCallable`. Two subscribers means two
  HTTP requests.
- **Hot:** the source is producing regardless, and subscribers see whatever is
  flowing at the time they arrive. A `Sinks.many()` fed by a Kafka listener, a
  `share()`d flux, a UI event stream.

Everything you get back from Spring's reactive APIs is cold by default. Making
something hot is a deliberate act: `share()`, `publish().refCount()`, `cache()`, or a
`Sinks` instance.

That distinction is exactly why Topic 105 exists. A cold source can be told to slow
down — it is generating on demand. A hot source cannot; it is going to produce whether
you are ready or not. **Backpressure is easy on cold sources and is the entire problem
on hot ones.**

### Schedulers, in one table

You met the threads in Topic 103. Reactor names them:

| Scheduler | Threads | For |
|---|---|---|
| `Schedulers.immediate()` | the calling thread | no switch at all |
| `Schedulers.single()` | one reusable thread | low-volume serialised work |
| `Schedulers.parallel()` | `availableProcessors()` threads | **CPU-bound** work. Never blocking work. |
| `Schedulers.boundedElastic()` | elastic, bounded (a large cap), with a queue | **blocking** work. The sanctioned offload. |
| `Schedulers.newBoundedElastic(n, q, name)` | exactly `n` | blocking work with a **deliberate** bound |
| the event loop | `reactor-http-nio-*` | wherever you did not switch |

And the two operators that use them, which are constantly confused:

- **`subscribeOn(scheduler)`** — affects where the **subscription** happens, and
  therefore where the **source** does its work. It affects the whole chain upward from
  the source. Position in the chain barely matters, and only the first one takes
  effect.
- **`publishOn(scheduler)`** — affects where everything **downstream of this point**
  runs. Position matters completely. You can have several.

```java
Mono.fromCallable(this::blockingJdbcQuery)     // runs on jdbcScheduler
    .subscribeOn(jdbcScheduler)                //   <- because of this
    .map(this::toDto)                          // still on jdbcScheduler
    .publishOn(Schedulers.parallel())          //   <- switch here
    .map(this::expensiveCpuTransform)          // now on parallel-N
    .subscribe();
```

---

## Why does it matter?

**1. The silent no-op is a data-loss bug with no symptom.**
A `Mono` that writes an audit row, publishes an event, or debits a wallet, and is never
subscribed, does nothing. No exception, no metric, no log. You find it from a business
number weeks later — the same failure shape as Topic 40's self-invocation trap, and for
a structurally similar reason: you wrote something that looks like a call and is
actually a description.

**2. Assembly vs subscription decides which thread runs your code.**
Topic 103 taught you that blocking the wrong thread stalls every connection on a loop.
Whether your blocking call lands on a loop or on `boundedElastic` is decided by whether
it runs at assembly time or subscription time, and by where `subscribeOn` sits. Getting
this wrong is how a correct-looking `subscribeOn` fails to move anything.

**3. `flatMap` vs `concatMap` is a correctness decision that looks like a style
choice.** Ledger entries applied out of order, inventory decrements racing, state
transitions arriving backwards — all of them are `flatMap` where `concatMap` was
required, and all of them pass tests with one element.

**4. It is the vocabulary for the rest of the phase.** Backpressure (105), threading
models (106), the Loom comparison (107) and reactive debugging (108) are all stated in
terms of assembly, subscription, signals and demand. Without this document they are
noises.

**5. Interviewers use "how is a Mono different from a Promise" as a calibration
question.** "It's basically the same" is a mid answer. Cold/lazy, retry-by-
resubscription, cancellation, and the silent-no-op hazard is a senior one.

---

## Machine-level reality

### What `.map()` actually allocates

```java
Flux<String> f = Flux.just(1, 2, 3)
                     .map(i -> i * 2)
                     .filter(i -> i > 2)
                     .map(Object::toString);
```

At assembly time this creates **four objects**, each holding a reference to the one
before it:

```
FluxMapFuseable(  source = FluxFilterFuseable(  source = FluxMapFuseable(  source = FluxArray ) ) )
      ^ Object::toString              ^ i > 2                  ^ i * 2            ^ [1,2,3]
```

Each operator is itself a `Publisher`. Assembly is **just object construction** — it
allocates a handful of small objects and returns. It touches no thread pool, issues no
I/O, and can happen on any thread including the event loop, safely.

That is why `Flux<String> f = ...` on its own line is free, and why building the same
chain in a hot loop is a real (if small) allocation cost — Topic 68's material.

### What `subscribe()` actually does — the two-phase walk

This is the mechanism. Learn it and half of Topics 105 and 108 become obvious.

**Phase 1 — subscription travels UPSTREAM.**

```
f.subscribe(consumer)
  -> FluxMapFuseable.subscribe(LambdaSubscriber)
       creates MapSubscriber(downstream = LambdaSubscriber)
       -> FluxFilterFuseable.subscribe(MapSubscriber)
            creates FilterSubscriber(downstream = MapSubscriber)
            -> FluxMapFuseable.subscribe(FilterSubscriber)
                 creates MapSubscriber2(downstream = FilterSubscriber)
                 -> FluxArray.subscribe(MapSubscriber2)
                      creates ArraySubscription
```

You started at the bottom of the chain and walked to the top. Each operator built a
`Subscriber` instance. **This is where the runtime objects come from — assembly built
`Publisher`s, subscription builds `Subscriber`s.**

**Phase 2 — `onSubscribe` travels DOWNSTREAM, then demand travels UPSTREAM.**

```
ArraySubscription
  -> MapSubscriber2.onSubscribe(sub)
       -> FilterSubscriber.onSubscribe(sub)
            -> MapSubscriber.onSubscribe(sub)
                 -> LambdaSubscriber.onSubscribe(sub)
                      calls sub.request(Long.MAX_VALUE)   <-- DEMAND, going UP
```

Then, and only then, `FluxArray` starts calling `onNext` downward.

Three consequences you will use constantly:

1. **`subscribe()` with a lambda requests `Long.MAX_VALUE`** — "unbounded, send me
   everything". That is the default and it is why most people never notice backpressure
   exists. Topic 105 is what happens when you cannot say that.
2. **`Context` is attached to the subscription and travels upstream**, which is why
   `contextWrite(...)` affects operators **above** it and not below. That sentence is
   the single most confusing thing about Reactor and it falls straight out of this
   diagram. Topic 108 uses it.
3. **`onNext` runs on whatever thread called it.** The source's thread propagates
   downward through every operator until a `publishOn` changes it. So `map` does not
   have a thread of its own; it runs on whoever handed it an element.

### Signals, and what each one actually does

| Signal | Direction | What the operator typically does |
|---|---|---|
| `subscribe(Subscriber)` | downstream → upstream | build my `Subscriber`, subscribe it to my source |
| `onSubscribe(Subscription)` | upstream → downstream | store the subscription, pass a wrapped one down |
| `onNext(T)` | upstream → downstream | apply the transform, call `downstream.onNext(result)` |
| `onError(Throwable)` | upstream → downstream | **terminal.** Pass it on; do not emit again. |
| `onComplete()` | upstream → downstream | **terminal.** Pass it on. |
| `request(long n)` | downstream → upstream | adjust my demand accounting, call `upstream.request(...)` |
| `cancel()` | downstream → upstream | stop the source; release resources |

The demand accounting inside an operator is the part people never look at, and it is
worth seeing once:

- **`map`** is 1:1. It passes `request(n)` straight through unchanged. Demand in,
  demand out.
- **`filter`** is 1:0-or-1. When it drops an element it must call
  `upstream.request(1)` itself, or the stream stalls — the downstream requested one
  element and got nothing.
- **`flatMap`** is 1:N. It requests a prefetch amount from upstream, subscribes to
  each inner publisher, and manages a queue plus per-inner demand. This is why
  `flatMap` has both a `concurrency` and a `prefetch` parameter, and why an
  unbounded inner source inside `flatMap` is a memory hazard.
- **`buffer(n)`** turns `request(1)` downstream into `request(n)` upstream. Demand is
  *multiplied*.

**That is what "backpressure propagates" means, concretely: every operator translates
downstream demand into upstream demand.** And the corollary that Topic 105 is built
on: **any operator or bridge that cannot do that translation is where backpressure
dies.**

### Errors are terminal, and that surprises people

```java
Flux.just(1, 2, 0, 4)
    .map(i -> 100 / i)          // throws on the third element
    .subscribe(System.out::println, e -> System.out.println("err " + e));
```

This emits 100, 50, then `onError`. **The 4 is never processed.** An `onError` is a
terminal signal for the whole chain; it is not a per-element skip.

That is the opposite of a `for` loop with a try/catch inside it, and it is a common
source of "why did my batch stop halfway". If you want per-element resilience you must
say so explicitly, and the idiom is to move the error handling *inside* the inner
publisher:

```java
Flux.just(1, 2, 0, 4)
    .flatMap(i -> Mono.fromCallable(() -> 100 / i)
                      .onErrorResume(e -> Mono.empty()))   // contained per element
    .subscribe(System.out::println);
```

The three error operators, which are not interchangeable:

| Operator | Meaning |
|---|---|
| `onErrorReturn(fallbackValue)` | swallow the error, emit one fallback, complete |
| `onErrorResume(e -> otherPublisher)` | swallow the error, switch to another source |
| `onErrorMap(e -> new DomainException(e))` | translate the error; it stays an error |
| `doOnError(e -> log.error(...))` | observe only. **Does not handle it.** |

`doOnError` followed by nothing is a very common mistake: the log line appears, the
error still propagates, and someone believes it was handled.

### Fusion — why the operator count is not the allocation count

Reactor performs **operator fusion**. Adjacent compatible operators can be collapsed so
that elements are not passed through a queue at every step. Two kinds:

- **Macro-fusion:** at assembly time, `Mono.just(x).map(f)` can be recognised and
  replaced with a simpler single operator.
- **Micro-fusion:** at subscription time, operators that implement `Fuseable`
  negotiate a mode (`SYNC`, `ASYNC`) so that a downstream operator can poll elements
  directly from the upstream queue rather than receiving `onNext` calls.

You do not configure this. You need to know it exists for exactly two reasons:

1. It is why a 12-operator chain is not 12 queues and 12 allocations per element.
2. It is why the class names in a reactive stack trace say `FluxMapFuseable` and
   `...$MapFuseableSubscriber` rather than anything you wrote, and why some operators
   you expect to see are missing from the trace. Topic 108's problem, previewed here.

### Where the `Context` lives

Reactor's `Context` is an immutable key-value map carried **in the subscription**, not
in a `ThreadLocal`. `contextWrite(ctx -> ctx.put("correlationId", id))` creates a new
`Context` that is visible to operators **upstream** of the `contextWrite` call, because
the context is assembled during the upstream subscription walk.

This is the correct design — a chain that hops threads cannot use `ThreadLocal` — and
it is completely counter-intuitive the first time. Topic 108 is the full treatment; the
fact you need here is: **the context is part of the subscription, and the subscription
travels upstream.**

---

## Example 1 — minimal

### 1a — the no-op, in six lines

```java
package com.orderflow.lab.reactor;

import reactor.core.publisher.Mono;

public class SilentNoOp {

    public static void main(String[] args) {
        Mono<String> m = Mono.fromCallable(() -> {
            System.out.println("SIDE EFFECT: writing audit row");
            return "written";
        });

        System.out.println("built the Mono. Did the side effect happen?");
        // ... and we never subscribe.
        System.out.println("done.");
    }
}
```

Run it. **"SIDE EFFECT" is never printed.** The `Mono` was constructed, held, and
discarded. In TypeScript the equivalent `Promise` would have run before the second
line executed.

Now add one line at the end:

```java
        m.subscribe();
```

and the side effect appears. That one line is the difference between a working audit
trail and an empty table.

### 1b — assembly time vs subscription time, made visible

```java
package com.orderflow.lab.reactor;

import reactor.core.publisher.Mono;

public class AssemblyVsSubscription {

    static String expensive(String label) {
        System.out.println("  EVALUATED: " + label
                + " on " + Thread.currentThread().getName());
        return label;
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("--- building chains (assembly time) ---");

        Mono<String> eager = Mono.just(expensive("just"));          // runs NOW
        Mono<String> lazy  = Mono.fromCallable(() -> expensive("fromCallable"));
        Mono<String> defer = Mono.defer(() -> Mono.just(expensive("defer")));

        System.out.println("--- chains built. Now subscribing. ---");

        eager.subscribe();
        lazy.subscribe();
        defer.subscribe();

        System.out.println("--- subscribing a SECOND time ---");
        eager.subscribe();
        lazy.subscribe();
        defer.subscribe();
    }
}
```

**What to look for**, and this is the whole lesson in one program:

| Line | When it evaluates | On the second subscribe |
|---|---|---|
| `Mono.just(expensive("just"))` | **during assembly**, before "chains built" is printed | **does not re-evaluate** — the value was captured once |
| `Mono.fromCallable(() -> expensive(...))` | at each subscription | re-evaluates |
| `Mono.defer(() -> ...)` | at each subscription | re-evaluates |

`Mono.just(someMethodCall())` is the trap. It looks lazy. Its argument is an ordinary
Java expression, evaluated eagerly, on whatever thread assembled the chain — which in
a WebFlux controller is an **event-loop thread**. If that method blocks, you have Topic
103's Trap 1, and a `subscribeOn` further down the chain will not save you, because the
work already happened.

### 1c — `.log()`, the instrument you will use most

```java
Flux.just("SKU-1001", "SKU-1002", "SKU-1003")
    .log("catalogue")
    .map(String::toLowerCase)
    .take(2)
    .subscribe(System.out::println);
```

`.log()` prints every signal crossing that point. The shape of the output is:

```
[main] INFO catalogue - onSubscribe(FluxArray.ArraySubscription)
[main] INFO catalogue - request(2)
[main] INFO catalogue - onNext(SKU-1001)
[main] INFO catalogue - onNext(SKU-1002)
[main] INFO catalogue - cancel()
```

*(Illustration of the format, not captured output. Your logger name, thread name and
subscription class will differ.)*

**Read that trace like this:**

- `onSubscribe` first, always. If you never see it, **nothing subscribed** — that is
  the silent no-op, caught.
- `request(2)`, not `request(unbounded)`, because `take(2)` bounded the demand. Demand
  travelling upstream, visible.
- `cancel()` after the second element, because `take(2)` cancelled the source once it
  had what it needed.

Put `.log("name")` at two points in a chain and the difference between the two traces
tells you exactly which operator changed the demand or the threading.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow`'s order-detail read path, at the Topic 65 baseline.

- `GET /orders/{id}` must return the order, its lines, the product name and current
  price for each line, and the payment status.
- Dataset: 100k products, 1M orders, 5M order lines. Median order has 5 lines; the
  p99 order has 40.
- Product details come from a **separate catalogue service** over HTTP (this is the
  new part; previously it was a join). Median 8 ms, p99 40 ms.
- Payment status comes from the payments service, median 6 ms.
- The catalogue service has a hard limit: **8 concurrent connections per caller.**
  Exceeding it returns 429 and its own p99 collapses.
- Baseline target for this endpoint: p99 under 250 ms at 200 requests/second.
- The ledger side of this feature must apply status transitions **in order**. Applying
  `SETTLED` before `AUTHORIZED` corrupts the payment state machine.

### The version that ships and takes down the catalogue service

```java
package com.orderflow.orders;

import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
public class OrderDetailController {

    private final OrderReactiveRepository orders;
    private final CatalogueClient catalogue;
    private final PaymentClient payments;
    private final AuditWriter audit;

    // constructor omitted

    @GetMapping("/orders/{id}")
    public Mono<OrderDetail> orderDetail(@PathVariable long id) {

        audit.recordRead(id);                     // DEFECT 1: returns a Mono. Dropped.

        return orders.findById(id)
            .flatMap(order ->
                Flux.fromIterable(order.lines())
                    .flatMap(line -> catalogue.product(line.productId())   // DEFECT 2
                                              .map(p -> enrich(line, p)))
                    .collectList()
                    .flatMap(lines -> payments.statusFor(order.paymentId())
                                              .map(st -> new OrderDetail(order, lines, st)))
            );
    }
}
```

### What actually happens under the baseline

**Defect 1 — the audit that never happens.**
`audit.recordRead(id)` returns a `Mono<Void>`. The return value is discarded. **No
audit row is ever written.** There is no compiler warning (Java has no
`@MustBeConsumed`), no runtime error, and no log line. The audit table is simply empty,
and the read-access compliance report that depends on it silently reports zero reads
across the entire service.

You will not find this from an alert. You will find it when someone asks why the audit
table has no rows for an endpoint that gets 200 requests per second.

**Defect 2 — unbounded fan-out into a service with a connection limit.**
`flatMap` with no concurrency argument subscribes to up to 256 inner publishers
concurrently. The p99 order has 40 lines, so a single request can open 40 concurrent
catalogue calls. At 200 requests/second with a median of 5 lines, steady state is
roughly 1,000 catalogue calls per second, and bursts of 40 per request.

Against a service that permits 8 concurrent connections:

| Symptom | Where you see it |
|---|---|
| Catalogue returns 429 for a growing fraction of calls | error rate on the catalogue client |
| Order-detail p99 goes from 250 ms to seconds | your own percentile panel |
| Catalogue service's own p99 collapses for **every** caller | their dashboard, their incident channel |
| Retries (if configured) amplify the load further | Topic 111's retry-storm shape |
| Under enough load, connection-pool exhaustion in the HTTP client | `WebClient` connection provider metrics |

The bad part: this passes every test. A test order has 2 lines and no concurrency.

### The corrected version

```java
package com.orderflow.orders;

import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
public class OrderDetailController {

    /** Matched to the catalogue service's documented limit. Not a guess, a contract. */
    private static final int CATALOGUE_CONCURRENCY = 8;

    private final OrderReactiveRepository orders;
    private final CatalogueClient catalogue;
    private final PaymentClient payments;
    private final AuditWriter audit;

    // constructor omitted

    @GetMapping("/orders/{id}")
    public Mono<OrderDetail> orderDetail(@PathVariable long id) {

        return orders.findById(id)
            .switchIfEmpty(Mono.error(new OrderNotFoundException(id)))
            .flatMap(this::enrichOrder)
            // FIX 1: the audit Mono is COMPOSED into the chain, so it is subscribed.
            //        flatMap(detail -> audit...thenReturn(detail)) keeps the result.
            .flatMap(detail -> audit.recordRead(id).thenReturn(detail))
            .timeout(Duration.ofMillis(900))
            .name("orders.detail")                 // for Micrometer, Topic 118
            .metrics()
            .checkpoint("orderDetail");            // for Topic 108
    }

    private Mono<OrderDetail> enrichOrder(Order order) {
        // FIX 2: bounded concurrency, matched to the downstream's limit.
        //        flatMapSequential preserves line order without serialising the calls.
        Mono<List<EnrichedLine>> lines =
            Flux.fromIterable(order.lines())
                .flatMapSequential(line ->
                        catalogue.product(line.productId())
                                 .map(p -> enrich(line, p))
                                 .onErrorResume(e -> Mono.just(degraded(line))),
                        CATALOGUE_CONCURRENCY)
                .collectList();

        Mono<PaymentStatus> status = payments.statusFor(order.paymentId());

        // Independent calls run concurrently; zip waits for both.
        return Mono.zip(lines, status)
                   .map(t -> new OrderDetail(order, t.getT1(), t.getT2()));
    }
}
```

Five decisions in that code, each of which is a separate lesson:

1. **`flatMap(detail -> audit.recordRead(id).thenReturn(detail))`** — the audit `Mono`
   is now part of the chain, so subscribing to the response subscribes to it. The
   `thenReturn(detail)` preserves the value that `audit` discards, because
   `Mono<Void>` completing empty would otherwise swallow the whole response. This is
   the number-one place people get the fix half right and end up with an empty 200.
2. **`flatMapSequential(..., 8)`** — concurrency is bounded to the downstream's stated
   limit, and output order matches line order. `flatMap` would have been unordered;
   `concatMap` would have serialised to one at a time and turned a 40-line order into
   40 × 8 ms = 320 ms.
3. **`onErrorResume` inside the inner publisher** — one failing product lookup
   degrades one line instead of failing the whole response. Error containment must be
   inside the inner `Mono`, because `onError` is terminal for whatever chain it reaches.
4. **`Mono.zip(lines, status)`** — the two independent calls proceed concurrently.
   Chaining them with `flatMap` would have made them sequential for no reason.
5. **`timeout`, `name`/`metrics`, `checkpoint`** — the operational trio. `timeout`
   bounds the blast radius of a slow downstream; `name`+`metrics` gives Micrometer a
   per-chain timer (Topic 118); `checkpoint` gives Topic 108 something to say when it
   fails at 3am.

### The ordered path — where `concatMap` is mandatory

The same service applies payment status transitions to the ledger. Here order is not a
preference:

```java
/**
 * Payment state transitions MUST be applied in emission order.
 * SETTLED before AUTHORIZED corrupts the state machine and the row is then
 * unrecoverable without a manual repair script.
 */
public Mono<Void> applyTransitions(Flux<StatusTransition> transitions) {
    return transitions
            .concatMap(t -> ledger.apply(t))      // ONE AT A TIME. Deliberate.
            .then();
}
```

`concatMap` costs you throughput: one in-flight call at a time. That is the price of
ordering, and it is the correct price here. The engineer who "optimises" this to
`flatMap` during a latency push will produce a corruption bug that appears under load,
disappears in staging, and takes days to attribute.

Write the reason in the code, as above. A comment that says *what would break* survives
a refactor; a comment that says "use concatMap here" does not.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a `Mono` built and never subscribed

**This is the defining bug of this topic.**

**Wrong:**

```java
@PostMapping("/orders")
public Mono<OrderId> place(@RequestBody PlaceOrderCommand cmd) {
    inventory.reserve(cmd.sku(), cmd.units());        // returns Mono<Void>. Dropped.
    walletClient.debit(cmd.customerId(), cmd.total()); // returns Mono<Void>. Dropped.
    return orders.insert(cmd);
}
```

**Exact symptom:** the endpoint returns 200 with a valid order id. **The inventory was
never reserved and the wallet was never debited.** No exception. No error metric. No
log line. Nothing in any dashboard.

You discover it from data: inventory counts that never decrease, wallet balances that
never move, an order table that grows while every other table stays still. On
`orderflow` specifically, the observable is that oversell becomes unlimited because
nothing ever decrements stock.

**Root cause:** `Mono` is cold. `inventory.reserve(...)` built a blueprint and threw it
away. In TypeScript the equivalent call would have executed; in Java it did not.

**Fix:** compose them into the returned chain.

```java
@PostMapping("/orders")
public Mono<OrderId> place(@RequestBody PlaceOrderCommand cmd) {
    return inventory.reserve(cmd.sku(), cmd.units())
            .then(walletClient.debit(cmd.customerId(), cmd.total()))
            .then(orders.insert(cmd));
}
```

`then(other)` means "when I complete, ignore my value and continue with `other`". For
`Mono<Void>` sequencing that is exactly right.

**How to make this impossible rather than merely fixed:**

- Turn on your IDE's "result of method call ignored" inspection for `Publisher`
  return types, and make it an error.
- Add ErrorProne or SpotBugs to the build with a check-return-value rule. Reactor's
  types are annotated in a way that static analysers can use.
- In code review, treat **any bare statement whose expression type is `Mono` or
  `Flux`** as a defect on sight. That single rule catches nearly all instances.

The last one costs nothing and is the highest-value review habit in this phase.

---

### Trap 2 — `Mono.just(expensiveCall())` — eager evaluation at assembly time

**Wrong:**

```java
public Mono<Order> load(long id) {
    return Mono.just(orderJpaRepository.findById(id).orElseThrow())   // blocking, NOW
               .subscribeOn(Schedulers.boundedElastic());             // too late
}
```

**Exact symptom:** exactly Topic 103's Trap 1 — a blocked event loop, bimodal latency,
a p99 that will not explain itself — **despite the presence of a `subscribeOn` that
appears to handle it.** BlockHound throws on the loop thread. The `subscribeOn` in the
code is what makes this so hard to see in review: it looks handled.

**Root cause:** `Mono.just(x)` takes a **value**, not a supplier. Java evaluates
arguments before the call. `findById` therefore ran on the thread that assembled the
chain — the event loop — before `Mono.just` even existed. `subscribeOn` schedules the
*subscription*, and there is nothing left to schedule.

**Fix:**

```java
public Mono<Order> load(long id) {
    return Mono.fromCallable(() -> orderJpaRepository.findById(id).orElseThrow())
               .subscribeOn(Schedulers.boundedElastic());
}
```

`fromCallable` defers the call to subscription time, so `subscribeOn` has something to
move.

**The general rule:** `Mono.just` / `Flux.just` are for values you **already have**.
The moment the argument is a method call that does work, you want `fromCallable`,
`fromSupplier` or `defer`. Grep your codebase for `Mono.just(` followed by anything
containing `(` — it is a surprisingly effective audit.

---

### Trap 3 — `flatMap` where order matters

**Wrong:**

```java
Flux.fromIterable(statusTransitions)      // AUTHORIZED, CAPTURED, SETTLED
    .flatMap(t -> ledger.apply(t))        // three concurrent calls
    .then();
```

**Exact symptom:** intermittent, load-dependent, and never reproducible in staging.
Payment rows in impossible states — `SETTLED` with no capture, or a status that has
gone backwards. Under low load it works, because each call finishes before the next
starts by luck. Under load, network jitter reorders them.

The tell in the data: the corruption rate correlates with throughput, and the affected
rows cluster around latency spikes on the ledger service.

**Root cause:** `flatMap` subscribes to inner publishers concurrently and emits results
**in completion order, not source order**. Nothing in the type system says the
operations were meant to be ordered.

**Fix, and choose deliberately:**

| Requirement | Operator |
|---|---|
| Each must finish before the next starts (true sequencing) | **`concatMap`** |
| Concurrency is fine, but output must be in source order | `flatMapSequential(f, n)` |
| Order genuinely does not matter | `flatMap(f, n)` — but bound `n` |

For a state machine, `concatMap`. Note that `flatMapSequential` gives you ordered
*output* while still running the calls concurrently — which is wrong for state
transitions, because the *side effects* still interleave. Ordered output is not
ordered execution. That distinction has bitten people who reached for
`flatMapSequential` thinking it was the safe choice.

**Prevention:** a comment stating what breaks, plus a test that asserts on the *order
of side effects* (record the calls in a list and assert the sequence), not just on the
final value. A test on the final value passes with either operator.

---

### Trap 4 — unbounded `flatMap` concurrency against a limited downstream

**Wrong:**

```java
Flux.fromIterable(allOrderLines)          // up to 40 per request, thousands per second
    .flatMap(line -> catalogue.product(line.productId()))
    .collectList();
```

**Exact symptom:** 429s or connection-pool exhaustion at the downstream, its p99
collapsing for every caller including ones unrelated to you, and — if the downstream is
a database — connection-pool timeouts of exactly the shape from Topic 109. Your own
latency gets worse, not better, which is the confusing part: you added concurrency and
went slower.

**Root cause:** `flatMap`'s default concurrency is `Queues.SMALL_BUFFER_SIZE` (256 at
time of writing). You did not choose 256; you inherited it. Reactive frameworks make
fan-out syntactically free, which means the constraint has to come from you.

**Fix:** always pass a concurrency argument, derived from the downstream's actual
capacity:

```java
.flatMap(line -> catalogue.product(line.productId()), CATALOGUE_CONCURRENCY)
```

and additionally bound the HTTP client's connection provider so the limit is enforced
at the transport layer too:

```java
ConnectionProvider provider = ConnectionProvider.builder("catalogue")
        .maxConnections(8)
        .pendingAcquireMaxCount(64)
        .pendingAcquireTimeout(Duration.ofMillis(500))
        .build();
```

Belt and braces, deliberately: the operator bound expresses intent, the connection
bound enforces it even if someone adds a second call site.

**The rule to carry:** an unbounded `flatMap` is an unbounded thread pool with better
manners. Everything Topic 90 says about pool sizing applies, and the number should come
from the downstream's capacity, not from your convenience.

---

### Trap 5 — subscribing twice to a cold `Mono` with side effects

**Wrong:**

```java
Mono<PaymentResult> auth = paymentGateway.authorize(cmd);   // cold

auth.subscribe(r -> metrics.record(r));       // subscription 1
return auth.map(this::toResponse);            // subscription 2, when the framework
                                              // subscribes to the returned Mono
```

**Exact symptom:** **the customer is charged twice.** Two HTTP calls to the payment
gateway, two authorisations, two entries on the customer's statement. Your logs show
one order. The gateway's logs show two.

If the downstream is idempotent you get away with it and the symptom becomes merely a
doubled call rate — which you will attribute to something else entirely.

**Root cause:** a cold publisher re-executes on every subscription. Two `subscribe`
calls means two executions. Nothing warns you, and it looks exactly like the Promise
idiom of attaching two `.then()` handlers to one promise — which does *not* re-execute.

This one is especially cruel because the Promise habit it comes from is correct in
TypeScript.

**Fix, three options with different meanings:**

```java
// A. Compose instead of subscribing twice. Almost always the right answer.
return paymentGateway.authorize(cmd)
        .doOnNext(metrics::record)
        .map(this::toResponse);

// B. If you genuinely need to share one execution among several subscribers:
Mono<PaymentResult> shared = paymentGateway.authorize(cmd).cache();

// C. If it's a fan-out to independent consumers:
Flux<PaymentResult> hot = paymentGateway.authorize(cmd).flux().share();
```

Prefer A. `cache()` and `share()` are correct tools with real semantics (`cache`
replays to late subscribers; `share` does not) but reaching for them to fix an
accidental double subscription is treating the symptom.

**Review rule:** a local variable of type `Mono`/`Flux` that is used more than once is
worth a second look every time.

---

### Trap 6 — `block()` inside a reactive chain

**Wrong:**

```java
.map(order -> {
    PaymentStatus s = payments.statusFor(order.paymentId()).block();   // <--
    return new OrderDetail(order, s);
})
```

**Exact symptom:** either Topic 103's Trap 1 in full (loop stalled, bimodal p99), or —
if you are on a recent Reactor and the thread is one it knows must not block — an
immediate, explicit failure:

```
java.lang.IllegalStateException: block()/blockFirst()/blockLast() are blocking,
  which is not supported in thread reactor-http-nio-3
```

*(Illustration of the format, not captured output.)*

That exception is a **gift**. Reactor detects `block()` on threads marked
non-blocking and fails loudly rather than silently stalling. It cannot detect JDBC or
`RestTemplate` — that is what BlockHound is for.

**Root cause:** `block()` subscribes and then parks the calling thread until a terminal
signal arrives. On an event loop that is precisely the forbidden operation.

**Fix:** compose. `block()` inside a chain is always avoidable:

```java
.flatMap(order -> payments.statusFor(order.paymentId())
                          .map(s -> new OrderDetail(order, s)))
```

**Where `block()` is legitimate:** in a `main` method, in a test, in a
`CommandLineRunner`, or at a genuine boundary where you must hand a value to a
blocking API — and then on a thread that is allowed to block. Never inside an
operator, never on a request path.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM and will not print output and
call it captured. What follows is the source, the command, what to look for, and how to
read every result you might get.

### Setup

```bash
mkdir -p ~/java-lab/104 && cd ~/java-lab/104

curl https://start.spring.io/starter.zip \
  -d dependencies=webflux \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=reactor-lab \
  -d type=maven-project -o reactor-lab.zip && unzip reactor-lab.zip -d reactor-lab
cd reactor-lab

./mvnw -q dependency:tree | grep -i reactor    # record the versions you actually have
```

Set a log pattern that shows the thread, because half of this topic is "which thread":

```properties
logging.pattern.console=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{15} - %msg%n
logging.level.reactor=INFO
```

### Proof 1 — nothing runs until you subscribe

Run `SilentNoOp` from Example 1a.

| What you see | What it means |
|---|---|
| "SIDE EFFECT" is **not** printed | Correct. Cold and lazy, confirmed. This is the whole topic in one observation. |
| "SIDE EFFECT" **is** printed | You used `Mono.just(sideEffect())` rather than `fromCallable`, so the argument was evaluated eagerly at assembly. That is Trap 2, and finding it this way is a good outcome. |
| It prints after you add `.subscribe()` | Subscription is what starts execution. Confirmed. |

### Proof 2 — read a signal trace with `.log()`

```java
Flux.range(1, 5)
    .log("source")
    .map(i -> i * 10)
    .log("afterMap")
    .take(2)
    .log("afterTake")
    .subscribe(v -> System.out.println("got " + v));
```

**What to look for:** compare the three traces. Read them as follows.

| What you see | What it means |
|---|---|
| `onSubscribe` appears in all three, innermost first | The subscription walk, observed. It went from `afterTake` upward to `source`. |
| `request(2)` at `source` but `request(unbounded)` at `afterTake` | `take(2)` translated unbounded downstream demand into bounded upstream demand. **Demand accounting, visible.** |
| `cancel()` at `source` after two elements | `take` cancelled upstream once satisfied. Cancellation propagating upward. |
| No `onSubscribe` anywhere | Nothing subscribed. If this is production code, you have found Trap 1. |
| `onNext` lines all on the same thread | No scheduler switch. Expected without `publishOn`/`subscribeOn`. |
| `onNext` on different threads before and after a `publishOn` | The switch, observed. Note which side changed — everything **downstream** of `publishOn`. |

`.log()` is verbose and allocates; it is a debugging tool, not a production one. Topic
108 covers the production-safe alternatives.

### Proof 3 — `subscribeOn` vs `publishOn`, settled empirically

```java
package com.orderflow.lab.reactor;

import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;

public class WhereDidItRun {

    static <T> T tag(String stage, T value) {
        System.out.printf("%-12s %-20s %s%n", stage, Thread.currentThread().getName(), value);
        return value;
    }

    public static void main(String[] args) throws InterruptedException {
        Flux.range(1, 3)
            .map(i -> tag("map1", i))
            .subscribeOn(Schedulers.newSingle("SUBSCRIBE-ON"))
            .map(i -> tag("map2", i))
            .publishOn(Schedulers.newSingle("PUBLISH-ON"))
            .map(i -> tag("map3", i))
            .subscribe(i -> tag("consume", i));

        Thread.sleep(500);      // main would otherwise exit first
    }
}
```

| What you see | What it means |
|---|---|
| `map1` and `map2` on `SUBSCRIBE-ON`, `map3` and `consume` on `PUBLISH-ON` | The expected result. **`subscribeOn` affects everything from the source; `publishOn` affects everything after it.** |
| `map1` on `SUBSCRIBE-ON` even though `subscribeOn` appears *after* it in the code | Correct and counter-intuitive. `subscribeOn` affects the **subscription**, which travels upstream, so its position in the chain barely matters. |
| Adding a second `subscribeOn` changes nothing | Only the first one (closest to the source, in subscription order) wins. |
| Everything on `main` | You forgot the sleep and the program exited, or the source is synchronous and fused. Add the sleep. |

Run this once and you will never confuse the two operators again. Reading about them
does not work; the table does.

### Proof 4 — `flatMap` vs `concatMap` vs `flatMapSequential`

```java
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import java.time.Duration;

public class OrderingProof {
    static Mono<String> call(String id, int millis) {
        return Mono.delay(Duration.ofMillis(millis)).thenReturn(id + "(" + millis + "ms)");
    }

    public static void main(String[] args) throws InterruptedException {
        var inputs = java.util.List.of("A", "B", "C");
        var delays = java.util.Map.of("A", 300, "B", 100, "C", 200);

        System.out.println("--- flatMap ---");
        Flux.fromIterable(inputs).flatMap(id -> call(id, delays.get(id)))
            .subscribe(System.out::println);
        Thread.sleep(600);

        System.out.println("--- concatMap ---");
        Flux.fromIterable(inputs).concatMap(id -> call(id, delays.get(id)))
            .subscribe(System.out::println);
        Thread.sleep(800);

        System.out.println("--- flatMapSequential ---");
        Flux.fromIterable(inputs).flatMapSequential(id -> call(id, delays.get(id)))
            .subscribe(System.out::println);
        Thread.sleep(600);
    }
}
```

| What you see | What it means |
|---|---|
| `flatMap` emits B, C, A | Completion order. All three ran concurrently; the fastest finished first. |
| `concatMap` emits A, B, C and takes ~600 ms total | One at a time. Ordered, and the durations add up. |
| `flatMapSequential` emits A, B, C in ~300 ms | Concurrent execution, reordered output. **Note the total time** — that is the difference from `concatMap`, and it is the reason to choose it. |
| All three produce A, B, C | Your delays are too small or equal; make them clearly different so the effect is visible. |

Also time each block. The wall-clock difference between `concatMap` (~600 ms) and
`flatMapSequential` (~300 ms) is the concrete cost of ordering-by-serialisation versus
ordering-by-buffering, and it is the number to bring to a design discussion.

### Proof 5 — `StepVerifier`, so this is a test and not a script

```xml
<dependency>
  <groupId>io.projectreactor</groupId>
  <artifactId>reactor-test</artifactId>
  <scope>test</scope>
</dependency>
```

```java
@Test
void transitions_are_applied_in_order() {
    List<String> applied = new CopyOnWriteArrayList<>();

    Flux<String> chain = Flux.just("AUTHORIZED", "CAPTURED", "SETTLED")
            .concatMap(t -> Mono.delay(randomDelay())
                                .doOnNext(x -> applied.add(t))
                                .thenReturn(t));

    StepVerifier.create(chain)
            .expectNext("AUTHORIZED", "CAPTURED", "SETTLED")
            .verifyComplete();

    // The critical assertion: SIDE EFFECT order, not just output order.
    assertThat(applied).containsExactly("AUTHORIZED", "CAPTURED", "SETTLED");
}
```

| What you see | What it means |
|---|---|
| Both assertions pass | `concatMap` is doing what you needed. |
| `expectNext` passes but `applied` is out of order | You used `flatMapSequential`. Output order is right and **execution order is not** — exactly the distinction from Trap 3. This test is the only thing that catches it. |
| A timeout | The chain never completed. Usually a missing terminal signal or a source that never completes. |

`StepVerifier.withVirtualTime(...)` lets you test hour-long timeouts in milliseconds by
replacing the clock. Use it for retry-with-backoff tests, which are otherwise
untestable.

### Proof 6 — catch an unsubscribed publisher automatically

There is no built-in Reactor check for "you dropped a `Mono`". Your defences, in order
of value:

```bash
# 1. IDE inspection: "Result of method call ignored" for Publisher-returning methods.
#    IntelliJ: Settings > Editor > Inspections > Java > Probable bugs.
#    Set it to ERROR for org.reactivestreams.Publisher and subtypes.

# 2. Static analysis in the build:
#    ErrorProne's CheckReturnValue / SpotBugs RV_RETURN_VALUE_IGNORED.
#    Reactor's API is annotated in a way these tools can use.

# 3. The review rule, which costs nothing:
grep -rn --include=*.java -E '^\s+(inventory|wallet|audit|payments|ledger)\.[a-zA-Z]+\(.*\);\s*$' src/main/java
```

| What you see | What it means |
|---|---|
| Grep hits on lines that are bare method calls to reactive services | Candidate Trap 1 sites. Check each one's return type. |
| ErrorProne flags a dropped `Mono` at build time | **The best outcome.** Make it an error, not a warning. |
| No tooling available | The review rule still works: any statement whose expression type is `Mono`/`Flux` and which is not returned or assigned is a defect. |

---

## Failure drill

**Mandatory.** Produce both failures yourself and write down what you saw before
reading the analysis.

### The scenario

Two failures that share one root cause — "I wrote something that looks like a call and
is actually a description".

- **Part A:** an audit row that is never written, in an endpoint that returns 200.
- **Part B:** a ledger that applies three transitions in the wrong order under load.

### Part A — the side effect that never happened

`AuditWriter.java`:

```java
package com.orderflow.lab.drill;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

@Component
public class AuditWriter {

    private static final Logger log = LoggerFactory.getLogger(AuditWriter.class);

    /** Stands in for a database insert. Records what actually happened. */
    private final List<String> rows = new CopyOnWriteArrayList<>();

    public Mono<Void> recordRead(long orderId) {
        return Mono.fromRunnable(() -> {
            rows.add("read:" + orderId);
            log.info("AUDIT WRITTEN for order {} on {}", orderId,
                     Thread.currentThread().getName());
        });
    }

    public int rowCount() { return rows.size(); }
}
```

`DrillController.java`:

```java
package com.orderflow.lab.drill;

import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
public class DrillController {

    private final AuditWriter audit;

    public DrillController(AuditWriter audit) { this.audit = audit; }

    /** BROKEN. The audit Mono is built and dropped. */
    @GetMapping("/broken/{id}")
    public Mono<String> broken(@PathVariable long id) {
        audit.recordRead(id);                                  // <-- the drill
        return Mono.just("order " + id);
    }

    /** FIXED. The audit Mono is composed into the returned chain. */
    @GetMapping("/fixed/{id}")
    public Mono<String> fixed(@PathVariable long id) {
        return audit.recordRead(id).thenReturn("order " + id);
    }

    /** WRONG FIX. Subscribes, but detaches the chain. Look closely. */
    @GetMapping("/wrongfix/{id}")
    public Mono<String> wrongFix(@PathVariable long id) {
        audit.recordRead(id).subscribe();                      // fire and forget
        return Mono.just("order " + id);
    }

    @GetMapping("/auditcount")
    public Mono<Integer> auditCount() { return Mono.just(audit.rowCount()); }
}
```

Commands:

```bash
./mvnw spring-boot:run

# 100 requests to each endpoint
for i in $(seq 1 100); do curl -s -o /dev/null localhost:8080/broken/$i;  done
curl -s localhost:8080/auditcount; echo   # <-- record this number

# restart, then:
for i in $(seq 1 100); do curl -s -o /dev/null localhost:8080/fixed/$i;   done
curl -s localhost:8080/auditcount; echo

# restart, then:
for i in $(seq 1 100); do curl -s -o /dev/null localhost:8080/wrongfix/$i; done
curl -s localhost:8080/auditcount; echo
```

**What to capture:** the three counts, the HTTP status of every request, and whether
any "AUDIT WRITTEN" line appeared in the logs for `/broken`.

| What you see | What it means |
|---|---|
| `/broken` → 100 successful responses, **audit count 0**, no AUDIT log lines | **The drill has fired.** A fully successful endpoint that silently did none of its work. There is no error anywhere to alert on. |
| `/fixed` → 100 responses, audit count 100 | Composition works. The response chain's subscription drove the audit. |
| `/wrongfix` → 100 responses, audit count 100 | It *works* — and this is the dangerous part. See the analysis below. |
| `/broken` gives a compiler or IDE warning | Good — your tooling is configured. Most is not, by default. |

**Now the important part — why `/wrongfix` is not a fix.**

`audit.recordRead(id).subscribe()` does execute. But it detaches the audit from the
request lifecycle entirely:

- If the request is **cancelled** (client disconnects, timeout fires), the audit still
  runs — you have logged a read that did not complete.
- If the audit **fails**, the error goes to Reactor's global "dropped error" hook and
  produces at most a warning log. The response is still 200. You have no way to fail
  the request when the audit fails.
- There is **no backpressure and no ordering** between the audit and the response.
- The Reactor `Context` (correlation id, trace id, security context) from the request
  is **not propagated** into the detached subscription, so the audit row is written
  without the correlation id. That is Topic 108's exact problem, arriving early.

So: `/broken` fails silently, `/wrongfix` succeeds while quietly discarding
cancellation, error propagation and context, and `/fixed` is correct. Bare
`.subscribe()` in application code is a smell for these four reasons; write it down.

### Part B — the order that was not preserved

Add to `DrillController`:

```java
    private final List<String> ledger = new CopyOnWriteArrayList<>();

    private Mono<String> applyTransition(String t, int millis) {
        return Mono.delay(java.time.Duration.ofMillis(millis))
                   .doOnNext(x -> ledger.add(t))
                   .thenReturn(t);
    }

    @GetMapping("/ledger/{mode}")
    public Mono<List<String>> runLedger(@PathVariable String mode) {
        ledger.clear();
        var flux = reactor.core.publisher.Flux.just("AUTHORIZED", "CAPTURED", "SETTLED");
        var delays = java.util.Map.of("AUTHORIZED", 300, "CAPTURED", 100, "SETTLED", 50);

        var chain = switch (mode) {
            case "flatmap"    -> flux.flatMap(t -> applyTransition(t, delays.get(t)));
            case "concatmap"  -> flux.concatMap(t -> applyTransition(t, delays.get(t)));
            case "sequential" -> flux.flatMapSequential(t -> applyTransition(t, delays.get(t)));
            default -> throw new IllegalArgumentException(mode);
        };

        return chain.collectList().map(emitted -> List.copyOf(ledger));
    }
```

```bash
curl -s localhost:8080/ledger/flatmap;    echo
curl -s localhost:8080/ledger/concatmap;  echo
curl -s localhost:8080/ledger/sequential; echo
```

| What you see | What it means |
|---|---|
| `flatmap` → `["SETTLED","CAPTURED","AUTHORIZED"]` | **The drill has fired.** The ledger applied a settlement before the authorisation. On a real payment state machine this row is now corrupt. |
| `concatmap` → `["AUTHORIZED","CAPTURED","SETTLED"]` | Correct ordering, achieved by serialising. Note the total wall time: ~450 ms. |
| `sequential` → `["SETTLED","CAPTURED","AUTHORIZED"]` | **The important result.** `flatMapSequential` ordered the *emissions* but the *side effects* still ran concurrently and out of order. Ordered output is not ordered execution. |
| All three ordered | Your delays are equal or too small. Make `AUTHORIZED` clearly the slowest. |

### What the drill proves

Both halves are the same lesson wearing different clothes:

> **In Reactor, writing the call is not making the call.** Assembly builds a
> description. Subscription runs it. Everything about *whether*, *when*, *how many
> times*, *on which thread* and *in what order* your code executes is decided by the
> chain you composed, not by the order the statements appear in the file.

Carry one sentence out of this: *a bare `Mono` statement is dead code, and
`flatMapSequential` orders emissions, not effects.*

---

## Measurement

### The instrument for each claim

| Claim you want to make | Instrument | Never use |
|---|---|---|
| "This publisher is actually subscribed" | `.log()`, or `doOnSubscribe(s -> log...)` | reading the code |
| "This ran on thread X" | `Thread.currentThread().getName()` inside the operator | the operator's position in the file |
| "Demand is bounded here" | `.log()` and read the `request(n)` lines | the operator's name |
| "Order is preserved" | a `StepVerifier` test asserting on the **side-effect list** | asserting on the emitted values |
| "Concurrency is bounded to 8" | the downstream's connection metrics, plus a `Semaphore`-style counter in a test | the `flatMap` argument alone |
| "This chain is faster than the MVC path" | k6 open-model run against the Topic 65 baseline | a `System.nanoTime()` loop |
| "No operator blocks a loop" | BlockHound (Topic 103) | inspection |

### Per-chain metrics, the production-safe instrument

```java
return orders.findById(id)
        .flatMap(this::enrichOrder)
        .name("orders.detail")     // becomes the Micrometer meter name
        .tag("path", "detail")
        .metrics();                // registers timers for this chain
```

`.name()` + `.metrics()` gives you a timer per chain, split by terminal signal
(complete / error / cancel). The **cancel** count is the one to watch and the one
nobody looks at: a rising cancel rate means clients are disconnecting or timeouts are
firing before your chain finishes. Topic 118 covers cardinality — do not tag with an
order id.

### Comparing against the Topic 65 baseline

Same rules as Topic 103, restated because they are the ones people break:

1. **Open-model arrival rate** in k6 (`constant-arrival-rate`), never a fixed VU loop.
   A closed loop hides tail latency through coordinated omission.
2. **Same dataset:** 100k products, 1M orders, 5M order lines.
3. **Same cache state**, warmed identically or recorded as cold in both runs.
4. **Record the configuration:** heap, collector, container CPU limit,
   `reactor.netty.ioWorkerCount`, `flatMap` concurrency bounds, HTTP client
   `maxConnections`. A reactive benchmark whose fan-out bounds you did not record is
   not reproducible, and the gate rule is ±10%.
5. **p50/p95/p99/p999 plus error rate plus cancel rate**, per endpoint. Never a mean.

The honest expectation, stated before you measure so you cannot rationalise
afterwards: **rewriting a blocking endpoint as a reactive chain against the same
blocking database will not make it faster.** It changes the threading model. If your
numbers show a large win, look for a methodology error — most often an accidental
change in fan-out concurrency or in cache state.

### The standing rule: a naive `System.nanoTime()` loop is WRONG

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 100_000; i++) {
    Flux.range(1, 10).map(x -> x * 2).reduce(Integer::sum).block();
}
System.out.println((System.nanoTime() - start) / 100_000 + " ns/op");
```

Wrong for Topic 77's four JIT reasons — dead-code elimination, constant folding,
on-stack replacement, cold-JIT and profile pollution — and for three that are specific
to Reactor:

1. **Assembly cost and subscription cost are different things**, and this loop
   measures their sum. Which one dominates depends entirely on chain length, and you
   cannot tell from one number.
2. **Fusion changes with the shape of the chain.** A chain of fusable operators
   collapses; add one non-fusable operator and the whole cost profile changes. A
   microbenchmark of `map.map.map` tells you almost nothing about `map.flatMap.map`.
3. **`.block()` serialises everything**, which defeats the only property you might
   have wanted to measure.

Use JMH with `@Fork(3)`, `@State(Scope.Benchmark)` and `Blackhole` for operator-level
cost (Topic 77), and k6 with an open arrival model for service-level numbers (Topic
65). And be honest about which question you are answering: operator overhead is
nanoseconds, and your database call is hundreds of microseconds.

---

## Practice exercises

### 1 — Easy: translate and predict

**Part A.** Translate these five RxJS snippets to Reactor. For each, state whether the
Reactor version is an exact equivalent or differs, and how.

```ts
1.  of(1,2,3).pipe(map(x => x*2)).subscribe(console.log)
2.  from(ids).pipe(mergeMap(id => http.get(`/p/${id}`), 4)).subscribe()
3.  timer(0, 1000).pipe(take(5)).subscribe(console.log)
4.  src$.pipe(catchError(() => of(null))).subscribe()
5.  const shared$ = src$.pipe(shareReplay(1))
```

**Part B.** For each of these, predict what is printed **before** running it. Then run
it and explain any surprise in one sentence.

```java
a.  Mono.just(System.currentTimeMillis()).delayElement(Duration.ofSeconds(1))
        .subscribe(t -> System.out.println(System.currentTimeMillis() - t));

b.  Mono<String> m = Mono.fromCallable(() -> { System.out.println("ran"); return "x"; });
    m.subscribe(); m.subscribe();

c.  Flux.just(1,2,3).doOnNext(System.out::println).subscribe();
    Flux.just(1,2,3).doOnNext(System.out::println);

d.  Flux.just(1,2,0,4).map(i -> 10/i).onErrorReturn(-1).subscribe(System.out::println);

e.  Mono.just("a").subscribeOn(Schedulers.parallel())
        .map(s -> Thread.currentThread().getName()).subscribe(System.out::println);
```

Snippet (c) is the one that matters. Explain the difference in one sentence, and then
say what tooling would have caught the second line in a real codebase.

### 2 — Medium: the audit (combines Topics 01–103)

This service is part of `orderflow`'s checkout path. It contains **eight** defects.
Five are from this topic; three are from earlier topics. For each: name the topic,
state the **observable** symptom in production, and write the fix.

```java
package com.orderflow.checkout;

import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.HashMap;
import java.util.Map;

@Service
public class CheckoutService {

    private final Map<Long, Double> priceCache = new HashMap<>();
    private final InventoryClient inventory;
    private final WalletClient wallet;
    private final CatalogueClient catalogue;
    private final OrderRepository orders;
    private final AuditWriter audit;

    // constructor omitted

    public Mono<OrderId> checkout(CheckoutCommand cmd) {

        audit.recordAttempt(cmd.customerId());

        Double cached = priceCache.get(cmd.productId());
        double unitPrice = cached != null
                ? cached
                : catalogue.priceOf(cmd.productId()).block();
        priceCache.put(cmd.productId(), unitPrice);

        double total = unitPrice * cmd.quantity();

        Mono<Void> reserve = inventory.reserve(cmd.productId(), cmd.quantity());
        reserve.subscribe();

        return Flux.fromIterable(cmd.lines())
                .flatMap(line -> applyLine(cmd.orderId(), line))
                .then(wallet.debit(cmd.customerId(), total))
                .then(orders.insert(cmd));
    }

    private Mono<Void> applyLine(Long orderId, CheckoutLine line) {
        return Mono.just(orders.reserveLine(orderId, line));
    }

    public boolean isSameOrder(Long a, Long b) {
        return a == b;
    }
}
```

Hints, in the order to think about them. One defect makes the whole method block an
event-loop thread — find it first, because its symptom masks everything else. One is a
Topic 01 boxing comparison that passes every test with small ids. One is a Topic 92
concurrent-collection hazard on a shared field. One is a Topic 79 unbounded-growth
leak in the same field. Two are dropped publishers. One is eager evaluation at
assembly time. One is unbounded fan-out with a money side effect.

For each defect, also answer: **would a unit test have caught it?** At least four
would not, and being able to say which is the point of the exercise.

### 3 — Hard: production simulation on the `orderflow` baseline

**Part A — establish.** Bring up the Topic 65 stack and re-run the recorded baseline
for `GET /orders/{id}`. Confirm ±10%. Record p50/p95/p99/p999 and error rate.

**Part B — build the enrichment.** Extract product lookup into a separate catalogue
service (a second container is fine; a WireMock or a tiny Spring app both work) with:

- a configurable per-caller concurrency limit of 8, returning 429 above it;
- median 8 ms, p99 40 ms latency (inject the distribution deliberately).

Reimplement `GET /orders/{id}` as the **naive** version from Example 2 — unbounded
`flatMap`, dropped audit `Mono`.

**Part C — capture the damage.** Run the k6 mix (70/20/10) at the baseline arrival
rate with the order-detail endpoint included. Capture:

- p50/p95/p99/p999 for order detail **and** for the unrelated catalogue-read endpoint;
- the catalogue service's 429 rate and its own p99;
- the audit row count (expect zero, and confirm it);
- the maximum observed in-flight catalogue calls (instrument the client).

**Part D — fix in two independent steps, measuring after each.** First bound the
`flatMap` concurrency to 8 and re-measure. Then compose the audit `Mono` and
re-measure. Produce a table: change → order-detail p99 → 429 rate → audit rows.

State explicitly which of the two fixes moved the latency numbers and which moved
none of them. **The audit fix should move no latency number at all**, and being able
to say "this fix is correctness-only, here are the numbers proving it changed nothing
about performance" is exactly the kind of statement that distinguishes a senior
engineer.

**Part E — the ordering question.** The p99 order has 40 lines. Compare three
implementations of line enrichment at the baseline arrival rate:

1. `concatMap` (ordered, serialised)
2. `flatMapSequential(f, 8)` (ordered output, bounded concurrency)
3. `flatMap(f, 8)` + a client-side sort by line number

Produce p50/p99 for each, and then answer: which would you ship, and what would have to
be true about the data for your answer to change? Note that (3) is only viable because
line number is a stable sort key — say what happens if it is not.

**Part F — argue against yourself.** You have a bounded, ordered, instrumented reactive
enrichment path. Make the strongest possible case that this endpoint should have been
written imperatively on virtual threads (Topic 101) with a bounded semaphore for the
catalogue calls. Then state what would have to be true for the reactive version to win.
**Keep this answer. Topic 107 makes you defend it with numbers.**

---

## Interview questions

### Q1 — "How is a `Mono` different from a `Promise`?"

**Mid-level answer:** "They're similar — both represent a future value. A `Mono` has
more operators and can be empty."

**Senior answer:** "The operator surface is the superficial difference. The one that
changes how you write code is that a **`Mono` is cold and lazy while a `Promise` is
eager**. A `Promise` starts its work when you construct it; a `Mono` is a blueprint
that does nothing until something calls `subscribe()`. So building a `Mono` and not
subscribing is a silent no-op — no exception, no log, the side effect just never
happens. That's the single most common bug people bring over from a Promise
background, and it's a data bug rather than a crash, which makes it expensive.

Three things fall out of laziness that are genuinely better. **Retry works**: retrying
means re-subscribing, which re-runs the recipe, whereas you can't retry a `Promise` —
you have to retry the function that made it. **Cancellation is real**: `cancel()`
propagates upstream and the HTTP client can actually abort, where a `Promise` has no
cancellation and `AbortController` is a side channel. And **you can compose before
anything runs**, which is what makes operators like `timeout` and `retryWhen` possible
at all.

One thing that's worse: subscribing twice to a cold `Mono` executes it twice. Attaching
two `.then()` handlers to a `Promise` doesn't. So the Promise habit of holding a
reference and consuming it in two places is a double-charge bug in Reactor.

And the thing neither has, which is why Reactor exists at all: **backpressure**. A
`Subscription` carries `request(n)` upstream so the consumer can tell the producer how
much it can accept. Promises have no concept of that, and neither does RxJS in any
meaningful way."

**What separates them:** cold/lazy stated as the *primary* difference, the three
consequences named, the double-subscription hazard volunteered, and finishing on
backpressure — which sets up the next question and shows you know what the
specification is actually for.

**Follow-up:** "Show me code that silently does nothing." Any dropped `Mono<Void>`.

---

### Q2 — "What is the difference between assembly time and subscription time?"

**Mid-level answer:** "Assembly is when you build the chain, subscription is when it
runs."

**Senior answer:** "Right, and the mechanism matters because three separate classes of
bug come out of it.

At assembly, each operator allocates a small `Publisher` object holding a reference to
its upstream. You've built a linked list. Nothing has executed.

At subscription, the call walks that list from the bottom upward — each operator
creating a `Subscriber` that subscribes to its upstream — until it reaches the source.
Then `onSubscribe` propagates back downward, and the bottom subscriber calls
`request(n)`, which travels back **up**. Only then do elements start flowing down.

Three consequences. First, `Mono.just(someBlockingCall())` evaluates its argument at
**assembly**, on whatever thread built the chain — an event loop, in a WebFlux
controller — so a `subscribeOn` further down moves nothing and you get a stalled loop
with an apparently correct fix in the code. `fromCallable` or `defer` is what you
wanted.

Second, `subscribeOn` affects the subscription, which travels upstream, so its position
in the chain barely matters and only the first one wins. `publishOn` affects the
downstream signal path, so its position matters completely. That asymmetry confuses
everyone once and is obvious once you've seen the walk.

Third — and this is the one that matters for observability — the Reactor `Context` is
carried in the subscription, so `contextWrite` affects operators **above** it, not
below. That's why context propagation in Reactor reads backwards, and it's why
`ThreadLocal` can't do the job: the chain hops threads between operators."

**What separates them:** describing the two-phase walk mechanically, and then deriving
three specific bugs from it rather than listing facts. The `Context` direction is the
detail almost nobody volunteers.

**Follow-up:** "Why can't `ThreadLocal` carry the correlation id?" Because `publishOn`
and `flatMap` move execution between threads, and a `ThreadLocal` set on one is
invisible on another. Topic 108.

---

### Q3 — "When would you use `concatMap` instead of `flatMap`?"

**Mid-level answer:** "`concatMap` preserves order and `flatMap` doesn't, so use
`concatMap` when order matters."

**Senior answer:** "Order is the headline; there are actually three axes and I'd want
to name all of them because picking on one gets you the wrong operator.

`flatMap` subscribes to inner publishers concurrently — up to 256 by default, which is
a number you inherit rather than choose — and emits in **completion** order.
`concatMap` runs **one at a time** and emits in source order. `flatMapSequential` runs
concurrently but buffers to emit in source order.

The trap is that people reach for `flatMapSequential` when they want ordering and
concurrency, and it's the wrong choice for **side effects**, because it orders the
emissions and not the execution. If I'm applying payment state transitions to a ledger,
`flatMapSequential` will still run `SETTLED` before `AUTHORIZED`; only `concatMap`
gives me sequenced execution. Ordered output is not ordered execution, and that
distinction has cost me real time.

So: sequenced side effects → `concatMap`. Independent work where I only need ordered
results → `flatMapSequential` with a bound. Genuinely order-independent → `flatMap`
with a bound.

And in all three cases I'd pass an explicit concurrency argument matched to the
downstream's capacity, because the default 256 will happily open 256 concurrent calls
against a service with a 10-connection pool. That's not a reactive problem, it's the
same pool-sizing question as any thread pool — the framework just makes the fan-out
free to write.

To test it I'd assert on the order of recorded **side effects**, not on the emitted
values, because a test on emitted values passes with `flatMapSequential` and the bug
ships."

**What separates them:** the three-way distinction, the ordered-output-versus-ordered-
execution insight, the default-concurrency-is-256 fact, and proposing a test that
actually catches it.

**Follow-up:** "What does `concatMap` cost you?" Throughput: strictly one in-flight
call, so latencies add. For a 40-element source against an 8 ms service that is 320 ms
instead of 40.

---

### Q4 — "Your reactive endpoint returns 200 but nothing was written to the database. Where do you start?"

**Mid-level answer:** "Check the database connection and the transaction
configuration."

**Senior answer:** "Before either of those I'd check whether the publisher was ever
subscribed, because in Reactor that is by far the most likely cause and it produces
exactly this signature: a successful response and no side effect.

The specific shape is a statement like `inventory.reserve(sku, units);` on its own
line. It returns a `Mono<Void>`, the value is discarded, nothing subscribes, and the
work never happens. Java has no unused-result error for it, so it compiles clean and
the endpoint returns 200.

I'd confirm it in about a minute: add `.log()` or a `doOnSubscribe` at the top of the
suspect publisher and look for an `onSubscribe` signal. If there isn't one, that's the
answer. Then I'd grep the file for any bare statement whose expression type is a
`Publisher`.

The fix is composition — `then`, `flatMap`, `thenReturn` — and specifically not a bare
`.subscribe()`, which does execute but detaches the work from the request lifecycle:
you lose cancellation, the error goes to a global dropped-error hook instead of failing
the request, and the Reactor `Context` — correlation id, trace id, security context —
isn't propagated into the detached subscription.

To stop it recurring I'd turn on the IDE's ignored-return-value inspection for
`Publisher` types and add an ErrorProne or SpotBugs check-return-value rule to the
build. This is a silent failure, so it needs an active check; you can't test for a call
you didn't think to make."

**What separates them:** going to the framework-specific cause first rather than the
generic one, naming the one-minute diagnostic, explaining precisely why `.subscribe()`
is not the fix, and proposing tooling because the failure is silent. That last move —
"a silent failure needs an active check" — is the same instinct as Topic 40's
`isAopProxy` assertion.

**Follow-up:** "Why is `.subscribe()` in a controller a smell?" Cancellation, error
propagation, context, and ordering — all four.

---

### Q5 — "How do you test a reactive chain?"

**Mid-level answer:** "Use `StepVerifier` and assert on the emitted values."

**Senior answer:** "`StepVerifier` for signals, plus assertions on side effects,
because they answer different questions.

`StepVerifier.create(chain).expectNext(a, b, c).verifyComplete()` verifies the
**signal** contract — the values, their order, the terminal signal, and that there
isn't an extra element after completion. It also has `expectError`,
`expectNextCount`, and `thenCancel` for cancellation paths, which people almost never
test and which are where the interesting bugs are.

But signal assertions miss a whole class of bug. If ordering of **effects** matters —
a ledger, a state machine, anything with writes — I record the effects in a
`CopyOnWriteArrayList` and assert on that list separately, because
`flatMapSequential` produces correct emission order with incorrect execution order and
a value-only test passes.

For anything time-based I use `StepVerifier.withVirtualTime`, which replaces the
scheduler's clock, so a retry with a five-minute backoff is testable in milliseconds
instead of being deleted from the suite for being slow.

And two things that aren't `StepVerifier`. **BlockHound** in the test suite, so a
blocking call on a non-blocking thread fails the build rather than showing up as a p99
mystery in production. And a static check or IDE inspection for dropped publishers,
because 'the code never subscribed' is not something a test can catch — the test
subscribes to whatever you hand it, so a test of a broken method can pass while
production does nothing."

**What separates them:** distinguishing signal tests from effect tests, virtual time,
and knowing which failure modes are outside what tests can reach. Naming BlockHound and
the dropped-publisher gap is the part that lands.

**Follow-up:** "How would you test that a client disconnect cancels the upstream call?"
`StepVerifier`'s `thenCancel().verify()`, plus a `doOnCancel` that records the event.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A cold `Mono` re-executes on every subscription. Derive from that single fact why
   `retry()` is possible in Reactor and impossible on a `Promise` — and then name the
   bug that the same fact causes.

2. `subscribeOn` affects the whole chain from the source regardless of where you put
   it; `publishOn` affects only what is below it. Both facts follow from one property
   of the subscription walk. Name the property, then predict what two `publishOn`
   calls in sequence do.

3. Reactor's `Context` travels **upstream** with the subscription. Why is that the
   right design given that operators may hop threads? Name one thing it makes harder
   than `AsyncLocalStorage` would.

4. `onError` is terminal for the whole chain. Argue that this is the right default.
   Then argue it is the wrong default. Which side is stronger for a batch job, and does
   your answer change for a request/response endpoint?

5. `flatMap`'s default concurrency is 256. Suppose it had been 1 instead. Name one
   thing that would improve and one thing that would get worse — and say which default
   you would have chosen for a framework used by people who have not read this
   document.

6. Operator fusion means a 12-operator chain is not 12 queues. What does that buy, and
   what does it cost you at 3am when you are reading a stack trace? (Topic 108 answers
   the second half; try to predict it.)

7. You could imagine a Java compiler warning for "a `Publisher` expression whose value
   is discarded". Java has no such warning. Sketch what it would take to add one at the
   language level, and say why a library annotation plus a static analyser is the
   answer the ecosystem actually reached for.

---

## Quick reference card

### Creation

```java
Mono.just(value)                  // value ALREADY computed. Eager argument.
Mono.fromCallable(() -> work())   // lazy, may throw
Mono.fromSupplier(() -> value)    // lazy, must not throw checked
Mono.fromRunnable(() -> effect()) // lazy, Mono<Void>
Mono.defer(() -> buildMono())     // lazy, rebuilds the whole Mono per subscription
Mono.empty() / Mono.error(e) / Mono.never()
Mono.fromFuture(completableFuture)         // Topic 91 bridge

Flux.just(a, b, c) / Flux.fromIterable(list) / Flux.range(1, n)
Flux.fromStream(() -> stream)     // supplier form: re-subscribable
Flux.interval(Duration.ofSeconds(1))
Flux.generate(...)                // synchronous, HONOURS demand
Flux.create(sink -> ..., overflowStrategy)  // push source. Topic 105.
Sinks.many().multicast().onBackpressureBuffer()   // programmatic hot source
```

### Transformation

```java
.map(f)                    // 1:1, synchronous
.filter(p)
.flatMap(f)                // 1:N, CONCURRENT (default 256), interleaved
.flatMap(f, n)             // bounded concurrency  <- prefer this
.flatMapSequential(f, n)   // concurrent, ordered OUTPUT (not ordered effects)
.concatMap(f)              // one at a time, ordered execution AND output
.switchMap(f)              // cancel previous on each new element
.then() / .thenReturn(v) / .thenMany(flux)
.zipWith(other) / Mono.zip(a, b)  // wait for both, run concurrently
.collectList() / .collectMap(k)
.buffer(n) / .window(n) / .groupBy(k)
.distinct() / .take(n) / .skip(n) / .limitRate(n)
```

### Errors, resilience, lifecycle

```java
.onErrorReturn(fallback)        // swallow, emit one value, complete
.onErrorResume(e -> other)      // swallow, switch source
.onErrorMap(e -> new DomainEx(e))  // translate; still an error
.doOnError(e -> log.error(...)) // OBSERVE ONLY. Does not handle.
.retry(3)
.retryWhen(Retry.backoff(3, Duration.ofMillis(100)).jitter(0.5))
.timeout(Duration.ofMillis(900))
.defaultIfEmpty(v) / .switchIfEmpty(otherPublisher)
.doOnSubscribe / .doOnNext / .doOnCancel / .doFinally
```

### Threading

```java
.subscribeOn(scheduler)    // WHERE THE SOURCE RUNS. First one wins. Position ~irrelevant.
.publishOn(scheduler)      // WHERE EVERYTHING BELOW RUNS. Position matters. Stackable.

Schedulers.parallel()                    // CPU work. availableProcessors() threads.
Schedulers.boundedElastic()              // BLOCKING work. Large default bound.
Schedulers.newBoundedElastic(n, q, name) // blocking work with a DELIBERATE bound.
Schedulers.single() / Schedulers.immediate()
```

### Debugging and observability

```java
.log("name")                       // every signal. Verbose. Dev only.
.doOnEach(sig -> ...)              // signal-level side effect, context-aware
.name("chain").metrics()           // Micrometer timers per chain (Topic 118)
.checkpoint("where")               // assembly info for stack traces (Topic 108)
Hooks.onOperatorDebug()            // global, expensive (Topic 108)
StepVerifier.create(chain)...      // tests
StepVerifier.withVirtualTime(...)  // time-based tests without waiting
BlockHound.install()               // catch blocking on non-blocking threads (Topic 103)
```

### The rules

```
NEVER   leave a Mono/Flux statement unconsumed. Compose it or return it.
NEVER   use Mono.just(f()) where f() does work. Use fromCallable / defer.
NEVER   call .block() inside an operator or on a request path.
NEVER   call flatMap without a concurrency bound against a limited downstream.
ALWAYS  choose concatMap when the SIDE EFFECTS must be ordered.
ALWAYS  assert on side-effect order in tests, not just emitted values.
ALWAYS  contain per-element errors inside the inner publisher, not outside it.
REMEMBER onError is terminal for the chain. doOnError does not handle anything.
```

---

## When would I use this at work?

**1. Reviewing a pull request that adds a call to a reactive service.**
A diff adds `notifications.send(orderId);` on its own line inside a handler. It looks
like a call. It is a dropped `Mono`, and the notification will never be sent — with a
200 response and no error anywhere. Catching that in review costs ten seconds; catching
it in production means discovering that a month of notifications were never sent. This
is the highest-frequency payoff of this topic and it becomes automatic once the pattern
is in your eye.

**2. Diagnosing a downstream service you accidentally overloaded.**
The catalogue team reports that their p99 collapsed and the traffic is coming from your
service. You know that an unbounded `flatMap` opens up to 256 concurrent inner
subscriptions, so you find the fan-out, bound it to their documented limit, and bound
the HTTP client's connection provider as well. You can then explain — with the
before/after 429 rate — that the framework made the fan-out syntactically free and the
constraint had to come from you. That is a much better conversation than "we'll add a
retry."

**3. Deciding whether a corruption bug is a race or an ordering bug.**
Payment rows appear in impossible states, only under load, never in staging. Most teams
start with database isolation levels and locking. You check the operator first: a
`flatMap` where the state machine needed `concatMap` produces exactly this signature —
load-correlated, latency-correlated, invisible at concurrency one. You can prove it with
a test that asserts on side-effect order, and the fix is one word. Knowing to look
there first is worth days.

---

## Connected topics

**Prerequisites:**
- **21–22 — Lambdas and method references:** every operator takes one. `flatMap`'s
  argument is a `Function<T, Publisher<R>>`, and the four method-reference forms show
  up constantly.
- **23 — Streams and laziness:** the closest thing already in your Java vocabulary. A
  `Stream` is lazy until a terminal operation, exactly as a `Flux` is lazy until
  subscription. The difference is that a `Stream` is pull-only and synchronous, with no
  concept of demand crossing a thread boundary.
- **26 — `Optional`:** `Mono` is `Optional` that can also be asynchronous and can fail.
  The `map`/`flatMap` distinction is the same distinction.
- **91 — `CompletableFuture`:** the eager, non-composable-before-execution cousin.
  `Mono.fromFuture` and `Mono.toFuture` are the bridges, and the differences (cold vs
  hot, cancellation, retry) are the same list as the Promise comparison.
- **103 — NIO and the event loop:** which thread runs an operator, and why blocking one
  is catastrophic. Assembly-time evaluation lands on the loop; that is why Trap 2
  matters.

**This unlocks:**
- **105 — Backpressure:** `request(n)`, the signal you saw in the `.log()` traces, and
  what happens when a source cannot honour it. This is the direct continuation — read
  it next.
- **106 — WebFlux vs MVC:** who subscribes to the `Mono` your controller returns, and
  what the threading model costs.
- **107 — Loom vs reactive:** whether you needed any of this. Composition and
  backpressure are the two things this document gives you that virtual threads do not.
- **108 — Debugging reactive:** why the stack trace names `FluxMapFuseable` and not
  your file, why `checkpoint()` helps, and how the `Context` you met here carries a
  correlation id across thread hops.
- **111 — Resilience4j:** `retryWhen(Retry.backoff(...).jitter(...))` versus the
  annotation-driven equivalents, and why jitter is not optional.
- **113–114 — Kafka:** a consumer is a hot push source. Everything about cold-versus-hot
  in this document decides whether you can backpressure it.
- **118 — Metrics:** `.name().metrics()`, and the cancel-rate signal nobody watches.
- **119–120 — Tracing and MDC:** `Context` propagation, which is why Reactor has its own
  context type at all.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0; Reactor's version
comes from the Boot BOM and is deliberately not quoted here. Two things in this document
are hedged rather than asserted: the exact default `flatMap` concurrency and prefetch
constants (256 / `Queues.SMALL_BUFFER_SIZE` at time of writing — implementation details,
not compatibility promises), and the precise mapping of RxJS `exhaustMap` onto a Reactor
operator, which has no exact 1:1. Both are settled by the Reactor reference documentation
for your resolved version. Everything else here — cold and lazy, the two-phase
subscription walk, demand travelling upstream, the terminal-signal contract, and the
`flatMap`/`concatMap`/`flatMapSequential` distinction — is specification-level and has
been stable since Reactor 3.0.*
