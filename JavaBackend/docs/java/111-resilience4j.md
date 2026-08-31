# 111 — Resilience4j: Circuit Breakers, Bulkheads, Retries with Jitter, Rate Limiting

## Phase: 11 — Distributed Systems & Production
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: wrap the `orderflow` payment gateway. The gateway is the one call in the order-placement path that leaves the process, so it is the one that can convert a downstream blip into a full `orderflow` outage. You will make it fail on purpose, prove the retry storm, and then prove the breaker opens.

---

## Mechanical statement

Read this twice. Everything else is an elaboration of it.

> **Every Resilience4j decorator is a wrapper in a chain. The ORDER is the
> semantics.**
>
> `Retry(CircuitBreaker(call))` — the retry loop is **outside**. Each attempt
> passes through the breaker, so the breaker records every attempt; and when the
> breaker is open, the retry's next attempt fails fast on
> `CallNotPermittedException` instead of touching the network. The breaker sees one
> logical failure per *retried attempt* and can open.
>
> `CircuitBreaker(Retry(call))` — the retry loop is **inside**. The breaker sees
> one call, which the retry loop turns into four network attempts before returning.
> If the retry eventually succeeds, **the breaker records a success** and never
> learns the dependency is degraded. If it fails, the breaker records exactly one
> failure for four attempts, so it needs four times as many client requests to
> reach its threshold. **The breaker is blind, and it takes four times as long to
> open.**
>
> This is not a style preference. The two orderings produce different failure
> behaviour under the same outage, and only one of them protects the dependency.

Two corollaries you should be able to state without thinking:

> **A retry is a load multiplier.** `maxAttempts = 4` against a dependency that is
> already failing means every client request becomes four requests. At the moment
> the dependency most needs less load, you give it 4×. Retries without a circuit
> breaker and a concurrency bound convert a partial outage into a total one.

> **Resilience4j's annotations are AOP proxies (Topic 40).** `@CircuitBreaker` on a
> method called from inside the same bean does nothing at all. No exception, no
> log line, no protection.

---

## The bridge from what you know

You already know what a circuit breaker is, what a bulkhead is for, and why
exponential backoff needs jitter. This section does not re-teach any of that. It is
only the four things that are different in Java with Resilience4j, and each of them
is a place where correct understanding of the pattern still produces broken code.

### Difference 1 — the decorators are explicit and ordered, and you choose the order

In Node you probably used `cockatiel`, `opossum`, or a hand-rolled wrapper. The
composition was a function you wrote, so the nesting was visible on the page:

```ts
const guarded = retry(policy, () => breaker.execute(() => http.post(url, body)));
//               ^ outside                ^ inside
```

You could see which was outside. In Spring, you write:

```java
@Retry(name = "payments")
@CircuitBreaker(name = "payments")
@Bulkhead(name = "payments")
public PaymentResult charge(ChargeCommand cmd) { ... }
```

Three annotations on one method, in an arbitrary source order that **does not
determine the nesting**. The nesting is determined by the **aspect order** of the
Resilience4j aspects, which is a configuration property, not the order you typed
them in.

Resilience4j's documented default nesting is:

```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( yourMethod ) ) ) ) )
```

Retry outermost, bulkhead innermost. **That default is correct** — it is the
ordering the Mechanical statement argues for. The danger is not the default; it is
that the default is invisible, and that someone who "fixes" a problem by nudging
`resilience4j.retry.retryAspectOrder` can silently invert it.

Aspect order in Spring is a number where **higher means outer** (higher precedence
runs first). The properties exist per module:

```yaml
resilience4j:
  retry:
    retryAspectOrder: <n>
  circuitbreaker:
    circuitBreakerAspectOrder: <n>
  bulkhead:
    bulkheadAspectOrder: <n>
```

I am not going to state the exact default numeric values, because they have changed
across releases and an invented number here would be worse than useless. **Print
them** — the Hands-on proof section shows how — and assert the resulting order in a
test.

### Difference 2 — it is proxy-based, so Topic 40 applies in full

`@CircuitBreaker`, `@Retry`, `@Bulkhead`, `@RateLimiter` and `@TimeLimiter` are all
implemented by Spring AOP aspects. Everything from Topic 40 carries over:

- **Self-invocation bypasses them.** `this.charge(cmd)` from another method in the
  same bean gets no breaker, no retry, no bulkhead.
- **`private` and `final` methods cannot be advised.**
- **A method called from `@PostConstruct`** runs before the proxy exists from that
  bean's own point of view (Topic 37).
- The bean's runtime class will be `...$$SpringCGLIB$$...` if a proxy exists at
  all — which is how you check.

The failure mode is the worst kind: your code looks protected, your config looks
right, the Actuator endpoint shows a circuit breaker named `payments` sitting in
`CLOSED` forever with zero recorded calls, and the payment gateway takes you down.
The zero-call count is the tell.

**The functional API has no such problem** and is why I use it for the paths that
matter:

```java
var decorated = Decorators.ofSupplier(() -> gateway.charge(cmd))
        .withBulkhead(bulkhead)          // innermost
        .withCircuitBreaker(breaker)
        .withRetry(retry)                // outermost -- added last
        .decorate();
```

In the `Decorators` builder each `withX` wraps what came before, so **the last one
added is the outermost**. The order is on the page and cannot be changed by a
property file.

### Difference 3 — the breaker state lives in this JVM only

`CircuitBreakerRegistry` holds one `CircuitBreaker` instance per name, in the heap
of this process. There is no coordination between pods.

Run `orderflow` on 8 pods and you have **8 independent breakers**, each with its own
sliding window, each needing to independently accumulate `minimumNumberOfCalls`
before it will evaluate its failure rate.

Two consequences that people get wrong:

- **Sizing.** If you set `minimumNumberOfCalls: 100` and your traffic is spread
  over 8 pods, each pod needs 100 calls of its own. At a low request rate to that
  dependency, a pod may take minutes to reach the threshold — during which it keeps
  calling a dead dependency. Size the window against **per-pod** rate, not total.
- **Observability.** "The circuit breaker is open" is a per-pod statement. Your
  dashboard must break the state metric out by pod, or you will see a state gauge
  that is simultaneously 0 and 1 and conclude the metric is broken.

There is no distributed circuit breaker in Resilience4j and you should not build
one. A breaker that requires a network call to consult has reintroduced the
dependency it was protecting you from.

### Difference 4 — a semaphore bulkhead does not create a thread boundary

Resilience4j has two bulkheads and they are not variants of the same thing:

