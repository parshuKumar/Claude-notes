# 40 — Proxying Mechanics: JDK Dynamic Proxies vs CGLIB, and the Self-Invocation Trap

## Phase: 4 — Spring Core
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: no new `orderflow` capability. This is the prerequisite that makes Topics 45, 54, 55, 110 and 111 debuggable instead of magical.

---

## Mechanical statement

Read this twice before continuing. Everything else in the document is an elaboration
of it.

> **Spring hands callers a DIFFERENT OBJECT than the one you wrote.**
>
> A **JDK dynamic proxy** is a runtime-generated class that implements the same
> *interfaces* as your bean and dispatches every interface method to an
> `InvocationHandler`.
>
> A **CGLIB proxy** is a runtime-generated **subclass** of your bean's class. It
> overrides every non-`final`, non-`private`, non-`static` method and delegates to a
> `MethodInterceptor`, which eventually calls your real object.
>
> **Either way, a call that originates from `this` inside your bean never touches the
> proxy.** It is a direct virtual dispatch on the target object. Every annotation you
> wrote — `@Transactional`, `@Cacheable`, `@Async`, `@Validated`, `@PreAuthorize`,
> `@Retry` — is implemented by that proxy, and is therefore silently absent from
> self-calls.

There is no error. There is no warning. There is no log line. The annotation is
simply metadata that nothing read.

---

## The bridge from what you know

### The analogue that is genuinely honest: `Proxy` in JavaScript

You already have this object:

```ts
const target = {
  charge(amountMinor: number) { return `charged ${amountMinor}`; },
};

const proxied = new Proxy(target, {
  get(obj, prop, receiver) {
    const original = Reflect.get(obj, prop, receiver);
    if (typeof original !== 'function') return original;
    return (...args: unknown[]) => {
      console.log(`ENTER ${String(prop)}`);
      try   { return original.apply(obj, args); }
      finally { console.log(`EXIT ${String(prop)}`); }
    };
  },
});

proxied.charge(1999);   // logs ENTER, charged 1999, EXIT
target.charge(1999);    // logs NOTHING — you called the target directly
```

That last line is the entire topic. **You already understand the self-invocation
trap.** You have just never been in a situation where the framework handed you the
proxy and kept the target hidden, so you never had to think about which one `this`
points at.

Java's `java.lang.reflect.Proxy` is the direct equivalent: a handler object with one
`invoke(proxy, method, args)` entry point, wrapping a target.

**Verdict: HONEST analogue.** Use it.

### What has NO JavaScript equivalent: CGLIB

JavaScript has no way to say "generate, at runtime, a new class that `extends`
`OrderService`, override every method, and instantiate it without running the
constructor". You can approximate it with prototype tricks, but there is no bytecode
generation and no subclassing of a compiled class.

CGLIB does exactly that. It writes a new `.class` file into memory whose bytecode
says `class OrderService$$SpringCGLIB$$0 extends OrderService`, defines it into a
`ClassLoader`, and creates an instance.

**Verdict: NO ANALOGUE.** This is the part you have to learn from scratch, and it is
the part that produces the `final`-method and constructor surprises later in this
document.

### The bridge that is PARTIAL — and the difference is the whole lesson

You will be tempted to map Spring AOP advice onto NestJS interceptors. Resist it just
long enough to see where the mapping breaks.

```ts
// NestJS
@Injectable()
export class TimingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const started = Date.now();
    return next.handle().pipe(tap(() => console.log(Date.now() - started)));
  }
}

@Controller('orders')
@UseInterceptors(TimingInterceptor)
export class OrdersController {
  @Post() place(@Body() dto: PlaceOrderDto) { return this.orders.place(dto); }
}
```

Nest wires that interceptor into the **request pipeline at the route boundary**. The
framework owns the call site: an HTTP request arrives, Nest runs guards, then
interceptors, then your handler. Your `OrdersController` object is the object you
wrote. It is not wrapped.

So in Nest, when `OrderService.place()` internally calls `this.reserveStock()`,
nothing happens — and **you never expected anything to happen**, because you never
put an interceptor on `reserveStock`. Interceptors live at routes. Routes are entry
points. Entry points are not reachable from inside your own class.

Spring is different in exactly one way, and that one way generates most of the
"Spring is magic and I hate it" experiences in the industry:

**Spring's advice lives on the object itself, not at a route boundary.** The bean in
the container is a wrapper. Every collaborator that injected `OrderService` holds the
wrapper. And `this` inside `OrderService` is *not* the wrapper.

So in Spring you *can* put `@Transactional` on `reserveStock()`. It looks like it
should work. It compiles. Nothing complains. And it does nothing at all when called
from `place()`.

| | NestJS interceptor | Spring AOP advice |
|---|---|---|
| Where it is attached | route / controller handler | **the bean object** |
| What holds it | the framework's request pipeline | a proxy object in the container |
| Reachable by a self-call | no, and you never expected it to be | **no, and you absolutely expected it to be** |
| Failure mode when misused | you notice, because there is no route | **silent** |

**Verdict: PARTIAL, and the delta is the lesson.** The shape is the same — an
interceptor chain around a call. The attachment point is different, and that
difference is why this trap catches TypeScript engineers specifically. Your instincts
are calibrated for a world where cross-cutting concerns live at the edge. Spring puts
them on the object.

### TS decorators vs Java annotations — say this out loud once

```ts
function Transactional(): MethodDecorator {
  return (target, key, descriptor) => {
    const original = descriptor.value;         // this CODE RUNS at class definition
    descriptor.value = function (...args) { /* wrap */ };
    return descriptor;
  };
}
```

A TypeScript decorator is a **function that executes** when the class is defined. It
can physically replace the method on the prototype. By the time anyone calls the
method, the wrapping already happened — including for `this.method()` calls, because
the prototype itself was rewritten.

```java
@Transactional
public void reserveStock(String sku, int units) { ... }
```

A Java annotation **executes nothing**. It is a run of bytes in the class file's
`RuntimeVisibleAnnotations` attribute. It is inert. Something else must read it
reflectively and *build a different object*. That something else is
`AbstractAutoProxyCreator`, a `BeanPostProcessor`, and the different object is the
proxy.

**This is the root cause of the whole topic.** A TS decorator can rewrite the method
in place, so self-calls are covered. A Java annotation cannot touch your method at
all, so the only place the behaviour can live is a *wrapper* — and self-calls skip
wrappers.

**Verdict: PARTIAL, and knowing the delta explains the trap without memorising it.**

---

## What is this?

Spring implements cross-cutting behaviour by **replacing your bean with a wrapper**
during startup. There are exactly two wrapping strategies in Spring AOP:

**1. JDK dynamic proxy.** Requires your class to implement at least one interface.
`java.lang.reflect.Proxy.newProxyInstance(loader, interfaces, handler)` generates a
class that implements those interfaces and extends `java.lang.reflect.Proxy`. Every
interface method body is: pack the arguments into an `Object[]`, call
`handler.invoke(this, method, args)`. Spring's handler is `JdkDynamicAopProxy`, which
walks the advice chain and then reflectively invokes the method on the real target.

**2. CGLIB proxy.** Requires only that your class is not `final`. CGLIB generates a
**subclass** at runtime, overriding every non-`final`, non-`private`, non-`static`,
visible method. Each override calls a `MethodInterceptor`. Spring's is
`CglibAopProxy.DynamicAdvisedInterceptor`, which walks the same advice chain and then
calls the method on the real target.

### Which one do you get?

The decision lives in `DefaultAopProxyFactory`. In plain Spring Framework:

```
if (proxyTargetClass is true) -> CGLIB
else if (target class is an interface, or is already a JDK proxy, or is a lambda) -> JDK
else if (target class implements at least one non-internal interface) -> JDK
else -> CGLIB
```

**But you are on Spring Boot, and Boot changes the default.** Since Boot 2.0,
`AopAutoConfiguration` sets `spring.aop.proxy-target-class=true` unless you say
otherwise, which means `@EnableAspectJAutoProxy(proxyTargetClass = true)`. The same
default is applied to `@EnableTransactionManagement` and `@EnableCaching` through
Boot's auto-configuration.

> **The practical rule for a Boot 3.x/4.x application: you get CGLIB, even when your
> class implements an interface.** You will see `$$SpringCGLIB$$` in class names far
> more often than `$Proxy`. Boot made that choice because interface-only proxying
> broke too many things — chiefly, you cannot cast a JDK proxy to the concrete class,
> and half the ecosystem does exactly that.

You force JDK proxies back on with:

```properties
spring.aop.proxy-target-class=false
```

That single line is the most useful diagnostic in this whole topic, and you will use
it in the Hands-on section.

`[BOOT 3.x DELTA]` — none for this default. `spring.aop.proxy-target-class=true` has
been Boot's default since 2.0 and is unchanged in 3.x and 4.x. What *did* change: on
Boot 2.x you will more often meet `$$EnhancerBySpringCGLIB$$` in class names, because
Spring Framework 5.3 renamed the generated-class marker to `$$SpringCGLIB$$`. Same
mechanism, different string to grep for.

---

## Why does it matter?

**1. It is the single most common "why did my annotation do nothing" in Java.**

Every proxy-based feature inherits this: `@Transactional` (54–55), `@Cacheable`
(110), `@Async`, `@Validated` (45), `@PreAuthorize` (57), Resilience4j's
`@CircuitBreaker` and `@Retry` (111), `@Scheduled`. If you do not know this
mechanism, each of those is a separate mystery. If you do, they are one fact.

**2. The failure is silent and the data damage is permanent.**

