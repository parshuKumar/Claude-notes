# 41 — AOP: Aspects, Pointcuts, Advice, and Ordering

## Phase: 4 — Spring Core
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: an audit-logging aspect over every `orderflow` command method, and a timing aspect whose measurements become Topic 118's RED metrics. This is the last topic in Phase 4 — after it, `orderflow` can wire beans, place an order against a stubbed inventory, and observe itself.

---

## Mechanical statement

> **An aspect is not a language feature.**
>
> Each piece of advice you write (`@Before`, `@Around`, `@AfterReturning`,
> `@AfterThrowing`, `@After`) is compiled into an ordinary Java class implementing
> `org.aopalliance.intercept.MethodInterceptor`. Spring pairs it with a `Pointcut` —
> a predicate over methods — to form an `Advisor`.
>
> At startup, `AnnotationAwareAspectJAutoProxyCreator` (a `BeanPostProcessor`, Topic
> 37) asks every advisor "do you match any method on this bean?". If any does, the
> bean is replaced by a **proxy** (Topic 40).
>
> At call time, the proxy builds a **list** of the matching interceptors, sorted by
> `@Order`, and walks it. `ReflectiveMethodInvocation.proceed()` holds an index; each
> call to `proceed()` advances it and invokes the next interceptor. When the index
> runs off the end, the target method is invoked.
>
> **That is all AOP is: a sorted list of objects, walked recursively, with your method
> at the bottom.** `@Order` decides the sort. Lower value = earlier in the list =
> **outermost**.

Everything else — pointcut syntax, advice types, the AspectJ expression language — is
notation for building that list.

---

## The bridge from what you know

### The analogue that is nearly exact: `next.handle()` and `proceed()`

```ts
// NestJS
@Injectable()
export class TimingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const started = process.hrtime.bigint();
    return next.handle().pipe(
      tap({
        next:  () => this.record(ctx, started, 'ok'),
        error: () => this.record(ctx, started, 'error'),
      }),
    );
  }
}
```

```java
// Spring
@Aspect
@Component
@Order(10)
public class TimingAspect {

    @Around("@annotation(com.orderflow.audit.Command)")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long started = System.nanoTime();
        String outcome = "ok";
        try {
            return pjp.proceed();                 // <-- next.handle()
        } catch (Throwable t) {
            outcome = "error";
            throw t;
        } finally {
            record(pjp, started, outcome);
        }
    }
}
```

Line for line:

| NestJS | Spring | Verdict |
|---|---|---|
| `NestInterceptor.intercept` | `@Around` advice method | **HONEST** |
| `next.handle()` | `pjp.proceed()` | **HONEST** — both call the next thing in the chain |
| `ExecutionContext` | `ProceedingJoinPoint` | **PARTIAL** — `pjp` gives method, args, target; `ctx` gives the HTTP layer too |
| Interceptor order = array order in `@UseInterceptors` | `@Order`, lower = outer | **PARTIAL** — Spring's is global, Nest's is per-route |
| `.pipe(tap(...))` for the after-phase | `finally` block, or `@AfterReturning`/`@AfterThrowing` | **PARTIAL** — Nest's is reactive, Spring's is synchronous |
| `@UseInterceptors(X)` on a controller | a **pointcut expression** | **NO ANALOGUE** — see below |

### The two differences that matter

**1. Attachment: explicit list vs. predicate.**

Nest attaches interceptors by writing them down: `@UseInterceptors(TimingInterceptor)`
on a controller, or `APP_INTERCEPTOR` globally. You can read a controller and see
exactly what wraps it.

Spring attaches advice by **evaluating a predicate against every method of every
bean**. Nothing is written at the call site. A `@Transactional` you have never seen,
in a class you have never opened, may be wrapped by an aspect declared in a different
module. This is more powerful and considerably harder to reason about — and it is why
"which aspects apply to this method?" is a question you need a *command* to answer,
not a file to read. The Hands-on section gives you that command.

**2. Where it attaches — and you already know this one.**

Nest wires interceptors at the **route boundary**, inside the framework's request
pipeline. Spring wires advice onto the **bean object**, as a proxy. Therefore
`this.method()` inside your own class bypasses every aspect in the application. That
is Topic 40, and it applies to every aspect you write here without exception. If you
skipped Topic 40, stop and read it — this document assumes it.

### One more honest anchor: RxJS operator chains

`ReflectiveMethodInvocation.proceed()` walking a sorted interceptor list is
structurally the same thing as an RxJS `pipe()` composing operators around a source.
Order matters for the same reason: each operator sees the world as transformed by the
ones outside it. If you have ever debugged a `pipe(retry(3), timeout(1000))` versus
`pipe(timeout(1000), retry(3))`, you have already debugged aspect ordering.

---

## What is this?

Aspect-Oriented Programming lets you attach behaviour to methods you did not write, by
describing *which* methods rather than *listing* them.

Five vocabulary words. Define them once, use them precisely:

| Term | What it actually is |
|---|---|
| **Join point** | A point in program execution where advice can be attached. **In Spring AOP this is always, and only, a method execution.** AspectJ proper also supports field access, constructor calls and more. Spring does not. |
| **Pointcut** | A predicate that selects join points. Written in AspectJ's expression language. |
| **Advice** | The code that runs at a matched join point. Five kinds, below. |
| **Aspect** | A class holding pointcuts and advice. Marked `@Aspect`, and it must also be a bean (`@Component`). |
| **Advisor** | Spring's internal pairing of one pointcut with one advice. This is the object that actually goes in the sorted list. |

### The five advice types

```java
@Before("...")           void  a(JoinPoint jp)                    // runs before; cannot change args or block the call
@AfterReturning("...")   void  b(JoinPoint jp, Object result)     // runs only on normal return
@AfterThrowing("...")    void  c(JoinPoint jp, Throwable ex)      // runs only on a throw; CANNOT suppress it
@After("...")            void  d(JoinPoint jp)                    // runs either way — a finally block
@Around("...")           Object e(ProceedingJoinPoint pjp)        // wraps the call; the only one that can
                                                                  // change args, change the result, suppress
                                                                  // an exception, or skip the call entirely
```

`@Around` can do everything the other four can. The other four exist because they are
harder to get wrong: you cannot forget to return the result from a `@Before`.

> **Use the weakest advice that does the job.** `@Around` for anything that must wrap
> (timing, retry, transaction-like behaviour). `@Before`/`@AfterReturning` for anything
> that only observes. This is the same instinct as preferring `const` to `let`.

### The pointcut designators you will actually use

```java
// 1. Methods carrying a specific annotation — the one to prefer.
@annotation(com.orderflow.audit.Command)

// 2. Methods on a class carrying an annotation.
@within(org.springframework.stereotype.Service)

// 3. Method signature matching.
execution(public * com.orderflow.orders.OrderService.place(..))
execution(* com.orderflow..*Service.*(..))

// 4. Everything inside a package or type.
within(com.orderflow.payments..*)

// 5. Bean name matching (Spring-specific, not AspectJ).
bean(*Repository)

// 6. Argument matching, and binding arguments into your advice method.
args(com.orderflow.orders.PlaceOrderCommand)

// 7. Proxy type vs target type — a genuine Topic 40 distinction.
this(com.orderflow.payments.PaymentGateway)     // matches the PROXY's type
target(com.orderflow.payments.PaymentGateway)   // matches the TARGET's type
```

`execution(* com.orderflow..*Service.*(..))` reads as: any return type, any class in
`com.orderflow` or below whose simple name ends in `Service`, any method, any
arguments. `..` in a package position means "and subpackages"; `..` in an argument
position means "any number of arguments of any type".

### Named pointcuts

Do not repeat expressions. Name them:

```java
@Aspect
@Component
public class OrderflowPointcuts {

    @Pointcut("@annotation(com.orderflow.audit.Command)")
    public void commandMethod() {}          // the body is always empty and always ignored

    @Pointcut("within(com.orderflow.payments..*)")
    public void inPayments() {}

    @Pointcut("commandMethod() && inPayments()")
    public void paymentCommand() {}
}
```

Then reference them by fully-qualified method name from any aspect:

```java
@Around("com.orderflow.aop.OrderflowPointcuts.paymentCommand()")
```

The empty method is pure syntax — a place to hang the annotation and give the
expression a compiler-checked name. Renaming it is a compile error at every reference,
which is exactly what you want and what a raw string does not give you.

---

## Why does it matter?

**1. Cross-cutting concerns are the largest source of copy-paste in a service.**
Audit, timing, correlation IDs, retry, authorization checks. Written by hand, they
appear at the top and bottom of every method and get forgotten in exactly the method
where they mattered. As an aspect, they are declared once and cannot be forgotten.

**2. Every proxy-based Spring feature you will meet is an aspect.**
`@Transactional` is `TransactionInterceptor`. `@Cacheable` is `CacheInterceptor`.
`@Validated`, `@Async`, `@PreAuthorize`, `@CircuitBreaker`, `@Retry` — all
`MethodInterceptor`s on the same chain. Once you can read the chain, all of them stop
being magic simultaneously, and you can reason about how they *combine*.