| | `SemaphoreBulkhead` (default) | `FixedThreadPoolBulkhead` |
|---|---|---|
| Mechanism | A `java.util.concurrent.Semaphore` (Topic 97) | A separate bounded `ThreadPoolExecutor` with a bounded queue |
| Which thread runs the call | **The caller's thread** | A bulkhead thread |
| Return type | Anything | Must be `CompletableFuture` (`@Bulkhead(type = THREADPOOL)`) |
| Protects | The **downstream** from too many concurrent calls | The downstream **and** the caller's thread pool from being consumed |
| Cost | Almost nothing | A thread hop; loses `ThreadLocal`s (Topic 57's `SecurityContext`, Topic 120's MDC, Topic 119's trace) |

The semaphore bulkhead is the one you want by default. It bounds concurrency
without a thread hop, so your `SecurityContext`, MDC and trace context survive.

But be precise about what it does: with a semaphore bulkhead, **requests over the
limit block the caller's thread** for up to `maxWaitDuration`, then throw
`BulkheadFullException`. On a thread-per-request server that means Tomcat threads
are still occupied. The bulkhead protects the *payment gateway*; it protects
`orderflow`'s thread pool only insofar as `maxWaitDuration` is short.

Set `maxWaitDuration: 0` if you want immediate rejection, which is usually right for
a synchronous HTTP path.

---

## What is this?

Resilience4j is a small, functional-style fault-tolerance library. It is the
successor to Hystrix (which Netflix put into maintenance in 2018 and Spring Cloud
removed). Five independent modules, each usable alone:

| Module | Question it answers | State it keeps |
|---|---|---|
| **CircuitBreaker** | Is this dependency healthy enough to call? | A sliding window of recent outcomes, plus a state machine |
| **Retry** | Should I try again? | None across calls — per-invocation only |
| **Bulkhead** | How many of these may be in flight at once? | A permit count |
| **RateLimiter** | How many of these may start per unit time? | A token/cycle accounting |
| **TimeLimiter** | How long am I willing to wait? | None — it is a `Future.get(timeout)` |

### The circuit breaker state machine, precisely

Three operational states plus three manual ones.

```
                 failure rate >= threshold
    CLOSED  ------------------------------->  OPEN
      ^                                         |
      |                                         | waitDurationInOpenState elapses
      | failure rate < threshold                v
      +---------------------------------  HALF_OPEN
                                                |
                 failure rate >= threshold      |
                <-------------------------------+
                        back to OPEN
```

- **CLOSED** — calls pass through. Outcomes are recorded in the sliding window.
- **OPEN** — calls are **rejected immediately** with `CallNotPermittedException`.
  Nothing reaches the dependency. This is the point: the dependency gets to
  recover, and your caller gets a fast failure instead of a slow one.
- **HALF_OPEN** — a fixed number of trial calls
  (`permittedNumberOfCallsInHalfOpenState`) are allowed through. Their outcomes
  decide whether to close or re-open.
- Plus `DISABLED`, `FORCED_OPEN` and `METRICS_ONLY` — the last is genuinely useful:
  it records everything and never rejects, so you can watch what a breaker *would*
  have done before you let it act.

Two thresholds, not one:

- **`failureRateThreshold`** — percentage of recorded calls that failed.
- **`slowCallRateThreshold`** with **`slowCallDurationThreshold`** — percentage of
  calls that took longer than a duration you name.

The slow-call threshold is the one people omit, and it is the one that matters most
in practice. **A dependency that is slow but not failing is the dominant real-world
failure mode**, and it is far more dangerous than one that returns errors: slow
calls hold your threads and your connections (Topic 55) while a failure rate of 0%
keeps the breaker resolutely closed.

### Sliding windows

- **`COUNT_BASED`** with `slidingWindowSize: N` — the last N calls. A ring buffer.
- **`TIME_BASED`** with `slidingWindowSize: N` — the last N **seconds** of calls,
  held as N per-second partial aggregations.

`minimumNumberOfCalls` gates evaluation: below it, the failure rate is reported as
`-1` and the breaker will not open regardless. This is what prevents one failure at
2am from opening a breaker on a low-traffic endpoint.

For a low-and-bursty dependency, count-based is more predictable. For a steady
high-rate dependency, time-based bounds how stale the window can be.

### Retry, and the two arithmetic facts

```yaml
resilience4j:
  retry:
    instances:
      payments:
        maxAttempts: 3
```

**`maxAttempts` is TOTAL attempts, not additional retries.** `maxAttempts: 3` means
one original call plus two retries: a **3× load multiplier**. `maxAttempts: 4` is
the "three retries" of the interview cliché and is a **4×** multiplier.

Second fact: **the retry's wait blocks the calling thread**. With the synchronous
API there is no scheduler; `Retry` sleeps. Three retries at 1s, 2s, 4s means a
request thread is held for at least 7 seconds beyond the call durations. On a
Tomcat pool of 200 threads, a dependency that is timing out at 3s with 3 retries
holds each request for ~16 seconds, so 200 threads sustain roughly 12 requests per
second before the pool is exhausted and *every* endpoint on the service starts
queueing. This is Topic 55's lesson with a different cause.

### Jitter

```java
IntervalFunction.ofExponentialRandomBackoff(
        Duration.ofMillis(200),   // initial interval
        2.0,                      // multiplier
        0.5);                     // randomization factor: +/- 50%
```

Without the randomization factor, every client that failed at the same instant
retries at the same instant. The failing dependency comes back up and is hit by
the entire fleet simultaneously — and fails again, resynchronising everyone for the
next wave. That is the **retry storm**, and it is the assigned failure drill.

### Rate limiter and time limiter, briefly

- **RateLimiter** — `limitForPeriod` calls per `limitRefreshPeriod`, with
  `timeoutDuration` as how long a caller waits for a permit before
  `RequestNotPermitted`. Use it when the dependency has a **contractual** rate limit
  you must not exceed. Do not use it as a substitute for a bulkhead: a rate limiter
  bounds *starts per second*, a bulkhead bounds *concurrent in flight*. A dependency
  that gets slow will blow through a concurrency bound while staying under a rate
  bound.
- **TimeLimiter** — only meaningful with `CompletableFuture`; it cancels the future
  on timeout. For a synchronous `RestClient` call the timeout you actually need is
  the **HTTP client's** connect and read timeout, not a `TimeLimiter`. Setting a
  `TimeLimiter` and leaving the HTTP read timeout unset is a common and useless
  combination.

---

## Why does it matter?

**1. The payment gateway is the only external call in `orderflow`'s write path.**
Everything else is Postgres, Redis and Kafka inside your own network. An external
payment provider is a third party with its own incidents, its own deploys, and its
own rate limits. It is the single most likely cause of an `orderflow` outage that
originates outside `orderflow`.

**2. Without a bound, one slow dependency takes down every endpoint.** This is
Topic 55's mechanic and Topic 109's arithmetic, arriving from a new direction. A
payment call that goes from 80ms to 8 seconds does not slow down order placement —
it holds Tomcat threads, and once they are all held, `GET /api/products` (70% of
your load mix, touching nothing but Redis) starts timing out too. Customers see a
totally broken site because one third party is slow.

**3. Retries are the most commonly added and least commonly measured "reliability"
change.** "We added retries" is said in postmortems more often than almost any
other sentence, and roughly half the time the retries made the incident longer.
Being able to say *why*, with the multiplier arithmetic, is a senior-level
distinction.

**4. It is where circuit-breaker theory meets Java's proxy semantics.** You know
the pattern. The thing that will break your implementation is `@CircuitBreaker` on
a self-invoked method, or a retry inside the breaker. Neither is a distributed
systems question; both are Java questions.

---

## Machine-level reality

### The decorator chain is nested lambdas, and nothing more

There is no bytecode magic. `Decorators.ofSupplier(...).withCircuitBreaker(cb)`
returns a new `Supplier` whose `get()` is approximately:

```java
// The SHAPE of CircuitBreaker.decorateSupplier -- read the real source, Topic 125.
static <T> Supplier<T> decorateSupplier(CircuitBreaker cb, Supplier<T> supplier) {
    return () -> {
        cb.acquirePermission();                 // throws CallNotPermittedException if OPEN
        long start = cb.getCurrentTimestamp();
        try {
            T result = supplier.get();          // the INNER supplier
            cb.onResult(cb.getCurrentTimestamp() - start, cb.getTimestampUnit(), result);
            return result;
        } catch (Exception e) {
            cb.onError(cb.getCurrentTimestamp() - start, cb.getTimestampUnit(), e);
            throw e;
        }
    };
}
```

`Retry.decorateSupplier` is the same shape with a loop and a sleep. So
`withBulkhead(...).withCircuitBreaker(...).withRetry(...)` builds:

```
retrySupplier.get()
  -> loop {
       breakerSupplier.get()
         -> acquirePermission()
         -> bulkheadSupplier.get()
              -> semaphore.tryAcquire()
              -> gateway.charge(cmd)       <-- the actual network call
     }
```

**Read that stack and the ordering question answers itself.** The retry loop
re-enters `acquirePermission()` on each attempt. Once the breaker opens, the second
and third attempts throw `CallNotPermittedException` immediately and cost nothing.
Invert the order and the loop is *inside* `acquirePermission`, so the permission is
acquired once and the loop hammers the network under a single permit.

For the annotation form, the same nesting is produced by Spring AOP: each
Resilience4j aspect is a `MethodInterceptor` in the proxy's interceptor chain
(Topic 41), ordered by its `@Order`. Same structure, chosen by a property file
instead of by your code.

### Where each state machine lives

| Component | Where the state is | Concurrency mechanism | Scope |
|---|---|---|---|
| CircuitBreaker | `CircuitBreakerStateMachine`, an `AtomicReference<CircuitBreakerState>` in a registry-held singleton | CAS on the state reference; the sliding window uses atomic/aggregating structures rather than a lock | **Per JVM, per breaker name.** 8 pods = 8 breakers |
| Sliding window (count) | A fixed-size ring buffer of `Measurement` objects, plus a running total | Updated under the breaker's own synchronisation | Per breaker |
| Sliding window (time) | N per-second partial aggregations in a circular array; the head advances on the clock | Same | Per breaker |
| Retry | A per-invocation `Retry.Context` — attempt count and the last exception | None needed; it is stack-local | **Per call.** The registry-held `Retry` object holds only config and metrics |
| SemaphoreBulkhead | `java.util.concurrent.Semaphore` — an AQS-backed permit count (Topic 94, Topic 97) | AQS: CAS on the permit state, park/unpark on contention | Per JVM, per bulkhead name |
| ThreadPoolBulkhead | A `ThreadPoolExecutor` with a bounded `ArrayBlockingQueue` | The executor's own lock | Per JVM |
| RateLimiter (`AtomicRateLimiter`) | An immutable `State` record in an `AtomicReference`, holding the current cycle and remaining permits, advanced from `System.nanoTime()` | CAS loop; a losing thread recomputes and retries | Per JVM |

Three things fall out of that table.

**The breaker is a shared mutable singleton on the hot path.** Every call to the
payment gateway CASes the same `AtomicReference` and touches the same sliding
window. At `orderflow`'s baseline (10% of the mix is order placement, each placing
one payment) that is a low rate and contention is irrelevant. If you were to put a
breaker around a call made thousands of times per second per pod, the window update
becomes a contended shared cache line — Topic 96's false-sharing territory. Measure
before assuming; do not put a breaker around an in-process call.

**The retry has no cross-call memory.** It cannot know that the last 50 calls all
failed. Only the breaker knows that. This is precisely why the two are complements
and why a retry alone is not a resilience strategy: it is a load multiplier with no
off switch.

**`AtomicRateLimiter` uses `System.nanoTime()`**, which is a monotonic clock and not
wall-clock time. That is correct for interval measurement (Topic 77) and it means a
clock adjustment on the host does not break your rate limiter.

### What "the breaker opened" costs the caller

When OPEN, `acquirePermission()` throws `CallNotPermittedException` — a
`RuntimeException`. That is a **cheap** failure: no socket, no DNS, no TLS
handshake, no thread parked. Latency for a rejected call is microseconds, which is
why an open breaker makes your p99 *better*, not worse, during a downstream outage.

That improvement in the latency graph is a signal people misread. A p99 that
suddenly drops while the error rate climbs is the shape of "the breaker opened".
Learn that shape; it is on your dashboard within a minute of a downstream failure.

---

## Example 1 — minimal

### A dependency note, stated honestly

Resilience4j's Spring Boot starter artifact id has historically tracked the Boot
major version (`resilience4j-spring-boot2`, then `resilience4j-spring-boot3`).
**I do not know with certainty what the Boot 4 artifact id is on the version you
will use, and I am not going to guess one.** Check the Resilience4j release notes
or Maven Central before adding the dependency; if only a Boot 3 starter exists for
your Resilience4j version, the functional API below works on any Boot version with
no starter at all, and is what I would use in that case anyway.

The core modules are unambiguous:

```xml
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-circuitbreaker</artifactId>
</dependency>
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-retry</artifactId>
</dependency>
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-bulkhead</artifactId>
</dependency>
<!-- Micrometer binding for the metrics in the Measurement section -->
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-micrometer</artifactId>
</dependency>
```

Resilience4j is **not** in the Spring Boot BOM, so you must supply its version.
Manage it in `dependencyManagement` once (Topic 32), not per module.

### The functional API, with the order on the page

```java
package com.orderflow.payments;

import io.github.resilience4j.bulkhead.*;
import io.github.resilience4j.circuitbreaker.*;
import io.github.resilience4j.core.IntervalFunction;
import io.github.resilience4j.decorators.Decorators;
import io.github.resilience4j.retry.*;

public class GuardedPaymentGateway {

    private final PaymentGateway delegate;
    private final Retry retry;
    private final CircuitBreaker breaker;
    private final Bulkhead bulkhead;

    public GuardedPaymentGateway(PaymentGateway delegate) {
        this.delegate = delegate;

        this.breaker = CircuitBreaker.of("payments", CircuitBreakerConfig.custom()
                .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
                .slidingWindowSize(20)
                .minimumNumberOfCalls(10)
                .failureRateThreshold(50f)
                .slowCallDurationThreshold(Duration.ofSeconds(2))
                .slowCallRateThreshold(50f)
                .waitDurationInOpenState(Duration.ofSeconds(10))
                .permittedNumberOfCallsInHalfOpenState(3)
                // A declined card is NOT a gateway failure. See Trap 5.
                .ignoreExceptions(PaymentDeclinedException.class)
                .recordExceptions(IOException.class, TimeoutException.class,
                                  GatewayUnavailableException.class)
                .build());

        this.retry = Retry.of("payments", RetryConfig.custom()
                .maxAttempts(3)                       // 3 TOTAL = 3x multiplier
                .intervalFunction(IntervalFunction
                        .ofExponentialRandomBackoff(Duration.ofMillis(200), 2.0, 0.5))
                .retryExceptions(IOException.class, TimeoutException.class)
                .ignoreExceptions(PaymentDeclinedException.class,
                                  CallNotPermittedException.class)  // see Trap 2's fix
                .build());

        this.bulkhead = Bulkhead.of("payments", BulkheadConfig.custom()
                .maxConcurrentCalls(16)
                .maxWaitDuration(Duration.ZERO)       // reject immediately, do not queue
                .build());
    }

    public PaymentResult charge(ChargeCommand cmd) {
        Supplier<PaymentResult> decorated =
            Decorators.ofSupplier(() -> delegate.charge(cmd))
                      .withBulkhead(bulkhead)        // innermost: bounds concurrency
                      .withCircuitBreaker(breaker)   // middle: sees each attempt
                      .withRetry(retry)              // OUTERMOST: added last
                      .decorate();
        return decorated.get();
    }
}
```

**The one line to remember from this whole example:** in the `Decorators` builder,
**the last `withX` is the outermost wrapper**. `withRetry` last means
`Retry(CircuitBreaker(Bulkhead(call)))`, which is what the Mechanical statement
says you want.

### The annotation form, for comparison