A `@Transactional` method that does not roll back writes half a business operation
and leaves it there. There is no exception to alert on, no error metric, no log line.
You find it from a reconciliation report weeks later, and by then you have thousands
of rows in a state your invariants say is impossible.

**3. It changes what your object graph actually contains.**

The bean in the container is not the class you wrote. That has consequences you will
meet directly: `ClassCastException` on a cast that "obviously" works, `getClass()`
returning a name you do not recognise, field access returning `null`, `@Value` fields
appearing unset, `instanceof` behaving unexpectedly, a constructor running twice.
Every one of those is a two-hour debugging session if you do not have the mechanism,
and a thirty-second diagnosis if you do.

**4. It is a staple senior interview filter.**

"Your `@Transactional` did nothing — walk me through why" separates people who use
Spring from people who understand it. There is no way to bluff it.

---

## Machine-level reality

This section is what makes Topic 40 a differentiator. Everything here is checkable,
and where I am not certain I say so and give you the command.

### How a JDK dynamic proxy is actually made

```java
Object proxy = java.lang.reflect.Proxy.newProxyInstance(
        classLoader,                                  // where to define the new class
        new Class<?>[] { PaymentGateway.class },      // interfaces to implement
        invocationHandler);                           // the single dispatch point
```

At runtime the JDK synthesises a class whose bytecode is roughly:

```java
final class $Proxy37 extends java.lang.reflect.Proxy implements PaymentGateway {

    private static final Method m3;   // PaymentGateway.authorize(AuthorizationRequest)

    $Proxy37(InvocationHandler h) { super(h); }

    public final AuthorizationResult authorize(AuthorizationRequest r) {
        try {
            return (AuthorizationResult) super.h.invoke(this, m3, new Object[] { r });
        } catch (RuntimeException | Error e) { throw e; }
          catch (Throwable t) { throw new UndeclaredThrowableException(t); }
    }
    // plus overrides of equals, hashCode, toString, all routed to h.invoke
}
```

Four facts fall out of that bytecode, and each is a real-world consequence:

1. **It `extends java.lang.reflect.Proxy`.** Java has single inheritance, so the proxy
   *cannot* also extend your class. Casting a JDK proxy to your concrete class throws
   `ClassCastException`. This is why Boot defaults to CGLIB.
2. **It only implements the interfaces you passed.** A `public` method on your class
   that is not on any interface does not exist on the proxy. Callers holding the
   interface type cannot reach it at all.
3. **Every generated method is `final`.** You cannot subclass a JDK proxy.
4. **Arguments are packed into an `Object[]`.** Primitives are boxed on every call —
   Topic 01's allocation, once per invocation, per primitive.

The proxy class's package and module placement varies by JDK version and by whether
the proxied interfaces are public and in the unnamed module. **I am not going to
assert the exact package name for JDK 25.** The reliable marker is the **simple
name**, which starts with `$Proxy`. Do not pattern-match the package. The Hands-on
section gives you the command that prints the real answer on your machine.

### How a CGLIB proxy is actually made

Spring bundles a repackaged copy of CGLIB under `org.springframework.cglib` so you
never add the dependency yourself. It generates, defines and instantiates a subclass:

```java
// conceptual shape of the generated class
public class OrderService$$SpringCGLIB$$0 extends OrderService {

    private MethodInterceptor CGLIB$CALLBACK_0;   // = DynamicAdvisedInterceptor

    @Override
    public void place(PlaceOrderCommand cmd) {
        MethodInterceptor mi = this.CGLIB$CALLBACK_0;
        if (mi != null) {
            mi.intercept(this, METHOD_place, new Object[] { cmd }, METHODPROXY_place);
        } else {
            super.place(cmd);
        }
    }

    // NOT overridden: final methods, private methods, static methods,
    //                 package-private methods from another package
}
```

Six facts, each with a consequence:

1. **`final` methods cannot be overridden.** The JVM forbids it — a class file that
   overrides a `final` method fails verification with `VerifyError`. So CGLIB simply
   *skips* them. **No error. No exception. Your annotation is ignored.** Spring's
   `CglibAopProxy` may emit an INFO-level message when it notices a `final` method it
   wanted to advise, but it is easy to miss and it is not a failure. Verify by
   behaviour, not by the absence of a log line.
2. **`final` classes cannot be subclassed at all.** Here Spring *does* fail loudly, at
   startup, with `AopConfigException: Could not generate CGLIB subclass of class ...`.
   Note that **records are implicitly `final`** — a record can never be CGLIB-proxied.
3. **`private` methods are not inherited**, so there is nothing to override. Also,
   Spring's `AnnotationTransactionAttributeSource` only reads transaction metadata
   from `public` methods in proxy mode, so a `private @Transactional` method is
   ignored twice over.
4. **`static` methods are not virtually dispatched.** An override is impossible by
   definition.
5. **The proxy has its own copy of every field, and they are all empty.** The
   generated subclass inherits the field *declarations* from your class, but Spring
   never populates them — your dependencies were injected into the *target* instance,
   which the proxy holds a reference to. So `proxy.someField` reads `null` or `0`.
   Anything that touches a field directly instead of through a method sees an empty
   object. This is why Spring insists you access state through methods.
6. **The proxy needs an instance of the subclass, and creating one normally would run
   your constructor a second time.** This is the "constructors run twice-ish" fact.

### Why constructors run twice-ish under CGLIB

Instantiating `OrderService$$SpringCGLIB$$0` with `new` would invoke its constructor,
which must call `super(...)` — your constructor — again. So without mitigation, a
CGLIB-proxied bean's constructor runs **twice**: once for the real target, once for
the proxy shell.

Spring mitigates this with **Objenesis**, a library that allocates an instance of a
class *without running any constructor* (it uses JVM-specific tricks — on HotSpot,
`ReflectionFactory.newConstructorForSerialization`, the same mechanism deserialization
uses). Spring's `ObjenesisCglibAopProxy` is the default. When Objenesis succeeds, your
constructor runs exactly once.

When Objenesis **fails or is unavailable**, Spring falls back to
`enhancer.create(...)`, which does invoke the constructor. Then it really does run
twice, on a second object whose fields are then discarded.

The practical rule this produces:

> **A proxied bean's constructor must be side-effect-free.** No registering with a
> global registry, no starting a thread, no incrementing a counter, no writing a row.
> Put that work in `@PostConstruct` or `InitializingBean` — those run on the target
> exactly once. This is a Topic 37 fact you now have a mechanism for.

`@Configuration` classes are a special case you have already been relying on: they are
CGLIB-proxied so that a `@Bean` method calling another `@Bean` method returns the
singleton instead of a new object. That is *why* `@Configuration` classes cannot be
`final`, and why `@Configuration(proxyBeanMethods = false)` — "lite mode" — changes
inter-bean-method calls into ordinary Java calls that construct new instances. Same
mechanism, visible in a place you have already met.

### Where in the lifecycle the proxy is created — and why `@PostConstruct` cannot see it

From Topic 37's ordering:

```
instantiate
  -> populate properties (@Autowired / constructor args are already done)
     -> *Aware callbacks
        -> BeanPostProcessor.postProcessBeforeInitialization
           -> @PostConstruct
              -> InitializingBean.afterPropertiesSet
                 -> BeanPostProcessor.postProcessAfterInitialization   <-- PROXY CREATED HERE
                    -> the container stores THIS object as the bean
```

`AbstractAutoProxyCreator` is a `BeanPostProcessor`. Its `wrapIfNecessary` runs in
`postProcessAfterInitialization`. Two consequences that people rediscover the hard way:

1. **Any method called from `@PostConstruct` gets no advice**, even if you go through
   another bean, because *your own* proxy does not exist yet at that instant. A
   `@Transactional` warm-up method invoked from `@PostConstruct` is not transactional.
   Use `ApplicationRunner`, `CommandLineRunner`, or an
   `ApplicationListener<ContextRefreshedEvent>` instead — those fire after the whole
   context is proxied. (This is the fix for the Topic 37 catalogue warm-up hook.)
2. **During a circular dependency, Spring may expose an *early* reference to the
   unproxied target** via `getEarlyBeanReference`. That is why circular references
   became a startup failure by default in Boot 2.6 (Topic 39, Trap 3): a collaborator
   could end up holding the raw target while everyone else holds the proxy, and its
   calls would silently get no advice. You now have the mechanism behind that rule.

### The cost of the indirection, per call

A direct call:

```
invokevirtual OrderService.place   ->  your bytecode
```

A CGLIB-proxied call:

```
invokevirtual OrderService.place            (dispatches to the SUBCLASS override)
  -> allocate Object[] for the arguments    (+ boxing of any primitives — Topic 01)
  -> DynamicAdvisedInterceptor.intercept
     -> build a CglibMethodInvocation
        -> ReflectiveMethodInvocation.proceed()   (once per advice, recursively)
           -> ... your interceptor chain ...
              -> invoke the method on the real target
                 -> your bytecode
```

So: one extra virtual dispatch, one array allocation, boxing per primitive argument,
one object per invocation for the `MethodInvocation`, and one stack frame per advice.

**Whether Spring's CGLIB path dispatches the final call through CGLIB's generated
`FastClass`/`MethodProxy` (index-based, no reflection) or through plain
`Method.invoke` has changed across Spring versions, and I am not certain which
applies on Framework 7.0.** It matters only for a micro-benchmark, never for
correctness. Settle it on your machine by reading
`CglibAopProxy$CglibMethodInvocation` in the sources your build resolved, or by
looking at an async-profiler flame graph under load (Topic 78). Do not take my word
for it, and do not take a blog post's word for it either.