**3. Ordering bugs are silent and expensive.**
When two aspects wrap the same method, the order changes the semantics. Cache outside
transaction versus inside transaction is the difference between a correct cache and a
cache serving data that was rolled back. Nothing throws. The failure drill in this
document produces exactly that, on purpose.

**4. A careless pointcut degrades the whole application.**
`execution(* com.orderflow..*(..))` proxies almost every bean in the context. Startup
slows, every bean acquires Topic 40's constraints, and the self-invocation trap
suddenly applies to code that never asked for it. A pointcut is a global statement and
should be written like one.

---

## Machine-level reality

### From `@Aspect` to a list of objects

Startup, in order:

1. **`@EnableAspectJAutoProxy`** registers `AnnotationAwareAspectJAutoProxyCreator` as
   a `BeanPostProcessor`. Spring Boot does this for you via `AopAutoConfiguration`
   whenever `spring-boot-starter-aop` (i.e. `aspectjweaver`) is on the classpath.
2. The creator scans all beans for `@Aspect`. For each advice method it builds one
   **`Advisor`** = `AspectJExpressionPointcut` + an advice object.
3. **The advice objects are `MethodInterceptor`s.** `@Around` becomes
   `AspectJAroundAdvice`, which implements `MethodInterceptor` directly. `@Before`
   becomes `AspectJMethodBeforeAdvice`, wrapped by `MethodBeforeAdviceInterceptor` to
   fit the same shape. `@AfterReturning` and `@After` are adapted the same way. **Every
   advice type ends up as a `MethodInterceptor`.** There is no second mechanism.
4. In `postProcessAfterInitialization` — the same hook that creates the proxies in
   Topic 40 — the creator asks each advisor's pointcut whether it matches any method
   on the bean. If yes, the bean is wrapped.

Two things follow from step 4 that people get wrong:

- **Pointcut matching happens at startup, per bean, per method.** A pointcut like
  `execution(* com.orderflow..*(..))` is evaluated against every method of every bean
  in your context. On a large application that is measurable startup time, and you
  will see it in the context-load duration of your test suite.
- **A bean matched by *any* advisor gets a proxy for *all* its methods.** The
  interceptor chain per method may be empty, but the proxy exists, and so does every
  Topic 40 constraint.

### The chain walk, per invocation

```java
// conceptual, from ReflectiveMethodInvocation
public Object proceed() throws Throwable {
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();                    // <- YOUR method, via reflection
    }
    Object interceptorOrMatcher =
        this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    // (dynamic pointcuts like args() are re-evaluated here, per call)
    return ((MethodInterceptor) interceptorOrMatcher).invoke(this);
}
```

So a call through a proxy with three aspects looks like this on the stack:

```
proxy.place(cmd)
  DynamicAdvisedInterceptor.intercept
    ReflectiveMethodInvocation.proceed()          index 0
      TimingAspect around                @Order(10)
        pjp.proceed()
          ReflectiveMethodInvocation.proceed()    index 1
            AuditAspect around           @Order(20)
              pjp.proceed()
                ReflectiveMethodInvocation.proceed()  index 2
                  TransactionInterceptor.invoke      @Order(LOWEST_PRECEDENCE)
                    begin transaction
                      OrderService.place(cmd)     <- your code, at last
                    commit or rollback
```

Read that stack once and the ordering rule stops needing memorisation: **lower
`@Order` value = earlier in the list = further out = starts first and finishes last.**
It is nesting, not sequencing.

### Two costs per call, and where they come from

1. **Chain-walk cost.** One `ReflectiveMethodInvocation` object, plus one stack frame
   per advice, plus the `Object[]` argument array from Topic 40, plus boxing of any
   primitive arguments (Topic 01). Tens of nanoseconds total. Irrelevant.
2. **Advice-body cost.** Whatever *you* wrote. An aspect that calls
   `Arrays.toString(pjp.getArgs())` or serialises arguments to JSON can cost
   microseconds to milliseconds. **This is the cost that actually shows up in a flame
   graph**, and it is entirely under your control.

### Static vs dynamic pointcut matching

Most designators (`execution`, `within`, `@annotation`, `@within`, `bean`) are
**static**: matched once at proxy-creation time. Their per-call cost is zero.

`args(...)`, `@args(...)` and `if(...)` are **dynamic**: the actual argument values
must be examined, so matching runs **on every invocation**. In the chain-walk code
above, that is the "dynamic method matchers" branch.

> Prefer static designators. If you need to inspect arguments, match statically with
> `@annotation` and do the inspection inside your advice body — where you can see and
> profile the cost.

### Ordering *within* one aspect

Two pieces of advice in the *same* aspect class at the same join point are the case
where I will not assert a rule from memory. Spring's documented guidance has changed:
it was "undefined, refactor into separate aspects", and later versions introduced
deterministic ordering by advice type (around, before, after, after-returning,
after-throwing). **I am not certain which precise rule applies on Framework 7.0.**

The command that settles it is three lines of probe code, in the Hands-on section
below. The design rule that makes the question irrelevant:

> **One concern per aspect class, with an explicit `@Order`.** Then ordering is
> something you declared rather than something you inherited.

`@Order` on the aspect class is the supported, portable mechanism. An aspect may also
implement `Ordered` and return the value from `getOrder()`, which is what you need if
the order must be configurable.

---

## Example 1 — minimal

```java
package com.orderflow.lab;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Aspect                 // marks it as an aspect
@Component              // and it must ALSO be a bean, or nothing reads it
@Order(10)
public class TraceAspect {

    @Around("execution(* com.orderflow.lab.GreetingService.*(..))")
    public Object trace(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("-> " + pjp.getSignature().toShortString());
        try {
            Object result = pjp.proceed();
            System.out.println("<- " + pjp.getSignature().toShortString() + " = " + result);
            return result;                       // MUST return it. See Trap 2.
        } catch (Throwable t) {
            System.out.println("!! " + pjp.getSignature().toShortString()
                    + " threw " + t.getClass().getSimpleName());
            throw t;                             // MUST rethrow, or you swallow it.
        }
    }
}
```

```java
@Service
public class GreetingService {
    public String greet(String customer) { return "hello " + customer; }
}
```

Both `@Aspect` **and** `@Component` are required. `@Aspect` alone does nothing —
Spring only inspects beans, and without `@Component` the class is never registered. A
silently-inert aspect is almost always this.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow`, sized from the Phase 7 gate:

- **12 command methods** across orders, inventory, wallet and payments. A *command* is
  a method that changes state; queries are excluded.
- **400 commands/second** at flash-sale peak.
- Compliance requires an audit record of **who attempted what, when, and whether it
  succeeded**, retained for seven years.
- The timing data must feed Micrometer so Topic 118 can build RED dashboards, with a
  hard rule: **no unbounded tag values.** A tag per order ID creates one Prometheus
  time series per order — 400 new series per second — and kills the monitoring system
  before it kills the application.
- Log budget: at 400 rps and ~250 bytes per audit line, that is 100 KB/s, roughly
  **8.6 GB per day**. That number decides the log format and what may be included.

### Step 1 — declare what a "command" is, in the type system

```java
package com.orderflow.audit;

import java.lang.annotation.*;

/**
 * Marks a state-changing operation. Audited and timed.
 * The presence of this annotation is a deliberate declaration, not a naming convention.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface Command {

    /** Stable, low-cardinality name used as a metric tag and audit action. */
    String value();
}
```

This is the single most important design decision in this document.

The tempting alternative is `execution(* com.orderflow..*Service.*(..))` — "audit
everything in a service". Do not. Three reasons:

1. It matches **queries** too, so you audit and time 100k catalogue reads per minute
   alongside 400 real commands.
2. It proxies nearly every bean in the context, imposing Topic 40's constraints
   everywhere.
3. It couples your audit policy to a **naming convention**. Rename a class from
   `OrderService` to `OrderCoordinator` and you silently stop auditing orders. There is
   no compile error and no test failure.

`@Command("order.place")` is explicit, greppable, compiler-anchored, and carries the
low-cardinality metric name with it.

### Step 2 — the shared pointcut library

```java
package com.orderflow.aop;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class OrderflowPointcuts {

    /** Any method annotated @Command. Static match: zero per-call cost. */
    @Pointcut("@annotation(com.orderflow.audit.Command)")
    public void commandMethod() {}

    /** Excludes anything already inside the aop package, so aspects never advise aspects. */
    @Pointcut("!within(com.orderflow.aop..*)")
    public void notAnAspect() {}

    @Pointcut("commandMethod() && notAnAspect()")
    public void auditableCommand() {}
}
```

The `notAnAspect()` guard looks like paranoia until an aspect calls a collaborator
that is itself advised by the same aspect, and you get a `StackOverflowError` at
startup. Write it once, forget about it.

### Step 3 — the timing aspect, outermost

```java
package com.orderflow.aop;

import com.orderflow.audit.Command;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

