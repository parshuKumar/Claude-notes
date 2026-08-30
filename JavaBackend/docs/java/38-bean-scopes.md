# 38 — Bean Scopes vs Nest Provider Scopes; Scoped Proxies

## Phase: 4 — Spring Core
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: `orderflow` gains a **request-scoped correlation and tenant context** on the order API — one object per HTTP request, carrying the correlation id and the calling merchant's tenant id, readable from the service layer without threading it through every method signature.

---

## ELI5 anchor

Imagine a hotel.

The **front desk** is one desk. It exists before you arrive and it will still be there
after you leave. Everyone who walks in talks to the same desk. That is a **singleton**
bean: one instance, created at startup, shared by everybody.

A **luggage locker** is assigned to you when you check in and emptied when you check out.
Your locker holds your things. My locker holds mine. Neither of us can see inside the
other's. That is a **request-scoped** bean: one instance per HTTP request, thrown away at
the end of the request.

A **complimentary pen** is handed to you fresh every time you ask for one. Nobody keeps
track of it. If you leave it behind, nobody comes to collect it. That is a **prototype**
bean: a new instance every time it is asked for, and — the part that surprises people —
the hotel does *not* clean it up afterwards.

Now the problem this whole topic exists to solve.

The front desk needs to be able to look in "the current guest's locker". But the front desk
was built before any guest existed. It cannot be handed a locker at construction time,
because at construction time there is no guest.

So Spring gives the front desk **an intercom handset bolted to the wall**. The handset is
always the same handset. The front desk holds it forever. But when you pick it up and
speak, it connects to *whoever is currently checked in on this line*. The handset holds no
information at all; it only knows how to find the right locker at the moment you use it.

That handset is a **scoped proxy**. It is the answer to "how does a thing that lives
forever talk to a thing that lives for 40 milliseconds".

And here is the bug that this topic is really about: if the front desk picks up the
handset **once**, writes down the guest's name on a sticky note, and then reads the sticky
note for the next ten thousand guests — nothing crashes. There is no error. Every guest
after the first is simply treated as the first guest. In `orderflow` terms: every customer
sees the first customer's tenant.

---

## The bridge from what you know

### Nest provider scopes ≈ Spring bean scopes: PARTIAL, and the mechanism is completely different

You already know this problem. NestJS has exactly the same three scopes, with almost the
same names:

```ts
// NestJS
@Injectable()                                    // DEFAULT — singleton
export class ProductCatalogService {}

@Injectable({ scope: Scope.REQUEST })            // one per request
export class RequestContextService {
  constructor(@Inject(REQUEST) private readonly req: Request) {}
  get tenantId() { return this.req.headers['x-tenant-id'] as string; }
}

@Injectable({ scope: Scope.TRANSIENT })          // new instance per injection site
export class ScratchBuffer {}
```

| Nest scope | Spring scope | Same? |
|---|---|---|
| `Scope.DEFAULT` | `singleton` | Yes, in effect |
| `Scope.REQUEST` | `request` | Same intent, **different machinery** |
| `Scope.TRANSIENT` | `prototype` | Close, but not identical — see below |
| — | `session`, `application`, `websocket` | No Nest equivalent out of the box |

So far so familiar. Now the part that actually matters.

### The difference: Nest **bubbles scope UP**. Spring injects a **proxy**.

This is the single sentence to carry out of this section.

**In Nest**, if `OrderService` injects a `REQUEST`-scoped provider, then `OrderService`
*itself becomes request-scoped*. And if `OrderController` injects `OrderService`, the
controller becomes request-scoped too. The scope propagates upward through the injection
chain until it reaches the top. Nest documents this explicitly as "bubbling up".

```
RequestContextService  (REQUEST)
        ^ injected by
OrderService           -> becomes REQUEST, automatically
        ^ injected by
OrderController        -> becomes REQUEST, automatically
```

Consequence: on **every HTTP request**, Nest constructs a fresh `OrderController`, a fresh
`OrderService`, and a fresh `RequestContextService`. Three constructions per request, plus
anything else in that subtree. Nest is explicit that this has a performance cost and tells
you to use request scope sparingly.

**In Spring**, scope does not bubble. `OrderPlacementService` stays a singleton — created
once, at startup, and never again. Instead, Spring injects into it a **scoped proxy**: a
single stateless object that, on every method call, looks up "the request-scoped instance
belonging to the request currently running on this thread" and forwards the call there.

```
RequestContext (request scope)   <-- N live instances, one per in-flight request
        ^ resolved per call by
RequestContext$$SpringCGLIB$$0   <-- ONE proxy object, injected once, holds no state
        ^ injected once at startup into
OrderPlacementService (singleton)  <-- ONE instance, forever
```

### Why the difference matters — cost profiles, honestly

Neither approach is free, and they are expensive in different places. State both, because
an interviewer will ask which is better and the answer is "they trade different things".

| | NestJS bubbling | Spring scoped proxy |
|---|---|---|
| **Objects constructed per request** | the whole injection subtree above the request-scoped provider — potentially dozens | exactly one: the request-scoped bean itself |
| **Per-call overhead** | none once constructed; calls are plain method calls | one proxy dispatch + one map lookup + one `ThreadLocal` read, **per method call** |
| **Where the cost lands** | request setup (allocation, GC pressure — Topic 68) | steady-state call path (indirection — Topic 40) |
| **Correctness** | correct by construction; you cannot get a stale instance | correct **if** you never dereference and cache the result |
| **What can silently break** | nothing scope-related; you get a fresh graph | caching the resolved value in a singleton field (Trap 1), `final` methods (Trap 5), off-thread access (Trap 4) |
| **Visible in code review?** | yes — `scope: Scope.REQUEST` is on the provider and its effect is documented | **no** — a singleton looks like a singleton; the proxy is invisible in the source |
| **Startup cost** | unaffected | unaffected — singletons are still built once |

**Verdict: PARTIAL analogue.** The vocabulary transfers perfectly. The mechanism does not,
and the mechanism is where the bugs live. Nest's approach cannot produce a stale-instance
bug, because there is no long-lived holder to go stale. Spring's approach cannot produce a
per-request-allocation problem, because nothing is reallocated. You are trading an
allocation problem for a staleness problem, and this document is about the staleness
problem.

### `Scope.TRANSIENT` vs `prototype` — close, not equal

Nest's `TRANSIENT` means "each consumer that injects this gets its own dedicated
instance" — the instance is per *injection site*, and it lives as long as its consumer.

Spring's `prototype` means "every time this bean is **requested from the container**, build
a new one". If a singleton injects a prototype, the request happens **once**, at singleton
construction, and the singleton holds that one instance forever (Trap 2).

The practical overlap is large. The practical difference is that Spring's prototype does
*not* give you a new instance per call unless you ask the container again per call — with
`ObjectProvider`, `@Lookup`, or a factory. That distinction is Trap 2 and it catches
people coming from Nest specifically, because `TRANSIENT` reads like "new every time".

### What has NO analogue in your world

Three things.

**1. The scoped proxy object itself.** Nest has no equivalent because it does not need one.
There is no runtime-generated subclass sitting between a singleton and a per-request
object. If you want the mental picture, it is closest to a JavaScript `Proxy` whose `get`
trap resolves the target from `AsyncLocalStorage` on every property access — but nobody
writes that, and Nest certainly does not.

**2. `singleton` means *per `ApplicationContext`*, not per JVM.** A classical singleton in
any language means "one per process". A Spring singleton means "one per container". If your
application starts two contexts — a parent and child context, or a test suite with several
cached contexts (Topic 60) — you have two instances of that "singleton". This surprises
people who reach for a singleton bean as a global counter.

**3. Session and application scope.** Nest has no built-in session-scoped provider. Spring
has `session` (per `HttpSession`) and `application` (per `ServletContext`). You will
almost certainly not use them in a stateless API, but you need to know they exist, and you
need to know that `session` scope is why some Spring applications need sticky sessions in
the load balancer.

### The `@Inject(REQUEST)` habit you must unlearn

In Nest you routinely do this:

```ts
constructor(@Inject(REQUEST) private readonly req: Request) {}
```

There is no equivalent worth using in Spring. You *can* inject `HttpServletRequest` into a
singleton — Spring will hand you a scoped-proxy-like wrapper that resolves per thread — but
it drags the servlet API into your service layer and makes the class untestable without a
mock request.

The Spring-idiomatic move is to define your **own** small request-scoped domain object
holding exactly the fields you need (`correlationId`, `tenantId`, `customerId`), populate
it once in a filter, and inject that. Your service layer then depends on a
`RequestContext` you own, not on `jakarta.servlet.http.HttpServletRequest`. That is
Example 2.

---

## What is this?

A **scope** is the rule the container uses to answer one question: *when someone asks for
this bean, do I return an existing instance or build a new one, and how long does the one I
build survive?*

A bean **definition** (Topic 36) is the recipe. A scope decides how many cakes get baked
from that recipe and who gets which one.

### The six built-in scopes

| Scope name | Constant | One instance per | Available in | Destruction callbacks run? |
|---|---|---|---|---|
| `singleton` | `ConfigurableBeanFactory.SCOPE_SINGLETON` | `ApplicationContext` | everywhere | **Yes** — `@PreDestroy` on context close |
| `prototype` | `ConfigurableBeanFactory.SCOPE_PROTOTYPE` | *request to the container* | everywhere | **No** — see Trap 3 |
| `request` | `WebApplicationContext.SCOPE_REQUEST` | HTTP request | web contexts only | **Yes** — at request completion |
| `session` | `WebApplicationContext.SCOPE_SESSION` | `HttpSession` | web contexts only | **Yes** — at session invalidation |
| `application` | `WebApplicationContext.SCOPE_APPLICATION` | `ServletContext` | web contexts only | **Yes** — at context shutdown |
| `websocket` | `"websocket"` | WebSocket session | WebSocket-enabled apps | Yes |