The honest bottom line, stated now so the Measurement section can prove it: **this
overhead is in the tens-to-low-hundreds of nanoseconds. A Postgres round trip is
200,000 to 2,000,000 nanoseconds. You do not learn proxies for performance. You learn
them for correctness.**

---

## Example 1 — minimal

Two classes, one annotation, one surprise.

```java
package com.orderflow.lab;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProxyDemoService {

    /** Called from outside -> goes THROUGH the proxy. */
    public String outer() {
        return "outer -> " + inner();      // self-call: 'this.inner()'
    }

    /** Advised in theory. Never advised when reached via outer(). */
    @Transactional
    public String inner() {
        return "inner";
    }
}
```

```java
package com.orderflow.lab;

import org.springframework.aop.framework.AopProxyUtils;
import org.springframework.aop.support.AopUtils;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class ProxyProbe implements CommandLineRunner {

    private final ProxyDemoService service;

    public ProxyProbe(ProxyDemoService service) { this.service = service; }

    @Override
    public void run(String... args) {
        System.out.println("class name        = " + service.getClass().getName());
        System.out.println("isAopProxy        = " + AopUtils.isAopProxy(service));
        System.out.println("isCglibProxy      = " + AopUtils.isCglibProxy(service));
        System.out.println("isJdkDynamicProxy = " + AopUtils.isJdkDynamicProxy(service));
        System.out.println("ultimateTarget    = "
                + AopProxyUtils.ultimateTargetClass(service).getName());
    }
}
```

Run it. `class name` will not be `com.orderflow.lab.ProxyDemoService`. That is the
whole point: **the object you were handed is not the object you wrote**, and one
`println` proves it. The `ultimateTargetClass` line tells you what is underneath —
that method is the one to reach for when you need the real class for logging,
metrics tags, or a `switch` on type.

> `CommandLineRunner` rather than `@PostConstruct`, deliberately. See the lifecycle
> note above: at `@PostConstruct` time your own proxy does not exist yet.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` order placement, sized against the Phase 7 gate targets:

- **400 order placements/second** at flash-sale peak; **1M existing orders**,
  **100k products**.
- Placement must be atomic across four writes: reserve inventory, debit wallet,
  insert payment, insert order. Partial completion is a **money bug**, not a latency
  bug.
- The team wants an audit row written for every placement attempt, **including failed
  ones**, so the audit write must survive the rollback of the placement.

That last requirement is exactly the situation where an engineer reaches for
`REQUIRES_NEW` on an internal method — and hits this trap at full force.

### The code that looks right and destroys data

```java
package com.orderflow.orders;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final InventoryService inventory;
    private final WalletService wallet;
    private final PaymentService payments;
    private final OrderRepository orders;
    private final OrderAuditRepository auditRepo;

    public OrderService(InventoryService inventory,
                        WalletService wallet,
                        PaymentService payments,
                        OrderRepository orders,
                        OrderAuditRepository auditRepo) {
        this.inventory = inventory;
        this.wallet = wallet;
        this.payments = payments;
        this.orders = orders;
        this.auditRepo = auditRepo;
    }

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {

        writeAuditRow(cmd);                       // <-- SELF-CALL. Propagation ignored.

        inventory.reserve(cmd.sku(), cmd.units());
        wallet.debit(cmd.customerId(), cmd.totalMinorUnits());
        var paymentId = payments.authorize(cmd);
        return orders.insert(cmd, paymentId);
    }

    /** Intended: always persist the attempt, even if placement rolls back. */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeAuditRow(PlaceOrderCommand cmd) {
        auditRepo.insertAttempt(cmd.customerId(), cmd.sku(), cmd.units());
    }
}
```

### What actually happens at 400 rps

`writeAuditRow` is reached via `this.writeAuditRow(cmd)`. That is an `invokevirtual`
on the target object. The proxy is not in the call path, so
`Propagation.REQUIRES_NEW` is never read by anything. The audit insert joins the
**outer** transaction.

Result: when a placement fails — insufficient stock, declined card, insufficient
wallet balance — the outer transaction rolls back and **takes the audit row with
it**. The audit table contains only successes.

The observable symptom is not an exception. It is a **business number**: at a 3%
decline rate and 400 rps, roughly 12 audit rows per second silently vanish, about
1 million per day, and the fraud team's dashboard shows a decline rate of exactly
zero. Someone will spend a week investigating the fraud pipeline before anyone looks
at `OrderService`.

There is a second, worse variant of this same mistake. Suppose someone "optimises" by
moving the wallet debit inline:

```java
    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        inventory.reserve(cmd.sku(), cmd.units());
        debitWalletInNewTransaction(cmd);        // self-call again
        ...
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void debitWalletInNewTransaction(PlaceOrderCommand cmd) { ... }
```

Now the intent was "the debit commits independently". The reality is that it joins the
outer transaction. If a later step throws, the debit rolls back too. That is arguably
the *correct* behaviour by luck — which is the most dangerous outcome of all, because
the code and the behaviour disagree and nobody notices until the requirement changes.

### The fix, and why it is a design improvement rather than a workaround

```java
package com.orderflow.orders;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

/**
 * A separate bean, therefore a separate proxy, therefore a real transaction boundary.
 * The class name states the guarantee: this audit row survives the caller's rollback.
 */
@Service
public class OrderAuditWriter {

    private final OrderAuditRepository auditRepo;

    public OrderAuditWriter(OrderAuditRepository auditRepo) { this.auditRepo = auditRepo; }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordAttempt(PlaceOrderCommand cmd) {
        auditRepo.insertAttempt(cmd.customerId(), cmd.sku(), cmd.units());
    }
}
```

```java
@Service
public class OrderService {

    private final OrderAuditWriter auditWriter;    // injected -> this is the PROXY
    // ... other collaborators

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        auditWriter.recordAttempt(cmd);            // goes THROUGH the proxy. Works.
        inventory.reserve(cmd.sku(), cmd.units());
        wallet.debit(cmd.customerId(), cmd.totalMinorUnits());
        var paymentId = payments.authorize(cmd);
        return orders.insert(cmd, paymentId);
    }
}
```

The rule this teaches, and the one to carry into Topics 54–55:

> **A transaction boundary is a design concept. If two pieces of work need different
> transactional guarantees, they belong in different objects.** The proxy limitation
> is not arbitrary — it is the container forcing you to make the boundary explicit in
> your object graph instead of hiding it in an annotation on a private-ish method.

**One real cost, stated honestly:** `REQUIRES_NEW` takes a **second connection from
the pool while still holding the first**. At 400 rps with a HikariCP pool of 20, that
is a deadlock generator. The right production answer here is probably not
`REQUIRES_NEW` at all — it is writing the audit row *before* opening the placement
transaction, or publishing an event consumed after commit. Topic 55 makes that
argument in full. The proxy fix makes your code do what it says; the design question
is separate and still open.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the self-invocation trap itself

**Wrong:**

```java
@Service
public class InventoryService {
    public void reserveAll(List<Reservation> reservations) {
        reservations.forEach(this::reserveOne);       // self-call
    }

    @Transactional
    public void reserveOne(Reservation r) { repo.decrement(r.sku(), r.units()); }
}
```

**Exact symptom:** a partial reservation persists after a failure. Concretely: five
reservations, the third throws, and reservations one and two are **still in the
database**. No exception is logged beyond the original one. `SHOW TRANSACTION` -style
investigation shows the writes were auto-committed one at a time.

A second, sharper symptom for confirming the diagnosis: put
`TransactionSynchronizationManager.isActualTransactionActive()` at the top of
`reserveOne` and log it. It prints `false`.

**Root cause:** `this::reserveOne` is a method reference bound to the **target**
instance, not the proxy. `@Transactional` is implemented by the proxy. The proxy is
not in the call path.

**Fix:** move `reserveOne` to a collaborator bean, or move the annotation to
`reserveAll` if one transaction for all five is the correct semantics. (It usually
is. Ask what the boundary should be before reaching for a mechanism.)

---

### Trap 2 — a `final` method silently gets no advice

**Wrong:**

```java
@Service
public class PricingService {

    @Cacheable("product-prices")
    public final long priceOf(long productId) {       // 'final' for "safety"
        return repo.priceOf(productId);
    }
}
```

**Exact symptom:** nothing at startup. The application runs. Every call hits Postgres.
`GET /products` p99 sits at 180 ms instead of the 12 ms you expected, and the cache
metrics show **zero hits and zero misses** — not a low hit rate, *no activity at
all*. A cache with no misses either is never called or is not wired.

**Root cause:** CGLIB generates a subclass. A `final` method cannot be overridden — the
JVM rejects such a class file at verification. So CGLIB skips it, and the subclass
simply inherits your original implementation. No override means no interceptor means
no cache.

Spring's `CglibAopProxy` *may* log an INFO-level line noting a final method it could
not proxy. It is easy to miss, it is not an error, and I would not build a diagnosis
around it. Turn on `logging.level.org.springframework.aop=DEBUG` to raise your odds,
then verify by behaviour.

**Fix:** remove `final`. If you genuinely want the method non-overridable, make the
*class* the boundary a different way — but note that a `final` class cannot be
CGLIB-proxied at all, which at least fails loudly.

**The general form of this trap, worth memorising as a set:**

| Modifier | CGLIB | JDK proxy | Fails loudly? |
|---|---|---|---|
| `public` | advised | advised **if on an interface** | — |
| `protected` | overridable, but `@Transactional` metadata is only read from `public` methods in proxy mode | not on the interface, so invisible | **no** |
| package-private | advised only if the proxy lands in the same package | invisible | **no** |
| `private` | not inherited — nothing to override | invisible | **no** |
| `static` | not virtually dispatched | invisible | **no** |
| `final` method | cannot be overridden — **skipped** | n/a (interface methods are never final) | **no** |
| `final` class | cannot be subclassed | fine, if it implements an interface | **yes** — `AopConfigException` at startup |

Five of those seven rows fail silently. That table is the reason this topic exists.

---

### Trap 3 — `private @Transactional`

**Wrong:**

```java
@Service
public class WalletService {
    public void settle(long walletId) { applyLedgerEntries(walletId); }