/**
 * Order 10 -> outermost of our aspects.
 * Deliberate: the measurement must INCLUDE transaction begin and commit, because
 * commit is real latency the customer waits for. If this sat inside the transaction
 * interceptor it would under-report p99 by the commit cost, which is exactly the
 * cost that spikes under load.
 */
@Aspect
@Component
@Order(10)
public class CommandTimingAspect {

    private final MeterRegistry meters;

    public CommandTimingAspect(MeterRegistry meters) { this.meters = meters; }

    @Around("com.orderflow.aop.OrderflowPointcuts.auditableCommand()")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {

        Command command = ((MethodSignature) pjp.getSignature())
                .getMethod().getAnnotation(Command.class);

        Timer.Sample sample = Timer.start(meters);
        String outcome = "success";
        try {
            return pjp.proceed();
        } catch (Throwable t) {
            outcome = "failure";
            throw t;                                        // never swallow
        } finally {
            sample.stop(Timer.builder("orderflow.command")
                    .tag("command", command.value())        // BOUNDED: 12 values
                    .tag("outcome", outcome)                // BOUNDED: 2 values
                    // NEVER: .tag("orderId", ...) or .tag("customerId", ...)
                    .publishPercentileHistogram()           // server-side percentiles — Topic 118
                    .register(meters));
        }
    }
}
```

Total time series: 12 commands x 2 outcomes = **24**, plus histogram buckets. That is
a number you can defend in a capacity review. `.tag("orderId", ...)` would produce
34 million series per day. Topic 118 has the drill that proves it.

### Step 4 — the audit aspect

```java
package com.orderflow.aop;

import com.orderflow.audit.Command;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Aspect
@Component
@Order(20)                       // inside timing, outside the transaction interceptor
public class CommandAuditAspect {

    private static final Logger log = LoggerFactory.getLogger("orderflow.audit");

    private final CurrentActor actor;          // request-scoped, from Topic 38

    public CommandAuditAspect(CurrentActor actor) { this.actor = actor; }

    @Around("com.orderflow.aop.OrderflowPointcuts.auditableCommand()")
    public Object audit(ProceedingJoinPoint pjp) throws Throwable {

        Command command = ((MethodSignature) pjp.getSignature())
                .getMethod().getAnnotation(Command.class);

        // NOTE: we log the ACTION and the ACTOR. We do NOT log pjp.getArgs().
        // See Trap 5 — arguments are PII, are unbounded in size, and calling
        // toString() on a JPA entity can trigger lazy loading (Topic 49).
        try {
            Object result = pjp.proceed();
            log.info("action={} actor={} outcome=success", command.value(), actor.id());
            return result;
        } catch (Throwable t) {
            log.warn("action={} actor={} outcome=failure error={}",
                    command.value(), actor.id(), t.getClass().getSimpleName());
            throw t;
        }
    }
}
```

At 400 rps and ~120 bytes per line this is roughly 4 GB/day — inside budget, and
structured for the JSON logging you will add in Topic 120.

### Step 5 — make the ordering explicit rather than inherited

```java
package com.orderflow.config;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.Ordered;
import org.springframework.transaction.annotation.EnableTransactionManagement;

@Configuration
@EnableTransactionManagement(order = 100)     // outside the cache interceptor
@EnableCaching(order = 200)                   // inside the transaction interceptor
public class AspectOrderConfig { }
```

Do not skip this because "the defaults are probably fine". They are not defined.
Both `@EnableTransactionManagement` and `@EnableCaching` default their advisor order
to `Ordered.LOWEST_PRECEDENCE` — the same value — so the relative order between the
transaction and cache interceptors is a **tie**, resolved by advisor registration
order, which is not part of any contract. The failure drill below is what a tie
resolved the wrong way costs you.

### Step 6 — apply it

```java
@Service
public class OrderService {

    @Command("order.place")
    @Transactional
    public OrderId place(PlaceOrderCommand cmd) { ... }

    @Command("order.cancel")
    @Transactional
    public void cancel(OrderId id, CancellationReason reason) { ... }

    // No @Command: this is a query. Not audited, not timed here.
    public OrderView findById(OrderId id) { ... }
}
```

The resulting chain for `place`:

```
timing (10) -> audit (20) -> transaction (100) -> cache (200) -> OrderService.place
```

### The honest limitation of this design

The audit aspect sits **outside** the transaction interceptor. So it logs
`outcome=success` when `place` returns normally — but the transaction commits *after*
that, inside the transaction interceptor, which is further in. If the commit fails
(a deferred constraint, a connection loss, a rollback-only flag set deeper in the
call), the audit log says success for an operation that did not happen.

Moving the audit aspect inside the transaction does not fix it either — it just moves
the same gap.

The correct fix, when the audit record must be exactly right, is
`TransactionSynchronizationManager.registerSynchronization(...)` with an
`afterCompletion(int status)` callback, which fires with the real commit-or-rollback
outcome. That is Topic 54's material. State the limitation in a comment now; do not
pretend the aspect is stronger than it is. **An audit log that is right 99.9% of the
time and silently wrong the rest is worse than one that documents its own gap.**

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the pointcut that matches everything

**Wrong:**

```java
@Around("execution(* com.orderflow..*.*(..))")
public Object audit(ProceedingJoinPoint pjp) throws Throwable { ... }
```

**Exact symptoms, in the order you meet them:**

- Application startup time roughly doubles, and the test suite's context-load time
  with it. Measure with `--debug` or `-Dspring.context.startup.enabled` and
  `/actuator/startup`.
- A `ClassCastException` in code that previously worked, because a bean that was never
  proxied now is (Topic 40, Trap 4).
- A `@Value`-injected field reads `null` somewhere, because something touched a field
  on the proxy rather than the target.
- In the worst case: `StackOverflowError` at startup, because the aspect calls a
  collaborator that is itself matched by the same pointcut, recursing.

**Root cause:** the pointcut is evaluated against every method of every bean, and a
match on any method proxies the whole bean. You have applied a global change with a
local-looking annotation.

**Fix:** match on an annotation you control (`@annotation(com.orderflow.audit.Command)`)
and add `&& !within(com.orderflow.aop..*)`. If you genuinely need package-wide
matching, scope it tightly: `within(com.orderflow.orders.command..*)`.

**How to check what you actually matched:**

```properties
logging.level.org.springframework.aop=DEBUG
```

and look for the beans reported as proxied. If the list is longer than you expected,
your pointcut is wrong.

---

### Trap 2 — `@Around` that does not return `proceed()`'s result

**Wrong:**

```java
@Around("com.orderflow.aop.OrderflowPointcuts.auditableCommand()")
public Object audit(ProceedingJoinPoint pjp) throws Throwable {
    log.info("start");
    pjp.proceed();                 // result discarded
    log.info("end");
    return null;                   // or falls off the end of a void-ish path
}
```

**Exact symptom:** every advised method returns `null`. Concretely: `POST /orders`
returns HTTP 200 with a body of `null`, or the caller NPEs on
`orderService.place(cmd).value()`. The method's own logging shows it computed the right
answer. The aspect logs "start" and "end" happily.

The variant with a primitive return type is louder and easier:

```
java.lang.NullPointerException: Cannot invoke "java.lang.Long.longValue()"
  because the return value of "...place(...)" is null
```

— Topic 01's unboxing NPE, arriving from an unexpected direction.

**Root cause:** `@Around` advice **is** the method as far as the caller is concerned.
Whatever it returns is what the caller gets. `proceed()` returns the target's value and
you threw it away.

**Fix:** `return pjp.proceed();`, or capture it in a variable and return that. And
prefer `@Before` + `@AfterReturning` when you only observe — those cannot express this
bug.

**Test that catches it forever:**

```java
@Test
void aspect_preserves_the_return_value() {
    assertThat(orderService.place(validCommand())).isNotNull();
}
```

---

### Trap 3 — the `@Cacheable` / `@Transactional` ordering tie

**Wrong:** relying on the default.

```java
@Service
public class ProductCatalogService {

    @Cacheable("product-prices")
    @Transactional
    public long priceOf(String sku) { return repo.priceOf(sku); }
}
```

**Exact symptom:** a price served from the cache that does not exist in the database.
`select price from product where sku = 'SKU-4471'` returns 1999. `GET /products/SKU-4471`
returns 2499, consistently, until the TTL expires. Restarting the pod "fixes" it. No
exception anywhere, in any log, at any level.

**Root cause:** two independent facts compounding.

1. Both `@EnableTransactionManagement` and `@EnableCaching` default their advisor
   order to `Ordered.LOWEST_PRECEDENCE`. They tie. The resolution depends on advisor
   registration order, which is not a contract.
2. **Spring's default `CacheManager` is not transaction-aware.** A `@Cacheable` entry
   is written when the advised method *returns*, not when the transaction *commits*.
   If the cache interceptor sits inside the transaction interceptor, the entry is
   written before the commit decision — and any later rollback leaves it behind.

**Fix, in two parts, and you need both:**

```java
@EnableTransactionManagement(order = 100)   // transaction OUTSIDE
@EnableCaching(order = 200)                 // cache INSIDE
```

Ordering ensures that anything the transaction machinery throws — including
`UnexpectedRollbackException` at commit time — is not swallowed by a cache write that
already happened.

```java
@Bean
public CacheManager cacheManager(CacheManager delegate) {
    return new TransactionAwareCacheManagerProxy(delegate);   // defers puts to afterCommit
}
```

This is the part that actually closes the hole: a transaction-aware cache manager
registers a `TransactionSynchronization` and performs the put in `afterCommit`. Redis
users get the same via `RedisCacheManager.builder(...).transactionAware()`.

Topic 110 goes further — including why this still does not save you from a cache
populated by a *different* transaction that later rolls back.

---

### Trap 4 — an aspect on a self-called method

**Wrong:**

```java
@Service
public class InventoryService {