```java
@Service
public class AnnotatedPaymentService {

    // Source order of these three annotations is IRRELEVANT to nesting.
    // Nesting comes from the aspect order properties.
    @Retry(name = "payments")
    @CircuitBreaker(name = "payments", fallbackMethod = "chargeFallback")
    @Bulkhead(name = "payments")
    public PaymentResult charge(ChargeCommand cmd) {
        return gateway.charge(cmd);
    }

    // Fallback signature: same parameters, plus the Throwable, same return type.
    // A mismatched signature fails at RUNTIME, not compile time.
    private PaymentResult chargeFallback(ChargeCommand cmd, CallNotPermittedException e) {
        return PaymentResult.deferred(cmd.orderId());
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payments:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        registerHealthIndicator: true
        ignoreExceptions:
          - com.orderflow.payments.PaymentDeclinedException
  retry:
    instances:
      payments:
        maxAttempts: 3
        waitDuration: 200ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true          # THE JITTER. Do not omit.
        randomizedWaitFactor: 0.5
        retryExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - io.github.resilience4j.circuitbreaker.CallNotPermittedException
          - com.orderflow.payments.PaymentDeclinedException
  bulkhead:
    instances:
      payments:
        maxConcurrentCalls: 16
        maxWaitDuration: 0
```

**Fallback methods are where correctness quietly leaks.** A fallback catches the
exception and returns a value, which means the caller cannot tell a real result
from a fallback. For a *read* (return a cached catalogue page) that is fine. For
`charge`, returning "success" from a fallback is fraud and returning "failed" may be
a lie — the gateway might have charged the card and then timed out. The only honest
fallback for a payment is one that returns a **third state**: "unknown, reconcile
later". That state has to exist in your domain model, which makes it a design
decision, not a configuration one.

### Run it

```bash
curl -s localhost:8080/actuator/circuitbreakers | jq .
curl -s localhost:8080/actuator/circuitbreakerevents/payments | jq '.circuitBreakerEvents[-5:]'
curl -s localhost:8080/actuator/retryevents/payments | jq '.retryEvents[-5:]'
curl -s localhost:8080/actuator/bulkheads | jq .
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,circuitbreakers,circuitbreakerevents,retryevents,bulkheads,ratelimiters
  health:
    circuitbreakers:
      enabled: true
```

**WHAT TO LOOK FOR** in `/actuator/circuitbreakers`:

| What you see | What it means |
|---|---|
| Your breaker name is listed with state `CLOSED` | The registry has it. It exists. |
| The name is **absent** | The instance was never created — a typo between the `@CircuitBreaker(name=...)` and the `instances:` key, or the starter is not on the classpath. |
| State `CLOSED` and, in `/circuitbreakerevents`, **zero events after traffic** | The aspect never ran. Self-invocation, or the bean is not proxied. **This is Trap 4 and it is the single most important thing this endpoint tells you.** |
| `failureRate: -1` | Fewer than `minimumNumberOfCalls` recorded. Not a bug; the window is not yet evaluable. |

---

## Example 2 — production scenario (on the project spine)

### The constraints, as numbers

From Topic 65's recorded baseline and Topic 109's pool work:

- Load mix **70% catalogue read / 20% order read / 10% order placement**, driven
  open-loop by k6 at a fixed arrival rate.
- Order placement is the only path that calls the payment gateway. So the gateway
  call rate is **10% of total arrival rate**, per pod after load balancing.
- `spring.datasource.hikari.maximum-pool-size: 16` on 8 vCPU (Topic 109).
- Tomcat `server.tomcat.threads.max` default 200.
- The payment provider's stated contract: **200 requests/second per API key**, p99
  under 400ms, and a documented 30-second incident-recovery expectation.
- `orderflow`'s SLO (Topic 130): 99.5% of `POST /api/orders` succeed, p99 under
  1.5s.

Those numbers determine every setting below. Copying a config from a blog does not.

### Sizing each guard from the numbers

**Bulkhead — `maxConcurrentCalls`.** This is the number that actually protects you.
Derive it from Little's Law: at the provider's normal p99 of 400ms, sustaining the
gateway rate you need requires `rate x latency` concurrent calls. But the number you
must size for is the **degraded** case: if the provider goes to 4 seconds, a
concurrency bound of 16 caps you at 4 calls/second and *rejects the rest
immediately* — which is exactly what you want, because the alternative is 200
Tomcat threads all parked in a socket read.

```
maxConcurrentCalls = 16
```

Why 16 and not 200: **16 is less than Hikari's pool size and far less than Tomcat's
thread count.** The bulkhead's job is to guarantee that a payment-gateway incident
can never consume more than 16 of the 200 request threads, leaving 184 for the 90%
of traffic that does not touch payments. Write that sentence in the config comment;
it is the entire justification.

**Retry — `maxAttempts`.** The provider is HTTP over the internet, so transient
network failures are real and a retry is justified. But: 10% of arrival rate is
already the gateway load, and `maxAttempts: 3` makes the worst case **3× that**.
Check it against the provider's 200 rps limit and your own peak. If 3× peak exceeds
200 rps you must either lower `maxAttempts` or add a `RateLimiter` at 200/s so you
degrade rather than get rate-limited by the provider (whose 429s you would then have
to decide whether to retry — do not; retrying a 429 is the definition of making it
worse).

**Circuit breaker — the window.** Per-pod gateway rate at baseline is
`0.10 x arrival_rate / 8 pods`. Set `minimumNumberOfCalls` so that a pod reaches it
within a few seconds at that rate. If a pod only sees 2 gateway calls per second,
`minimumNumberOfCalls: 100` means 50 seconds of calling a dead provider before the
breaker can even evaluate. Use a **time-based** window in that case and accept a
smaller minimum:

```yaml
slidingWindowType: TIME_BASED
slidingWindowSize: 30        # last 30 seconds
minimumNumberOfCalls: 10     # per POD
```

**`waitDurationInOpenState`.** The provider says 30 seconds to recover. Setting 5
seconds means six probe waves during a single incident, each of which re-hammers a
recovering provider. Setting 120 seconds means you stay down two minutes after they
recover. **10–30 seconds with `permittedNumberOfCallsInHalfOpenState: 3`** is the
defensible middle: the half-open probe is 3 calls, not a flood.

### The production configuration

```yaml
resilience4j:
  circuitbreaker:
    configs:
      external-http:                       # a shared base config
        slidingWindowType: TIME_BASED
        slidingWindowSize: 30
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallDurationThreshold: 1s      # provider p99 is 400ms; 1s is degraded
        slowCallRateThreshold: 50
        waitDurationInOpenState: 20s
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        registerHealthIndicator: true
        eventConsumerBufferSize: 100
    instances:
      payments:
        baseConfig: external-http
        ignoreExceptions:
          - com.orderflow.payments.PaymentDeclinedException
          - com.orderflow.payments.InsufficientFundsException

  retry:
    configs:
      external-http:
        maxAttempts: 3
        waitDuration: 200ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5
    instances:
      payments:
        baseConfig: external-http
        retryExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
        ignoreExceptions:
          - io.github.resilience4j.circuitbreaker.CallNotPermittedException
          - com.orderflow.payments.PaymentDeclinedException
          - com.orderflow.payments.PaymentAlreadyProcessedException

  bulkhead:
    instances:
      payments:
        maxConcurrentCalls: 16     # < hikari(16) and << tomcat(200): a payment
        maxWaitDuration: 0         # incident cannot consume the whole server

  ratelimiter:
    instances:
      payments:
        limitForPeriod: 180        # provider contract is 200/s; leave headroom
        limitRefreshPeriod: 1s
        timeoutDuration: 0         # do not queue; reject and let the caller decide
```

### The HTTP client timeouts, which are not optional

A circuit breaker cannot protect you from a call that never returns. If the socket
read has no timeout, the thread is held forever and the breaker's slow-call
threshold never fires because the call never completes.

```java
@Bean
RestClient paymentRestClient(RestClient.Builder builder) {
    var factory = new SimpleClientHttpRequestFactory();   // or the Apache/JDK factory
    factory.setConnectTimeout(Duration.ofMillis(500));
    factory.setReadTimeout(Duration.ofSeconds(2));        // < slowCallDurationThreshold? No --
                                                          // see the note below
    return builder.baseUrl(providerBaseUrl)
                  .requestFactory(factory)
                  .build();
}
```

The relationship between the read timeout and `slowCallDurationThreshold` deserves
a sentence, because getting it backwards makes one of them useless:

- **read timeout > slowCallDurationThreshold**: slow calls are *recorded as slow*
  and count toward `slowCallRateThreshold`. The breaker learns about degradation
  before calls start failing. This is what you want.
- **read timeout < slowCallDurationThreshold**: every slow call becomes a
  `SocketTimeoutException` before it can be classified as slow. The slow-call
  threshold never fires and only the failure threshold does. The setting is dead
  configuration.

So: `slowCallDurationThreshold: 1s`, `readTimeout: 2s`. Slow calls between 1s and 2s
are recorded as slow; anything beyond 2s is a failure. Both thresholds are live.

### Where the guard goes in the transaction

This is the part that connects to Phase 5 and it is where the real damage happens.

```java
// WRONG -- and it is Topic 55's drill wearing a Resilience4j costume.
@Transactional
public OrderId place(PlaceOrderCommand cmd) {
    var order = orders.save(Order.from(cmd));
    inventory.reserve(cmd.sku(), cmd.quantity());
    var result = guardedGateway.charge(chargeFrom(order));   // <-- HTTP inside the tx
    payments.save(Payment.from(result));
    return order.id();
}
```

A `@Transactional` method holds its Hikari connection for its entire duration. The
retry sleeps inside it. With `maxAttempts: 3` and exponential backoff, a degraded
provider holds a database connection for seconds per request. Pool size is 16. At
16 concurrent order placements, **every connection in the pool is parked waiting on
a third-party HTTP call**, and every endpoint in `orderflow` — including the 70%
catalogue read — fails with a Hikari timeout.

The bulkhead of 16 does not save you here, because 16 concurrent payment calls is
exactly enough to hold all 16 connections.

```java
// RIGHT -- the transaction ends before the network call begins.
public OrderId place(PlaceOrderCommand cmd) {
    OrderId id = orderTxService.createPendingOrderAndReserveStock(cmd);  // tx 1, short
    PaymentResult result = guardedGateway.charge(chargeFrom(id, cmd));   // NO tx open
    orderTxService.applyPaymentOutcome(id, result);                      // tx 2, short
    return id;
}
```

Three consequences you must accept, not work around:

1. **There is now an observable intermediate state**: an order in `PENDING_PAYMENT`
   with stock reserved and no payment. That is a saga (Topic 117), and it needs a
   compensation for the case where `charge` never returns an answer.
2. **`charge` must be idempotent**, because the retry may deliver the same charge
   twice. That is Topic 116, and it is Trap 3 below.
3. **`applyPaymentOutcome` may never run** if the pod dies between step 2 and step
   3. You need a reconciliation job that finds `PENDING_PAYMENT` orders older than
   N minutes and asks the provider what happened. This is not optional and it is
   the thing teams skip.

Note that `orderTxService` must be a **separate bean**, not another method on this
class — self-invocation would lose the `@Transactional` proxy (Topic 40, again).

### The `[BOOT 3.x DELTA]` box

| Concern | Boot 3.x / Security 6 era | Boot 4.1 | Note |
|---|---|---|---|
| Starter artifact | `resilience4j-spring-boot3` | **Verify the current artifact id** — it has tracked the Boot major | Do not assume; check Maven Central |
| Actuator endpoint ids | `circuitbreakers`, `circuitbreakerevents`, `retryevents`, `bulkheads`, `ratelimiters` | Same ids | Exposure config unchanged |
| Metrics binding | `resilience4j-micrometer` | Same | Boot 4 ships an OpenTelemetry starter; the Micrometer names below still apply |
| Config prefix | `resilience4j.*` | Same | The library's own namespace, not Spring's |
| `RestTemplate` vs `RestClient` | `RestTemplate` common | `RestClient` is the current synchronous client; Boot 4 also auto-configures **HTTP Service Clients** (annotated interfaces) | If you adopt HTTP Service Clients, the Resilience4j decoration point moves — you decorate the interface's *implementation* bean or wrap the call site, not the interface |

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — retries added "for reliability" amplify the outage