    @Transactional
    private void applyLedgerEntries(long walletId) { ... }
}
```

**Exact symptom:** identical to Trap 1 — partial writes persist, no error. Plus, in
most IDEs, a greyed-out or warning-marked annotation that everyone has learned to
ignore.

**Root cause:** two independent reasons, either of which is sufficient.
(a) A `private` method is not inherited, so the CGLIB subclass has nothing to
override. (b) `AnnotationTransactionAttributeSource` only reads transaction metadata
from `public` methods when operating in proxy mode, so even a `protected` method would
be skipped.

**Fix:** make the method `public` on a collaborator bean. If it is private because it
is an implementation detail, that is a signal that the *transaction boundary* is in
the wrong place, not that the method needs a visibility change.

---

### Trap 4 — casting the bean to its concrete class

**Wrong:**

```java
@Autowired private PaymentGateway gateway;   // interface type

void reconcile() {
    var adyen = (AdyenPaymentGateway) gateway;   // "I know it's the Adyen one"
    adyen.downloadSettlementFile();
}
```

**Exact symptom, under `spring.aop.proxy-target-class=false` (JDK proxies):**

```
java.lang.ClassCastException: class jdk.proxy2.$Proxy63 cannot be cast to
  class com.orderflow.payments.adyen.AdyenPaymentGateway
```

**Exact symptom under Boot's default (CGLIB):** it *works*, because the CGLIB proxy
really is a subclass of `AdyenPaymentGateway`. So the bug is invisible until someone
sets `proxy-target-class=false`, or adds an interface-only proxy somewhere, or
upgrades a library. Then it breaks in an environment-specific way.

**Root cause:** a JDK proxy extends `java.lang.reflect.Proxy` and can only implement
interfaces. Java's single inheritance makes the cast impossible.

**Fix:** put `downloadSettlementFile()` on the interface, or introduce a second
interface (`SettlementSource`) and inject that. If you truly need the concrete type
for a framework reason, `AopProxyUtils.ultimateTargetClass(bean)` gives you the class,
and `((Advised) bean).getTargetSource().getTarget()` gives you the instance — but
reaching for either in application code is almost always a design smell.

---

### Trap 5 — a side-effecting constructor on a proxied bean

**Wrong:**

```java
@Service
public class MetricsRegistrar {
    public MetricsRegistrar(MeterRegistry registry) {
        registry.gauge("orderflow.inventory.reserved", this, MetricsRegistrar::reserved);
        ACTIVE_SERVICES.add(this);          // static registry
    }
}
```

**Exact symptom:** the gauge reports `0` forever, or a duplicate-registration warning
from Micrometer, or `ACTIVE_SERVICES` contains two entries where you expected one.
Under Micrometer specifically you get a gauge bound to an object whose fields are all
`null`, which reports `NaN` or `0` and never changes.

**Root cause:** if Objenesis is unavailable or fails, CGLIB instantiates the proxy
subclass by calling its constructor, which calls `super(...)` — your constructor —
a second time, on a *different* object whose fields Spring never populates. That
second object escapes into the registry via `this`.

More generally: **`this` escaping from a constructor is a bug in plain Java too**
(Topic 17 — unsafe publication). Proxying makes it worse by giving `this` two possible
identities.

**Fix:** move registration into `@PostConstruct` or an `ApplicationRunner`. Those run
on the target instance, exactly once, after fields are populated.

**How to check whether it is actually happening to you:** put a
`System.out.println("ctor " + System.identityHashCode(this))` in the constructor and
count the lines at startup. Two lines for one bean means the Objenesis path was not
taken.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM and will not print output and
call it captured. What follows is the exact source, the exact command, what to look
for, and how to read every result you might get.

### Setup

```bash
mkdir -p ~/java-lab/40 && cd ~/java-lab/40
java --version           # expect 21 or 25

curl https://start.spring.io/starter.zip \
  -d dependencies=web,jdbc,h2 \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=proxy-lab \
  -d type=maven-project -o proxy-lab.zip && unzip proxy-lab.zip -d proxy-lab
cd proxy-lab
```

Add `src/main/resources/schema.sql`:

```sql
create table if not exists order_audit (
  id     identity primary key,
  ref    varchar(64) not null,
  at_ts  timestamp default current_timestamp
);
```

and `src/main/resources/application.properties`:

```properties
spring.datasource.url=jdbc:h2:mem:orderflow;DB_CLOSE_DELAY=-1
spring.sql.init.mode=always
```

> H2 in memory, not Postgres, deliberately: this drill is about the proxy, and a
> containerised database is noise. `orderflow` moves to Postgres at Topic 47.

### Proof 1 — the bean is not your class

Use `ProxyDemoService` and `ProxyProbe` from Example 1.

```bash
./mvnw spring-boot:run
```

**What to look for:** the five printed lines.

| What you see | What it means |
|---|---|
| `class name = com.orderflow.lab.ProxyDemoService$$SpringCGLIB$$0` | The default on Boot 2.x–4.x. A runtime-generated **subclass**. You are holding a CGLIB proxy. |
| `class name = ...$$EnhancerBySpringCGLIB$$...` | Same thing on Spring Framework < 5.3. The marker string was renamed; the mechanism is identical. |
| `class name` contains `$Proxy` (e.g. `jdk.proxy2.$Proxy63`) | A **JDK dynamic proxy**. Something set `proxy-target-class=false`, or you are outside Boot's auto-config. |
| `class name = com.orderflow.lab.ProxyDemoService` (no marker) | **No proxy at all.** Either no advice applies to this bean, or you have no `@EnableTransactionManagement`/starter on the classpath. If you expected advice, this is your bug. |
| `isCglibProxy = true`, `isJdkDynamicProxy = false` | Confirms the reading above programmatically. Prefer these over string-matching the class name in real code. |
| `ultimateTarget = com.orderflow.lab.ProxyDemoService` | The real class underneath. This is what you want for logging, metric tags, and any `switch` on type. |

**Also print the module**, because I told you not to trust the JDK proxy package name:

```java
System.out.println("module = " + service.getClass().getModule());
System.out.println("simple = " + service.getClass().getSimpleName());
```

The simple name is the stable marker. `$ProxyN` → JDK. `...$$SpringCGLIB$$N` → CGLIB.

### Proof 2 — force JDK proxies and watch the class name change

Give `ProxyDemoService` an interface:

```java
public interface ProxyDemo { String outer(); String inner(); }

@Service
public class ProxyDemoService implements ProxyDemo { /* as before */ }
```

Inject it **by the interface** in the probe (`ProxyDemo service`), then:

```bash
./mvnw spring-boot:run                                    # CGLIB (Boot default)
./mvnw spring-boot:run -Dspring-boot.run.arguments=--spring.aop.proxy-target-class=false
```

| What you see | What it means |
|---|---|
| Run 1: `$$SpringCGLIB$$`; run 2: `$Proxy` | Expected. You have directly observed Boot's default and overridden it. **This is the single most useful diagnostic command in this topic.** |
| Both runs show CGLIB | You are still injecting the **concrete class** somewhere. A JDK proxy cannot satisfy a concrete-class injection point, so Spring falls back to CGLIB. Search for `ProxyDemoService` as a field/parameter type. |
| Run 2 fails with `NoSuchBeanDefinitionException` or a `ClassCastException` | Something else in the app casts the bean to its concrete class — Trap 4. You just found a latent bug. |

You can also set it as a JVM system property, which is useful when you cannot edit the
config of a running service:

```bash
java -Dspring.aop.proxy-target-class=true -jar target/proxy-lab-0.0.1-SNAPSHOT.jar
```

### Proof 3 — see the proxy being created

```properties
logging.level.org.springframework.aop=DEBUG
logging.level.org.springframework.transaction.interceptor=TRACE
```

```bash
./mvnw spring-boot:run 2>&1 | grep -iE "proxy|advis|final"
```

| What you see | What it means |
|---|---|
| Lines about creating an implicit proxy for a bean, naming your class | Confirms *which* beans got wrapped. Beans not listed got no proxy — if you expected advice on one of them, that is the bug. |
| A line mentioning a method being skipped because it is `final` | You have caught Trap 2 in the log. Treat this as a lucky bonus, not a reliable check. |
| Nothing at all | Your logger name or level is wrong, or no proxying is happening. Check that a starter that enables AOP is on the classpath. |
| With transaction TRACE: `Getting transaction for [...]` naming your method | The proxy really is intercepting. **Absence of this line for a method you annotated is the direct proof of the self-invocation trap.** |

### Proof 4 — programmatic assertions you can keep

These belong in a test, not just a scratch run:

```java
import static org.assertj.core.api.Assertions.assertThat;
import org.springframework.aop.support.AopUtils;
import org.springframework.aop.framework.AopProxyUtils;

@SpringBootTest
class ProxyAssertions {

    @Autowired ProxyDemoService service;