    public void reserveAll(List<Reservation> rs) {
        rs.forEach(this::reserveOne);       // self-call
    }

    @Command("inventory.reserve")
    public void reserveOne(Reservation r) { repo.decrement(r.sku(), r.units()); }
}
```

**Exact symptom:** `orderflow_command_seconds_count{command="inventory.reserve"}`
reads **0** on the dashboard while inventory is demonstrably being reserved. The audit
log has no `action=inventory.reserve` lines at all. Everything works; nothing is
observed.

The reason this is worse than a broken feature: an observability gap looks identical
to "that code path is never used". Someone will eventually delete a metric or an alert
because "it never fires".

**Root cause:** Topic 40. `this::reserveOne` binds to the target instance. Advice lives
on the proxy.

**Fix:** annotate the method that is actually called from outside — `reserveAll` — or
extract `reserveOne` into a collaborator bean. Prefer annotating the entry point: one
audit record per business operation is usually more useful than one per item anyway.

---

### Trap 5 — an aspect that logs the arguments

**Wrong:**

```java
log.info("action={} args={}", command.value(), Arrays.toString(pjp.getArgs()));
```

**Exact symptoms, and there are four distinct ones:**

1. **A PII incident.** `PlaceOrderCommand` contains a customer ID, a delivery address
   and, in some codebases, a payment token. Those are now in your log aggregation
   system, replicated, retained for seven years and searchable by everyone with log
   access. This is a reportable data-protection breach, not a bug.
2. **Log volume explosion.** A command carrying a 200-line order becomes a 40 KB log
   entry. At 400 rps that is 16 MB/s — 1.4 TB/day — and your log pipeline drops
   messages, including the ones you needed.
3. **A lazy-loading side effect.** If an argument is a JPA entity, `toString()` may
   touch a lazy association and trigger a SELECT — one extra query per command, or a
   `LazyInitializationException` if the session has closed. **Your observability code
   has changed the behaviour of the system it observes.** Topics 49 and 50.
4. **Latency.** Serialisation is real work in the request path. This is the one aspect
   cost that shows up in a flame graph.

**Root cause:** advice runs in the request path with full access to arguments, and
`getArgs()` makes it trivially easy to log everything.

**Fix:** log the *action name* and the *actor*, both bounded and non-sensitive. If you
need an identifier, pass it explicitly and deliberately:

```java
@Command(value = "order.place", auditKey = "#cmd.idempotencyKey()")
```

and evaluate that one expression, rather than reaching for the whole argument array.
Never call `toString()` on an entity from an aspect.

---

## Hands-on proof

Every command below is one **you** run. No output is reproduced here as if captured.

### Setup

```bash
mkdir -p ~/java-lab/41 && cd ~/java-lab/41
curl https://start.spring.io/starter.zip \
  -d dependencies=web,aop,jdbc,h2,cache,actuator \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=aop-lab \
  -d type=maven-project -o aop-lab.zip && unzip aop-lab.zip -d aop-lab
cd aop-lab
```

`spring-boot-starter-aop` is what pulls in `aspectjweaver` and triggers
`AopAutoConfiguration`. Without it, `@Aspect` classes are silently inert — which is the
first thing to check when an aspect "does nothing".

### Proof 1 — is my aspect even wired?

```java
@Component
class AspectInventory implements CommandLineRunner {
    private final ApplicationContext ctx;
    AspectInventory(ApplicationContext ctx) { this.ctx = ctx; }

    public void run(String... args) {
        ctx.getBeansWithAnnotation(org.aspectj.lang.annotation.Aspect.class)
           .forEach((name, bean) -> System.out.println("ASPECT " + name
                   + " -> " + AopProxyUtils.ultimateTargetClass(bean).getName()));
    }
}
```

```bash
./mvnw spring-boot:run | grep "^ASPECT"
```

| What you see | What it means |
|---|---|
| Your aspect listed | It is a bean and Spring can see it. Move on to Proof 2. |
| Nothing listed | The class is missing `@Component` (or is outside the scanned package — Topic 36). This is the most common cause of "my aspect does nothing". |
| Aspects listed, but no advice runs | The pointcut does not match. Go to Proof 2. |

### Proof 2 — which beans got proxied, and by what?

```properties
logging.level.org.springframework.aop=DEBUG
logging.level.org.springframework.aop.aspectj=TRACE
```

```bash
./mvnw spring-boot:run 2>&1 | grep -iE "creating implicit proxy|advisor|candidate"
```

| What you see | What it means |
|---|---|
| Proxy-creation lines naming exactly the beans you expected | Your pointcut is correctly scoped. |
| Dozens of beans you did not expect | Trap 1. Your pointcut is too broad. Narrow it before doing anything else. |
| Your target bean absent from the list | The pointcut does not match it. Check package spelling, the annotation's fully-qualified name, and that the method is `public` and non-`final` (Topic 40). |

Then confirm the runtime object directly, exactly as in Topic 40:

```java
System.out.println(orderService.getClass().getName());   // expect $$SpringCGLIB$$
System.out.println(AopUtils.isAopProxy(orderService));
```

### Proof 3 — print the actual advisor chain for a bean

This is the command that answers "what wraps this method?", which has no equivalent
file to read.

```java
import org.springframework.aop.framework.Advised;
import org.springframework.core.annotation.AnnotationAwareOrderComparator;

@Component
class ChainPrinter implements CommandLineRunner {
    private final OrderService orderService;
    ChainPrinter(OrderService orderService) { this.orderService = orderService; }

    public void run(String... args) {
        if (orderService instanceof Advised advised) {
            System.out.println("TARGET: " + advised.getTargetClass().getName());
            int i = 0;
            for (var advisor : advised.getAdvisors()) {
                System.out.printf("  [%d] %s  advice=%s  order=%s%n",
                        i++,
                        advisor.getClass().getSimpleName(),
                        advisor.getAdvice().getClass().getName(),
                        AnnotationAwareOrderComparator.INSTANCE
                                .getClass().getSimpleName());   // see note below
            }
        } else {
            System.out.println("NOT PROXIED — nothing advises this bean.");
        }
    }
}
```

```bash
./mvnw spring-boot:run | sed -n '/^TARGET:/,/^$/p'
```

| What you see | What it means |
|---|---|
| A numbered list of advisors | **The list is printed in execution order.** Index 0 is outermost. This is the interceptor chain from the Machine-level section, made visible. |
| `TransactionInterceptor` appearing after `CacheInterceptor` | Cache is outside transaction. Decide whether that is what you want — see the Failure drill. |
| `NOT PROXIED` | No advisor matched. Nothing you annotated is doing anything. |
| Your aspect's advice class named `AspectJAroundAdvice` | Confirms the mechanical statement: your `@Around` really is a `MethodInterceptor` in a list. |

> To read each advisor's numeric order, cast to `org.springframework.core.Ordered`
> where the advisor implements it, or call `getOrder()` on
> `DefaultPointcutAdvisor`-style advisors. Not every advisor exposes one; where it does
> not, position in the list is the answer that matters.

### Proof 4 — settle the within-one-aspect ordering question yourself

I declined to assert this from memory. Here is the three-minute experiment.

```java
@Aspect @Component @Order(10)
class OrderProbeAspect {
    @Around("execution(* com.orderflow.lab.GreetingService.greet(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("AROUND enter"); try { return pjp.proceed(); }
        finally { System.out.println("AROUND exit"); }
    }
    @Before("execution(* com.orderflow.lab.GreetingService.greet(..))")
    public void before() { System.out.println("BEFORE"); }
    @AfterReturning("execution(* com.orderflow.lab.GreetingService.greet(..))")
    public void afterReturning() { System.out.println("AFTER_RETURNING"); }
    @After("execution(* com.orderflow.lab.GreetingService.greet(..))")
    public void after() { System.out.println("AFTER"); }
}
```

```bash
./mvnw spring-boot:run | grep -E "AROUND|BEFORE|AFTER"
```

**How to read it:** write down the sequence you observe. That is the rule **on your
version**. Then re-run after splitting each advice into its own `@Aspect` class with
explicit `@Order` values and confirm the sequence is now the one you declared. The
second run is the design lesson: you should not be reading this from a log, you should
be reading it from your own `@Order` annotations.

### Proof 5 — measure what a broad pointcut costs at startup

```bash
# baseline: narrow @annotation pointcut
./mvnw spring-boot:run 2>&1 | grep -i "Started .* in "