**Wrong approach**

```yaml
resilience4j:
  retry:
    instances:
      payments:
        maxAttempts: 4
        waitDuration: 1s          # fixed. no backoff, no jitter.
```

No circuit breaker. No bulkhead. This is what gets added after the first
transient-failure incident, and it is added by someone acting in good faith.

**Exact symptom**

The provider has a partial outage — say it is returning 503 for 30% of requests
because one of its shards is down. Before your change, 30% of `orderflow`'s
payments fail. After your change:

1. Your request rate to the provider goes to **4× your normal rate**, because every
   failing request is retried three more times.
2. The provider, now at 4× load, degrades further. The 30% failure rate becomes
   80%.
3. Which means more of your requests retry. Your multiplier approaches the full 4×
   on nearly all traffic.
4. Every retried request holds a Tomcat thread through three 1-second waits plus
   four call durations.

The measurements that show it, at your Topic 65 baseline:

| Instrument | What you see | What it means |
|---|---|---|
| `resilience4j_retry_calls_total{kind="failed_with_retry"}` climbing | Retries are happening and not helping | The retries are pure amplification |
| Outbound request rate to the provider (their dashboard, or your client-side counter) | ~4× the order-placement rate | The multiplier, measured |
| `hikaricp_connections_pending` climbing, `http_server_requests` p99 climbing on **`GET /api/products`** | A payments incident is degrading an endpoint that touches no payment code | Thread-pool exhaustion. The blast radius is the whole service. |
| Provider returns 429 | You have crossed their rate limit | You are now being throttled *because of your own retries* |

**Root cause**

Two independent errors compounding.

**(a) The multiplier.** `maxAttempts: 4` is one call plus three retries: a **4×
load multiplier applied precisely when the dependency is least able to serve
load**. A retry is only useful if the failure is *transient and independent*. A
failure caused by overload is neither: it is caused by the load you are adding.

**(b) No jitter.** `waitDuration: 1s` fixed means every client that failed at
t=0 retries at t=1s, t=2s, t=3s. Two hundred concurrent requests become 200
simultaneous retries, three times. The provider sees three sharp spikes rather than
a smeared load. When it briefly recovers, the next synchronised wave knocks it over
again. This is the retry storm and it is the assigned Failure drill.

**Fix — four changes, none of which is "remove the retries"**

```yaml
resilience4j:
  retry:
    instances:
      payments:
        maxAttempts: 3                       # 1. 3x, not 4x
        waitDuration: 200ms
        enableExponentialBackoff: true       # 2. spread the attempts out
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true           # 3. THE JITTER: desynchronise clients
        randomizedWaitFactor: 0.5
        retryExceptions:                     # 4. retry only what a retry can fix
          - java.io.IOException
          - java.net.SocketTimeoutException
        ignoreExceptions:
          - com.orderflow.payments.PaymentDeclinedException
```

Plus the two things that are not retry configuration and matter more:

- **A circuit breaker outside the retry** so the amplification has a hard stop.
  Once the breaker opens, attempts 2 and 3 cost nothing and reach nothing.
- **A bulkhead** so the concurrent in-flight count is bounded regardless.

State the invariant you are enforcing: *the maximum load `orderflow` can place on
the payment provider is `maxConcurrentCalls` in flight, and zero while the breaker
is open.* That is a number you can give the provider's team. "We retry three times"
is not.

**And a rule with no exceptions: never retry a 429 or a 503 with a `Retry-After`
header on your own schedule.** The dependency has told you when to come back.
Honouring it is the whole point of the header; ignoring it is the definition of
making an incident worse.

---

### Trap 2 — retrying INSIDE the circuit breaker, so the breaker never sees the failures

**Wrong approach**

```java
// The Decorators builder: LAST added is OUTERMOST.
// This puts the breaker outside and the retry inside -- the wrong way round.
Supplier<PaymentResult> decorated =
    Decorators.ofSupplier(() -> gateway.charge(cmd))
              .withRetry(retry)             // added first  -> INNER
              .withCircuitBreaker(breaker)  // added last   -> OUTER
              .decorate();
```

Or, with annotations, the same inversion achieved by "tidying up" the aspect order:

```yaml
resilience4j:
  circuitbreaker:
    circuitBreakerAspectOrder: <higher than retry's>   # breaker becomes outermost
```

**Exact symptom**

The provider is hard down. Every call fails. And yet:

```bash
curl -s localhost:8080/actuator/circuitbreakers | jq '.circuitBreakers'
```

| What you see | What it means |
|---|---|
| State `CLOSED`, `bufferedCalls` far lower than your request rate | The breaker is counting **one call per client request**, not one per attempt. It is seeing a third of the failures it should. |
| State `CLOSED` while `resilience4j_retry_calls_total{kind="successful_with_retry"}` is high | Worse: the retry is *succeeding* on attempt 2 or 3, so the breaker records **successes**. A dependency failing two thirds of the time looks perfectly healthy to the breaker. |
| The breaker eventually opens, but much later than you expected | The arithmetic: with `maxAttempts: 3`, it takes 3× the client request volume to accumulate `minimumNumberOfCalls`. |
| Latency p99 on `POST /api/orders` is enormous while error rate is low | Every request is paying three attempts plus two backoff waits. The breaker's job — fast failure — is not happening. |

The second row is the genuinely dangerous one. A breaker that records successes
during a degradation is not a broken breaker; it is a breaker doing exactly what it
was asked, on the wrong input.

**Root cause**

The decorator nesting. With `CircuitBreaker(Retry(call))`:

```
breakerSupplier.get()
  -> acquirePermission()          // ONCE
  -> retrySupplier.get()
       -> loop { gateway.charge() }   // 3 network attempts under ONE permission
  -> onSuccess() or onError()     // ONE recorded outcome
```

The permission is acquired once, the loop hammers the network under it, and one
outcome is recorded. The breaker's sliding window is measuring *client requests*,
not *dependency calls*. Its failure-rate threshold is therefore about a quantity you
did not mean.

With `Retry(CircuitBreaker(call))`:

```
retrySupplier.get()
  -> loop {
       breakerSupplier.get()
         -> acquirePermission()      // EVERY attempt
         -> gateway.charge()
         -> onError()                // EVERY attempt recorded
     }
```

Every attempt is measured, and once the breaker opens, the remaining attempts throw
`CallNotPermittedException` immediately.

**Fix**

Put the retry outermost, and — this is the part people miss — **tell the retry not
to retry `CallNotPermittedException`**:

```java
var retryConfig = RetryConfig.custom()
        .maxAttempts(3)
        .intervalFunction(IntervalFunction
                .ofExponentialRandomBackoff(Duration.ofMillis(200), 2.0, 0.5))
        .ignoreExceptions(CallNotPermittedException.class)   // <-- essential
        .build();
```

Without that line, an open breaker produces `CallNotPermittedException`, the retry
treats it as a retryable failure, and you burn all three attempts and both backoff
sleeps to achieve nothing. The request takes ~600ms to fail in a way that could have
taken 50 microseconds. It is not a correctness bug, but at 10% of your load mix it
is a visible p99 regression during exactly the incident where you wanted fast
failure.

**Prove the order rather than trusting it:**

```java
@Test
void retry_is_outside_the_circuit_breaker() {
    var breaker = CircuitBreaker.ofDefaults("t");
    var retry   = Retry.of("t", RetryConfig.custom().maxAttempts(3)
                       .waitDuration(Duration.ofMillis(1)).build());
    var attempts = new AtomicInteger();

    Supplier<String> s = Decorators
            .ofSupplier(() -> { attempts.incrementAndGet(); throw new IOException(); })
            .withCircuitBreaker(breaker)
            .withRetry(retry)          // outermost
            .decorate();

    assertThatThrownBy(s::get).isInstanceOf(IOException.class);

    assertThat(attempts.get()).isEqualTo(3);
    // The DECISIVE assertion: the breaker recorded THREE calls, not one.
    assertThat(breaker.getMetrics().getNumberOfBufferedCalls()).isEqualTo(3);
}
```

`getNumberOfBufferedCalls() == 3` is the assertion that fails if anyone inverts the
order. Put it in the codebase.

---

### Trap 3 — retrying a non-idempotent payment call

**Wrong approach**

```java
@Retry(name = "payments")
public PaymentResult charge(ChargeCommand cmd) {
    // POST /v1/charges  { "amount": 4999, "currency": "GBP", "source": "..." }
    return restClient.post().uri("/v1/charges").body(cmd).retrieve().body(PaymentResult.class);
}
```

The retry fires on `SocketTimeoutException`. That looks safe: the call timed out,
so presumably nothing happened.

**Exact symptom**

The customer is charged twice for one order. Your database has one `payment` row.
The provider's dashboard has two charges with the same amount, seconds apart, for
the same card.

You will not find this in your logs, because from `orderflow`'s point of view
nothing went wrong: attempt 1 timed out, attempt 2 succeeded, one `PaymentResult`
was returned, one row was written. **The bug is invisible from inside the service
and is reported by the customer or by the provider's duplicate-charge alerting.**

Reproduce it deliberately:

```bash
# Point the gateway at a stub that sleeps past your read timeout, then returns 200
# and records the charge. Place one order.
curl -s -X POST localhost:8080/api/orders -H "Authorization: Bearer $CUST" \
     -H 'Content-Type: application/json' \
     -d '{"productSku":"SKU-1001","quantity":1}'

# Then count what the stub actually recorded.
curl -s localhost:9099/stub/charges | jq 'length'
```

| What you see | What it means |
|---|---|
| The stub recorded **more than one** charge for one order | The retry duplicated a side effect. This is the bug, demonstrated. |
| The stub recorded exactly one | Either the retry did not fire, or your idempotency key worked. Check `retryevents` to tell which. |

**Root cause**

A timeout is **not** evidence that the request did not happen. It is evidence that
you did not receive a response. The three possibilities are indistinguishable from
the client:

1. The request never arrived. Retry is safe.
2. The request arrived, was processed, and the **response** was lost. Retry
   duplicates the effect.
3. The request arrived and is **still being processed**. Retry races it.

You cannot tell which. Therefore: **retrying a non-idempotent operation is always
unsafe, and a timeout is the most dangerous case, not the safest.** The intuition
"it timed out, so it probably didn't happen" is backwards for a slow dependency,
where a timeout most often means the work *did* happen and took too long.

**Fix**

Make the operation idempotent at the provider, then retry freely. Every serious
payment provider supports an idempotency key header for exactly this reason.

```java
public PaymentResult charge(ChargeCommand cmd) {
    return restClient.post()
        .uri("/v1/charges")
        // The key is DERIVED FROM THE ORDER and is STABLE ACROSS RETRIES.
        // Generating it inside this method with UUID.randomUUID() defeats the
        // entire mechanism -- every attempt would get a different key.
        .header("Idempotency-Key", cmd.idempotencyKey())
        .body(cmd)
        .retrieve()
        .body(PaymentResult.class);
}
```

```java
public record ChargeCommand(OrderId orderId, Money amount, String source) {
    /** Deterministic: same order + same attempt semantics = same key, forever. */
    public String idempotencyKey() {
        return "orderflow-charge-" + orderId().value();
    }
}
```

Three rules that make this actually work:

1. **The key is derived from the business operation, not generated per attempt.**
   `UUID.randomUUID()` inside the retried method is the single most common way this
   is broken.