    @Test
    void the_bean_is_proxied() {
        assertThat(AopUtils.isAopProxy(service)).isTrue();
        assertThat(AopProxyUtils.ultimateTargetClass(service))
                .isEqualTo(ProxyDemoService.class);
    }
}
```

**Why this is worth keeping:** it is a regression test for "someone made this class
`final`" and "someone removed the annotation that caused proxying". Both are silent
failures otherwise. This is the cheapest insurance in the topic.

### Proof 5 — dump the generated CGLIB bytecode and read it

```bash
mkdir -p /tmp/cglib-dump
./mvnw spring-boot:run \
  -Dspring-boot.run.jvmArguments="-Dcglib.debugLocation=/tmp/cglib-dump"
ls -R /tmp/cglib-dump
javap -p /tmp/cglib-dump/com/orderflow/lab/ProxyDemoService\$\$SpringCGLIB\$\$0.class
```

**What to look for** in the `javap` output:

- The class declaration: `extends com.orderflow.lab.ProxyDemoService`. **That word
  `extends` is the whole CGLIB story on one line.**
- An override of `outer()` and an override of `inner()`.
- Fields named `CGLIB$CALLBACK_0` and similar.
- **Any method you marked `final` is absent from the override list.** That absence is
  the mechanical proof of Trap 2.

| What you see | What it means |
|---|---|
| The dump directory is populated | Confirmed. You are looking at a class that did not exist before the JVM started. |
| Empty directory | The system property name depends on the CGLIB build Spring ships. Fall back to `-Djdk.proxy.debug` for JDK proxies, or simply use `javap` on the class object via a reflection dump — the class-name proof from Proof 1 is sufficient evidence on its own. |

> **Honest note:** `cglib.debugLocation` is a property of the CGLIB code Spring
> repackages, and I am not certain it is still honoured on every Spring 7.x build.
> Spring has discussed replacing CGLIB with ByteBuddy for years. If the dump is empty,
> do not conclude anything about proxying — check
> `unzip -l ~/.m2/repository/org/springframework/spring-core/*/spring-core-*.jar | grep -ci cglib`
> to see which generator your build actually ships, and rely on Proof 1 instead.

### Proof 6 — count constructor invocations

Add to `ProxyDemoService`:

```java
public ProxyDemoService() {
    System.out.println("CTOR identity=" + System.identityHashCode(this));
}
```

```bash
./mvnw spring-boot:run 2>&1 | grep "CTOR identity"
```

| What you see | What it means |
|---|---|
| One line | Objenesis allocated the proxy without running a constructor. The normal, healthy case. |
| Two lines with **different** identity hashes | The Objenesis path was not taken; the proxy subclass's constructor ran too. Any side effect in your constructor just happened twice, on an object whose fields are empty. |
| Two lines with the **same** hash | Not a proxying effect. You have two `@Bean` definitions or a duplicated component scan — Topic 36. |

---

## Failure drill

**Mandatory.** Do not read the fixes until you have produced the failure yourself and
written down what you saw. The point is not the knowledge; it is the memory of the
row that should not have been there.

### The scenario

A `@Transactional` method is called from a sibling method in the same class. It writes
a row and then throws. The transaction "rolls back". The row is still in the database.

### Setup

`src/main/java/com/orderflow/lab/OrderAuditService.java`:

```java
package com.orderflow.lab;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Service
public class OrderAuditService {

    private final JdbcTemplate jdbc;

    public OrderAuditService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    /** Entry point. Called from outside, so THIS call goes through the proxy. */
    public void placeOrder(String ref) {
        recordAndFail(ref);                       // <-- self-invocation
    }

    /** Annotated. Called from placeOrder(), so the proxy is bypassed. */
    @Transactional
    public void recordAndFail(String ref) {
        System.out.println("  txActive inside recordAndFail = "
                + TransactionSynchronizationManager.isActualTransactionActive());
        jdbc.update("insert into order_audit(ref) values (?)", ref);
        throw new IllegalStateException("payment declined for " + ref);
    }

    public int countRows(String ref) {
        return jdbc.queryForObject(
                "select count(*) from order_audit where ref = ?", Integer.class, ref);
    }
}
```

`src/main/java/com/orderflow/lab/DrillRunner.java`:

```java
package com.orderflow.lab;

import org.springframework.aop.framework.AopProxyUtils;
import org.springframework.aop.support.AopUtils;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class DrillRunner implements CommandLineRunner {

    private final OrderAuditService audit;

    public DrillRunner(OrderAuditService audit) { this.audit = audit; }

    @Override
    public void run(String... args) {
        System.out.println("=== PROXY IDENTITY ===");
        System.out.println("  getClass()        = " + audit.getClass().getName());
        System.out.println("  isAopProxy        = " + AopUtils.isAopProxy(audit));
        System.out.println("  isCglibProxy      = " + AopUtils.isCglibProxy(audit));
        System.out.println("  isJdkDynamicProxy = " + AopUtils.isJdkDynamicProxy(audit));
        System.out.println("  ultimateTarget    = "
                + AopProxyUtils.ultimateTargetClass(audit).getName());

        System.out.println("=== RUN A: self-invocation ===");
        try { audit.placeOrder("SELF-CALL"); }
        catch (RuntimeException e) { System.out.println("  caught: " + e.getMessage()); }
        System.out.println("  rows for SELF-CALL = " + audit.countRows("SELF-CALL"));

        System.out.println("=== RUN B: through the proxy ===");
        try { audit.recordAndFail("VIA-PROXY"); }
        catch (RuntimeException e) { System.out.println("  caught: " + e.getMessage()); }
        System.out.println("  rows for VIA-PROXY = " + audit.countRows("VIA-PROXY"));
    }
}
```

### Commands

```bash
./mvnw spring-boot:run
```

Then, with transaction tracing on, to see the boundary decisions:

```bash
./mvnw spring-boot:run \
  -Dspring-boot.run.arguments="--logging.level.org.springframework.transaction.interceptor=TRACE"
```

### What to capture

Write these five things down before reading further:

1. The exact value of `getClass()`.
2. `txActive inside recordAndFail` for **RUN A**.
3. `rows for SELF-CALL`.
4. `txActive inside recordAndFail` for **RUN B**.
5. `rows for VIA-PROXY`.

### How to read it

| What you see | What it means |
|---|---|
| `getClass()` contains `$$SpringCGLIB$$` | You hold a runtime-generated subclass. The advice lives here, not on your class. |
| RUN A: `txActive = false`, `rows for SELF-CALL = 1` | **The drill has fired.** The insert auto-committed because no transaction was ever started. The row survived the exception. This is the row that should not exist. |
| RUN B: `txActive = true`, `rows for VIA-PROXY = 0` | The control case. The same method, reached through the proxy, is transactional and rolls back correctly. |
| RUN A and RUN B both give 0 rows | Your `placeOrder` is not doing a self-call — check you are calling `recordAndFail(ref)` and not `this.audit.recordAndFail(ref)` or similar. Or something is weaving with AspectJ. |
| RUN A and RUN B both give 1 row | No transaction manager is active at all. Confirm `spring-boot-starter-jdbc` is present and that `@EnableTransactionManagement` is auto-configured (it is, via `TransactionAutoConfiguration`). |
| With TRACE on: `Getting transaction for [...recordAndFail]` appears for RUN B only | **This is the cleanest possible evidence.** The interceptor logged that it started a transaction exactly once, for the call that went through the proxy. |

### Now fix it — all four ways, then a recommendation

#### Fix 1 — self-injection

```java
@Service
public class OrderAuditService {

    private final JdbcTemplate jdbc;
    private final OrderAuditService self;          // the PROXY, not 'this'

    public OrderAuditService(JdbcTemplate jdbc, @Lazy OrderAuditService self) {
        this.jdbc = jdbc;
        this.self = self;
    }

    public void placeOrder(String ref) {
        self.recordAndFail(ref);                   // goes through the proxy
    }
}
```

`@Lazy` is required: without it this is a constructor-level circular dependency and
fails at startup with `BeanCurrentlyInCreationException` (Topic 39, Trap 3). With it,
Spring injects a lazy-resolving proxy that hands you the real bean — which is itself
the AOP proxy.

**Verdict:** it works. It is also the single most confusing three lines a future
reader will meet. A field named `self` on a class is a strong signal that the class is
doing two jobs.

#### Fix 2 — deferred lookup via `ObjectProvider` or `ApplicationContext`

```java
@Service
public class OrderAuditService {

    private final JdbcTemplate jdbc;
    private final ObjectProvider<OrderAuditService> selfProvider;

    public OrderAuditService(JdbcTemplate jdbc,
                             ObjectProvider<OrderAuditService> selfProvider) {
        this.jdbc = jdbc;
        this.selfProvider = selfProvider;
    }

    public void placeOrder(String ref) {
        selfProvider.getObject().recordAndFail(ref);
    }
}
```

No `@Lazy` needed, because `ObjectProvider` is a handle and resolves nothing at
construction time. There is also a variant using `AopContext.currentProxy()`:

```java
// requires @EnableAspectJAutoProxy(exposeProxy = true)
((OrderAuditService) AopContext.currentProxy()).recordAndFail(ref);
```

**Verdict on both:** they work, and they make the intent slightly more visible than
`self`. `AopContext.currentProxy()` additionally couples your business code to Spring
AOP internals and needs a global flag switched on. I would not merge it.

#### Fix 3 — extract a collaborator bean  ← **the one to actually pick**

```java
@Service
public class OrderAuditWriter {
    private final JdbcTemplate jdbc;
    public OrderAuditWriter(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @Transactional
    public void recordAndFail(String ref) {
        jdbc.update("insert into order_audit(ref) values (?)", ref);
        throw new IllegalStateException("payment declined for " + ref);
    }
}

@Service
public class OrderAuditService {
    private final OrderAuditWriter writer;      // injected -> the proxy
    public OrderAuditService(OrderAuditWriter writer) { this.writer = writer; }