# then change the pointcut to execution(* com.orderflow..*.*(..)) and re-run
./mvnw spring-boot:run 2>&1 | grep -i "Started .* in "
```

Better, with the startup endpoint:

```properties
management.endpoints.web.exposure.include=startup,beans
```

```bash
curl -s localhost:8080/actuator/startup | jq '.timeline.events
  | sort_by(-.duration) | .[0:15] | .[] | {name: .startupStep.name, duration}'
```

| What you see | What it means |
|---|---|
| Startup time noticeably higher with the broad pointcut | Expected. Pointcut matching runs per method per bean at startup. |
| No measurable difference | Your context is small. Repeat on a real application before concluding broad pointcuts are free — the effect scales with bean count. |
| `spring.beans.smart-initialize` steps dominated by proxy creation | You have found the cost directly. |

---

## Failure drill

**Mandatory.** Produce the failure yourself before reading the fix. The value is in
having seen a cache serve a price that is not in the database.

### The scenario

A price is computed inside a transaction and cached. The transaction rolls back. The
cache keeps the value and serves it to every subsequent request.

### Setup

`schema.sql`:

```sql
create table if not exists product (
  sku   varchar(32) primary key,
  price bigint not null           -- minor units. Never a double. (Topic 01)
);
insert into product (sku, price) values ('SKU-4471', 1999);
```

`application.properties`:

```properties
spring.datasource.url=jdbc:h2:mem:orderflow;DB_CLOSE_DELAY=-1
spring.sql.init.mode=always
spring.cache.type=simple
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.cache=TRACE
```

`CatalogConfig.java` — **the ordering that causes the bug**:

```java
package com.orderflow.lab;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.annotation.EnableTransactionManagement;

@Configuration
@EnableTransactionManagement(order = 100)   // transaction OUTSIDE
@EnableCaching(order = 200)                 // cache INSIDE  <-- the hazard
public class CatalogConfig { }
```

`PriceService.java`:

```java
package com.orderflow.lab;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class PriceService {

    private final JdbcTemplate jdbc;

    public PriceService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    /** Reads the CURRENT price. Cached. Called from inside a transaction below. */
    @Cacheable("product-prices")
    public long priceOf(String sku) {
        System.out.println("  DB READ for " + sku);
        return jdbc.queryForObject("select price from product where sku = ?", Long.class, sku);
    }

    public long readFromDbDirect(String sku) {
        return jdbc.queryForObject("select price from product where sku = ?", Long.class, sku);
    }
}
```

`PriceAdjustmentService.java` — writes, then reads through the cache, then fails:

```java
package com.orderflow.lab;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class PriceAdjustmentService {

    private final JdbcTemplate jdbc;
    private final PriceService prices;

    public PriceAdjustmentService(JdbcTemplate jdbc, PriceService prices) {
        this.jdbc = jdbc;
        this.prices = prices;
    }

    /**
     * A price-change job: write the new price, re-read it through the normal
     * (cached) read path to validate, then fail a sanity check.
     */
    @Transactional
    public void applyPriceChange(String sku, long newPrice) {
        jdbc.update("update product set price = ? where sku = ?", newPrice, sku);

        long readBack = prices.priceOf(sku);          // through the proxy -> CACHED NOW
        System.out.println("  read back inside tx = " + readBack);

        if (readBack > 2000) {                        // sanity check fails
            throw new IllegalStateException("price sanity check failed: " + readBack);
        }
    }
}
```

`CacheDrillRunner.java`:

```java
package com.orderflow.lab;

import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class CacheDrillRunner implements CommandLineRunner {

    private final PriceAdjustmentService adjust;
    private final PriceService prices;

    public CacheDrillRunner(PriceAdjustmentService adjust, PriceService prices) {
        this.adjust = adjust;
        this.prices = prices;
    }

    @Override
    public void run(String... args) {
        System.out.println("=== BEFORE ===");
        System.out.println("  db     = " + prices.readFromDbDirect("SKU-4471"));

        System.out.println("=== APPLY PRICE CHANGE (will roll back) ===");
        try { adjust.applyPriceChange("SKU-4471", 2499L); }
        catch (RuntimeException e) { System.out.println("  caught: " + e.getMessage()); }

        System.out.println("=== AFTER ROLLBACK ===");
        System.out.println("  db     = " + prices.readFromDbDirect("SKU-4471"));
        System.out.println("  cached = " + prices.priceOf("SKU-4471"));
    }
}
```

### Commands

```bash
./mvnw spring-boot:run
```

Then, to see the transaction and cache decisions interleaved:

```bash
./mvnw spring-boot:run 2>&1 | grep -E "DB READ|read back|caught|db     |cached |Getting transaction|Completing transaction|Cache entry"
```

### What to capture

Write down all five before reading on:

1. `db` before the change.
2. `read back inside tx`.
3. Whether a second `DB READ for SKU-4471` line appears at the end.
4. `db` after the rollback.
5. `cached` after the rollback.

### How to read it

| What you see | What it means |
|---|---|
| `db = 1999` before | Baseline. |
| `read back inside tx = 2499` | The read inside the transaction saw its own uncommitted write. Correct behaviour — read-your-own-writes. |
| `caught: price sanity check failed: 2499` | The transaction rolled back. |
| `db = 1999` after | **The database is correct.** The rollback worked perfectly. |
| `cached = 2499`, with **no** second `DB READ` line | **The drill has fired.** The cache is serving a price that has never existed in the database and never will. Every request now gets 2499 until the TTL expires or the pod restarts. |
| `cached = 1999` with a second `DB READ` line | The cache entry was not written, or was evicted. Check that `@EnableCaching` is present, that `spring.cache.type=simple`, and that the ordering in `CatalogConfig` is as shown. |
| TRACE shows the cache put happening between `Getting transaction` and `Completing transaction` | **The cleanest evidence available.** The cache write occurred *inside* the transaction's lifetime but was not covered by it — because the cache is not transactional. |

Now query it from the other side to make the damage concrete:

```bash
# in a second terminal, if you exposed an endpoint:
curl -s localhost:8080/products/SKU-4471       # returns 2499
# and the database:
#   select price from product where sku = 'SKU-4471';   -> 1999
```

Two sources of truth, disagreeing, with no error anywhere. At `orderflow`'s scale this
is a customer being charged a price that no system of record contains, and it will be
found by a finance reconciliation, not by monitoring.

### The fix — part 1: ordering

```java
@Configuration
@EnableTransactionManagement(order = 100)
@EnableCaching(order = 50)                  // cache now OUTSIDE the transaction
public class CatalogConfig { }
```

Re-run. Now the chain is `cache -> transaction -> method`. When the transaction
interceptor rolls back and rethrows, the exception passes *through* the cache
interceptor, which does not write on an exception path. **`cached` should now read
1999.**

Confirm the chain flipped, using Proof 3's `ChainPrinter`: `CacheInterceptor` should
appear at a lower index than `TransactionInterceptor`.

### The fix — part 2: make the cache transaction-aware (the real fix)

Ordering closes *this* hole. It does not close the general one, because the cache
still has no idea a transaction exists. Consider: the write happens, the method
returns, the cache interceptor is outside and writes on the way out — but the outer
transaction belongs to a **caller further up**, which rolls back later. Ordering
between these two advisors cannot help, because the rollback happens outside both.

```java
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        var simple = new ConcurrentMapCacheManager("product-prices");
        return new TransactionAwareCacheManagerProxy(simple);
    }
}
```

`TransactionAwareCacheManagerProxy` registers a `TransactionSynchronization` and
defers every put to `afterCommit`. If no transaction is active it writes immediately,
so non-transactional call paths are unaffected. For Redis:
`RedisCacheManager.builder(factory).transactionAware().build()`.

Re-run the drill with the **original** (bad) ordering plus the transaction-aware cache
manager. `cached` should read 1999 even then. That is how you know which fix is doing
the work.

### What the fix proves

Three things, in increasing order of importance:

1. **`@Order` is not cosmetic.** Two integers decided whether your cache and your
   database agree.
2. **The defaults do not decide this for you.** Both advisors default to
   `Ordered.LOWEST_PRECEDENCE`. You were relying on a tie-break that is not a contract.
   Whatever you observed on your machine is not guaranteed on the next Spring version
   or after a bean-name change.
3. **Ordering is a mitigation; transaction-awareness is the fix.** The strongest
   version of this answer in an interview is "I'd set the order explicitly *and* use a
   transaction-aware cache manager, because ordering only protects the case where both
   interceptors are on the same method."

---

## Measurement

### The wrong way, and why

```java
// WRONG. Do not use this number for anything.
long t0 = System.nanoTime();
for (int i = 0; i < 5_000_000; i++) service.pricedMethod("SKU-4471");
System.out.println((System.nanoTime() - t0) / 5_000_000 + " ns/op");
```

Four independent failure modes, and the result blends all of them in unknown
proportions:

1. **Dead-code elimination** — the return value is unused, so C2 can prove the call
   has no observable effect and delete the loop body. You time an empty loop.
2. **Constant folding** — a constant argument to a pure method lets the JIT hoist the
   result out of the loop entirely.
3. **On-stack replacement** — the loop begins interpreted and is swapped for compiled
   code mid-execution. Your average mixes interpreter, C1 and C2 in a ratio that
   depends on the iteration count you happened to choose.
4. **Cold JIT and profile pollution** — measuring the un-advised variant first makes
   the call site monomorphic; measuring the advised variant afterwards in the same JVM
   pollutes the profile. Both numbers are wrong, in opposite directions.

Topic 77 is this in full. Treat any quoted "Spring AOP costs N ns" that came from a
`nanoTime` loop as fiction.

### The right shape

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import org.springframework.aop.framework.ProxyFactory;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)
@State(Scope.Benchmark)
public class AspectOverheadBenchmark {

    public interface Pricer { long priceOf(long productId); }

    public static class RealPricer implements Pricer {
        public long priceOf(long productId) { return productId * 199L + 7L; }
    }

    private Pricer direct;
    private Pricer oneNoOpAdvice;
    private Pricer threeNoOpAdvices;
    private Pricer adviceThatSerialisesArgs;

    @Setup(Level.Trial)
    public void setUp() {
        RealPricer target = new RealPricer();
        direct = target;

        oneNoOpAdvice     = proxy(target, 1, false);
        threeNoOpAdvices  = proxy(target, 3, false);
        adviceThatSerialisesArgs = proxy(target, 1, true);
    }

    private Pricer proxy(RealPricer target, int adviceCount, boolean serialise) {
        var pf = new ProxyFactory(target);
        pf.setProxyTargetClass(true);
        for (int i = 0; i < adviceCount; i++) {
            pf.addAdvice((org.aopalliance.intercept.MethodInterceptor) inv -> {
                if (serialise) {
                    // the realistic cost of an audit aspect that logs arguments
                    String ignored = java.util.Arrays.toString(inv.getArguments());
                    if (ignored.isEmpty()) throw new IllegalStateException();
                }
                return inv.proceed();
            });
        }
        return (Pricer) pf.getProxy();
    }

    @Benchmark public void a_direct(Blackhole bh)        { bh.consume(direct.priceOf(4471L)); }
    @Benchmark public void b_oneAdvice(Blackhole bh)     { bh.consume(oneNoOpAdvice.priceOf(4471L)); }
    @Benchmark public void c_threeAdvices(Blackhole bh)  { bh.consume(threeNoOpAdvices.priceOf(4471L)); }
    @Benchmark public void d_serialising(Blackhole bh)   { bh.consume(adviceThatSerialisesArgs.priceOf(4471L)); }
}
```