2. **The key must be stable across process restarts.** If `orderflow` crashes and a
   reconciliation job re-attempts the charge, it must produce the same key. Derive
   it from the order id, or store it on the order row.
3. **If the provider does not support idempotency keys, you must not retry the
   charge.** Retry the *query* instead: on timeout, do not re-POST; GET the charge
   status by your reference and act on what you find. This is more work and it is
   the only correct option.

Now generalise the rule, because it applies to every retry you will ever configure:

| Operation | Retryable? |
|---|---|
| `GET /api/products/{sku}` | Yes — no side effect |
| `POST /v1/charges` without an idempotency key | **No** |
| `POST /v1/charges` with a stable idempotency key | Yes |
| A Kafka produce with `enable.idempotence=true` | Yes (Topic 114) |
| `INSERT` guarded by a unique constraint | Yes — the second one fails cleanly (Topic 116) |
| `UPDATE inventory SET qty = qty - 1` | **No** — not idempotent |
| `UPDATE inventory SET qty = 5 WHERE version = 7` | Yes — conditional and absolute (Topic 52) |

The pattern: **an operation is retry-safe when repeating it produces the same final
state.** Absolute assignments and constraint-guarded inserts are; relative
adjustments are not.

Forward reference: Topic 116 builds the server side of this — how `orderflow` itself
honours an `Idempotency-Key` header on `POST /orders` and `POST /payments`.

---

### Trap 4 — `@CircuitBreaker` on a self-invoked method, so nothing is protected

**Wrong approach**

```java
@Service
public class PaymentService {

    public PaymentResult payForOrder(Order order) {
        return charge(chargeFrom(order));      // <-- 'this.charge(...)'
    }

    @CircuitBreaker(name = "payments", fallbackMethod = "fallback")
    @Retry(name = "payments")
    @Bulkhead(name = "payments")
    public PaymentResult charge(ChargeCommand cmd) {
        return gateway.charge(cmd);
    }

    private PaymentResult fallback(ChargeCommand cmd, Throwable t) {
        return PaymentResult.deferred(cmd.orderId());
    }
}
```

Every annotation is correct. Every configuration key is correct. Nothing works.

**Exact symptom**

The provider goes down. The breaker never opens. There is no fallback. The bulkhead
does not bound anything. Requests pile up until Tomcat's thread pool is exhausted.

The diagnostic that settles it in five seconds:

```bash
curl -s localhost:8080/actuator/circuitbreakerevents/payments | jq '.circuitBreakerEvents | length'
```

| What you see | What it means |
|---|---|
| `0` after you have definitely made payment calls | **The aspect never ran.** Nothing was decorated. This is the whole diagnosis. |
| Events present, all `SUCCESS` | The aspect runs; the calls are genuinely succeeding. Different problem. |
| The breaker `payments` is **not listed** in `/actuator/circuitbreakers` at all | The instance was never created — a name typo, or the starter is missing |

Confirm the cause:

```java
@Component
class ProxyProbe {
    ProxyProbe(PaymentService paymentService) {
        System.out.println("PaymentService is " + paymentService.getClass().getName());
    }
}
```

`...PaymentService$$SpringCGLIB$$0` means a proxy exists and the internal call
bypassed it. A plain class name means no aspect matched at all — check that the
Resilience4j starter is on the classpath.

**Root cause**

Topic 40, verbatim. The advice lives on the proxy. `payForOrder` calls
`this.charge(...)`, a plain virtual invocation on the target object. The proxy is
not in the call path, so no interceptor runs.

The private `fallback` method compounds it: even when the proxy *is* used,
Resilience4j resolves the fallback reflectively by name and signature, so a typo,
a wrong parameter list, or the wrong `Throwable` type produces a runtime failure
rather than a compile error.

**Fix, in preference order**

*Fix 1 — the functional API on a dedicated collaborator.* Best, because the
decoration is visible and the boundary is a real object:

```java
@Component
public class GuardedPaymentGateway {          // one bean, one job
    // as in Example 1: Decorators.ofSupplier(...).withBulkhead().withCircuitBreaker().withRetry()
}

@Service
public class PaymentService {
    private final GuardedPaymentGateway guarded;   // injected -> a real call boundary
    public PaymentResult payForOrder(Order order) {
        return guarded.charge(chargeFrom(order));
    }
}
```

*Fix 2 — extract the annotated method to a separate bean.* Same effect, keeps the
annotations.

*Fix 3 — self-injection or `ObjectProvider`.* Works, and makes reviewers ask why.

*What does not work:* moving the annotations onto `payForOrder`. That changes what
is guarded — now the whole method including any database work is inside the
bulkhead, which is a different and usually worse design.

**Standing rule:** put a `getNumberOfBufferedCalls() > 0` assertion in an
integration test for every breaker you configure. A breaker with zero recorded calls
is not a healthy breaker; it is a breaker that is not there.

---

### Trap 5 — the breaker counts business outcomes as failures

**Wrong approach**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payments:
        failureRateThreshold: 50
        # recordExceptions / ignoreExceptions omitted entirely
```

By default the breaker records **every** exception as a failure. Your gateway
throws `PaymentDeclinedException` when the card is declined — a completely normal,
completely healthy outcome that happens on a few percent of real traffic and spikes
during, say, a fraud-rule change at the issuer.

**Exact symptom**

An issuer starts declining a category of cards. Decline rate goes from 3% to 60%.
The breaker sees a 60% failure rate, crosses its 50% threshold, and **opens**.

Now `orderflow` rejects **100%** of payment attempts, including the 40% that would
have succeeded. You have converted a partial degradation into a total outage, in
the name of resilience.

```bash
curl -s localhost:8080/actuator/circuitbreakerevents/payments \
  | jq -r '.circuitBreakerEvents[] | "\(.type) \(.errorMessage // "")"' | tail -20
```

| What you see | What it means |
|---|---|
| Many `ERROR` events whose message is a **business** decline, then a `STATE_TRANSITION` to OPEN | The breaker is counting declines. This is the bug. |
| `ERROR` events that are `SocketTimeoutException` / `IOException` | Genuine transport failures. The breaker is doing its job. |
| A `STATE_TRANSITION` to OPEN with `failureRate` just over threshold and a mix of both | You have a real problem *and* a classification problem. Fix the classification first or you cannot see the real one. |

**The mirror-image error is equally common:** listing `recordExceptions` so narrowly
that a genuine failure mode is excluded. If you record only `IOException` and the
provider starts returning HTTP 500 — which your client turns into an
`HttpServerErrorException` — the breaker records **successes** and never opens.

**Root cause**

The breaker's question is *"is the dependency healthy?"*, not *"did this business
operation succeed?"* A declined card is a healthy dependency answering correctly. A
500 is an unhealthy dependency. An exception type is a poor proxy for that
distinction unless you make the mapping explicit.

**Fix**

Classify explicitly, in both directions, and prefer a predicate for anything
subtle:

```java
CircuitBreakerConfig.custom()
    .failureRateThreshold(50f)
    // Health signals only.
    .recordExceptions(
        java.io.IOException.class,
        java.net.SocketTimeoutException.class,
        org.springframework.web.client.HttpServerErrorException.class,  // 5xx
        GatewayUnavailableException.class)
    // Business outcomes -- never a health signal.
    .ignoreExceptions(
        PaymentDeclinedException.class,
        InsufficientFundsException.class,
        InvalidCardException.class)
    .build();
```

When the mapping is not expressible by type — for instance HTTP 429, which is the
provider telling you *you* are the problem — use a predicate:

```java
.recordFailurePredicate(throwable -> {
    if (throwable instanceof HttpClientErrorException e) {
        // 429: back off, yes -- but it is not the provider being unhealthy.
        // Let the RateLimiter and the retry's Retry-After handling deal with it.
        return e.getStatusCode() != HttpStatus.TOO_MANY_REQUESTS
            && e.getStatusCode().is5xxServerError();
    }
    return throwable instanceof IOException;
})
```

And design the domain so the distinction exists at all. If `charge` returns
`PaymentResult.declined(...)` instead of throwing, the breaker never sees declines
and the classification problem disappears. **Modelling business outcomes as return
values and infrastructure failures as exceptions makes this trap structurally
impossible** — which is a Topic 09 and Topic 28 argument arriving with a
distributed-systems consequence. A sealed `PaymentResult` over records is the
cleanest form:

```java
public sealed interface PaymentResult {
    record Captured(String providerRef, Money amount) implements PaymentResult {}
    record Declined(String reasonCode) implements PaymentResult {}
    record Deferred(OrderId orderId) implements PaymentResult {}   // "unknown, reconcile"
}
```

Only transport problems throw. The breaker's `recordExceptions` list then writes
itself.

---

## Hands-on proof

### Proof 1 — print the actual aspect order at startup

Do not trust the documented default; read it from your running application.

```java
@Component
class AspectOrderProbe {

    AspectOrderProbe(ApplicationContext ctx) {
        ctx.getBeansOfType(Ordered.class).forEach((name, bean) -> {
            if (bean.getClass().getName().contains("resilience4j")) {
                System.out.printf("order=%d  %s%n", bean.getOrder(), bean.getClass().getName());
            }
        });
    }
}
```

**WHAT TO LOOK FOR:** one line per Resilience4j aspect with its numeric order.

| What you see | What it means |
|---|---|
| Retry's order is the **highest** number | Retry is outermost. Correct — this matches the Mechanical statement. |
| CircuitBreaker's order is higher than Retry's | The breaker is outermost. **This is Trap 2.** Fix the `*AspectOrder` properties. |
| Bulkhead's order is the **lowest** | Bulkhead is innermost. Correct. |
| No lines printed | The aspects are not registered — the starter is missing, or these beans do not implement `Ordered` on your version. Fall back to Proof 2, which measures behaviour rather than configuration. |

Proof 2 is the more reliable instrument, because it measures what actually happens
rather than what is configured.

### Proof 2 — assert the ordering behaviourally

The unit test from Trap 2's fix is the proof. Its decisive assertion is
`breaker.getMetrics().getNumberOfBufferedCalls()`:

- `== maxAttempts` → retry is outside the breaker. Correct.
- `== 1` → retry is inside the breaker. Trap 2.

Run it against the **annotation** configuration too, by calling through the Spring
proxy in a `@SpringBootTest` and reading the registry:

```java
@SpringBootTest
class AnnotationOrderTest {

    @Autowired PaymentService service;                  // the PROXY
    @Autowired CircuitBreakerRegistry breakers;
    @MockitoBean PaymentGateway gateway;

    @Test
    void breaker_records_every_attempt() {
        when(gateway.charge(any())).thenThrow(new IOException("down"));
        var before = breakers.circuitBreaker("payments")
                             .getMetrics().getNumberOfBufferedCalls();

        assertThatThrownBy(() -> service.charge(aCharge()));

        var after = breakers.circuitBreaker("payments")
                            .getMetrics().getNumberOfBufferedCalls();
        assertThat(after - before).isEqualTo(3);   // maxAttempts, not 1
    }
}
```

`@MockitoBean` is the Boot 3.4+/4.x replacement for `@MockBean`. On Boot 3.3 and
earlier it is `@MockBean`.

### Proof 3 — watch the breaker transition, live

```bash
watch -n 1 "curl -s localhost:8080/actuator/circuitbreakers \
  | jq -r '.circuitBreakers | to_entries[] | \"\(.key) \(.value.state)\"'"