    public void placeOrder(String ref) { writer.recordAndFail(ref); }
}
```

**Verdict: this is the recommendation.** Not because the others fail, but because it
is the only one that makes the *design* honest. A transaction boundary is a
contract about atomicity. Contracts belong on objects. The other three fixes all
amount to "route my call back out through the wrapper", which works but leaves the
next reader with no signal that a boundary exists at all.

There is a second, practical reason: Fixes 1 and 2 are invisible to a reviewer.
Extracting a bean shows up in the diff as a new file with a name that states the
guarantee.

#### Fix 4 — AspectJ load-time weaving

The only fix that makes `this.method()` genuinely advised, because it stops using
proxies entirely.

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-aspects</artifactId>
</dependency>
```

```java
@Configuration
@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)
@EnableLoadTimeWeaving
public class WeavingConfig { }
```

```bash
java -javaagent:/path/to/spring-instrument-<version>.jar -jar target/proxy-lab.jar
```

AspectJ rewrites the **bytecode of your class itself** as it is loaded, injecting the
advice into the method body. There is no wrapper, so `this.recordAndFail()` is
advised. This is the closest Java gets to what a TypeScript method decorator does.

**Verdict: correct, and almost never worth it.** The costs are real: a `-javaagent`
on every JVM including your IDE, your tests and your container image; a `META-INF/`
`aop.xml`; class-load-time cost at startup; a debugging experience where the code you
step through is not the code you wrote; and a whole second AOP system for your team to
learn. Compile-time weaving via the AspectJ Maven plugin removes the agent but adds
build complexity and a compiler that is not `javac`.

Choose it when you have a genuine cross-cutting requirement that proxies cannot
express — advising `private` methods, field access, or object construction — and you
have decided the operational cost is worth it. Not to avoid extracting a class.

### What the fix proves

The fix is not the point. **The comparison is the point.**

Run A wrote a row that should not exist. Run B did not. Same method. Same annotation.
Same database. The only variable was **which object the call went through**. That is
the mechanical statement at the top of this document, demonstrated on data you can
`select` and see.

Carry one sentence out of this drill: *the annotation is not on the method, it is on
the wrapper.*

---

## Measurement

You will eventually be asked "how expensive are Spring proxies?" Here is how to answer
it honestly.

### First: why your instinct is wrong

The obvious approach:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    service.compute(i);
}
System.out.println((System.nanoTime() - start) / 10_000_000 + " ns/op");
```

Four independent reasons this lies, and you cannot tell which one is lying:

1. **Dead-code elimination.** You never use the return value. C2 proves the call has
   no observable effect and deletes the whole loop body. You measure an empty loop.
2. **Constant folding.** `i` is derived from a loop counter the JIT can reason about;
   if `compute` is small and pure, the result may be computed once or folded away.
3. **On-stack replacement.** The loop starts interpreted, gets compiled *while
   running*, and is replaced mid-flight. Your average blends interpreted, C1 and C2
   execution in a ratio that depends on the loop count you happened to pick.
4. **Cold JIT and profile pollution.** The first thousand iterations run interpreted.
   And if you benchmark the direct call first in the same JVM, the call site becomes
   monomorphic; benchmarking the proxied call afterwards in the same JVM pollutes the
   profile in the other direction. Both numbers are wrong, in opposite directions.

This is Topic 77's entire subject. Forward-reference it, and treat any blog post
quoting proxy overhead from a `nanoTime` loop as fiction.

### The correct shape: a JMH harness

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.lang.reflect.Proxy;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)                                    // 3 forks: separate JVMs, no profile pollution
@State(Scope.Benchmark)
public class ProxyOverheadBenchmark {

    public interface Pricer { long priceOf(long productId); }

    public static class RealPricer implements Pricer {
        public long priceOf(long productId) { return productId * 199L + 7L; }
    }

    private Pricer direct;
    private Pricer jdkProxied;
    private Pricer cglibProxied;

    @Setup(Level.Trial)
    public void setUp() {
        RealPricer target = new RealPricer();

        direct = target;

        // JDK dynamic proxy: a no-op handler, to isolate dispatch cost from advice cost
        jdkProxied = (Pricer) Proxy.newProxyInstance(
                Pricer.class.getClassLoader(),
                new Class<?>[] { Pricer.class },
                (proxy, method, args) -> method.invoke(target, args));

        // CGLIB proxy: build via Spring's ProxyFactory with proxyTargetClass = true
        var pf = new org.springframework.aop.framework.ProxyFactory(target);
        pf.setProxyTargetClass(true);
        pf.addAdvice((org.aopalliance.intercept.MethodInterceptor) inv -> inv.proceed());
        cglibProxied = (Pricer) pf.getProxy();
    }

    @Benchmark public void baselineDirect(Blackhole bh) { bh.consume(direct.priceOf(4471L)); }
    @Benchmark public void viaJdkProxy(Blackhole bh)    { bh.consume(jdkProxied.priceOf(4471L)); }
    @Benchmark public void viaCglibProxy(Blackhole bh)  { bh.consume(cglibProxied.priceOf(4471L)); }
}
```

| Annotation | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | Holds the objects in a field JMH controls, so the JIT cannot constant-fold them away. |
| `Blackhole.consume(...)` | Consumes the result in a way the JIT cannot prove is useless. Defeats dead-code elimination. |
| `@Warmup(iterations = 5)` | Lets C2 compile and reach steady state before anything is recorded. |
| `@Measurement(iterations = 10)` | Enough samples for JMH to report a confidence interval, not a single number. |
| `@Fork(3)` | Three **separate JVMs**. Defeats profile pollution and exposes run-to-run variance. A single fork can be silently wrong. |
| `@BenchmarkMode(AverageTime)` | Average nanoseconds per operation. Use `SampleTime` if you care about the tail. |

Run it:

```bash
./mvnw -Pbench clean verify
java -jar target/benchmarks.jar ProxyOverheadBenchmark -rf json -rff proxy-bench.json
```

**What to look for:** the difference between `baselineDirect` and the two proxied
variants, **and the error margin JMH prints next to each number**. If the margins
overlap, you have measured nothing. Report the interval, never the point estimate.

### The honest interpretation, stated before you run it

Add a fourth benchmark that does a single Postgres `select 1` round trip. Put the
numbers side by side:

| Operation | Order of magnitude |
|---|---|
| Direct virtual call | ~1 ns |
| Proxied call (dispatch + array + one no-op advice) | tens of ns |
| A `@Transactional` boundary (begin/commit, no I/O) | hundreds of ns to low µs |
| **A Postgres round trip on the same host** | **~200,000 ns (0.2 ms)** |
| A Postgres round trip across a network | 1,000,000+ ns |

At `orderflow`'s 400 rps with one database round trip per placement, proxy dispatch is
somewhere around **one ten-thousandth** of the request's latency budget. If you are
optimising it, you have run out of real problems.