| Element | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | JMH-owned fields, so the JIT cannot constant-fold the receivers. |
| `Blackhole.consume` | The result is genuinely consumed. Defeats dead-code elimination. |
| `@Warmup(5)` | Steady-state C2 code before anything is recorded. |
| `@Measurement(10)` | Enough samples for an error interval, not a single number. |
| `@Fork(3)` | Three separate JVMs. Exposes profile pollution as variance instead of hiding it. |
| Four variants, not two | Isolates chain-walk cost (a→b→c) from advice-body cost (b→d). |

```bash
./mvnw -Pbench clean verify
java -jar target/benchmarks.jar AspectOverheadBenchmark -rf json -rff aspect-bench.json
```

**What to look for:**

| Comparison | What it tells you |
|---|---|
| `a_direct` vs `b_oneAdvice` | The fixed cost of going through a proxy at all: dispatch, argument array, `MethodInvocation` allocation. |
| `b_oneAdvice` vs `c_threeAdvices` | The **marginal** cost per extra advice. Roughly linear — one frame and one `proceed()` each. |
| `b_oneAdvice` vs `d_serialising` | The cost of what **you wrote** in the advice body. Expect this gap to dwarf the other two. |
| Overlapping error intervals anywhere | You measured nothing there. Report the interval, never the point estimate. |

### The honest conclusion, stated up front

| Operation | Order of magnitude |
|---|---|
| Direct virtual call | ~1 ns |
| One no-op advice through a proxy | tens of ns |
| Three advices | still tens of ns |
| Advice that serialises arguments to a string | hundreds of ns to low µs |
| Advice that logs a JSON line synchronously | µs, and it does I/O |
| **A Postgres round trip** | **~200,000 ns** |

At `orderflow`'s 400 rps with one database round trip per command, the entire aspect
chain is a rounding error against the request budget — **unless the advice body does
real work**. That is the actionable finding: *the framework's overhead is negligible
and yours is not*.

Two things genuinely worth measuring, and neither is chain-walk cost:

1. **Startup cost of your pointcuts.** Proof 5. This is where a broad pointcut hurts,
   and it hurts your test suite most of all.
2. **Advice-body cost under load.** Take an async-profiler flame graph of `orderflow`
   at the Topic 65 baseline and look for your aspect's frames. If `CommandAuditAspect`
   is visible, it is doing too much. Topic 78.

> **The interview line:** "Aspect dispatch is tens of nanoseconds against a
> 200-microsecond database call, so I have never optimised it. What I *do* watch is
> what the advice body does — an audit aspect that serialises arguments was the most
> expensive thing in one of our flame graphs, and it was also a PII problem."

---

## Practice exercises

### 1 — Easy: build the chain and read it

1. Write three aspects — `A` with `@Order(1)`, `B` with `@Order(2)`, `C` with
   `@Order(3)` — each an `@Around` printing `enter X` / `exit X` around
   `GreetingService.greet(..)`.
2. Call `greet` once. Write down the exact printed sequence.
3. Predict what happens if you change `C` to `@Order(0)`. Then run it and check.
4. Add `ChainPrinter` from Proof 3 and confirm the advisor list order matches the
   printed sequence.
5. In one sentence, state the rule connecting `@Order` value to nesting depth — from
   what you observed, not from this document.

### 2 — Medium: the audit (combines Topics 01–40)

This aspect has **six** defects. Four are from this topic; two are from earlier ones.
For each: name the topic, state the *observable* production symptom, and write the fix.

```java
package com.orderflow.aop;

@Aspect
@Component
public class AuditAspect {

    private static final Logger log = LoggerFactory.getLogger(AuditAspect.class);

    private Map<String, Integer> callCounts = new HashMap<>();

    @Around("execution(* com.orderflow..*.*(..))")
    public Object audit(ProceedingJoinPoint pjp) throws Throwable {

        String method = pjp.getSignature().getName();
        Integer count = callCounts.get(method);
        callCounts.put(method, count + 1);

        log.info("calling {} with args {}", method, Arrays.toString(pjp.getArgs()));

        long start = System.currentTimeMillis();
        pjp.proceed();
        long elapsed = System.currentTimeMillis() - start;

        log.info("{} took {} ms", method, elapsed);
        return null;
    }
}
```

Hints, in the order to think about them: one defect makes **every advised method
return null** — find that first, it will mask everything else. One is a Topic 01
unboxing NPE. One is a Topic 38 shared-mutable-singleton race. One is a data-protection
incident. One will proxy the entire application. One risks infinite recursion.

### 3 — Hard: production simulation — instrument `orderflow` end to end

**Part A — build the spine.** Implement Example 2 in full: the `@Command` annotation,
`OrderflowPointcuts`, `CommandTimingAspect` (order 10), `CommandAuditAspect`
(order 20), and explicit `@EnableTransactionManagement(order = 100)` /
`@EnableCaching(order = 200)`. Annotate exactly the state-changing methods on
`OrderService`, `InventoryService` and `WalletService`.

**Part B — prove the chain.** Write a `@SpringBootTest` that:
1. asserts `orderService` is an AOP proxy;
2. casts it to `Advised` and asserts the advisor order is
   timing → audit → transaction → cache;
3. fails with a readable message if anyone later changes an `@Order`.

This test is the deliverable. It is the only thing that stops the failure drill from
recurring in six months.

**Part C — prove the metrics are bounded.** Place 1,000 orders across 50 customers and
20 products in a test. Then assert that
`meterRegistry.get("orderflow.command").timers()` has **at most 24** entries. Now
deliberately add `.tag("orderId", ...)` and watch the assertion fail with 1,000+
timers. Record the number. This is Topic 118's drill, arriving early and cheaply.

**Part D — reproduce the ordering bug on the spine.** Add `@Cacheable` to a
`ProductCatalogService.priceOf` method. Reproduce the failure drill inside `orderflow`
rather than in the lab project: a price change that rolls back, leaving a phantom
price in the cache. Then fix it both ways (ordering, and
`TransactionAwareCacheManagerProxy`) and write a test that fails if either fix is
removed.

**Part E — measure and decide.** Run the Part C test with the audit aspect logging
`pjp.getArgs()` and again with it logging only the action name. Record the difference
in total test wall time and in bytes of log output. Then answer: at 400 rps and 8.6 GB
of logs per day, what would you actually put in the audit record, and where would the
*detailed* record live instead? (There is a right answer and it involves the word
"outbox" — Topic 116.)