```

In another terminal, take the payment stub down, then bring it back.

**WHAT TO LOOK FOR** — the sequence of states, and the *time* between them:

| What you see | What it means |
|---|---|
| `CLOSED` → `OPEN` | The failure or slow-call rate crossed the threshold after `minimumNumberOfCalls`. |
| Stays `CLOSED` while everything fails | Either `minimumNumberOfCalls` not reached (check `failureRate: -1`), or the exception is not in `recordExceptions` (Trap 5), or the retry is inside (Trap 2). |
| `OPEN` → `HALF_OPEN` after `waitDurationInOpenState` | Automatic transition is on. Without `automaticTransitionFromOpenToHalfOpenEnabled`, the transition happens on the *next call*, not on a timer — so a quiet period leaves it showing OPEN. |
| `HALF_OPEN` → `CLOSED` after you restore the stub | Recovery worked. |
| `HALF_OPEN` → `OPEN` repeatedly | The dependency is not actually recovered, or `permittedNumberOfCallsInHalfOpenState` is too small to be statistically meaningful. |

### Proof 4 — the event stream, which is the best diagnostic in the library

```bash
curl -s localhost:8080/actuator/circuitbreakerevents/payments \
  | jq -r '.circuitBreakerEvents[] | "\(.creationTime) \(.type) \(.errorMessage // .stateTransition // "")"' \
  | tail -30
```

*Illustration of the event-line shape, not captured output:*

```
<timestamp> ERROR             java.net.SocketTimeoutException: Read timed out
<timestamp> ERROR             java.net.SocketTimeoutException: Read timed out
<timestamp> FAILURE_RATE_EXCEEDED
<timestamp> STATE_TRANSITION  CLOSED_TO_OPEN
<timestamp> NOT_PERMITTED
<timestamp> NOT_PERMITTED
```

`NOT_PERMITTED` events are calls the breaker rejected. Their **count** is the
measurement of how much load you prevented from reaching the dependency, and it is
the number to put in the incident review.

Note `eventConsumerBufferSize` (default is small): this endpoint shows only the
last N events per instance. It is a debugging tool, not an audit log. For the audit
log, register an event listener that writes to your structured logs (Topic 120):

```java
@Bean
RegistryEventConsumer<CircuitBreaker> breakerLogging() {
    return new RegistryEventConsumer<>() {
        @Override public void onEntryAddedEvent(EntryAddedEvent<CircuitBreaker> e) {
            e.getAddedEntry().getEventPublisher()
             .onStateTransition(ev -> log.warn("circuit_breaker_transition name={} from={} to={}",
                 ev.getCircuitBreakerName(),
                 ev.getStateTransition().getFromState(),
                 ev.getStateTransition().getToState()));
        }
        @Override public void onEntryRemovedEvent(EntryRemovedEvent<CircuitBreaker> e) { }
        @Override public void onEntryReplacedEvent(EntryReplacedEvent<CircuitBreaker> e) { }
    };
}
```

A state transition is an event a human should see. Log it at `WARN` with the
breaker name, and alert on it.

---

## Failure drill

**The assignment:** stub the payment gateway to fail. Retry without jitter from 200
concurrent requests. Observe the synchronised retry storm. Add jitter, a breaker and
a bulkhead. Re-measure.

This drill produces numbers. Every number below comes from **your** run. There are
no numbers printed in this section, deliberately.

### Step 0 — the stub

You need a payment gateway you control. A tiny WireMock or a plain Spring Boot app
on port 9099 is enough. It must do three things: fail on command, count what it
received, and report per-second arrival counts.

```java
// A minimal controllable stub. Run it as a separate process.
@RestController
class PaymentStub {

    private final AtomicBoolean healthy = new AtomicBoolean(true);
    private final AtomicLong delayMillis = new AtomicLong(50);
    private final Map<Long, LongAdder> arrivalsPerSecond = new ConcurrentHashMap<>();

    @PostMapping("/v1/charges")
    ResponseEntity<String> charge(@RequestHeader(value = "Idempotency-Key", required = false) String key)
            throws InterruptedException {
        arrivalsPerSecond.computeIfAbsent(Instant.now().getEpochSecond(),
                                          k -> new LongAdder()).increment();
        Thread.sleep(delayMillis.get());
        return healthy.get()
            ? ResponseEntity.ok("{\"status\":\"captured\"}")
            : ResponseEntity.status(503).body("{\"error\":\"unavailable\"}");
    }

    @PostMapping("/control/down")    void down()  { healthy.set(false); }
    @PostMapping("/control/up")      void up()    { healthy.set(true); }
    @PostMapping("/control/slow")    void slow(@RequestParam long ms) { delayMillis.set(ms); }

    /** THE INSTRUMENT: arrivals bucketed by second. The storm is visible here. */
    @GetMapping("/control/arrivals")
    Map<Long, Long> arrivals() {
        return arrivalsPerSecond.entrySet().stream()
            .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().sum(), (a,b)->a, TreeMap::new));
    }

    @PostMapping("/control/reset")   void reset() { arrivalsPerSecond.clear(); }
}
```

**`/control/arrivals` is the whole drill.** The retry storm is a *shape in a
per-second histogram*, and if you only measure totals you will not see it.

### Step 1 — the naive configuration

```yaml
resilience4j:
  retry:
    instances:
      payments:
        maxAttempts: 4
        waitDuration: 1s
        enableExponentialBackoff: false
        enableRandomizedWait: false     # NO JITTER -- this is the point
# no circuitbreaker instance
# no bulkhead instance
```

### Step 2 — fire 200 concurrent requests at a dead gateway

```bash
curl -X POST localhost:9099/control/reset
curl -X POST localhost:9099/control/down

# 200 concurrent order placements, arriving as close to simultaneously as possible.
# --parallel-immediate matters: it starts them together rather than ramping.
seq 200 | xargs -P 200 -I{} curl -s -o /dev/null \
  -X POST localhost:8080/api/orders \
  -H "Authorization: Bearer $CUST" \
  -H 'Content-Type: application/json' \
  -d '{"productSku":"SKU-1001","quantity":1}'

curl -s localhost:9099/control/arrivals | jq .
```

An alternative that is easier to make truly simultaneous, and which you already
have from Topic 65:

```javascript
// k6: 200 iterations that all start at once.
export const options = {
  scenarios: {
    burst: { executor: 'per-vu-iterations', vus: 200, iterations: 1, maxDuration: '60s' },
  },
};
```

### Step 3 — WHAT TO LOOK FOR

Read `/control/arrivals` as a per-second histogram and write down the shape.

| What you see | What it means |
|---|---|
| A tall spike at second `t`, then near-zero, then a **second tall spike** at `t+1`, another at `t+2`, another at `t+3` | **The retry storm.** All 200 clients failed together and retried together. Four distinct spikes = `maxAttempts: 4`. This is the thing you came to see. |
| Total arrivals ≈ 4 × 200 | The multiplier, measured directly. Write this number down. |
| Spikes of decreasing height | Some requests gave up early (a client timeout) or the load generator could not sustain the concurrency. Note it — it changes the interpretation. |
| A smooth plateau rather than spikes | Your requests did not actually start simultaneously. Fix the load generator before drawing conclusions. |

And on the `orderflow` side, during the same window:

```bash
curl -s localhost:8080/actuator/metrics/http.server.requests | jq .
curl -s localhost:8080/actuator/metrics/tomcat.threads.busy | jq .
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending | jq .
```

| What you see | What it means |
|---|---|
| `tomcat.threads.busy` at or near max for the duration of the storm | Every request thread is parked in a retry sleep or a socket read. |
| `hikaricp.connections.pending` above zero | If the gateway call is inside a transaction, you are also holding connections. This is Topic 55/109 and it means the blast radius includes the catalogue endpoint. |
| p99 on `GET /api/products` degraded | **The blast radius, quantified.** A payment incident is now a catalogue incident. This is the number that makes the case to your team. |

### Step 4 — add jitter only, and re-measure

```yaml
        maxAttempts: 4
        waitDuration: 200ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5
```

| What you see | What it means |
|---|---|
| The per-second histogram is **smeared** instead of spiked; peak arrivals per second drops substantially | Jitter desynchronised the clients. Record the peak-per-second before and after — that ratio is the value of jitter, in your numbers. |
| **Total** arrivals unchanged (still ≈ 4 × 200) | **Jitter changes the shape, not the volume.** This is the crucial observation of the whole drill: jitter alone does not reduce load, it only spreads it. If your dependency's problem is total capacity, jitter has not helped it at all. |
| The storm lasts longer in wall-clock time | Of course it does — you spread the same work out. |

### Step 5 — add the circuit breaker, and re-measure

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payments:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 20s
        automaticTransitionFromOpenToHalfOpenEnabled: true
  retry:
    instances:
      payments:
        ignoreExceptions:
          - io.github.resilience4j.circuitbreaker.CallNotPermittedException
```

| What you see | What it means |
|---|---|
| Arrivals stop entirely a fraction of a second into the storm | The breaker opened. **Total arrivals is now bounded by roughly `minimumNumberOfCalls` plus in-flight**, not by 4 × 200. Compare this total to step 2's. That ratio is what a circuit breaker buys. |
| One small spike at `t + waitDurationInOpenState` | The half-open probe: `permittedNumberOfCallsInHalfOpenState` calls, not a flood. |
| `circuitbreakerevents` shows a large `NOT_PERMITTED` count | The requests the breaker rejected without touching the network. |
| p99 on `POST /api/orders` **drops sharply** while the error rate is high | Fast failure. This is the shape to recognise on a dashboard. |

### Step 6 — add the bulkhead, and measure the blast radius

```yaml
  bulkhead:
    instances:
      payments:
        maxConcurrentCalls: 16
        maxWaitDuration: 0
```

Re-run the burst, and this time watch `GET /api/products` specifically.

| What you see | What it means |
|---|---|
| `tomcat.threads.busy` stays well below max during the storm | The bulkhead capped payment concurrency at 16. 184 threads remained available. |
| p99 on `GET /api/products` unchanged from the Topic 65 baseline | **Containment achieved.** A total payment-provider outage no longer affects 70% of your traffic. This is the deliverable of the drill. |
| `BulkheadFullException` count high | Requests rejected at the bulkhead. That is correct behaviour, and the count is your evidence of how much you shed. |

### Step 7 — the slow variant, which is more realistic

Repeat steps 2–6 with `/control/slow?ms=8000` and the gateway **healthy**. A slow
dependency is the more common and more dangerous case.

| What you see | What it means |
|---|---|
| With only a failure-rate threshold configured, the breaker stays `CLOSED` | The calls succeed, slowly. The failure rate is 0%. `failureRateThreshold` alone does not protect you from a slow dependency. |
| After adding `slowCallDurationThreshold` + `slowCallRateThreshold`, it opens | This is why both thresholds exist. Most teams configure only the first. |
| With no HTTP read timeout, threads are held for 8 seconds each | The breaker cannot classify a call that has not finished. **The client timeout is the foundation everything else rests on.** |

### Step 8 — write it up

Six numbers, from your runs, in a table: total arrivals at the stub, peak
arrivals/second, `orderflow` p99 on `POST /api/orders`, `orderflow` p99 on
`GET /api/products`, peak `tomcat.threads.busy`, and error rate — for each of:
naive, +jitter, +breaker, +bulkhead. That table is the artefact, and it is the exact
shape of evidence a production-readiness review (Topic 124) asks for.

---

## Measurement

### The standing rule first

> **A naive `System.nanoTime()` measurement of a decorated call is WRONG.** JIT
> warmup, on-stack replacement, dead-code elimination and the absence of
> steady-state make the first thousand iterations meaningless, and a single
> timing loop measures your loop as much as your code. Use JMH. This is **Topic
> 77**, and it is the measurement authority for this entire curriculum.