There are two situations where it genuinely matters, and you should be able to name
them: an advised method called **millions of times per request** inside a tight loop
(usually a design error — advise the loop, not the body), and a proxy whose *advice*
does real work per call (a `@Cacheable` computing an expensive SpEL key, or an audit
aspect serialising arguments to JSON — Topic 41's problem, not this one).

> **Say this in an interview and it lands:** "The reason to understand proxies is
> correctness, not speed. The overhead is nanoseconds against a millisecond-scale
> database call. The reason I care is that a proxy I do not understand silently
> deletes my transaction boundary."

---

## Practice exercises

### 1 — Easy: build the identity table

Create five beans, each in a different shape, and print
`getClass().getName()` plus the three `AopUtils` predicates for each:

1. A `@Service` with **no** annotations that trigger advice.
2. A `@Service` with a `@Transactional` method, no interface.
3. A `@Service` with a `@Transactional` method that **implements an interface**.
4. The same as (3), run with `--spring.aop.proxy-target-class=false`.
5. A `@Configuration` class, and the same `@Configuration` class with
   `proxyBeanMethods = false`.

Produce a table: bean → class name → proxy type → why. Then answer in one sentence
each: why does (1) differ from (2), and why does (5) differ between its two variants?

### 2 — Medium: the audit (combines Topics 01–39)

This class contains **six** defects. Four are from this topic; two are from earlier
topics. For each: name the topic, state the *observable* symptom in production, and
write the fix.

```java
package com.orderflow.inventory;

@Service
public final class InventoryService {

    @Autowired private InventoryRepository repo;

    private Map<String, Integer> reservedBySku = new HashMap<>();

    public InventoryService() {
        InventoryRegistry.register(this);
    }

    public void reserveAll(List<Reservation> reservations) {
        for (Reservation r : reservations) {
            this.reserveOne(r);
        }
    }

    @Transactional
    private void reserveOne(Reservation r) {
        repo.decrement(r.sku(), r.units());
        Integer current = reservedBySku.get(r.sku());
        reservedBySku.put(r.sku(), current == null ? r.units() : current + r.units());
    }

    @Cacheable("stock")
    public final Integer availableStock(String sku) {
        return repo.available(sku);
    }
}
```

Hints in the order to think about them: one defect makes the application **fail at
startup** — find that one first, because it hides the others. One is a Topic 01
unboxing NPE. One is a Topic 38 shared-mutable-singleton problem. Two are silent
no-advice traps from the modifier table in this document.

### 3 — Hard: production simulation — prove the boundary on `orderflow`

**Part A — reproduce.** Build the Example 2 `OrderService` with `writeAuditRow` as a
self-called `REQUIRES_NEW` method, backed by H2. Write an integration test that places
100 orders of which 20 fail (simulate a declined payment by throwing from
`PaymentService`). Assert on the audit row count. Record what you get.

**Part B — instrument.** Without changing any behaviour, add:
- a `@SpringBootTest` assertion that `orderService` is an AOP proxy and its
  `ultimateTargetClass` is `OrderService`;
- a log line inside `writeAuditRow` printing
  `TransactionSynchronizationManager.isActualTransactionActive()` and
  `getCurrentTransactionName()`.

Explain what the transaction *name* tells you that the boolean does not.

**Part C — fix, three ways, and measure the diff.** Implement Fix 1 (self-injection),
Fix 2 (`ObjectProvider`) and Fix 3 (extracted `OrderAuditWriter`). All three must make
the Part A test pass. Then produce a table comparing them on: lines changed, files
changed, how obvious the transaction boundary is to a reviewer who has not read this
document, and what happens if someone later removes the `@Transactional`.

**Part D — the connection-pool consequence.** Set `spring.datasource.hikari.maximum-`
`pool-size=5`. Run the Part A test at concurrency 10 with the `REQUIRES_NEW` fix in
place. Record what happens and why. (You are previewing Topic 55. Write down your
prediction *before* running it, then compare.)

**Part E — argue against yourself.** You recommended extracting `OrderAuditWriter`.
Make the strongest possible case that self-injection is the better choice for this
specific codebase. Then say what would have to be true about the team or the code for
that case to win.

---

## Interview questions

### Q1 — "Your `@Transactional` annotation did nothing. Walk me through why."

**Mid-level answer:** "Probably self-invocation — you called the method from within
the same class, so the proxy was bypassed. You need to call it from another bean."

**Senior answer:** "Self-invocation is the first thing I'd check, but I'd work through
a list, because there are at least six ways it can silently do nothing. In order of
how often I've actually seen them: (1) self-invocation — `this.method()` is a direct
virtual dispatch on the target, and the advice lives on the proxy wrapping it;
(2) the method is `private` or `protected` — proxy-mode transaction metadata is only
read from `public` methods, and a `private` method isn't inherited so CGLIB has
nothing to override; (3) the method is `final` — CGLIB can't override it, and it
skips it with no error; (4) the exception thrown was **checked**, and the default
rollback rule is unchecked exceptions only, so it committed; (5) the bean was
constructed with `new` somewhere — a hand-written `@Bean` method, for instance — so no
proxy exists at all; (6) it was called from `@PostConstruct`, where the bean's own
proxy doesn't exist yet, because proxies are created in
`postProcessAfterInitialization` which runs after `@PostConstruct`.

To diagnose it in under a minute I'd print `service.getClass().getName()` and look for
`$$SpringCGLIB$$` or `$Proxy`, and turn on
`logging.level.org.springframework.transaction.interceptor=TRACE` — if the
`Getting transaction for` line doesn't appear for that method, the interceptor never
ran, and now I know it's a routing problem rather than a rollback-rule problem. The
fix I'd merge is extracting a collaborator bean, because a transaction boundary is a
design concept and it should be visible as an object."

**What separates them:** the mid answer knows the headline. The senior answer has a
**ranked diagnostic list**, distinguishes "the interceptor never ran" from "it ran and
decided to commit", names the specific log switch, and treats the fix as a design
question. The `@PostConstruct` item in particular signals lifecycle knowledge that
almost nobody volunteers.

**Follow-up:** "You said checked exceptions commit by default. Why would anyone design
it that way?" (Because `@Transactional` follows EJB's convention, where a checked
exception is a modelled business outcome and an unchecked one is a failure. Whether
you agree is the interesting part of the answer.)

---

### Q2 — "When does Spring pick CGLIB over a JDK proxy?"

**Mid-level answer:** "If the class implements an interface it uses a JDK proxy,
otherwise CGLIB."

**Senior answer:** "That's the plain-Framework rule, but it's not what happens in a
Spring Boot app. `DefaultAopProxyFactory` picks JDK when the target implements at
least one non-internal interface and `proxyTargetClass` is false; CGLIB when
`proxyTargetClass` is true, or when there are no interfaces, or when the target is
already a JDK proxy or a lambda. The catch is that Spring Boot sets
`spring.aop.proxy-target-class=true` by default — since Boot 2.0 — so in practice
**you get CGLIB even with interfaces**. Boot made that choice because JDK proxies
break anything that casts the bean to its concrete class, and enough of the ecosystem
does that.

There's also a per-annotation override: `@EnableTransactionManagement(proxyTarget-`
`Class = true)`, `@EnableCaching(proxyTargetClass = true)`, and so on. Those matter
when you have mixed configuration.

I verify rather than reason about it: print `getClass().getName()` and look for
`$$SpringCGLIB$$` versus a `$Proxy` simple name, or call `AopUtils.isCglibProxy`. And
`spring.aop.proxy-target-class=false` is my go-to switch when I suspect something is
casting to a concrete class — it turns a latent bug into a `ClassCastException` at
startup."

**What separates them:** knowing Boot overrides the Framework default is the whole
answer. It is also the difference between someone who read the Spring docs and someone
who has debugged a Boot application.

**Follow-up:** "What breaks if you switch a service from CGLIB to JDK proxies?"
Concrete-class injection points, casts to the concrete type, and public methods not
declared on any interface becoming unreachable.

---

### Q3 — "Why can't you advise a `final` method? And what does Spring do about it?"

**Mid-level answer:** "Because CGLIB creates a subclass and you can't override a
`final` method."

**Senior answer:** "Right, and the important half is what Spring does about it:
**nothing**. There's no exception and no failed startup. CGLIB generates the subclass,
skips the `final` methods, and you inherit the original implementation — so your
annotation is metadata that nothing read. There may be an INFO-level line from
`CglibAopProxy`, but it's easy to miss and I wouldn't build a diagnosis on it.

Contrast that with a `final` **class**, which fails loudly at startup with
`AopConfigException: Could not generate CGLIB subclass`. So `final` at class level is
safe-by-failure and `final` at method level is dangerous-by-silence. Worth knowing:
records are implicitly `final`, so a record can never be CGLIB-proxied.

The underlying reason is JVM-level, not a CGLIB limitation: a class file that
overrides a `final` method fails verification. The same silent-skip applies to
`private`, `static`, and cross-package package-private methods. Five of the seven
modifier cases fail silently, which is why I keep a `@SpringBootTest` that asserts
`AopUtils.isAopProxy(bean)` on the beans whose advice actually matters — it's the only
regression test for someone adding `final` in a future refactor."

**What separates them:** "no error, no warning" is the answer. Then generalising to
the full modifier table, and — the part that gets people hired — proposing a *test*
that catches it, because a silent failure needs an active check.

**Follow-up:** "So how would you advise a `private` method if you truly had to?"
AspectJ weaving, and then a good candidate immediately explains why they would push
back on the requirement first.

---

### Q4 — "You have a `@Cacheable` method and a `@Transactional` method on the same bean, and the cache is serving stale data after a rollback. Where do you start?"

**Mid-level answer:** "Maybe the cache TTL is too long, or we need to evict on
update."

**Senior answer:** "Stale-after-rollback is a different failure from stale-after-write,
and TTL doesn't touch it. Both annotations are implemented as interceptors on the same
proxy, and they run in a chain whose order is determined by `@Order` on the
corresponding advisors. If the cache interceptor sits **outside** the transaction
interceptor, the sequence is: cache miss, transaction begins, method runs, transaction
rolls back, and then the cache interceptor stores the value it received before the
rollback was decided. You've cached something that never committed, and it will be
served until the TTL expires.

So I'd start by finding the actual order — `logging.level.org.springframework.aop=`
`DEBUG` shows the advisor chain — and then set the transaction advisor to a lower
order value so it wraps the cache advisor. That's Topic 41's material. The other thing
I'd check is that both annotations are actually being applied at all, because if
`@Cacheable` is on a self-called or `final` method it's doing nothing, and 'stale data'
would then be a misdiagnosis of 'no cache at all'."

**What separates them:** recognising this as an **ordering** problem in an interceptor
chain rather than a cache-configuration problem, and — the senior move — checking
whether the annotations are even in the call path before theorising about their
interaction.

**Follow-up:** "How do you make the cache write happen only after commit?" Ordering is
one answer; `TransactionSynchronizationManager.registerSynchronization` with an
`afterCommit` callback is the more robust one, and Topic 110 goes further.

---

### Q5 — "What is the runtime cost of a Spring proxy, and how would you find out?"

**Mid-level answer:** "It adds some overhead but it's small. I'd time it in a loop."

**Senior answer:** "A timing loop would give me a number and I wouldn't be able to
tell you what it measured. `System.nanoTime()` around a loop measures dead-code
elimination, constant folding, on-stack replacement and cold-JIT state in unknown
proportions — if the result isn't consumed, C2 deletes the loop body entirely and you
measure nothing. I'd use JMH: `@State(Scope.Benchmark)` so the objects can't be
folded, `Blackhole.consume` so the results can't be eliminated, warmup iterations to
reach steady state, and at least three forks so profile pollution between the direct
and proxied variants shows up as run-to-run variance instead of hiding in a single
number. And I'd report the confidence interval, not a point estimate.

But I'd also push back on the question. The mechanical cost is one extra virtual
dispatch, an `Object[]` allocation for the arguments with boxing for primitives, a
`MethodInvocation` object, and a stack frame per advice — tens of nanoseconds. A
Postgres round trip is around 200 microseconds. That's four orders of magnitude. In
our order service, proxy dispatch is roughly a ten-thousandth of the request budget.
The reason to understand proxies is correctness — a proxy I don't understand silently
deletes my transaction boundary — not throughput. If proxy overhead were genuinely my
bottleneck I'd want to know why an advised method is being called millions of times
per request, because that's usually the real bug."