Two rows there are worth stopping on.

**`singleton` is the default.** Every `@Component`, `@Service`, `@Repository`,
`@Controller` and `@Bean` you have written so far is a singleton. That is why the whole
container can be built at startup and why a bean's fields are shared by every concurrent
request — which makes **singleton bean fields a concurrency hazard**, and is the reason
service classes should be stateless apart from their injected collaborators.

**`prototype` does not get destruction callbacks.** This is not an oversight; it is a
documented design decision. Spring builds a prototype, injects it, applies post-processors,
hands it to you, and then **forgets it exists**. It keeps no reference, so it cannot call
`@PreDestroy` later. A prototype holding a resource is your problem to close. Trap 3.

### The scope mismatch problem, stated precisely

> **A bean can only hold a direct reference to a bean whose lifetime is at least as long as
> its own.**

A singleton lives for the life of the context. A request-scoped bean lives for ~40 ms. If
the singleton holds a direct reference to a request-scoped instance, then 40 ms later that
reference points at an object belonging to a request that finished long ago — and the
singleton keeps using it for every subsequent request.

Injecting **downward** in lifetime (singleton into request-scoped) is always fine. The
short-lived bean can hold the long-lived one; the long-lived one outlives it.

Injecting **upward** (request-scoped into singleton) is the problem, and there are exactly
two correct answers:

1. **A scoped proxy** — inject a stand-in that resolves the real instance per call.
2. **A deferred lookup** — inject `ObjectProvider<T>` and call `getObject()` at the moment
   you need it.

Everything else is a bug waiting for a production incident.

### What a scoped proxy actually is, mechanically