The specific hazard here: measuring "the overhead of a circuit breaker" with a
nanoTime loop around `breaker.executeSupplier(() -> {})`. JIT will inline the empty
supplier, scalar-replace the lambda (Topic 75), and possibly eliminate the whole
thing. You will measure single-digit nanoseconds and conclude the breaker is free.
It is nearly free, but that is not what you measured.

For the *system-level* questions in this topic — the ones that matter — the
instrument is not a microbenchmark at all. It is the Topic 65 load generator plus
the metrics below, compared against the recorded baseline.

### The Resilience4j metrics that matter

Expose them and check the names on your version rather than trusting mine:

```bash
curl -s localhost:8080/actuator/prometheus | grep -E '^resilience4j' | cut -d'{' -f1 | sort -u
```

*Illustration of the metric-name shape, not captured output:*

```
resilience4j_circuitbreaker_state
resilience4j_circuitbreaker_calls_seconds_count
resilience4j_circuitbreaker_failure_rate
resilience4j_circuitbreaker_slow_call_rate
resilience4j_circuitbreaker_buffered_calls
resilience4j_circuitbreaker_not_permitted_calls_total
resilience4j_retry_calls_total
resilience4j_bulkhead_available_concurrent_calls
resilience4j_bulkhead_max_allowed_concurrent_calls
resilience4j_ratelimiter_available_permissions
```

| Metric | What it answers | Alert on |
|---|---|---|
| `resilience4j_circuitbreaker_state` (gauge, tagged by state) | Which state, per pod | **Any pod not `closed` for more than one `waitDurationInOpenState`.** Break out by pod — the state is per-JVM. |
| `..._calls_seconds_count{kind="failed"}` / `{kind="successful"}` | The real success rate against the dependency | Rate of change, not absolute |
| `..._failure_rate`, `..._slow_call_rate` | How close to the threshold you are | A warning band below the threshold — this is your leading indicator |
| `..._not_permitted_calls_total` | How much load the breaker shed | Non-zero is an incident in progress |
| `resilience4j_retry_calls_total{kind=...}` | Four kinds: `successful_without_retry`, `successful_with_retry`, `failed_without_retry`, `failed_with_retry` | **`successful_with_retry` rising is your earliest warning of dependency degradation** — earlier than error rate, because the retries are hiding it from the caller |
| `resilience4j_bulkhead_available_concurrent_calls` | Headroom | Sustained zero means you are shedding load |

The `successful_with_retry` row is the one to internalise. When a dependency starts
degrading, your users see nothing at first, because retries are covering it. The
retry metric moves *before* the error rate does. It is the single most useful
resilience metric you will have.

### Deriving the load multiplier from your own metrics

```
attempts = retry_calls_total{successful_without_retry}
         + 2..N x (successful_with_retry + failed_with_retry)     [upper bound]
```

Better: instrument the outbound call directly with a Micrometer `Timer` around the
innermost supplier, and compare its count to the count of `POST /api/orders`. That
ratio **is** the multiplier, measured, with no arithmetic. Put it on the dashboard
next to the provider's rate limit.

### Comparing against the Topic 65 baseline

Run the recorded 70/20/10 mix at the recorded arrival rate for each of these
configurations, and fill in your own numbers:

| Configuration | p50 / p95 / p99 `POST /api/orders` | p99 `GET /api/products` | Error rate | Peak `tomcat.threads.busy` | Attempts at the provider |
|---|---|---|---|---|---|
| Baseline, gateway healthy, no guards | | | | | |
| Guards installed, gateway healthy | | | | | |
| Guards installed, gateway **down** | | | | | |
| No guards, gateway **down** | | | | | |
| Guards installed, gateway **slow (8s)** | | | | | |
| No guards, gateway **slow (8s)** | | | | | |

Two rows are the ones that justify the work:

- Row 2 minus row 1 is **the cost of the guards in the healthy case.** It should be
  small. If it is not, something is wrong — most likely a `ThreadPoolBulkhead`
  adding a thread hop where a semaphore would do.
- Row 3's `GET /api/products` p99 versus row 4's is **the blast-radius reduction**,
  which is the entire business case.

### Cardinality warning, forward-referencing Topic 118

Resilience4j metrics are tagged by **instance name**. That is bounded and fine. Do
**not** add a tag for the order id, the customer id, or the provider's error
message. One time series per label-set; an unbounded tag kills Prometheus, not your
app. Topic 118 has the drill.

---

## Practice exercises

### Easy — prove the ordering both ways

Write one test class with two tests, using the functional API:

1. `Retry(CircuitBreaker(failingCall))` — assert `attempts == 3` **and**
   `breaker.getMetrics().getNumberOfBufferedCalls() == 3`.
2. `CircuitBreaker(Retry(failingCall))` — assert `attempts == 3` **and**
   `getNumberOfBufferedCalls() == 1`.

Then add a third test: with the correct ordering and a breaker already OPEN, assert
that `attempts == 1` (the retry did not burn its budget on
`CallNotPermittedException`). Success criterion: you can state, from the assertions
alone, which ordering is in effect.

### Medium — the guarded gateway (combines Topics 40, 54, 55, 90, 97, 109)

Build `GuardedPaymentGateway` for `orderflow` and demonstrate all of:

1. The functional API with bulkhead innermost, breaker middle, retry outermost.
2. An integration test proving the breaker opens after `minimumNumberOfCalls` and
   that the `NOT_PERMITTED` count then rises.
3. A test proving that `@CircuitBreaker` on a self-invoked method records **zero**
   calls (Topic 40), and that the collaborator-bean version records the right
   number.
4. Move the gateway call **inside** a `@Transactional` method, run 20 concurrent
   placements against a gateway stubbed to take 5 seconds, and capture the Hikari
   pool exhaustion (Topic 55/109). Then move it out and show the pool is unaffected.
   Capture a thread dump (`jcmd <pid> Thread.print`) in the broken case and identify
   the threads parked in `HikariPool.getConnection`.
5. Explain, in three sentences, why `maxConcurrentCalls: 16` and
   `maximum-pool-size: 16` being equal is a hazard in the broken version and
   irrelevant in the fixed one.

### Hard — production simulation on the spine

Run `orderflow` at the Topic 65 baseline with the 70/20/10 mix and produce a written
result for each.

1. **Execute the full Failure drill** (steps 0–8) and deliver its six-number table.
2. **The slow-dependency variant.** Set the stub to 8s with no failures. Show that a
   `failureRateThreshold`-only breaker never opens, then add the slow-call
   thresholds and show it does. Report the time-to-open for each.
3. **Bulkhead sizing sweep.** With the gateway at 3-second latency, sweep
   `maxConcurrentCalls` over {4, 8, 16, 32, 64, 200} and record, for each: payment
   success rate, `BulkheadFullException` rate, and p99 on `GET /api/products`.
   Plot the last against the first. Identify the knee, and state the number you
   would ship and why.
4. **The reconciliation gap.** With the gateway configured to charge successfully
   but respond after your read timeout, place 50 orders. Count the charges at the
   stub versus the `payment` rows in Postgres. Quantify the money at risk per hour
   at your baseline rate. Then implement the idempotency key from Trap 3 and repeat.
5. **Break the ordering under load.** Invert the aspect order via the properties,
   re-run the drill, and measure how much later the breaker opens and how many extra
   requests reached the provider. Express the difference as "the retry-inside-breaker
   bug costs N extra requests to the dependency per incident minute" — in your
   numbers.
6. **Write the runbook entry.** Eight lines: how to see breaker state per pod, what
   `NOT_PERMITTED` rising means, how to force a breaker open
   (`FORCED_OPEN` via the registry) to shed load deliberately during a provider
   incident, and what the reconciliation job is called.

---

## Interview questions

### Q1 — "We added retries for reliability. What do you think?"

**MID-LEVEL:** "Retries are good for transient failures. I'd use exponential
backoff."

**SENIOR:** "It depends entirely on what else you added with them, because a retry
on its own is a load multiplier with no off switch. `maxAttempts: 4` means every
client request becomes four requests to the dependency at exactly the moment it is
least able to serve them — so a partial outage becomes a total one, and if the
dependency has a rate limit you will cross it and start getting 429s caused by your
own retries.

The minimum set is backoff **with jitter**, a circuit breaker outside the retry, and
a concurrency bulkhead. Jitter matters because without it every client that failed
at the same instant retries at the same instant, so the dependency gets synchronised
waves instead of smeared load — and it re-synchronises them each wave. But I'd be
precise: **jitter changes the shape of the load, not the volume.** Only the breaker
reduces volume.

And I'd check the operation is idempotent before retrying it at all. A timeout on a
POST is not evidence the request didn't happen — it is most often evidence that it
did and was slow. Retrying a charge without an idempotency key double-charges
customers."

**What separates them:** the mid-level answer treats retries as unambiguously good.
The senior answer states the multiplier arithmetic, distinguishes what jitter does
from what the breaker does, and gets to idempotency without being asked.

**Follow-up:** *"Your dependency's p99 goes from 200ms to 5s but it never returns an
error. Does your breaker open?"* — Not with only `failureRateThreshold`; the calls
succeed. You need `slowCallDurationThreshold` plus `slowCallRateThreshold`. And you
need an HTTP read timeout, or the calls never complete for the breaker to classify.

### Q2 — "Where does the retry go, inside or outside the circuit breaker?"

**MID-LEVEL:** "Outside, I think — that's the recommended order."

**SENIOR:** "Outside, and I can say what breaks if you get it wrong. Inside the
breaker, the breaker sees one call per client request while the retry loop makes
three network attempts under a single acquired permission. Two things go wrong.
First, the breaker's window is measuring the wrong quantity, so it needs three times
the client traffic to reach `minimumNumberOfCalls` and opens far too late. Second —
and this is the bad one — if the retry *succeeds* on attempt three, the breaker
records a **success**. A dependency failing two thirds of the time looks completely
healthy to the breaker.

With the retry outside, every attempt goes through `acquirePermission`, so every
attempt is recorded, and once the breaker opens the remaining attempts fail
immediately without touching the network. I'd also add
`ignoreExceptions(CallNotPermittedException.class)` to the retry config, or it burns
its whole budget and both backoff sleeps retrying a rejection.

Resilience4j's default aspect order is already retry-outermost, but I don't rely on
that — I assert `breaker.getMetrics().getNumberOfBufferedCalls() == maxAttempts` in
a test, because the order is a property file away from being inverted."

**What separates them:** the mid-level answer recites the recommendation. The senior
answer derives it from the decorator nesting, identifies the recorded-success
failure mode, and knows the `CallNotPermittedException` detail — which is the part
that only shows up when you have actually run it.

**Follow-up:** *"Where does the bulkhead go?"* — Innermost, so it bounds actual
in-flight calls to the dependency. Outside the breaker it would also count rejected
calls against its permits, which is both wrong and pointless.

### Q3 — "How do you know your circuit breaker is actually working?"

**MID-LEVEL:** "We configured it and the config looks right, and there's a health
indicator."

**SENIOR:** "Configuration is not evidence. The specific thing I check first is
`/actuator/circuitbreakerevents` — **a breaker with zero recorded events after real
traffic is not protecting anything.** That is the signature of the annotation being
on a self-invoked method, which is Spring's proxy semantics: the advice lives on the
proxy, and `this.charge(...)` never touches it. No exception, no log line, no
protection. I confirm by printing the bean's runtime class and looking for
`$$SpringCGLIB$$`.