**Part F — argue against yourself.** You used a `@Command` annotation rather than a
package or naming-convention pointcut. Make the strongest case for the pointcut-based
approach. Then say what would have to be true about the codebase for it to win.

---

## Interview questions

### Q1 — "What actually is a Spring aspect, mechanically?"

**Mid-level answer:** "It's a class annotated `@Aspect` where you write advice methods
with pointcut expressions, and Spring applies them to matching methods for
cross-cutting concerns like logging and transactions."

**Senior answer:** "Mechanically it's a list of `MethodInterceptor`s. At startup,
`AnnotationAwareAspectJAutoProxyCreator` — which is a `BeanPostProcessor` — turns each
advice method into an `Advisor`: a pointcut plus an advice object. `@Around` becomes
`AspectJAroundAdvice`, which implements `MethodInterceptor` directly; `@Before` and the
others are adapted into the same interface. Then it evaluates every pointcut against
every bean, and any bean with a match gets a proxy — Topic 40's proxy, which is the
important part. At call time, `ReflectiveMethodInvocation.proceed()` walks the sorted
interceptor list with an index, and when the index runs off the end it invokes your
target method by reflection.

Two consequences fall out of that and they are what I actually use day to day. First,
`@Order` sorts the list, and lower means earlier means **outermost** — it's nesting,
not sequencing. Second, because it's a proxy, self-invocation bypasses everything, and
`final`, `private` and `static` methods can't be advised at all. Spring AOP is also
method-execution join points only — no field access, no constructor interception. If I
need those I'm into AspectJ weaving, which is a much bigger commitment."

**What separates them:** "a sorted list of `MethodInterceptor`s walked by `proceed()`"
is a mechanism. "Cross-cutting concerns" is a definition from a textbook. The senior
answer also volunteers Spring AOP's limits — method execution only — without being
asked.

**Follow-up:** "Where in the bean lifecycle does that proxy get created?"
`BeanPostProcessor.postProcessAfterInitialization` — and a good candidate immediately
adds "which is why a `@Transactional` method called from `@PostConstruct` isn't
transactional."

---

### Q2 — "You have `@Cacheable` and `@Transactional` on the same method. Does order matter, and what is it by default?"

**Mid-level answer:** "Order matters — you want the transaction on the outside. Spring
handles it by default."

**Senior answer:** "It matters, and the default does **not** handle it. Both
`@EnableTransactionManagement` and `@EnableCaching` default their advisor order to
`Ordered.LOWEST_PRECEDENCE`, so they tie, and the resolution falls out of advisor
registration order — which isn't a contract. Whatever your app does today isn't
guaranteed after a Spring upgrade or a bean rename.

The failure mode when it goes the wrong way: the cache interceptor sits inside the
transaction interceptor, so the cache entry is written when the method returns, which
is *before* the commit decision. The transaction rolls back, the row reverts, and the
cache keeps serving a value that never existed in the database until the TTL expires.
Nothing throws. You find it from a reconciliation report.

I do two things. I set both orders explicitly in a config class so the intent is in
the repository and reviewable. And I use a transaction-aware cache manager —
`TransactionAwareCacheManagerProxy`, or `.transactionAware()` on the Redis builder —
which registers a `TransactionSynchronization` and defers the put to `afterCommit`.
Ordering alone only protects the case where both interceptors are on the same method;
if the rollback belongs to a caller further up the stack, ordering can't see it, and
transaction-awareness can. I'd also add a `@SpringBootTest` that casts the bean to
`Advised` and asserts the advisor order, because otherwise the next person to touch an
`@Order` value reintroduces it silently."

**What separates them:** knowing the defaults tie rather than assuming Spring "handles
it", distinguishing the mitigation from the fix, and proposing a regression test for a
silent failure. The last part is the strongest signal — silent failures need active
checks, and most candidates never say so.

**Follow-up:** "What if the cache is populated by a different transaction that later
rolls back?" That's the general problem transaction-awareness also cannot fully solve,
and it leads into cache stampede and invalidation — Topic 110.

---

### Q3 — "Write me a pointcut that audits every service method in `com.orderflow`. Then tell me why you would not ship it."

**Mid-level answer:** "`@Around(\"execution(* com.orderflow..*Service.*(..))\")` — that
matches every method on every class ending in `Service`."

**Senior answer:** "That's the expression, and I'd reject it in review, for three
reasons.

First, it's a naming convention masquerading as a policy. Rename `OrderService` to
`OrderCoordinator` and auditing silently stops, with no compile error and no failing
test. Audit requirements shouldn't depend on a suffix.

Second, it matches queries as well as commands. In our catalogue that's a hundred
thousand reads a minute being audited alongside four hundred real state changes per
second, which is a log-volume and cost problem, not just noise.

Third, matching any method on a bean proxies the whole bean, so every bean in the
package inherits Topic 40's constraints — self-invocation, no advice on `final`
methods, no casting to the concrete class. That's a global change expressed as a
local-looking annotation.

What I'd ship is a `@Command` annotation carrying a stable low-cardinality name, and
`@annotation(com.orderflow.audit.Command)` as the pointcut, plus
`&& !within(com.orderflow.aop..*)` so aspects never advise aspects — otherwise a
collaborator call inside the aspect can recurse into `StackOverflowError` at startup.
It's explicit, greppable, survives renames, and the annotation's value doubles as the
metric tag. And I'd verify what I actually matched with
`logging.level.org.springframework.aop=DEBUG` rather than trusting the expression."

**What separates them:** treating a pointcut as a **global policy statement** with
maintenance and cost consequences. Almost everyone can write the expression; very few
volunteer that a broad pointcut changes the proxying characteristics of an entire
package.

**Follow-up:** "How would you find out which beans your pointcut actually matched?"
The AOP DEBUG log, or `Advised.getAdvisors()` on the bean.

---

### Q4 — "Your `@Around` advice is running, but the endpoint returns null. What happened?"

**Mid-level answer:** "Maybe the service returned null, or the serialisation failed."

**Senior answer:** "Almost certainly the advice didn't return `pjp.proceed()`'s value.
`@Around` advice *is* the method from the caller's point of view — whatever it returns
is what the caller gets. If you call `proceed()` for its side effect and then fall
through to `return null`, every advised method returns null while the target method
demonstrably computed the right answer. That's a two-line fix and a genuinely confusing
hour if you don't know the shape.

The related mistakes in the same family: catching `Throwable` in the advice and not
rethrowing, which swallows every exception from every advised method and turns failures
into silent successes; and `@AfterThrowing`, which people expect can suppress an
exception — it can't, only `@Around` can.

I'd also say this is an argument for using the weakest advice that does the job.
`@Before` plus `@AfterReturning` cannot express either of those bugs, because neither
one controls the return value. I reach for `@Around` only when I genuinely need to
wrap — timing, retry, changing arguments."

**What separates them:** knowing that `@Around` *replaces* the method rather than
decorating it, and generalising to a design rule about advice-type selection.

**Follow-up:** "How would you catch this in CI?" A test asserting the return value of
an advised method is non-null — trivial, and nobody writes it until they have been
bitten.

---

### Q5 — "How expensive is Spring AOP?"

**Mid-level answer:** "There's some proxy overhead but it's usually acceptable. I'd
benchmark it in a loop if I was worried."

**Senior answer:** "A `nanoTime` loop would produce a number I couldn't interpret —
dead-code elimination, constant folding, on-stack replacement and cold-JIT state all
contribute in unknown proportions, and if the result isn't consumed, C2 deletes the
loop body entirely. I'd use JMH with `@State`, `Blackhole`, warmup, and at least three
forks so profile pollution between the advised and un-advised variants shows up as
variance rather than hiding inside one number. And I'd report the confidence interval.

But the more useful answer is to split the cost in two. The **framework's** cost per
call is one extra virtual dispatch, an `Object[]` for the arguments with boxing for
primitives, a `MethodInvocation` allocation, and one stack frame per advice — tens of
nanoseconds, roughly linear in the number of advices. Against a 200-microsecond
Postgres round trip that is four orders of magnitude down; I have never optimised it.

The cost that actually matters is **the advice body**. An audit aspect that calls
`Arrays.toString(getArgs())` or serialises to JSON is microseconds and shows up in a
flame graph — and if an argument is a JPA entity, `toString()` can trigger a lazy load,
so the observability code changes the behaviour of the thing it observes. That's the
one I look for with async-profiler under load.

There's also a startup cost nobody measures: pointcut matching runs per method per bean
at context creation. A broad `execution(* com.acme..*(..))` measurably slows startup,
and it slows every `@SpringBootTest` in the suite. That's usually the bigger real-world
bill."

**What separates them:** naming the four JIT effects, splitting framework cost from
advice-body cost, and volunteering the **startup** cost — which almost nobody
mentions and which is frequently the largest actual impact.

**Follow-up:** "So when has AOP overhead genuinely mattered to you?" A good answer
describes an advised method inside a tight loop, and notes the fix was to advise the
loop rather than the body.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `@Order(1)` runs "before" `@Order(2)`. But `@Order(1)`'s `@After` advice runs
   *after* `@Order(2)`'s. Explain why both statements are true using one word.