When you write `proxyMode = ScopedProxyMode.TARGET_CLASS`, Spring does the following at
bean-definition time (Topic 36's registration phase, before anything is instantiated):

1. It **renames** your bean definition to `scopedTarget.requestContext` and marks it with
   the real scope (`request`).
2. It registers a **second** bean definition under the original name `requestContext`,
   whose class is `ScopedProxyFactoryBean` and whose scope is **singleton**.
3. That factory bean produces an AOP proxy — a CGLIB subclass for `TARGET_CLASS`, a JDK
   interface proxy for `INTERFACES` — backed by a `SimpleBeanTargetSource` pointing at
   `scopedTarget.requestContext`.

So the object injected into your singleton is a singleton. It has no fields of its own that
matter. Every method call on it does this:

```
yourSingleton.ctx.tenantId()
  -> CGLIB override of tenantId() on RequestContext$$SpringCGLIB$$0
     -> TargetSource.getTarget()
        -> beanFactory.getBean("scopedTarget.requestContext")
           -> RequestScope.get(name, objectFactory)
              -> RequestContextHolder.currentRequestAttributes()   <-- ThreadLocal read
                 -> attributes.getAttribute(name, SCOPE_REQUEST)
                    -> present?  return it
                    -> absent?   objectFactory.getObject()  (build it now),
                                 store it on the request,
                                 register its destruction callback
     -> invoke tenantId() on the REAL instance
```

Four consequences fall straight out of that chain, and each becomes a trap later:

1. **`RequestContextHolder` is a `ThreadLocal`.** Request scope therefore has exactly the
   same thread-boundary problem as `SecurityContextHolder` from Topic 56 and the MDC from
   Topic 120. Off the request thread, there is no bound request, and you get an exception.
2. **The proxy is a CGLIB subclass**, so everything in Topic 40 applies: `final` classes
   cannot be proxied, `final` methods are not overridden and therefore not resolved, records
   are implicitly `final`, and the proxy's own inherited fields are empty.
3. **The lookup happens per method call**, not per injection. That is what makes it correct
   — and what makes caching the result a bug.
4. **The instance is created lazily**, on first access within a request. A request that
   never touches the bean never builds it.

### `[BOOT 3.x DELTA]`

Bean scopes and scoped proxies are one of the most stable corners of Spring. The
`@Scope`/`proxyMode` mechanism, the `@RequestScope`/`@SessionScope`/`@ApplicationScope`
composed annotations, and `ScopedProxyFactoryBean` behave the same on Boot 3.x
(Framework 6.x) and Boot 4.1 (Framework 7.0). Two things to be aware of when you meet
older code:

- **`javax.*` → `jakarta.*` (Boot 2.x → 3.x).** `RequestContextListener`,
  `RequestContextFilter` and `HttpServletRequest` moved package. A Boot 2.x example on the
  internet that imports `javax.servlet.http.HttpServletRequest` will not compile on 3.x or
  4.x. This is Topic 127.
- **`jakarta.inject.Provider<T>`** (the JSR-330 `Provider`) is supported alongside Spring's
  `ObjectProvider`, but on Boot 3+ the import is `jakarta.inject`, not `javax.inject`, and
  it requires the `jakarta.inject-api` dependency, which Boot does not pull in for you.
  Prefer `ObjectProvider` — it is on the classpath already and has a richer API.

I am **not** aware of scope-related behaviour changes in Framework 7.0, and I am not going
to assert that there are none. If you need certainty for a migration, the settling command
is to diff the reference documentation section "Bean Scopes" between the two versions and
run `mvn dependency:tree | grep spring-context` to confirm which Framework version your BOM
actually resolved.

---

## Why does it matter?

**1. The failure mode is shaped like a data leak, not like a crash.**

This is the reason this topic is in the curriculum at all. A misconfigured scope does not
throw. It serves customer B's request using customer A's tenant id. In a multi-tenant
`orderflow`, that means merchant B reading merchant A's orders. Your monitoring shows a
100% success rate. Your error budget is untouched. You find out from a support ticket, or
from a regulator.

**2. It is the mechanism behind every "how do I get the current X" question.**

Current user (Topic 56), current tenant, correlation id (Topic 120), trace context
(Topic 119), the database read-replica routing key. All of them are the same problem —
per-request state reachable from a long-lived object — and Spring offers exactly three
mechanisms: a scoped proxy, a `ThreadLocal` holder, or an explicit parameter. Knowing all
three and when each is right is a senior-level distinction.

**3. Scoped proxies inherit every constraint of Topic 40, silently.**

A `final` method on a request-scoped class returns the proxy's empty field values instead of
the real instance's. No error. This is the single least-known consequence of scoped proxies
and it is worth being the person in the room who knows it.

**4. It is where the Nest instinct actively misleads.**

Coming from Nest, "make it request-scoped" is a complete solution — the framework handles
the propagation. In Spring it is half a solution, and the missing half fails silently. Your
existing instinct is not wrong, it is *incomplete*, which is the most dangerous kind.

---

## Syntax breakdown

Only the constructs that are new. Everything else you have from Topics 36–39.

### `@Scope` on a component

```java
package com.orderflow.orders;

import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)     // "prototype"
public class OrderTotalCalculator { }
```

| Piece | Meaning |
|---|---|
| `@Scope(...)` | Sets the scope of **this bean definition**. Absent = `singleton`. |
| `ConfigurableBeanFactory.SCOPE_PROTOTYPE` | The constant for `"prototype"`. Prefer the constant over the string literal — a typo in a string scope name is a startup failure with a confusing message. |
| Where it goes | On the `@Component`-annotated class, **or** on a `@Bean` method in a `@Configuration` class. Both work; the `@Bean` method form scopes the bean the method produces. |

### `@Scope` with `proxyMode` — the form that matters

```java
package com.orderflow.web;

import org.springframework.context.annotation.Scope;
import org.springframework.context.annotation.ScopedProxyMode;
import org.springframework.stereotype.Component;
import org.springframework.web.context.WebApplicationContext;

@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { }
```

`ScopedProxyMode` has four values:

| Value | What Spring injects | When to use |
|---|---|---|
| `NO` | **the real instance**, resolved once at injection time | Default. Correct only when the injecting bean's scope is the same or shorter. |
| `TARGET_CLASS` | a **CGLIB subclass** proxy | The one you want, almost always. Works whether or not the class implements an interface. |
| `INTERFACES` | a **JDK dynamic proxy** implementing the class's interfaces | Only when you inject strictly by interface type and want no CGLIB. Injecting by concrete type then fails — Trap 5. |
| `DEFAULT` | usually equivalent to `NO` | Effectively "not specified". Do not write it. |

> **`proxyMode` defaults to `NO`.** Writing `@Scope("request")` with no `proxyMode` and
> injecting it into a singleton is the bug in Trap 1. This is the single most important
> line in this section.

### The composed annotations — use these instead

```java
import org.springframework.web.context.annotation.RequestScope;
import org.springframework.web.context.annotation.SessionScope;
import org.springframework.web.context.annotation.ApplicationScope;

@Component
@RequestScope                       // == @Scope(SCOPE_REQUEST, proxyMode = TARGET_CLASS)
public class RequestContext { }
```

`@RequestScope`, `@SessionScope` and `@ApplicationScope` are meta-annotated with
`proxyMode = ScopedProxyMode.TARGET_CLASS` already. **They are the safe default and you
should use them in preference to spelling out `@Scope`.** You cannot forget the proxy mode
if the annotation sets it for you.

There is no `@PrototypeScope` composed annotation, and no proxy mode is applied to
prototypes by default. That asymmetry is deliberate: prototype has no "current instance" to
resolve, so a proxy would have nothing sensible to look up.

### `ObjectProvider<T>` — the deferred lookup

```java
package com.orderflow.orders;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.stereotype.Service;

@Service
public class OrderPricingService {

    private final ObjectProvider<OrderTotalCalculator> calculators;

    public OrderPricingService(ObjectProvider<OrderTotalCalculator> calculators) {
        this.calculators = calculators;                 // a HANDLE, resolves nothing yet
    }

    public long total(PlaceOrderCommand cmd) {
        OrderTotalCalculator calc = calculators.getObject();   // a NEW prototype, per call
        return calc.compute(cmd);
    }
}
```

| Method | What it does |
|---|---|
| `getObject()` | Ask the container now. For a prototype, builds a new one. Throws if there is no such bean. |
| `getIfAvailable()` | Returns `null` if no bean is defined. Useful for optional collaborators. |
| `getIfUnique()` | Returns `null` if zero **or more than one** candidate exists. |
| `stream()` / `orderedStream()` | All candidates, the second in `@Order` order. This is how you inject "every implementation of `PaymentGateway`". |

`ObjectProvider` is injected as a singleton handle. It resolves nothing at construction
time, so it never triggers the "scope is not active" startup failure and never creates a
circular dependency.

### `@Lookup` — method injection

```java
@Service
public abstract class OrderPricingService {

    public long total(PlaceOrderCommand cmd) {
        return createCalculator().compute(cmd);
    }

    @Lookup
    protected abstract OrderTotalCalculator createCalculator();   // Spring implements this
}
```

Spring CGLIB-subclasses the class and implements the abstract method as a container lookup.
It works, and it exists mainly for pre-`ObjectProvider` code. **Prefer `ObjectProvider`:**
`@Lookup` forces the class to be non-`final` and abstract-ish, is invisible to a reader who
does not know the annotation, and inherits every Topic 40 proxy caveat.

### Registering a custom scope

Rarely needed, but knowing the shape tells you scopes are not magic:

```java
public interface Scope {
    Object get(String name, ObjectFactory<?> objectFactory);
    Object remove(String name);
    void registerDestructionCallback(String name, Runnable callback);
    Object resolveContextualObject(String key);
    String getConversationId();
}
```

```java
@Bean
static CustomScopeConfigurer scopeConfigurer() {
    CustomScopeConfigurer c = new CustomScopeConfigurer();
    c.addScope("batch-job", new BatchJobScope());
    return c;
}
```

`RequestScope` is a ~40-line implementation of that interface backed by
`RequestContextHolder`. That is genuinely all a scope is: a map with a lifetime policy.

Note the `static` on the `@Bean` method — `CustomScopeConfigurer` is a
`BeanFactoryPostProcessor`, and those must be instantiated before ordinary beans (Topic 35).

---

## Example 1 — minimal

Three classes. One singleton, one request-scoped bean, one controller. The point is to make
the proxy and the per-request resolution **visible**.

```java
package com.orderflow.lab;

import java.util.concurrent.atomic.AtomicLong;
import org.springframework.stereotype.Component;
import org.springframework.web.context.annotation.RequestScope;

/**
 * One instance per HTTP request. The instanceId lets us PROVE that, because
 * identityHashCode on the injected proxy would be identical every time.
 */
@Component
@RequestScope
public class RequestScopedCounter {

    private static final AtomicLong SEQ = new AtomicLong();

    private final long instanceId = SEQ.incrementAndGet();
    private int hits;

    public long instanceId() { return instanceId; }

    public int recordHit() { return ++hits; }
}
```

```java
package com.orderflow.lab;

import org.springframework.stereotype.Service;

/** A plain singleton. Created once, at startup. */
@Service
public class ScopeProbeService {

    private final RequestScopedCounter counter;      // <-- this is the PROXY

    public ScopeProbeService(RequestScopedCounter counter) {
        this.counter = counter;
    }

    public String describe() {
        return "injectedClass=" + counter.getClass().getName()
             + " instanceId="   + counter.instanceId()
             + " hitsThisRequest=" + counter.recordHit();
    }
}
```

```java
package com.orderflow.lab;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ScopeProbeController {

    private final ScopeProbeService probe;

    public ScopeProbeController(ScopeProbeService probe) { this.probe = probe; }

    @GetMapping("/lab/scope")
    public String scope() {
        return probe.describe() + " | second call: " + probe.describe();
    }
}
```

What this demonstrates, in one endpoint:

- `injectedClass` is **not** `com.orderflow.lab.RequestScopedCounter`. It carries a
  `$$SpringCGLIB$$` marker. The singleton is holding a proxy.
- `instanceId` is the **same** for both calls within one request, and **different** on the
  next request. That is per-request resolution, proven by a value the proxy cannot fake.
- `hitsThisRequest` goes `1` then `2` within a request, and resets to `1` on the next
  request. State does not leak across requests.

`SEQ` is `static`, so it survives across requests and gives you a monotonically increasing
instance number. Static state on a bean is normally a smell; here it is deliberate
instrumentation.

> Note the deliberate absence of `@PostConstruct` in `ScopeProbeService`. Reading
> `counter.instanceId()` from a `@PostConstruct` would resolve the scoped bean at startup,
> where there is no request — the exact mistake behind Trap 1. Topic 37 explains why
> `@PostConstruct` runs at the wrong moment for anything request-shaped.

---

## Example 2 — production scenario (on the project spine)

### The constraints, stated concretely

`orderflow` is now multi-tenant. Several merchant partners send orders through the same
deployment.

- **1,200 requests/second** at peak across 6 replicas, so ~200 rps per JVM. Order placement
  is 10% of that; the rest is catalogue and order reads.
- **~40 partner tenants.** Every row in `orders`, `order_line`, `payment` and `wallet`
  carries a `tenant_id`. Cross-tenant reads are a contractual breach, not a bug report.
- Every log line, every outbound HTTP call, every Kafka message and every audit row must
  carry a **correlation id**: taken from the inbound `X-Correlation-Id` header if present,
  generated if not, and echoed back on the response.
- The tenant id comes from a **JWT claim** (`tid`), not from a header — a header would be
  client-controlled and therefore forgeable. Topic 57 covers the token; this topic covers
  where the extracted value lives.
- SLO: p99 for `GET /api/orders` under 250 ms. Whatever mechanism we pick has to cost
  effectively nothing per call.

### The request-scoped context bean

```java
package com.orderflow.web;

import java.time.Instant;
import java.util.UUID;
import org.springframework.stereotype.Component;
import org.springframework.web.context.annotation.RequestScope;

/**
 * Per-request ambient context for orderflow.
 *
 * NOT final, NO final methods: it is CGLIB-proxied into singletons (Topic 40).
 * NOT a record, for the same reason — records are implicitly final.
 */
@Component
@RequestScope
public class RequestContext {

    private String correlationId;
    private String tenantId;
    private String customerId;
    private Instant receivedAt;

    /** Called exactly once per request, by RequestContextFilter below. */
    void initialise(String correlationId, String tenantId, String customerId, Instant receivedAt) {
        if (this.correlationId != null) {
            throw new IllegalStateException("RequestContext already initialised");
        }
        this.correlationId = correlationId;
        this.tenantId = tenantId;
        this.customerId = customerId;
        this.receivedAt = receivedAt;
    }

    public String correlationId() { return require(correlationId, "correlationId"); }
    public String tenantId()      { return require(tenantId, "tenantId"); }
    public String customerId()    { return require(customerId, "customerId"); }
    public Instant receivedAt()   { return receivedAt; }

    /** An immutable copy, safe to hand to another thread. See Trap 4. */
    public RequestSnapshot snapshot() {
        return new RequestSnapshot(correlationId, tenantId, customerId, receivedAt);
    }

    private static String require(String v, String field) {
        if (v == null) {
            throw new IllegalStateException(
                "RequestContext." + field + " was read before it was initialised — "
                + "either you are off the request thread, or the filter did not run "
                + "for this path");
        }
        return v;
    }
}
```

```java
package com.orderflow.web;

import java.time.Instant;

/** Immutable. This is what crosses a thread boundary, never RequestContext itself. */
public record RequestSnapshot(String correlationId,
                              String tenantId,
                              String customerId,
                              Instant receivedAt) {}
```

Four deliberate decisions worth defending in a code review:

**1. The class is not `final` and neither is any method.** A `final` method on a
CGLIB-proxied class is not overridden, so calling it through the proxy executes the parent
implementation **against the proxy's own fields**, which Spring never populated. It returns
`null`. Silently. This is Trap 5 and it is the least-known consequence of scoped proxies.

**2. It is a class, not a record.** Records are implicitly `final` and cannot be
CGLIB-proxied at all. That failure at least is loud — `AopConfigException` at startup — but
it removes the option entirely.

**3. Every getter throws if the field is unset**, with a message that names the two real
causes. Compare that to returning `null`: a `null` tenant id propagates into a query
predicate and either returns everything or nothing, and you debug it three layers away.
Fail at the read, name the cause.

**4. `snapshot()` exists from day one.** The moment anyone submits work to an executor, a
`CompletableFuture`, a `@Async` method or a virtual thread, they need an immutable value
object rather than a scope lookup. Providing it up front is cheaper than discovering the
need during an incident. Forward reference: Topic 119 makes the same argument for trace
context, Topic 91 for `CompletableFuture`, Topic 101 for virtual threads.

### Populating it — a filter, not a controller

```java
package com.orderflow.web;

import java.io.IOException;
import java.time.Clock;
import java.time.Instant;
import java.util.UUID;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.core.annotation.Order;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

@Component
@Order(20)      // AFTER Spring Security's chain, so the JWT is already validated
public class RequestContextFilter extends OncePerRequestFilter {

    static final String CORRELATION_HEADER = "X-Correlation-Id";

    private final RequestContext context;      // <-- the scoped PROXY, injected once
    private final Clock clock;

    public RequestContextFilter(RequestContext context, Clock clock) {
        this.context = context;
        this.clock = clock;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {

        String correlationId = header(request, CORRELATION_HEADER);
        if (correlationId == null || correlationId.length() > 64) {
            correlationId = UUID.randomUUID().toString();
        }

        String tenantId = "unknown";
        String customerId = "anonymous";
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getPrincipal() instanceof Jwt jwt) {
            tenantId = jwt.getClaimAsString("tid");
            customerId = jwt.getSubject();
        }

        // Touching the proxy HERE creates the real instance and binds it to this request.
        context.initialise(correlationId, tenantId, customerId, Instant.now(clock));

        response.setHeader(CORRELATION_HEADER, correlationId);
        chain.doFilter(request, response);
    }

    private static String header(HttpServletRequest r, String name) {
        String v = r.getHeader(name);
        return (v == null || v.isBlank()) ? null : v.trim();
    }
}
```

Three things worth noticing:

- **The filter injects the same scoped proxy the services do.** There is no special access
  path. `context.initialise(...)` resolves the instance for *this* request and populates it.
- **`@Order(20)` puts it after Spring Security.** The tenant comes from a validated JWT.
  If this filter ran first, `SecurityContextHolder` would be empty and every request would
  be tenant `unknown`. Ordering here is a correctness property, not a preference. Topic 56
  covers the chain; the exact integer matters only relative to the security filter
  registration order in your application, which you can print at startup.
- **A length cap on the inbound correlation id.** It is client-supplied and it ends up in
  every log line and in an outbound header. An unbounded client-supplied string in a log
  pipeline is a denial-of-service on your log storage, and a `\n` in it is log injection. A
  stricter version would validate it against a UUID pattern and reject anything else.

### Consuming it from the singleton service layer

```java
package com.orderflow.orders;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.orderflow.web.RequestContext;

@Service
public class OrderPlacementService {

    private final OrderRepository orders;
    private final InventoryService inventory;
    private final WalletService wallet;
    private final RequestContext context;     // scoped proxy. Injected ONCE. Never cached.

    public OrderPlacementService(OrderRepository orders,
                                 InventoryService inventory,
                                 WalletService wallet,
                                 RequestContext context) {
        this.orders = orders;
        this.inventory = inventory;
        this.wallet = wallet;
        this.context = context;
    }

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        String tenantId = context.tenantId();          // resolved NOW, for THIS request
        String correlationId = context.correlationId();

        inventory.reserve(tenantId, cmd.sku(), cmd.units());
        wallet.debit(tenantId, cmd.customerId(), cmd.totalMinor());

        Order order = new Order(correlationId, cmd.customerId(), context.receivedAt());
        order.setTenantId(tenantId);
        return orders.save(order).id();
    }
}
```

`OrderPlacementService` is a singleton. One instance, six replicas, 200 rps each. It reads
`context.tenantId()` on every call and gets the right answer every time, because the proxy
resolves per call.

**The rule to internalise, and the one Trap 1 breaks:**

> Read the scoped proxy **where you use the value**. Never assign it to a field, never
> capture it in a lambda that outlives the request, never memoise it "for performance". The
> lookup is a `ThreadLocal` read and a map get. It is not the thing making your p99 slow.

### The honest alternative comparison

A request-scoped bean is *one* of three ways to do this, and I am not going to pretend it
is unconditionally the best. Here is the actual decision:

| Mechanism | What it looks like | Cost | When it wins |
|---|---|---|---|
| **Request-scoped bean + proxy** | inject `RequestContext` | one CGLIB dispatch + `ThreadLocal` read + map get per call | Many consumers, deep call stacks, you want a typed domain object rather than the servlet API |
| **Explicit parameter** | `place(tenantId, cmd)` | zero | Few consumers, shallow stacks. **Always correct, always testable, never surprises anyone.** |
| **A `ThreadLocal` holder you own** | `TenantHolder.current()` | one `ThreadLocal` read | You need it outside the web layer too (a Kafka consumer, a batch job) where request scope does not exist |

**For `orderflow` I would pick the request-scoped bean for the web path**, because forty
call sites across five packages would otherwise all grow two extra parameters, and because
having the container manage the lifetime means the object is genuinely gone at request end
rather than leaking on a pooled thread.

**But note what it does not solve.** The moment `orderflow` gets a Kafka consumer
(Topic 113) or a scheduled settlement job, those code paths have no HTTP request, so
`RequestContext` throws there. At that point the right structure is a small `TenantHolder`
abstraction with two implementations — request-scoped for HTTP, explicitly-set for
consumers — and the service layer depending on the abstraction. That refactor is cheap if
you saw it coming and expensive if you did not. It is exactly the design question Topic 119
forces you to answer for trace context.

### The cost, stated honestly

Per call through the proxy: one virtual dispatch to the CGLIB override, one `ThreadLocal`
get, one `HashMap` get, one reflective or `MethodProxy` invocation on the target. That is
comfortably under a microsecond — call it tens to low hundreds of nanoseconds, the same
order as the Topic 40 numbers.

A Postgres round trip in `orderflow` is 200,000 to 2,000,000 nanoseconds. Order placement
does four of them. **The scoped proxy is not measurable against your SLO**, and any
optimisation that caches the resolved value to avoid it is trading a correctness guarantee
for nothing. If you want the number for your own machine rather than my estimate, Topic 77
tells you why a `System.nanoTime()` loop will not give it to you and what to use instead.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a singleton captures ONE request-scoped instance forever

This is the headline trap. It is the reason the topic exists.

**Wrong:**

```java
@Component
@Scope("request")                   // <-- proxyMode defaults to NO. Nothing warns you.
public class RequestContext { }

@Service
public class OrderPlacementService {
    private final RequestContext context;
    public OrderPlacementService(RequestContext context) { this.context = context; }
}
```

**Exact symptom — and there are two, which is what makes this dangerous.**

*Symptom A — the loud one, which you want.* The application fails to start:

```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name
  'orderPlacementService': Unsatisfied dependency expressed through constructor
  parameter <n>; nested exception is
  java.lang.IllegalStateException: No thread-bound request found: Are you referring to
  request attributes outside of an actual web request, or processing a request outside
  of the originally receiving thread?
```

*illustration of the message shape, not captured output*

Spring tried to build `orderPlacementService` at startup, needed a `request`-scoped bean,
found no active request, and refused. This is the good outcome: you fix it in three
minutes.

*Symptom B — the silent one, which ships to production.* The application **starts
normally**. Every endpoint returns 200. Then:

- Every response carries the **same** `X-Correlation-Id`, and it is not the one the client
  sent.
- Every log line for every request shows the **same** `tenantId`.
- Merchant B's `GET /api/orders` returns merchant A's orders.
- The audit table shows 4.3 million rows in a single day all attributed to one tenant.

Nothing throws. Success rate is 100%.

**Root cause.** With `proxyMode = NO`, Spring resolves the dependency **once**, at the
moment `OrderPlacementService` is constructed, and injects a plain reference to whatever
`RequestScope.get()` returned at that instant.

Symptom A happens when the singleton is built at startup: there is no request, so the
resolution fails loudly.

Symptom B happens when the singleton is built **during a request**, because then there *is*
an active request and the resolution succeeds. Four realistic ways that happens:

1. **`spring.main.lazy-initialization=true`.** Someone enabled it to cut startup time from
   9 s to 3 s. Now beans are constructed on first use — which for a web service means
   inside the first HTTP request. This is the nastiest version, because the change that
   caused it looks completely unrelated to security.
2. **`@Lazy` on the singleton bean** or on the injection point, added by someone
   "fixing" Symptom A without understanding it.
3. **The singleton is obtained from an `ObjectProvider` inside a handler**, so it is created
   on first request.
4. **`@Lazy` on a `@Configuration`-declared `@Bean`** somewhere up the chain.

The mechanism is identical in all four: at construction time a request was in flight, so
the reference resolved, and now it never changes.

> **Honest flag:** I have described the mechanism, which follows directly from
> `RequestScope.get()` returning the current request's instance and plain injection storing
> a plain reference. I have **not** verified the exact `spring.main.lazy-initialization`
> interaction on Boot 4.1 on a running JVM. Proof 3 in the Hands-on section is the settling
> experiment — it takes about four minutes and you should run it rather than trust this
> paragraph.

**Fix — and there are three, in order of preference:**

```java
// 1. BEST — the composed annotation sets proxyMode = TARGET_CLASS for you.
@Component
@RequestScope
public class RequestContext { }
```

```java
// 2. Explicit, when you want the reader to see the mechanism.
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { }
```

```java
// 3. Deferred lookup at the injection point. Verbose, but no proxy caveats at all.
@Service
public class OrderPlacementService {
    private final ObjectProvider<RequestContext> contextProvider;

    public OrderPlacementService(ObjectProvider<RequestContext> contextProvider) {
        this.contextProvider = contextProvider;
    }

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        RequestContext ctx = contextProvider.getObject();   // resolved per call
        ...
    }
}
```

**And the regression barrier, which is the part people skip:**

```java
@SpringBootTest
class ScopedProxyAssertions {

    @Autowired OrderPlacementService service;

    @Test
    void request_context_is_injected_as_a_scoped_proxy() {
        Object injected = ReflectionTestUtils.getField(service, "context");
        assertThat(AopUtils.isCglibProxy(injected))
            .as("RequestContext must be injected as a scoped proxy, "
              + "or every request sees the first request's tenant")
            .isTrue();
    }
}
```

That test costs four lines and catches the exact change — someone dropping
`@RequestScope` for `@Scope("request")` during a refactor — that produces a cross-tenant
data leak. There is no other automated signal for it.

---

### Trap 1b — the scoped proxy is correct, and the singleton caches the value anyway

Worth its own entry because the configuration is *right* and the bug is identical.

**Wrong:**

```java
@Service
public class OrderPlacementService {

    private final RequestContext context;          // correct scoped proxy
    private String tenantId;                       // "cache it, the proxy lookup is slow"

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        if (tenantId == null) {
            tenantId = context.tenantId();         // <-- resolved ONCE, kept FOREVER
        }
        ...
    }
}
```

Or the `@PostConstruct` variant, which is even more common because it looks like clean
initialisation:

```java
    @PostConstruct
    void init() {
        this.tenantId = context.tenantId();        // Topic 37: this runs at STARTUP
    }
```

**Exact symptom:** identical to Symptom B above — every request served with the first
request's tenant. The `@PostConstruct` variant instead usually reproduces Symptom A, because
`@PostConstruct` runs during startup where no request is bound.

**Root cause:** the proxy resolves per **call**. Assigning the result to a singleton field
converts a per-call lookup into a once-ever lookup. The proxy did its job; the field
undid it.

**Fix:** never store a value read through a scoped proxy in a field of a longer-lived bean.
Read it at the point of use. If you genuinely have a hot loop where you want to avoid
repeated lookups, read it into a **local variable** at the top of the method — locals die
with the call, fields do not.

```java
    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        final String tenantId = context.tenantId();     // local. Correct and fast.
        ...
    }
```

---

### Trap 2 — a prototype injected into a singleton is created exactly once

**Wrong:**

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class OrderTotalCalculator {
    private long runningTotal;                       // stateful, on purpose
    public void addLine(long unitMinor, int qty) { runningTotal += unitMinor * qty; }
    public long total() { return runningTotal; }
}

@Service
public class OrderPricingService {
    private final OrderTotalCalculator calculator;   // ONE instance, for the app's lifetime
    public OrderPricingService(OrderTotalCalculator calculator) { this.calculator = calculator; }

    public long price(PlaceOrderCommand cmd) {
        cmd.lines().forEach(l -> calculator.addLine(l.unitMinor(), l.qty()));
        return calculator.total();
    }
}
```

**Exact symptom:** order totals grow monotonically. The first customer is charged
correctly. The second is charged their own total plus the first customer's. By the
thousandth order the total is an absurd number and someone's card is declined for
£4,182,993.11. Under concurrency it is worse and non-deterministic: two threads increment
`runningTotal` on the same object with no synchronisation, so you also get lost updates
(Topic 88).

The direct confirmation: print `System.identityHashCode(calculator)` on each call. It is
**identical every time**. A prototype that is truly per-use would print a different value.

**Root cause:** scope governs *when the container is asked*, not *when the reference is
used*. The singleton asks once, at construction. Spring builds one prototype, injects it,
and never hears about it again. `prototype` does not mean "new per method call"; it means
"new per `getBean()`".

**Fix — three options, and the third is the right one:**

```java
// 1. ObjectProvider — ask the container per call.
private final ObjectProvider<OrderTotalCalculator> calculators;
public long price(PlaceOrderCommand cmd) {
    OrderTotalCalculator calc = calculators.getObject();   // genuinely new each time
    ...
}
```

```java
// 2. @Lookup — method injection. Works. Reads as magic.
@Lookup protected abstract OrderTotalCalculator createCalculator();
```

```java
// 3. BEST — do not make it a bean at all.
public long price(PlaceOrderCommand cmd) {
    var calc = new OrderTotalCalculator();     // it has no dependencies. Just build it.
    cmd.lines().forEach(l -> calc.addLine(l.unitMinor(), l.qty()));
    return calc.total();
}
```

**The judgement to carry:** a prototype bean is almost always a design smell. A class with
per-use mutable state and no injected dependencies is a *value object*, and value objects
are constructed with `new`. Putting it in the container buys you nothing and hands you this
trap. Prototype earns its place only when the object genuinely needs container-injected
collaborators *and* per-use state — which is rare.

---

### Trap 3 — `@PreDestroy` on a prototype never runs

**Wrong:**

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class SettlementFileWriter {

    private final BufferedWriter out;

    public SettlementFileWriter(@Value("${orderflow.settlement.dir}") String dir) throws IOException {
        this.out = Files.newBufferedWriter(Path.of(dir, UUID.randomUUID() + ".csv"));
    }

    @PreDestroy
    void close() throws IOException { out.close(); }     // NEVER CALLED
}
```

**Exact symptom:** the settlement job runs nightly. After eleven days the JVM dies with
`java.io.IOException: Too many open files`. Every unrelated endpoint fails, because the
process cannot open a socket either. `lsof -p <pid> | wc -l` shows thousands of entries,
mostly `.csv` files in the settlement directory, all with zero bytes on disk because
nothing was ever flushed.

**Root cause:** Spring does not retain a reference to prototype instances. It cannot,
without leaking them. Because it holds no reference, it cannot invoke a destruction
callback at any later point. The documented rule is: *for prototypes, Spring hands off the
object and the client code becomes responsible for its lifecycle.*

This asymmetry catches people because `@PostConstruct` **does** run on prototypes. Half the
lifecycle works. That is worse than neither half working.

**Fix — close it yourself, with the language construct designed for this:**

```java
public class SettlementFileWriter implements AutoCloseable {     // NOT a bean at all
    ...
    @Override public void close() throws IOException { out.close(); }
}

// caller
try (var writer = new SettlementFileWriter(dir)) {
    writer.write(rows);
}                                       // closed here, even on exception. Topic 08.
```

If it truly must be a container-managed prototype, wrap it: use `ObjectProvider` to obtain
it and a `try`/`finally` to dispose it, or register a `DisposableBeanAdapter` yourself. Both
are more code than just not making it a bean.

> **Contrast worth memorising:** request-scoped beans **do** get destruction callbacks.
> `RequestScope.registerDestructionCallback` stores the callback on the request attributes
> and `RequestContextHolder`'s cleanup runs it at request completion. So `@PreDestroy` on a
> `@RequestScope` bean fires; on a `prototype` bean it does not.

---

### Trap 4 — reading a request-scoped bean off the request thread

**Wrong:**

```java
@Service
public class OrderPlacementService {

    private final RequestContext context;
    private final AuditPublisher audit;

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        OrderId id = doPlace(cmd);
        audit.publishAsync(cmd, id);        // hands off to a thread pool
        return id;
    }
}

@Service
public class AuditPublisher {
    private final RequestContext context;

    @Async
    public void publishAsync(PlaceOrderCommand cmd, OrderId id) {
        String correlationId = context.correlationId();     // <-- BOOM, on a pool thread
        ...
    }
}
```

**Exact symptom:**

```
java.lang.IllegalStateException: No thread-bound request found: Are you referring to
  request attributes outside of an actual web request, or processing a request outside
  of the originally receiving thread? If you are actually operating within a web request
  and still receive this message, your code is probably running outside of
  DispatcherServlet: In this case, use RequestContextListener or RequestContextFilter
  to expose the current request.
```

*illustration of the message shape, not captured output*

The nasty part: because it happens on an `@Async` thread, the exception goes to
`AsyncUncaughtExceptionHandler` and **not** to your controller. The HTTP response is a
clean 201. The audit row is silently missing. You find it in a reconciliation report weeks
later — the same shape as Topic 40's audit bug.

**Root cause:** `RequestScope` reads `RequestContextHolder.currentRequestAttributes()`,
which is a `ThreadLocal`. `@Async` runs on a `TaskExecutor` thread that was created long
before this request existed and has no bound attributes. This is *exactly* the
`SecurityContextHolder` problem from Topic 56, on a different `ThreadLocal`.

There are in fact three `ThreadLocal`s in this stack that all break at the same boundary,
and you should learn them as one fact:

| Holder | Carries | Topic |
|---|---|---|
| `RequestContextHolder` | request attributes → request-scoped beans | **38 (here)** |
| `SecurityContextHolder` | the `Authentication` | 56 |
| `MDC` (SLF4J) | log context including the correlation id | 120 |
| OpenTelemetry `Context` | the active span | 119 |

Four, actually. All `ThreadLocal`. All silently empty on a pool thread.

**Fix — snapshot at the boundary, do not propagate the scope:**

```java
    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        RequestSnapshot snap = context.snapshot();   // read ON the request thread
        OrderId id = doPlace(cmd);
        audit.publishAsync(snap, cmd, id);           // pass an immutable value
        return id;
    }

    @Async
    public void publishAsync(RequestSnapshot snap, PlaceOrderCommand cmd, OrderId id) {
        // no scope lookup; the values travelled with the task
    }
```

**The alternative you will find on the internet and should mostly not use:**
`RequestContextHolder.setRequestAttributes(attrs, true)` with an inheritable holder, or a
`TaskDecorator` that copies attributes onto the pool thread. Both work. Both are fragile:
the request may have *completed* by the time the async task runs, at which point the
attributes object has been cleaned up and you are reading a corpse. Snapshotting an
immutable record has none of these problems and is trivially testable.

> **A `TaskDecorator` is the right tool for `MDC` and trace context**, where the values are
> genuinely just strings and the copy is safe. It is the wrong tool for a request-scoped
> *bean*, whose lifecycle is tied to a request that may already be over. Topic 119 draws
> the line properly.

---

### Trap 5 — a `final` method on a scoped-proxied class returns `null`

The least-known consequence of scoped proxies, and completely silent.

**Wrong:**

```java
@Component
@RequestScope
public class RequestContext {
    private String tenantId;
    public final String tenantId() { return tenantId; }    // 'final' for "safety"
}
```

**Exact symptom:** `context.tenantId()` returns `null` from every singleton, on every
request — even though the filter definitely called `initialise(...)` and a debugger
attached inside the filter shows the field correctly populated. Downstream, the tenant
predicate in the repository query becomes `tenant_id = null`, which in Postgres matches
**nothing**, so every merchant sees an empty order list and support tickets say "my orders
disappeared".

Or, if the code path instead does `if (tenantId == null) tenantId = "unknown"`, every row
is written with tenant `unknown` and the data is quietly corrupted rather than empty.

**Root cause:** exactly Topic 40. The scoped proxy is a CGLIB subclass. It overrides every
non-`final` method to do the target lookup. It **cannot** override a `final` method — the
JVM rejects such a class file at verification — so CGLIB skips it, and the subclass simply
inherits your implementation. That inherited implementation runs with `this` bound to **the
proxy object**, whose `tenantId` field Spring never populated. It reads `null`.

The general table from Topic 40 applies unchanged to scoped proxies:

| On the scoped class | Behaviour through the proxy | Fails loudly? |
|---|---|---|
| `public` non-`final` method | resolved to the real instance — correct | — |
| `final` method | **not overridden**; runs against the proxy's empty fields | **no** |
| `private` method | not inherited, not reachable from outside anyway | n/a |
| `final` class | cannot be subclassed — `AopConfigException` at startup | **yes** |
| a `record` | implicitly `final` — same startup failure | **yes** |
| direct field access (`ctx.tenantId`) | reads the proxy's empty field | **no** |
| `proxyMode = INTERFACES` + injection by concrete type | `NoSuchBeanDefinitionException` at startup | **yes** |

**Fix:** remove `final` from every method on a scoped-proxied class, do not make it a
record, and never expose public fields. Add a comment on the class saying *why*, because
the next person will want to add `final` back.

```java
/**
 * CGLIB-proxied into singletons (Topic 40). Therefore:
 *   - the class must not be final and must not be a record
 *   - no method may be final
 *   - no field may be read directly from outside
 * Breaking any of these returns null silently.
 */
@Component
@RequestScope
public class RequestContext { }
```

**Related, and worth knowing:** `equals`, `hashCode` and `toString` on a scoped proxy are
also intercepted, so they resolve to the target — but the proxy's *identity* is stable
across requests while the target's is not. Never use a scoped proxy as a `HashMap` key or
put it in a `Set`; you are keying on a stable stand-in for a value that changes per
request, which is Topic 13's mutable-key bug with a new dressing.

---

## Hands-on proof

Every command here is one **you** run. I have no JVM and will not print output and call it
captured. Below is the exact source, the exact command, what to look for, and how to read
every result you might get.

### Setup

```bash
mkdir -p ~/java-lab/38 && cd ~/java-lab/38
java --version           # expect 21 or 25

curl https://start.spring.io/starter.zip \
  -d dependencies=web \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=scope-lab \
  -d type=maven-project -o scope-lab.zip && unzip scope-lab.zip -d scope-lab
cd scope-lab
```

Use `RequestScopedCounter`, `ScopeProbeService` and `ScopeProbeController` from Example 1.

```bash
./mvnw spring-boot:run
```

### Proof 1 — the injected object is not your class

```bash
curl -s localhost:8080/lab/scope
```

**What to look for:** the `injectedClass=` value in the response body.

| What you see | What it means |
|---|---|
| `injectedClass=com.orderflow.lab.RequestScopedCounter$$SpringCGLIB$$0` | Correct. A CGLIB scoped proxy is injected. `@RequestScope` did its job. |
| `injectedClass` contains `$Proxy` | A JDK dynamic proxy — you used `proxyMode = INTERFACES`, or set `spring.aop.proxy-target-class=false`. Fine if you inject by interface only. |
| `injectedClass=com.orderflow.lab.RequestScopedCounter` (no marker) | **No proxy.** You wrote `@Scope("request")` without a `proxyMode`. If the app started at all, you are in Trap 1 Symptom B territory. Go to Proof 3. |
| The app failed to start with `No thread-bound request found` | Trap 1 Symptom A. The loud, good failure. Add `@RequestScope`. |

### Proof 2 — prove the instance changes per request and not within one

```bash
curl -s localhost:8080/lab/scope; echo
curl -s localhost:8080/lab/scope; echo
curl -s localhost:8080/lab/scope; echo
```

The response contains two `describe()` results per request. Compare `instanceId` across and
within requests.

| What you see | What it means |
|---|---|
| Within one response, both `instanceId` values equal; across responses they increase | **Correct.** One instance per request, reused within the request. This is the definition of request scope, observed. |
| `instanceId` identical across all three requests | The scoped bean is being resolved once and cached. Either no proxy (Proof 1) or a singleton field caching the value (Trap 1b). |
| `instanceId` differs *within* a single response | Not request scope — you have a prototype, or the scope name is misspelled. A misspelled scope name usually fails at startup, so check for `@Scope("prototype")` first. |
| `hitsThisRequest` shows `1` then `2`, resetting to `1` next request | State is per-request. Correct. |
| `hitsThisRequest` keeps climbing across requests | The same instance is being reused across requests. Same diagnosis as row 2. |

Run them concurrently to check the thread story too:

```bash
for i in $(seq 1 20); do curl -s localhost:8080/lab/scope & done; wait
```

Every response should carry a distinct pair of matching `instanceId`s. If two concurrent
responses share an `instanceId`, request scope is not isolating per request, which would
mean the scope is bound to something other than the request.

### Proof 3 — reproduce the silent leak, deliberately

This is the important one. Change the annotation to remove the proxy:

```java
@Component
@Scope("request")           // proxyMode defaults to NO
public class RequestScopedCounter { }
```

Run 3a — normal startup:

```bash
./mvnw spring-boot:run
```

Run 3b — with lazy initialisation, so the singleton is built inside the first request:

```bash
./mvnw spring-boot:run \
  -Dspring-boot.run.arguments=--spring.main.lazy-initialization=true
```

Then, for whichever run starts:

```bash
curl -s localhost:8080/lab/scope; echo
curl -s localhost:8080/lab/scope; echo
curl -s localhost:8080/lab/scope; echo
```

| What you see | What it means |
|---|---|
| Run 3a fails at startup with `No thread-bound request found` | Trap 1 Symptom A confirmed. This is the safe failure and the common experience. |
| Run 3b starts, and all three responses show the **same** `instanceId`, with `hitsThisRequest` climbing 1, 2, 3, 4, 5, 6 | **The leak, reproduced.** The singleton captured request #1's instance. In `orderflow` that is request #1's tenant id, served to every subsequent customer. Write down what you saw. |
| Run 3b also fails at startup | Lazy initialisation did not defer this particular bean on your version. The mechanism still holds; force it instead by putting `@Lazy` on `ScopeProbeService` and re-running. |
| Both runs behave identically to Proof 2 | You did not actually remove the proxy — check you edited the annotation and that a stale class is not on the classpath (`./mvnw clean spring-boot:run`). |

Now restore `@RequestScope` and confirm Proof 2's behaviour returns. **Do both directions.**
A drill you only run in the broken direction teaches you half of it.

### Proof 4 — destruction callbacks: request scope yes, prototype no

Add `@PreDestroy` to two beans and watch which one fires.

```java
@Component
@RequestScope
public class RequestScopedCounter {
    @PreDestroy void bye() { System.out.println("DESTROY request instanceId=" + instanceId); }
}

@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class ProtoBean {
    private static final AtomicLong SEQ = new AtomicLong();
    private final long id = SEQ.incrementAndGet();
    @PostConstruct void hi()  { System.out.println("CREATE  proto id=" + id); }
    @PreDestroy   void bye() { System.out.println("DESTROY proto id=" + id); }
}
```

Obtain a `ProtoBean` per request via an injected `ObjectProvider<ProtoBean>` in the
controller, then:

```bash
./mvnw spring-boot:run 2>&1 | grep -E "CREATE|DESTROY"
# in another shell:
curl -s localhost:8080/lab/scope > /dev/null
curl -s localhost:8080/lab/scope > /dev/null
```

| What you see | What it means |
|---|---|
| `DESTROY request instanceId=<n>` once per request | Request scope runs destruction callbacks at request completion. As documented. |
| `CREATE proto id=<n>` per request, and **no** `DESTROY proto` lines ever | **Trap 3, proven.** Spring built it, ran `@PostConstruct`, and forgot it. Anything that bean opened is now yours to close. |
| `DESTROY proto` lines appear | Something else is managing it — check you did not accidentally make it a singleton, or that no custom `BeanPostProcessor` is registering an adapter. |
| No `DESTROY request` lines either | The request completion hook is not running. Check you are on a servlet stack and not, for example, invoking the service from a plain unit test with no request bound. |

### Proof 5 — prove the thread boundary breaks it

```java
@Configuration
@EnableAsync
class AsyncConfig {}

@Service
public class AsyncProbe {
    private final RequestScopedCounter counter;
    public AsyncProbe(RequestScopedCounter counter) { this.counter = counter; }

    @Async
    public void offThread() {
        try {
            System.out.println("ASYNC instanceId=" + counter.instanceId());
        } catch (RuntimeException e) {
            System.out.println("ASYNC FAILED: " + e.getClass().getName() + ": " + e.getMessage());
        }
    }
}
```

Call `offThread()` from the controller, then:

```bash
curl -s localhost:8080/lab/scope > /dev/null
# watch the application log
```

| What you see | What it means |
|---|---|
| `ASYNC FAILED: java.lang.IllegalStateException: No thread-bound request found...` | **Trap 4, proven.** Request scope is `ThreadLocal`-backed and does not cross a thread boundary. Exactly Topic 56's `SecurityContext` problem. |
| `ASYNC instanceId=<n>` printed successfully | `@Async` did not actually run on another thread — check `@EnableAsync` is present and that you are not calling `offThread()` via a self-invocation (Topic 40), which would run it inline on the request thread. |
| Nothing printed at all and no error in the HTTP response | The exception went to the default `AsyncUncaughtExceptionHandler` and was logged at a level you are filtering out. **This is the production symptom** — a silent failure on a background thread with a clean 200 to the client. Raise the log level and look again. |

### Proof 6 — list every scoped bean in the context

Useful on a codebase you did not write:

```java
@Component
class ScopeInventory implements ApplicationRunner {
    private final ConfigurableListableBeanFactory factory;
    ScopeInventory(ConfigurableListableBeanFactory factory) { this.factory = factory; }

    @Override public void run(ApplicationArguments args) {
        for (String name : factory.getBeanDefinitionNames()) {
            String scope = factory.getBeanDefinition(name).getScope();
            if (scope != null && !scope.isEmpty() && !"singleton".equals(scope)) {
                System.out.println("SCOPE " + scope + " -> " + name);
            }
        }
    }
}
```

| What you see | What it means |
|---|---|
| A `scopedTarget.requestContext` entry with scope `request` | The scoped-proxy split, visible. The *other* definition — plain `requestContext` — is a singleton `ScopedProxyFactoryBean`, which is why it does not appear in this list. |
| A `request`-scoped bean with **no** matching `scopedTarget.` prefix | No proxy was created for it. If anything longer-lived injects it, you are in Trap 1. |
| `prototype`-scoped beans you did not expect | Audit each one: does anything inject it directly into a singleton (Trap 2), and does it hold a resource (Trap 3)? |

---

## Practice exercises

### 1 — easy: see all four behaviours, then predict them

Build the Example 1 lab and answer these **before** running each variant:

1. With `@RequestScope`, what will `instanceId` do across three sequential requests?
2. With `@Scope("request")` and no proxy mode, what happens at startup?
3. With `@Scope("prototype")` injected into a singleton, what will
   `System.identityHashCode` print across three requests?
4. With `@Scope("prototype")` obtained via `ObjectProvider.getObject()` per call, what
   changes?

Write your four predictions down. Then run each and mark yourself. Any prediction you got
wrong is a place your model is wrong, and it is cheaper to find it here than in an
interview.

**Done when:** you can state, without running anything, which of the four combinations
produces a startup failure, which produces a silent stale reference, and which is correct.

### 2 — medium: combines Topics 37, 39 and 40

Take the `orderflow` `RequestContext` from Example 2 and deliberately break it four ways,
one at a time, predicting the symptom first:

1. Make `RequestContext` a `record` instead of a class. **Predict** the failure and when it
   occurs (startup or first request), then run it.
2. Make `tenantId()` `final`. Predict what `OrderPlacementService` sees. Run it. This one is
   silent — you will need to print the value to observe it.
3. Move `context.tenantId()` into a `@PostConstruct` on `OrderPlacementService`. Predict
   which of Topic 37's lifecycle steps causes the failure, and name the exact step.
4. Change constructor injection to field injection (`@Autowired private RequestContext
   context;`). Does the scoped proxy still work? Does anything change about *when* the
   failure surfaces? Relate your answer to Topic 39's argument for constructor injection.

For each, write: the exact symptom, the lifecycle step or proxy mechanism responsible, and
the one-line fix.

**Done when:** you can explain (2) purely in terms of Topic 40's CGLIB override rules, with
no hand-waving about "the proxy not working".

### 3 — hard: production simulation — prove no cross-tenant leak under concurrent load

Build the full Example 2 stack: `RequestContext`, `RequestContextFilter`,
`OrderPlacementService`, and a `POST /api/orders` endpoint that writes an order row
carrying the tenant id.

**Requirements:**

1. Two tenants, `tenant-alpha` and `tenant-beta`. For the lab, take the tenant from an
   `X-Tenant-Id` header instead of a JWT claim (Topic 57 replaces this with the real thing;
   do not ship a header-based tenant).
2. A load script (k6, or `hey`, or a bash loop with `xargs -P`) that fires **200 concurrent
   requests, alternating tenants**, each with a unique `X-Correlation-Id`.
3. After the run, assert two things from the database:
   - Every row's `tenant_id` matches the tenant the corresponding request declared.
   - Every row's correlation id is **distinct**, and every response echoed back the
     correlation id it was sent.

**Then break it and prove your assertions catch it.** Change `@RequestScope` to
`@Scope("request")`, add `--spring.main.lazy-initialization=true`, and re-run. Your
assertions must fail. If they pass, they are not testing what you think.

**Then measure the cost.** Run the load with the scoped proxy and again with the tenant
passed as an explicit method parameter. Compare p50/p95/p99.

**Write down, in three sentences:** what the difference was, whether it was outside the
noise band, and what you would tell a colleague who wanted to remove the proxy "for
performance". If your load generator is a bash loop, note honestly that it is a coordinated
omission machine and the tail numbers are not trustworthy — Topic 65 builds the real one.

**Done when:** the assertion suite fails on the broken configuration and passes on the fixed
one, and you have a number (with a stated confidence) for the proxy's cost.

---

## Interview questions

### Q1 — "What happens if you inject a request-scoped bean into a singleton?"

**Mid-level answer:** "You get an error, because the request scope isn't active at startup.
You fix it with a scoped proxy."

**Senior answer:** "It depends on *when* the singleton is constructed, and that is the whole
answer. Without a scoped proxy, Spring resolves the dependency once at construction time. If
the singleton is built eagerly at startup there is no bound request, so you get a
`BeanCreationException` wrapping `IllegalStateException: No thread-bound request found` —
that is the loud, safe failure. But if the singleton happens to be constructed *during* a
request — because lazy initialisation is on, or someone added `@Lazy`, or it came from an
`ObjectProvider` inside a handler — the resolution succeeds and the singleton holds request
number one's instance forever. Nothing throws. In a multi-tenant service that is a
cross-tenant data leak with a 100% success rate on your dashboards. The fix is
`@RequestScope`, which sets `proxyMode = TARGET_CLASS` for you, or an `ObjectProvider` at
the injection point. And I'd add a `@SpringBootTest` assertion that the injected field is a
CGLIB proxy, because that is the only automated signal for the silent variant."

**What separates them:** the mid answer knows the fix. The senior answer knows there are
**two** symptoms, knows which configuration change flips between them, and knows the silent
one is the dangerous one. Naming `spring.main.lazy-initialization` as a way a startup
optimisation turns into a security incident is the detail that lands.

**Interviewer's follow-up:** *"You've got the proxy configured correctly and the bug still
happens. What are you looking for?"* — Two things: a singleton field caching the value read
through the proxy (Trap 1b), or a `final` method on the scoped class so CGLIB never
overrode it and you are reading the proxy's empty fields (Trap 5).

---

### Q2 — "How does Spring's request scope differ from Nest's REQUEST scope?"

**Mid-level answer:** "They both give you one instance per request. Nest calls it
`Scope.REQUEST` and Spring calls it `request`."

**Senior answer:** "Same intent, opposite mechanism, and the mechanisms have different
failure modes. Nest bubbles the scope *up* the injection chain: if a provider is
request-scoped, everything that injects it becomes request-scoped too, all the way to the
controller. So Nest constructs a whole subtree per request. Spring does not bubble at all —
the singleton stays a singleton, and Spring injects a scoped proxy that resolves the real
instance per method call from a `ThreadLocal`-backed holder. The trade is: Nest pays
allocation per request and is correct by construction, because there is no long-lived
reference to go stale. Spring pays a per-call indirection and can go stale, because a
singleton *can* cache what the proxy returned. In Nest the cost is visible in a profile; in
Spring the bug is invisible in a code review, because a singleton with a scoped proxy looks
exactly like a singleton with a singleton."

**What separates them:** the word "bubbles", and naming what each approach makes impossible.
An answer that only lists scope names is a documentation recital.

**Interviewer's follow-up:** *"Which would you rather have?"* — For a high-throughput
service, Spring's, because a per-request subtree rebuild at 1,200 rps is real GC pressure
and I can defend against the staleness bug with one assertion in a test. For a team new to
the framework, Nest's, because it cannot silently do the wrong thing.

---

### Q3 — "Why doesn't `@PreDestroy` run on a prototype bean?"

**Mid-level answer:** "Spring doesn't manage prototypes after creating them."

**Senior answer:** "Because Spring keeps no reference to them, and it keeps no reference on
purpose — retaining every prototype would be an unbounded leak, since the container has no
idea when you are finished with one. So the contract is explicitly asymmetric: Spring
instantiates it, injects it, runs `BeanPostProcessor`s and `@PostConstruct`, hands it over,
and forgets. `@PostConstruct` fires and `@PreDestroy` does not, which is worse than neither
firing because it looks like the lifecycle works. In practice this bites when a prototype
holds a file handle or a connection: you get `Too many open files` days later, and the stack
trace points at whatever unlucky code tried to open the next socket. My rule is that a
prototype holding a resource should not be a bean at all — it should implement
`AutoCloseable` and be created with `new` inside a try-with-resources. Request-scoped beans
*do* get destruction callbacks, because the scope has a defined end and registers them."

**What separates them:** explaining *why* the design is that way, and knowing the
request-scope contrast. The `AutoCloseable` recommendation shows judgement rather than
mechanism recall.

**Interviewer's follow-up:** *"How would you find this in an existing codebase?"* — Iterate
`getBeanDefinitionNames()`, filter to non-singleton scopes, and inspect each for a
`@PreDestroy` or a field of a `Closeable` type. Proof 6 is that code.

---

### Q4 — "You need the current tenant id in thirty service classes. How do you do it?"

**Mid-level answer:** "Make a request-scoped bean holding the tenant and inject it
everywhere."

**Senior answer:** "That is one of three options and I'd want to know two things before
choosing: does any non-HTTP code path need the same value, and does any of this run off the
request thread. If it's HTTP-only and on-thread, a `@RequestScope` bean populated in a
filter is the cleanest — one typed object, thirty injection points, no parameter churn.
If a Kafka consumer or a scheduled job also needs it, request scope does not exist there, so
I'd define a `TenantHolder` interface with a request-scoped implementation for HTTP and an
explicitly-set one for consumers, and have the services depend on the interface. And
whichever I pick, I make the values snapshot-able into an immutable record, because the
moment anyone submits work to an executor the `ThreadLocal` is empty and I need something to
pass. The option I'd argue *for* in a small codebase is just passing the tenant id as a
parameter — it is always correct, testable without a container, and visible in every
signature. Ambient context is a convenience that costs you a class of invisible bugs, and
thirty call sites is roughly where the trade tips."

**What separates them:** naming the two questions that decide it, and being willing to
defend the boring explicit-parameter option. Mentioning the snapshot for thread boundaries
shows they have hit this before.

**Interviewer's follow-up:** *"Say you go with the request-scoped bean. What do you write
down for the next person?"* — A class comment stating that it is CGLIB-proxied, so no
`final` methods, not a record, no public fields, and never cache what it returns in a
longer-lived object.

---

### Q5 — "What is `singleton` scope actually a singleton of?"

**Mid-level answer:** "There's one instance in the application."

**Senior answer:** "One per `ApplicationContext`, which is not the same as one per JVM.
That distinction matters in three places. In tests, the Spring test framework caches
contexts by configuration key, so a suite with several distinct configurations holds several
live contexts and several instances of your 'singleton' — which is exactly why static
mutable state shared between tests behaves strangely, and it is Topic 60. In a
parent/child context hierarchy, each context has its own. And in an app that starts a second
context programmatically, likewise. It also means a singleton bean is shared across every
concurrent request thread, so any mutable field on it is a data race — that is the practical
reason service classes are stateless apart from their injected collaborators, and it is the
first thing I check when someone reports intermittent wrong values."

**What separates them:** the per-context qualification, plus the immediate jump to "and
therefore its fields are shared across threads". The second half is the one that actually
prevents bugs.

**Interviewer's follow-up:** *"Give me a legitimate use for mutable state on a singleton."*
— A counter or cache using a thread-safe structure: `AtomicLong`, `LongAdder`, a
`ConcurrentHashMap`, or a Caffeine cache. The rule is not "no state", it is "no state
without a concurrency story", and the story has to be written down.

---

## Mental model checkpoint

Answer these out loud, without looking above. They are open questions; if an answer takes
you more than three sentences you have not compressed it yet.

1. A singleton and a request-scoped bean need to talk. In which direction is a plain
   reference safe, and why is the other direction unsafe — in terms of lifetimes, not in
   terms of Spring?

2. Spring injects a scoped proxy. Nest bubbles scope up the injection chain. State the one
   bug each approach makes impossible, and the one bug each approach makes possible.

3. `@Scope("request")` on a bean injected into a singleton. Give both possible symptoms, and
   name the configuration difference that decides which one you get.

4. A colleague adds `final` to a method on a `@RequestScope` class. Describe, mechanically,
   what the caller now receives and why nothing throws.

5. Why does `@PostConstruct` fire on a prototype but `@PreDestroy` not? Answer in terms of
   what the container is holding, not in terms of what the documentation says.

6. Name the four `ThreadLocal`-backed context holders in this stack that all break at the
   same boundary, and say what that boundary is.

7. You have a request-scoped bean and you need its value inside a `CompletableFuture`. Give
   the correct approach in one sentence, and say why copying the request attributes onto the
   pool thread is worse.

---

## Quick reference card

### The scopes

| Scope | Instance per | Web only? | `@PreDestroy` runs? |
|---|---|---|---|
| `singleton` (default) | `ApplicationContext` | no | yes, at context close |
| `prototype` | container request | no | **no** |
| `request` | HTTP request | yes | yes |
| `session` | `HttpSession` | yes | yes |
| `application` | `ServletContext` | yes | yes |
| `websocket` | WS session | yes | yes |

### Declaring a scope

```java
@Component @RequestScope                      // BEST for request scope
@Component @SessionScope
@Component @ApplicationScope
@Component @Scope(value = SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)

@Bean @RequestScope Foo foo() { ... }         // also works on @Bean methods
```

### `ScopedProxyMode`

| Value | Injects | Note |
|---|---|---|
| `NO` (**default**) | the real instance, once | the bug in Trap 1 |
| `TARGET_CLASS` | CGLIB subclass proxy | what you want |
| `INTERFACES` | JDK interface proxy | injection by concrete type then fails |
| `DEFAULT` | ~ `NO` | do not write it |

### Getting a fresh instance per call

```java
ObjectProvider<Thing> provider;   provider.getObject();      // preferred
@Lookup protected abstract Thing newThing();                 // legacy
new Thing();                                                 // best if it has no deps
```

### Rules for a class that will be scope-proxied

```
not final          not a record        no final methods
no public fields   never cached in a longer-lived bean
```

### Diagnostics

```java
bean.getClass().getName()                       // $$SpringCGLIB$$ = proxied
AopUtils.isCglibProxy(bean)
AopUtils.isJdkDynamicProxy(bean)
AopProxyUtils.ultimateTargetClass(bean)
beanFactory.getBeanDefinition(name).getScope()  // the declared scope
```

```properties
logging.level.org.springframework.beans.factory=DEBUG
logging.level.org.springframework.web.context.request=TRACE
spring.main.lazy-initialization=true            # flips Trap 1 from loud to silent
```

Look for a `scopedTarget.<beanName>` definition — its presence is the proxy split.

### Gotchas checklist

- `@Scope("request")` without `proxyMode` → Trap 1. Use `@RequestScope`.
- Value read through a proxy stored in a singleton field → Trap 1b.
- Prototype injected into a singleton → created once → Trap 2.
- `@PreDestroy` on a prototype → never runs → Trap 3.
- Scoped bean touched on an `@Async`/executor/virtual thread → `IllegalStateException` →
  Trap 4. Snapshot instead.
- `final` method or `record` on a scoped class → silent `null` or startup failure → Trap 5.
- Scoped proxy used as a map key or in a set → Topic 13's mutable-key bug.
- `singleton` is per context, not per JVM.
- Mutable field on a singleton is shared by every request thread.

---

## When would I use this at work?

**1. Multi-tenant request context — the `orderflow` case.** A `@RequestScope` bean holding
correlation id, tenant id and caller identity, populated in one filter after the security
chain, injected wherever needed. This is the default shape for any service where "who is
asking" must reach the persistence layer without appearing in forty signatures. The
non-negotiable part is the paired assertion test that the injection point is a proxy.

**2. Per-request accumulators for observability.** A request-scoped bean that collects
"how many database queries did this request issue", "how many cache misses", "which
downstream calls were made", and emits one structured log line at request completion. This
is genuinely per-request state with a defined end, so request scope is exactly right — and
the automatic destruction callback is what makes the emit-at-end pattern clean. It also
becomes your N+1 detector (Topic 50) with about ten more lines.

**3. Deciding *against* it.** The most valuable use of this topic at work is arguing someone
out of a request-scoped bean. When a colleague proposes ambient context for a value used in
three places, the honest position is: three explicit parameters are correct, testable
without a container, obvious to a reader, and cannot leak across tenants. Ambient context
earns its complexity somewhere north of a dozen consumers or a deep call stack you do not
control. Knowing the mechanism is what lets you make that argument on grounds other than
taste.

---

## Connected topics

**Prerequisites:**

- **35 — `ApplicationContext`**: the two-phase startup is why a scoped proxy can exist at
  all — the bean definition is *rewritten* into two definitions before anything is
  instantiated.
- **36 — Bean definition and component scanning**: `@Scope` is metadata on a
  `BeanDefinition`; `scopedTarget.<name>` is a second definition registered by the scoped
  proxy machinery.
- **37 — Bean lifecycle**: `@PostConstruct` runs at startup, which is why reading a
  request-scoped value from it fails; and the prototype half-lifecycle (`@PostConstruct` yes,
  `@PreDestroy` no) is a lifecycle fact.
- **39 — Injection styles**: `ObjectProvider` is the deferred-lookup form of constructor
  injection and the fix for both Trap 1 and Trap 2.
- **40 — Proxying**: a scoped proxy *is* a CGLIB proxy. Every `final`/`record`/field-access
  caveat transfers unchanged. Trap 5 is Topic 40 wearing a different hat.

**This unlocks / is used by:**

- **41 — AOP**: an audit aspect reading the correlation id from the request-scoped context
  is the first real consumer of this bean.
- **44 — REST controllers**: a custom `HandlerMethodArgumentResolver` is the alternative to
  a scoped bean for getting per-request values into a controller — same problem, different
  seam.
- **54–55 — `@Transactional`**: transaction-bound resources use `TransactionSynchronizationManager`,
  which is another `ThreadLocal`-scoped registry with the same lifetime reasoning.
- **56 — Spring Security filter chain**: `SecurityContextHolder` is the same `ThreadLocal`
  pattern; the `RequestContextFilter` here must be ordered *after* the security chain to see
  a validated principal.
- **57 — Authorization and JWT**: the tenant claim that populates `RequestContext` comes
  from the validated token, not from a client-controlled header.
- **60 — Spring test slices**: context caching is why "singleton" means per context, and
  why a `@MockitoBean` forks a second context with a second set of singletons.
- **65 — GATE, load baseline**: the cross-tenant assertion from Exercise 3 belongs in the
  load scenario; correctness under concurrency is a load-test result, not a unit-test one.
- **91 — `CompletableFuture`** and **101 — Virtual threads**: the same
  `ThreadLocal`-doesn't-propagate problem at larger scale. Virtual threads make it worse by
  multiplying the number of carriers.
- **109 — HikariCP**: a request-scoped bean holding a connection would pin it for the whole
  request — the same failure shape as `open-session-in-view` from Topic 49.
- **116 — Idempotency**: the idempotency key is per-request state with exactly this
  lifetime, and it is a good candidate for the request context object.
- **119 — Tracing and context propagation across threads**: the definitive treatment of the
  problem this topic introduces. Trace context, security context, MDC and request scope are
  four instances of one pattern, and 119 is where you build the propagation strategy for all
  of them.
- **120 — Logging and MDC**: the correlation id in `RequestContext` is what populates the
  MDC, and MDC leaking on pooled threads is the mirror image of Trap 4.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0, Jakarta EE 11
(`jakarta.servlet.*`). Scope semantics, `ScopedProxyMode` and the `@RequestScope` composed
annotation are stable across Framework 6.x and 7.0; the `javax` → `jakarta` package move
happened in Boot 3.0 and is Topic 127. Rather than trusting this document about which
objects your container actually built, print `getBeanDefinitionNames()` with each
definition's scope (Proof 6) and `getClass().getName()` on every injected scoped bean
(Proof 1) — those two lines settle every question in this topic on your machine.*