Beyond that I want three things. An integration test that stubs the dependency down
and asserts the state transition to OPEN and a rising `NOT_PERMITTED` count. A
`resilience4j_circuitbreaker_state` gauge on the dashboard broken out **by pod**,
because the breaker is per-JVM — eight pods is eight independent breakers, and if
you aggregate them the metric looks broken. And the failure drill run at least once
against the real load profile, so I have the blast-radius number rather than a
belief."

**What separates them:** the mid-level answer trusts configuration. The senior
answer names a specific falsifiable observation (zero events), knows why it happens
(proxies), and knows the breaker's scope is per-JVM — which changes both the sizing
and the dashboard.

**Follow-up:** *"Your breaker opens during a card-issuer problem and you go from 40%
success to 0%. What happened?"* — The breaker is counting `PaymentDeclinedException`
as a failure. Declines are a healthy dependency answering correctly. Move business
outcomes out of the exception channel — a sealed `PaymentResult` — or list them in
`ignoreExceptions`.

### Q4 — "What's a bulkhead and why not just use a rate limiter?"

**MID-LEVEL:** "A bulkhead isolates failures. A rate limiter limits requests per
second. They're similar."

**SENIOR:** "They bound different quantities and only one of them saves you when a
dependency gets slow. A rate limiter bounds **starts per second**. A bulkhead bounds
**concurrent in flight**. If the dependency's latency goes from 200ms to 8 seconds,
a limiter of 100/second happily lets 800 calls be in flight simultaneously — each
one holding a request thread — while staying perfectly within its limit. A bulkhead
of 16 caps in-flight at 16 no matter what the latency does. Little's Law is the
whole distinction: the limiter constrains the arrival rate, the bulkhead constrains
the arrival rate times the latency.

I size the bulkhead against the thread pool it is protecting, not against the
dependency's capacity: 16 concurrent payment calls out of 200 Tomcat threads means a
total payment outage can consume at most 8% of the server, and the 70% of traffic
that is catalogue reads is untouched. That containment number is the point.

I use a rate limiter as well when the provider has a **contractual** limit I must
not cross, with the limit set below theirs so I shed load myself rather than
collecting 429s.

And I'd use the semaphore bulkhead, not the thread-pool one, unless I specifically
need the thread boundary — the thread-pool variant loses `ThreadLocal`s, so the
security context, the MDC correlation id and the trace context all disappear."

**What separates them:** the mid-level answer describes both correctly and cannot
choose between them. The senior answer gives the Little's Law argument for why only
one handles the slow case, sizes it against the resource being protected, and knows
the `ThreadLocal` cost of the thread-pool variant.

**Follow-up:** *"`maxWaitDuration` — zero or non-zero?"* — Zero for a synchronous
HTTP request path: reject immediately so the caller can fail fast or fall back. A
non-zero wait converts a bulkhead into a queue, and an unbounded-ish queue in front
of a struggling dependency is Topic 90's lesson repeated.

### Q5 — "Your payment provider is down for twenty minutes. Walk me through what your service does."

**MID-LEVEL:** "The circuit breaker opens and we return an error to the customer."

**SENIOR:** "Minute zero: calls start failing. The retry fires with jittered
backoff, so we make about three attempts per request spread over a second or so.
After `minimumNumberOfCalls` fail within the sliding window — which on a per-pod
basis I've sized to be a few seconds at our rate — each pod's breaker opens
independently.

From then on, payment calls fail in microseconds with
`CallNotPermittedException`. `POST /api/orders` p99 actually *improves* while its
error rate goes to 100% — that inverted shape is what I'd expect on the dashboard
and it's a signal I'd want on-call to recognise. The bulkhead means we never held
more than 16 request threads on payments, so `GET /api/products` — 70% of our
traffic — is completely unaffected. That containment is the thing I'd want to be
true, and the failure drill is how I know it is.

Every 20 seconds each pod half-opens and sends three probe calls. Not a flood.

The part I'd actually want to talk about is what we tell the customer. The honest
answer for a payment is a third state, not success or failure: the order goes to
`PENDING_PAYMENT` and a reconciliation job retries later, because a timeout doesn't
tell us whether the charge happened. That state is a saga with a compensation, and
the compensation — releasing the reserved inventory — can itself fail, which is
Topic 117's problem.

And I'd know our revocation-style question: how many orders are sitting in
`PENDING_PAYMENT` and what's the money at risk. That's a query I'd have written
before the incident."

**What separates them:** the mid-level answer describes the mechanism. The senior
answer narrates it with timing, names the counter-intuitive latency signature,
quantifies the blast radius, and moves to the business consequence — the unknown
payment state — which is where the real work is.

**Follow-up:** *"One pod's breaker is open and the others are closed. Bug?"* — No.
Breakers are per-JVM with no coordination. Pods see different traffic and different
sliding windows. If it persists on one pod only, look at that pod's networking, not
at the breaker.

---

## Mental model checkpoint

1. Write the nesting produced by
   `Decorators.ofSupplier(s).withRetry(r).withCircuitBreaker(cb)`. Which is
   outermost? Is that what you want?
2. `maxAttempts: 4` against a failing dependency. What is the load multiplier, and
   what does jitter change about it?
3. A breaker is `CLOSED` with `failureRate: -1` while everything is failing. Give
   two causes.
4. Your dependency's latency goes to 10 seconds with a 0% error rate. Which of your
   guards fires, and which configuration key is required for it to?
5. `/actuator/circuitbreakerevents/payments` returns an empty list after an hour of
   production traffic. What is wrong, and what is the one-line confirmation?
6. Why does an open circuit breaker make your p99 *better*?
7. You have 8 pods. `minimumNumberOfCalls: 100`. Total gateway rate is 16/second.
   How long, roughly, before a pod's breaker can evaluate its failure rate?

---

## Quick reference card

**The order**

```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( call ) ) ) ) )
```

`Decorators` builder: **last `withX` added is outermost.**
Annotations: order comes from `resilience4j.<module>.<module>AspectOrder`, higher =
outer. Assert it, do not assume it.

**The five modules**

| Module | Bounds | Key settings |
|---|---|---|
| CircuitBreaker | Calls to an unhealthy dependency | `slidingWindowType/Size`, `minimumNumberOfCalls`, `failureRateThreshold`, `slowCallDurationThreshold` + `slowCallRateThreshold`, `waitDurationInOpenState`, `recordExceptions`/`ignoreExceptions` |
| Retry | Attempts per call | `maxAttempts` (**total**, not extra), `enableExponentialBackoff`, `enableRandomizedWait` + `randomizedWaitFactor`, `ignoreExceptions` (include `CallNotPermittedException`) |
| Bulkhead | Concurrent in-flight | `maxConcurrentCalls`, `maxWaitDuration` (use 0) |
| RateLimiter | Starts per period | `limitForPeriod`, `limitRefreshPeriod`, `timeoutDuration` |
| TimeLimiter | Wall-clock per call | `timeoutDuration` — needs `CompletableFuture`; prefer the HTTP client's read timeout for sync calls |

**Diagnostics**

```bash
curl -s localhost:8080/actuator/circuitbreakers | jq .
curl -s localhost:8080/actuator/circuitbreakerevents/payments | jq '.circuitBreakerEvents[-20:]'
curl -s localhost:8080/actuator/retryevents/payments | jq '.retryEvents[-20:]'
curl -s localhost:8080/actuator/bulkheads | jq .
curl -s localhost:8080/actuator/prometheus | grep '^resilience4j'
```

**Gotchas checklist**

- [ ] `circuitbreakerevents` is **non-empty** after real traffic. (Zero = Trap 4.)
- [ ] Retry is outermost; a test asserts `getNumberOfBufferedCalls() == maxAttempts`.
- [ ] `ignoreExceptions` on the retry includes `CallNotPermittedException`.
- [ ] Jitter is on (`enableRandomizedWait: true`).
- [ ] Every retried operation is idempotent, with a **stable** key.
- [ ] `slowCallDurationThreshold` **and** `slowCallRateThreshold` are set.
- [ ] The HTTP read timeout is **greater** than `slowCallDurationThreshold`.
- [ ] Business outcomes are in `ignoreExceptions` — or, better, are return values.
- [ ] `maxConcurrentCalls` is much smaller than the Tomcat thread count, and the
      containment ratio is written down.
- [ ] No external call inside a `@Transactional` method (Topic 55).
- [ ] Breaker state is dashboarded **per pod**.
- [ ] No unbounded metric tags (Topic 118).

---

## When would I use this at work?

**1. The first time you integrate any third party.** Payment provider, address
lookup, tax calculation, fraud scoring, a partner's inventory feed. The question
"what does our service do when this is down or slow for twenty minutes" has to have
an answer on day one, and the answer is a bulkhead plus a breaker plus an explicit
decision about the unknown-outcome state.

**2. In a postmortem, when someone proposes adding retries.** You will be the person
who can say "that is a 4× multiplier on a dependency that failed because of load"
with the drill's numbers behind it — and who can also say what *would* help. That
distinction is what senior means here.

**3. Sizing, during a capacity review.** `maxConcurrentCalls` is a containment
guarantee expressed as a number: "a total outage of X can consume at most N of our
M request threads." That sentence belongs in the production-readiness review (Topic
124) for every external dependency you have, and most teams cannot produce it.

---

## Connected topics

**Backwards**

- **40 — proxying and self-invocation.** Every Resilience4j annotation is an AOP
  proxy. Trap 4 is Topic 40 with a distributed-systems consequence.
- **41 — aspect ordering.** The decorator order is aspect order; the same
  `@Order` machinery as `@Cacheable`-outside-`@Transactional`.
- **09 / 28 — exception design and sealed types.** Whether declines are exceptions
  or return values decides whether Trap 5 is possible at all.
- **52 — locking.** Which operations are retry-safe: absolute and conditional
  updates are, relative adjustments are not.
- **54–55 — transactions.** An external call inside a transaction pins a connection
  for the whole retry sequence. The single most damaging way to combine these
  topics.
- **65 — the baseline.** Every number in Measurement is a delta against it.
- **90 / 97 — pools and `Semaphore`.** The semaphore bulkhead *is* Topic 97's
  `Semaphore`; `maxWaitDuration > 0` recreates Topic 90's queueing problem.
- **109 — HikariCP.** Bulkhead size versus pool size; both are concurrency bounds
  and they interact.
- **110 — caching.** A cached fallback is often the right response to an open
  breaker on a *read* path.

**Forwards**

- **114 / 115 — Kafka delivery and the outbox.** A retried produce is safe with an
  idempotent producer; the outbox relay's retry is safe because the sink is
  idempotent.
- **116 — idempotency.** The server side of Trap 3. `orderflow` must honour an
  `Idempotency-Key` the way you expect your provider to.
- **117 — sagas.** Moving the gateway call out of the transaction creates
  `PENDING_PAYMENT`. That is a saga step and it needs a compensation.
- **118 — metrics.** RED plus the Resilience4j series; `successful_with_retry` is
  your leading indicator. Watch the cardinality.
- **119 / 120 — tracing and MDC.** A `ThreadPoolBulkhead` crosses a thread boundary
  and drops both.
- **121 — Actuator and probes.** `registerHealthIndicator: true` puts the breaker in
  `/actuator/health`. Think hard before letting an open breaker fail **readiness** —
  it can remove every pod from the load balancer during a downstream incident.
- **123 — graceful shutdown.** In-flight retries hold threads through the drain
  period; the grace period must exceed the worst-case retry sequence.
- **130 — SLOs.** The breaker's thresholds should be derived from the error budget,
  not chosen by taste.
- **133 — postmortems.** "We added retries" is the most common contributing factor
  you will write up.