2. Spring AOP only supports method-execution join points. AspectJ supports field
   access, constructor execution and more. Derive Spring's limitation from the proxy
   mechanism alone — do not recall it.

3. `this(PaymentGateway)` matches the proxy's type; `target(PaymentGateway)` matches
   the target's. Construct a situation where those two designators give different
   answers for the same bean. (Hint: Topic 40, JDK versus CGLIB.)

4. A dynamic pointcut like `args(PlaceOrderCommand)` is re-evaluated on every call.
   Why can it not be resolved once at startup, when `@annotation(...)` can?

5. Your audit aspect sits outside the transaction interceptor and logs
   `outcome=success` when the method returns. Name the exact circumstance in which
   that log line is a lie, and then say why moving the aspect *inside* the transaction
   does not fix it.

6. A `@Cacheable` method whose cache is transaction-aware defers the put to
   `afterCommit`. What new failure mode does that introduce that the non-aware version
   did not have? (Think about what happens between the method returning and the commit
   landing, under concurrency.)

7. Nest attaches interceptors by writing them at the route; Spring attaches advice by
   evaluating a predicate. Name one maintenance property Nest's approach has that
   Spring's does not, and one capability Spring has that Nest cannot express. Then say
   which you would want in a 40-service codebase.

---

## Quick reference card

### Advice types

```java
@Before("pc()")                    void  x(JoinPoint jp)
@AfterReturning(value="pc()", returning="r")  void x(JoinPoint jp, Object r)
@AfterThrowing(value="pc()", throwing="e")    void x(JoinPoint jp, Throwable e)  // cannot suppress
@After("pc()")                     void  x(JoinPoint jp)                          // a finally
@Around("pc()")                    Object x(ProceedingJoinPoint pjp) throws Throwable
```

### Pointcut designators

```
@annotation(FQN)     method carries the annotation          STATIC   <- prefer this
@within(FQN)         declaring class carries it             STATIC
@target(FQN)         runtime target class carries it        dynamic-ish
within(pkg..*)       inside a package or type               STATIC
execution(sig)       method signature match                 STATIC
bean(namePattern)    Spring bean name (not AspectJ)         STATIC
args(Types)          runtime argument types                 DYNAMIC — per call
this(Type)           the PROXY's type
target(Type)         the TARGET's type
```

Combine with `&&`, `||`, `!`.

### `execution` signature grammar

```
execution( modifiers?  return-type  declaring-type?.method-name(params)  throws? )

execution(public * com.orderflow.orders.OrderService.place(..))
execution(* com.orderflow..*Service.*(..))       // .. in package = "and subpackages"
execution(* *.place(String, ..))                 // .. in params  = "and any more"
```

### Ordering

```java
@Order(10)   // LOWER value = HIGHER precedence = OUTERMOST = starts first, finishes last
@Order(20)
@EnableTransactionManagement(order = 100)
@EnableCaching(order = 200)
// Both default to Ordered.LOWEST_PRECEDENCE — they TIE. Set them explicitly.
```

### The `JoinPoint` API

```java
pjp.getSignature().getName()                       // method name
((MethodSignature) pjp.getSignature()).getMethod() // java.lang.reflect.Method -> annotations
pjp.getArgs()                                      // Object[] — do NOT log this
pjp.getTarget()                                    // the real bean
pjp.getThis()                                      // the proxy
pjp.proceed()                                      // call the next interceptor / the target
pjp.proceed(newArgs)                               // ... with different arguments
```

### Diagnostics

```properties
logging.level.org.springframework.aop=DEBUG
logging.level.org.springframework.aop.aspectj=TRACE
logging.level.org.springframework.cache=TRACE
logging.level.org.springframework.transaction.interceptor=TRACE
management.endpoints.web.exposure.include=startup,beans
```

```java
((Advised) bean).getAdvisors()                     // the chain, in execution order
((Advised) bean).getTargetClass()
AopUtils.isAopProxy(bean)
AopProxyUtils.ultimateTargetClass(bean)
```

### Gotchas checklist

- [ ] `@Aspect` **and** `@Component`. `@Aspect` alone is inert.
- [ ] `spring-boot-starter-aop` must be on the classpath.
- [ ] `@Around` must `return pjp.proceed()`. Forgetting returns `null` silently.
- [ ] Never swallow the exception in `@Around`. Rethrow.
- [ ] `@AfterThrowing` cannot suppress an exception. Only `@Around` can.
- [ ] Lower `@Order` = outermost. It is nesting, not sequencing.
- [ ] Transaction and cache advisor orders **tie** by default. Set both.
- [ ] Ordering mitigates; `TransactionAwareCacheManagerProxy` fixes.
- [ ] Prefer `@annotation(...)` over `execution(...)` over a package wildcard.
- [ ] Add `&& !within(your.aop.package..*)` to avoid advising your own aspects.
- [ ] Never log `pjp.getArgs()`. PII, volume, and lazy-loading side effects.
- [ ] Self-invocation bypasses every aspect. Always. (Topic 40.)
- [ ] Keep a test asserting the advisor order. It is the only defence against a silent
      reordering.

---

## When would I use this at work?

**1. Adding observability to a service you did not write.**
A legacy `orderflow` module has forty methods and no metrics. You add one annotation
and one aspect, and every command is timed and audited without touching forty files —
and without the risk of touching forty files. This is the highest-value, lowest-risk
application of AOP, and it is what the spine work in this document builds.

**2. Debugging "the annotation did nothing".**
Someone reports that `@Transactional` is not working. You cast the bean to `Advised`,
print the advisor list, and see either that the bean is not proxied at all or that the
transaction advisor is not in the chain for that method. Two minutes instead of an
afternoon. This works identically for `@Cacheable`, `@Async`, `@Validated`,
`@PreAuthorize` and `@Retry`, because they are all the same mechanism.

**3. Reviewing a proposed aspect.**
A colleague opens a PR with `execution(* com.orderflow..*(..))` and an
`Arrays.toString(getArgs())` log line. You can name the four consequences — startup
cost, whole-package proxying, PII in the log store, and a possible lazy-load
side effect — with specifics rather than "that feels broad". That review comment is
the difference between a mid-level and a senior engineer in the eyes of everyone
reading the thread.

---

## Connected topics

**Prerequisites:**
- **40 — Proxying mechanics**: mandatory. Every aspect in this document lives on a
  proxy, and the self-invocation trap applies to all of them.
- **37 — Bean lifecycle**: `AnnotationAwareAspectJAutoProxyCreator` is a
  `BeanPostProcessor` running in `postProcessAfterInitialization`.
- **39 — Injection styles**: `@Order` first appeared there for `List` injection
  ordering; it is the same comparator.
- **38 — Bean scopes**: the request-scoped `CurrentActor` the audit aspect injects.
- **36 — Component scanning**: an aspect outside the scanned packages is invisible.
- **01 — Boxing**: argument arrays box primitives on every advised call.

**This unlocks:**
- **45 — Bean Validation**: `@Validated` is another interceptor on this chain, and its
  ordering relative to transactions is a real decision.
- **54–55 — `@Transactional`**: `TransactionInterceptor` is the aspect you will spend
  the most time reasoning about. You now know what it is.
- **56–57 — Spring Security**: `@PreAuthorize` is a method interceptor too, and its
  order relative to transactions decides whether an unauthorized call has already
  opened a database transaction.
- **110 — Spring Cache and Redis**: the failure drill in this document is the
  introduction; Topic 110 covers stampede, invalidation and key design.
- **111 — Resilience4j**: `@Retry`, `@CircuitBreaker` and `@Bulkhead` are aspects, and
  their **decorator order** is exactly this topic's `@Order` problem with an outage
  attached — retry outside the breaker means the breaker never sees the failures.
- **116 — Transactional outbox**: where the detailed audit record should actually live,
  as Exercise 3 Part E asks you to work out.
- **118 — Micrometer metrics**: `CommandTimingAspect` becomes the RED metrics for
  every `orderflow` endpoint. The bounded-tag discipline you applied here is what
  keeps Prometheus alive.
- **119 — Distributed tracing**: an aspect is a natural place to open a span, and the
  `ThreadLocal` context propagation problem meets it immediately.
- **120 — Logging and MDC**: the audit aspect's correlation ID comes from MDC, which
  does not cross thread boundaries.
- **78 — Profiling**: how to find your advice body in a flame graph.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0, `jakarta.*`
throughout. Two points are deliberately hedged rather than asserted: the exact ordering
rule for multiple advice methods declared in a **single** aspect class on Framework
7.0 — Spring's documented behaviour has changed across versions, and Proof 4 settles it
on your build in three minutes — and the precise tie-break when the transaction and
cache advisors share `Ordered.LOWEST_PRECEDENCE`, which is exactly why you should set
both orders explicitly instead of learning the answer. Everything else here — the
interceptor-chain model, the designator semantics, `@Order` meaning nesting depth, and
the failure drill — has been stable since Spring 2.0 introduced `@AspectJ` support and
will still be true when you next debug a cache serving a price that is not in the
database.*