**What separates them:** naming the four specific JIT effects, insisting on forks and
intervals, and then **reframing the question** — the strongest available signal that
someone has done real performance work rather than read about it.

**Follow-up:** "When *would* proxy overhead matter?" A tight loop over an advised
method, or advice that itself does real work per call — serialising arguments,
evaluating an expensive SpEL cache key.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A JDK proxy `extends java.lang.reflect.Proxy`. Given Java's single inheritance,
   derive from that one fact *both* of the JDK proxy's practical limitations — without
   recalling them from the text.

2. Spring creates proxies in `postProcessAfterInitialization`. Suppose it created them
   in `postProcessBeforeInitialization` instead. Name one thing that would improve and
   one thing that would break.

3. A CGLIB proxy inherits your class's field *declarations* but Spring never populates
   them. Predict what `proxiedBean.someInjectedField` returns if you access it
   directly. Then explain why this almost never bites in practice, and name the one
   situation where it does.

4. TypeScript method decorators rewrite the prototype, so self-calls *are* intercepted.
   Java annotations cannot. Could Java have chosen the TypeScript approach? What would
   it have needed at the language level, and what would it have cost?

5. Boot defaults to `proxy-target-class=true`, meaning CGLIB even for interface-backed
   beans. Argue that this was the wrong default. Then argue it was right. Which side is
   stronger, and does your answer change for a library versus an application?

6. AspectJ load-time weaving makes self-invocation work. Given that, why is Spring AOP
   — the strictly weaker mechanism — still the default that ninety-plus percent of
   Spring applications use?

7. You add a `@SpringBootTest` asserting `AopUtils.isAopProxy(orderService)`. What
   class of future regression does that catch, and what class does it completely miss?
   Design a second assertion that closes part of the gap.

---

## Quick reference card

### The two proxy types

| | JDK dynamic proxy | CGLIB proxy |
|---|---|---|
| What it is | class implementing your **interfaces** | runtime-generated **subclass** |
| Requires | at least one interface | a non-`final` class |
| Extends | `java.lang.reflect.Proxy` | **your class** |
| Class name marker | simple name starts with `$Proxy` | `...$$SpringCGLIB$$N` (5.3+), `$$EnhancerBySpringCGLIB$$` (older) |
| Castable to your class | **no** — `ClassCastException` | yes |
| Advises non-interface methods | no | yes (if overridable) |
| Constructor re-run risk | none | yes, if Objenesis is unavailable |
| Boot default | only with `spring.aop.proxy-target-class=false` | **yes, the default** |

### What can and cannot be advised

```
CAN:      public non-final instance methods, called from ANOTHER bean
CANNOT:   final methods           -> silently skipped
CANNOT:   private methods         -> not inherited, and metadata not read
CANNOT:   static methods          -> not virtually dispatched
CANNOT:   any self-call           -> this.method() bypasses the proxy
CANNOT:   final classes           -> AopConfigException at startup (LOUD)
CANNOT:   records                 -> implicitly final
CANNOT:   calls made from @PostConstruct -> the proxy does not exist yet
```

### Diagnostic commands

```java
service.getClass().getName()                       // $$SpringCGLIB$$ or $Proxy?
service.getClass().getSimpleName()                 // the STABLE marker
AopUtils.isAopProxy(service)
AopUtils.isCglibProxy(service)
AopUtils.isJdkDynamicProxy(service)
AopProxyUtils.ultimateTargetClass(service)         // the real class underneath
TransactionSynchronizationManager.isActualTransactionActive()
TransactionSynchronizationManager.getCurrentTransactionName()
```

```properties
spring.aop.proxy-target-class=false                # force JDK proxies where possible
logging.level.org.springframework.aop=DEBUG
logging.level.org.springframework.transaction.interceptor=TRACE
```

```bash
-Dspring.aop.proxy-target-class=true               # as a JVM system property
-Dcglib.debugLocation=/tmp/cglib-dump              # dump generated bytecode (verify it works)
```

### The four fixes for self-invocation

| Fix | Works | Readable | Recommended |
|---|---|---|---|
| Self-injection (`@Lazy` on the constructor param) | yes | poor | no |
| `ObjectProvider` / `ApplicationContext` lookup | yes | fair | only where extraction is impossible |
| `AopContext.currentProxy()` (needs `exposeProxy=true`) | yes | poor | no |
| **Extract a collaborator bean** | yes | **good** | **yes** |
| AspectJ load-time weaving | yes, and fixes it globally | n/a | only for requirements proxies cannot express |

### Gotchas checklist

- [ ] The bean is not your class. Print `getClass().getName()` before theorising.
- [ ] `this.method()` never sees advice. Ever.
- [ ] `final` method → silently unadvised. `final` class → loud startup failure.
- [ ] `private`/`static` → silently unadvised.
- [ ] `@PostConstruct` self-calls get no advice — use `ApplicationRunner`.
- [ ] Proxied bean constructors must be side-effect-free.
- [ ] Never cast an injected bean to its concrete class.
- [ ] Never read a field directly off a proxied bean.
- [ ] Keep a test asserting `AopUtils.isAopProxy` on beans whose advice matters.
- [ ] Overhead is nanoseconds. Correctness is the reason you learned this.

---

## When would I use this at work?

**1. Diagnosing a silent data-integrity bug.**
Reconciliation shows orders with a payment row and no inventory reservation. Nobody
deployed a change. You print `orderService.getClass().getName()`, see the CGLIB
marker, then turn on transaction TRACE and notice the `Getting transaction for` line
never appears for the method in question. Ninety seconds to a diagnosis that
otherwise takes a day of reading business logic. This is the highest-value thing in
this document.

**2. Reviewing a pull request that adds `final`.**
Someone adds `final` to a `@Cacheable` method "for immutability". You catch it in
review. Without this topic, the symptom is a cache with a 0% hit rate discovered
weeks later during a load test, and by then nobody connects it to that diff.

**3. Making an architectural decision about AspectJ.**
A team proposes load-time weaving to "make `@Transactional` just work everywhere". You
can state precisely what it buys (advice on self-calls, `private` methods, field
access) and precisely what it costs (a `-javaagent` on every JVM including CI and
IDEs, class-load-time overhead, a second AOP model, and stepping through bytecode you
did not write) — and propose extracting three collaborator beans instead. Being able
to argue that trade with specifics rather than preference is a principal-track
conversation, and it starts here.

---

## Connected topics

**Prerequisites:**
- **17 — Immutability and safe publication**: why `this` escaping a constructor is
  already a bug before proxies double the number of possible identities for `this`.
- **35 — `ApplicationContext` and two-phase startup**: definitions are registered
  before instantiation, which is what makes post-processing — and therefore proxying —
  possible at all.
- **37 — Bean lifecycle**: proxies are created in
  `BeanPostProcessor.postProcessAfterInitialization`. That single ordering fact
  explains why `@PostConstruct` self-calls get no advice.
- **38 — Bean scopes**: scoped proxies are the *same mechanism* applied to a different
  problem. You have already been using proxies without naming them.
- **39 — Injection styles**: `@Lazy` injects a proxy; circular references expose the
  raw target. Both are this topic wearing a different hat.
- **01 — Boxing**: argument arrays box primitives on every proxied call.

**This unlocks:**
- **41 — AOP**: the interceptor chain the proxy walks, and `@Order` as the thing that
  sequences it. Read it next; it is the direct continuation.
- **45 — Bean Validation**: `@Validated` on a service is proxy-based. Everything here
  applies.
- **47 — Spring Data JPA**: repository interfaces are JDK proxies over
  `SimpleJpaRepository`. Now you know what that sentence means.
- **49 — Hibernate lazy associations**: a lazy `@ManyToOne` is a **different** kind of
  runtime-generated subclass proxy, from Hibernate rather than Spring. Same idea, and
  `entity.getClass()` will surprise you the same way.
- **54–55 — `@Transactional`**: the single largest consumer of this topic. Every trap
  in this document recurs there with money attached.
- **57 — Spring Security**: `@PreAuthorize` is proxy-based. A self-called
  `@PreAuthorize` method performs no authorization check — which is a security bug,
  not a correctness bug.
- **77 — JMH**: the correct way to measure the overhead the Measurement section
  sketched.
- **78 — Profiling**: where proxy frames show up in a flame graph, and how to tell
  advice cost from dispatch cost.
- **110 — Spring Cache**: `@Cacheable` inherits every constraint here, plus the
  ordering problem Topic 41 drills.
- **111 — Resilience4j**: `@CircuitBreaker` and `@Retry` are proxy-based annotations
  with the same self-invocation hole.
- **131 — Native images / AOT**: proxies are runtime bytecode generation, which is
  exactly what GraalVM native image cannot do. Spring's AOT engine pre-generates proxy
  classes at build time — a topic that is incomprehensible without this one.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0. Two things in
this document are deliberately hedged rather than asserted: the exact package of a
JDK-generated proxy class on JDK 25, and whether Spring 7.0's CGLIB path still
dispatches through CGLIB's `FastClass`/`MethodProxy` or through plain reflection.
Neither affects correctness, and each has a command in the Hands-on section that
settles it on your machine in under a minute. Everything else here — the mechanical
statement, the modifier table, the lifecycle position, and the drill — has been stable
across Spring 4, 5, 6 and 7, and will still be true when you next debug it at 2am.*
