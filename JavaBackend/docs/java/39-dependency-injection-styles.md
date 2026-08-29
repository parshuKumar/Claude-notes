# 39 — Injection Styles: Constructor vs Field vs Setter; `@Qualifier`, `@Primary`, Circular Deps

## Phase: 4 — Spring Core
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: refactor every `orderflow` service to constructor injection; introduce a `PaymentGateway` interface with two implementations, selected by `@Qualifier`

---

## ELI5 anchor

You are hiring someone to run a shop.

**Constructor injection** is handing them the keys, the till and the stock list **at
the door, before they are allowed in**. If you forgot the till, they never get
through the door. Nobody ever sees a shopkeeper standing behind an empty counter.

**Setter injection** is letting them in, then handing them things one at a time.
There is a window where they are in the shop with no till. If a customer walks in
during that window, they cannot serve them.

**Field injection** is letting them in empty-handed and then **reaching through the
wall with a magic arm** and placing the till on the counter from outside. It works.
It is invisible. And nobody looking at the shop from the street can tell what the
shopkeeper actually needs to do their job — you have to go inside and read the
walls.

Three consequences fall straight out of that picture, and they are the whole topic:

1. With the door check, a missing dependency is caught **at the door** — before the
   object exists in a broken state.
2. With the magic arm, you can never bolt anything down (`final`), because the arm
   has to be able to move it.
3. With the magic arm, a shopkeeper who needs **nine** things looks exactly like a
   shopkeeper who needs one. The door check makes the nine visible and embarrassing.

---

## The bridge from what you know

### What transfers almost completely

You are **not** learning DI. You have been writing this in NestJS for years:

```ts
@Injectable()
export class OrderService {
  constructor(
    private readonly inventory: InventoryService,
    private readonly wallet: WalletService,
  ) {}
}
```

Java/Spring:

```java
@Service
public class OrderService {

    private final InventoryService inventory;
    private final WalletService wallet;

    public OrderService(InventoryService inventory, WalletService wallet) {
        this.inventory = inventory;
        this.wallet = wallet;
    }
}
```

Same idea, more keystrokes. Nest's `private readonly x: X` in the parameter list is
sugar that declares the field, assigns it, and marks it read-only in one go. Java has
no such sugar (records do it, but a record is a data carrier, not a service). You
write the field, the parameter and the assignment by hand, or you let Lombok's
`@RequiredArgsConstructor` write them.

**Verdict: HONEST analogue.** Constructor injection means the same thing in both.

### What is genuinely different — resolution by TYPE, not by TOKEN

This is the difference that will actually catch you.

Nest resolves a dependency by an **injection token**. The token is usually the class
reference itself, or a string/symbol you supply:

```ts
constructor(@Inject('PAYMENT_GATEWAY') private readonly gateway: PaymentGateway) {}
```

The type annotation `: PaymentGateway` is erased at compile time and is not what Nest
looks up. The token is.

Spring resolves by **type first**. It looks at the parameter's declared type, finds
every bean assignable to it, and:

- exactly one candidate → inject it;
- zero candidates → fail at startup with `NoSuchBeanDefinitionException`;
- more than one candidate → try to narrow by `@Primary`, then by `@Qualifier`, then
  by matching the **parameter name** against the bean name. Still ambiguous → fail at
  startup with `NoUniqueBeanDefinitionException`.

That fallback to parameter name is worth internalising now, because it means
**renaming a constructor parameter can break your application** in a codebase with
two implementations of an interface and no explicit qualifier. There is no equivalent
hazard in Nest.

### The mapping table

| NestJS | Spring | Verdict |
|---|---|---|
| `constructor(private readonly x: X)` | constructor + `final` field + assignment | **HONEST** — same semantics |
| `@Inject(TOKEN)` | `@Qualifier("beanName")` | **PARTIAL** — Nest's token *is* the identity; Spring's qualifier only *narrows* a type-based match |
| `useClass` / `useFactory` / `useValue` | `@Bean` methods in a `@Configuration` class | **HONEST** — Topic 36 |
| `@Optional()` | `ObjectProvider<X>`, `Optional<X>`, or `@Autowired(required = false)` | **PARTIAL** |
| `forwardRef(() => X)` | `@Lazy` on one side of the cycle | **PARTIAL** — Nest's forwardRef fixes module-graph ordering; Spring's `@Lazy` inserts a *proxy*, which drags in Topic 40 |
| No field injection (there is no equivalent) | `@Autowired` on a private field | **NO ANALOGUE** — this is a Java-specific footgun you have never had access to |
| Property injection via `@Inject()` on a property | setter injection | **PARTIAL** — rare in both |

### The one that has no Nest equivalent at all

Field injection. Nest cannot reach into a private class property and set it, because
TypeScript decorators run at class-definition time and Nest's container has no
reflective write access to instance fields after construction. Java has
`java.lang.reflect.Field.setAccessible(true)`, and Spring uses it. That capability is
the entire reason field injection exists, and the entire reason it is a problem.

---

## What is this?

Spring has three places it can put a dependency into your bean:

1. **Constructor injection** — Spring calls your constructor with the resolved beans
   as arguments. The object is fully formed the moment it exists.
2. **Setter injection** — Spring calls your no-arg constructor first, then calls
   setters. There is a window where the object exists and is incomplete.
3. **Field injection** — Spring calls your no-arg constructor, then uses reflection
   to write directly into the private fields. Same window, plus invisibility.

On top of that, four annotations control *which* bean is chosen when the type alone
is not enough:

- `@Autowired` — "inject here". Optional on constructors since Spring 4.3.
- `@Qualifier("name")` — "of the candidates, pick this one".
- `@Primary` — "when nobody says which, pick me".
- `@Value("${...}")` — "inject a configuration value, not a bean".

And one interface, `ObjectProvider<T>`, for when the dependency is optional, plural,
or must be resolved *later* rather than at construction.

---

## Why does it matter?

Three concrete production failures, all of which trace back to the injection style.

**1. You ship an object that cannot work, and find out at request time.**

With field injection, `new OrderService()` succeeds and produces an instance whose
`inventory` field is `null`. That instance is perfectly constructible in a test, in a
static factory, in a `@Bean` method someone wrote by hand. The first NPE arrives at
2am from a code path nobody exercised. Constructor injection makes that instance
*impossible to create*.

**2. The god class hides.**

A class with nine field-injected dependencies looks like nine tidy one-line
declarations spread down the file. The same class with constructor injection has a
nine-parameter constructor that no reviewer can look at without flinching. The
discomfort is the feature. This is how single-responsibility violations get caught in
code review rather than in a refactor eighteen months later.

**3. You lose the memory-model guarantee from Topic 17.**

A field written by reflection after construction cannot be `final`. A non-final field
has **no final-field freeze**, so there is no guarantee that a thread which obtains a
reference to your bean sees the injected value rather than `null`. In practice Spring
publishes singletons through a `ConcurrentHashMap` whose synchronisation covers you —
so this is rarely the *proximate* cause of a bug. But you have traded a guarantee for
an accident of the container's implementation, and "it works because of something I
did not reason about" is exactly the class of thing that stops working under a
refactor. Topic 17 is the full argument; this is where it becomes a Spring decision.

---

## Syntax breakdown

### Constructor injection — the default, and no annotation needed

```java
package com.orderflow.orders;

import com.orderflow.inventory.InventoryService;
import com.orderflow.wallet.WalletService;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final InventoryService inventory;
    private final WalletService wallet;

    // No @Autowired. Since Spring 4.3, a class with exactly ONE constructor
    // has that constructor used for injection implicitly.
    public OrderService(InventoryService inventory, WalletService wallet) {
        this.inventory = inventory;
        this.wallet = wallet;
    }
}
```

| Bit of syntax | What it means |
|---|---|
| `@Service` | A `@Component` stereotype. Makes the class a scan candidate (Topic 36). `@Service` vs `@Component` is documentation, not behaviour. |
| `private final` | The field can never be reassigned, and gets the JMM freeze from Topic 17. Only possible with constructor injection. |
| no `@Autowired` | Spring 4.3+ rule: **one constructor → it is the injection point.** Two or more constructors → you must mark exactly one with `@Autowired`, or Spring uses the no-arg one if present, or fails. |
| parameter types | The lookup key. `InventoryService` is matched against every bean assignable to that type. |

### `@Autowired` — where you still need it

```java
@Service
public class PaymentService {

    private final PaymentGateway gateway;
    private final AuditLog audit;

    public PaymentService(PaymentGateway gateway) {      // used in production
        this(gateway, AuditLog.noop());
    }

    @Autowired                                            // disambiguates: use THIS one
    public PaymentService(PaymentGateway gateway, AuditLog audit) {
        this.gateway = gateway;
        this.audit = audit;
    }
}
```

Two constructors means Spring cannot guess. `@Autowired` on one of them names the
injection point. This is the only common reason to still write `@Autowired` in 2026.

`@Autowired` also has a `required` flag:

```java
@Autowired(required = false)
public void setFraudScorer(FraudScorer scorer) { this.scorer = scorer; }
```

If no `FraudScorer` bean exists, the setter is simply never called. Prefer
`ObjectProvider` (below) to this — `required = false` leaves you with a `null` field
and no signal.

### `@Qualifier` — narrowing among several candidates

```java
public interface PaymentGateway {
    AuthorizationResult authorize(PaymentRequest request);
}

@Component("stripeGateway")
public class StripePaymentGateway implements PaymentGateway { /* ... */ }

@Component("adyenGateway")
public class AdyenPaymentGateway implements PaymentGateway { /* ... */ }
```

```java
@Service
public class CardPaymentService {

    private final PaymentGateway gateway;

    public CardPaymentService(@Qualifier("adyenGateway") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

| Bit of syntax | What it means |
|---|---|
| `@Component("stripeGateway")` | Sets the bean **name** explicitly. Without it, the name is the class name with the first letter lower-cased: `stripePaymentGateway`. |
| `@Qualifier("adyenGateway")` on a parameter | Restrict the candidate set to beans whose name — or whose own `@Qualifier` value — is `adyenGateway`. |
| where it goes | On the **parameter**, not the constructor. Putting it on the constructor applies it to nothing useful. |

The string is the weak point: it is not checked by the compiler. Rename the bean,
and you get a startup failure — loud, but still a string-matching failure. The
stronger form is a **custom qualifier annotation**:

```java
package com.orderflow.payments;

import org.springframework.beans.factory.annotation.Qualifier;
import java.lang.annotation.*;

@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.PARAMETER, ElementType.METHOD, ElementType.FIELD })
public @interface Adyen { }
```

```java
@Adyen
@Component
public class AdyenPaymentGateway implements PaymentGateway { /* ... */ }

// injection point:
public CardPaymentService(@Adyen PaymentGateway gateway) { ... }
```

Now it is a type. Rename it and the compiler follows you. This is the version to use
in `orderflow`.

> **`@interface` is Java's syntax for declaring an annotation.** An annotation is
> inert metadata — a marker compiled into the class file. It does nothing at all
> until some component reads it via reflection. This is unlike a TypeScript
> decorator, which is a **function that executes** when the class is defined.
> `@Adyen` above never runs anything. Spring's `QualifierAnnotationAutowireCandidate`
> `Resolver` reads it during autowiring, and that read is the only thing that gives
> it meaning. Hold onto this distinction — Topic 40 is built on it.

### `@Primary` — the default winner

```java
@Primary
@Component
public class AdyenPaymentGateway implements PaymentGateway { /* ... */ }
```

Now any injection point asking for a bare `PaymentGateway` gets Adyen. Injection
points with an explicit `@Qualifier` still get what they asked for — **`@Qualifier`
at the injection point beats `@Primary` at the definition.**

Rules worth memorising:

- Exactly one `@Primary` per type. Two makes the ambiguity worse, not better, and
  fails at startup.
- `@Primary` says "sensible default". `@Qualifier` says "this specific one".
- Spring Framework 6.2 added `@Fallback`, the inverse: a bean marked `@Fallback` is
  only chosen if no non-fallback candidate exists. Useful for a stub gateway in a
  local profile. *I am confident this exists on the 6.2+/7.x line but check your exact
  version before relying on it — see the Hands-on section for the command.*

### `@Value` — configuration, not beans

```java
@Service
public class WalletService {

    private final long dailyDebitLimitMinorUnits;

    public WalletService(
            @Value("${orderflow.wallet.daily-limit-minor:500000}") long dailyLimit) {
        this.dailyDebitLimitMinorUnits = dailyLimit;
    }
}
```

| Bit of syntax | What it means |
|---|---|
| `${...}` | **Property placeholder.** Resolved from `application.yml`, environment variables, command line — the precedence chain is Topic 43. |
| `:500000` | Default if the property is absent. **Without a default, a missing property is a startup failure** — which is usually what you want. |
| `#{...}` | SpEL — Spring Expression Language. Evaluated, not looked up. `#{ 2 * T(java.lang.Runtime).getRuntime().availableProcessors() }`. Powerful and almost always a mistake in application code. |

`@Value` is fine for one or two values. For a group, use `@ConfigurationProperties`
(Topic 43) — you get a typed, validated object instead of scattered strings.

### `ObjectProvider<T>` — optional, plural, or deferred

```java
@Service
public class OrderService {

    private final ObjectProvider<FraudScorer> fraudScorers;

    public OrderService(ObjectProvider<FraudScorer> fraudScorers) {
        this.fraudScorers = fraudScorers;   // nothing resolved yet
    }

    public void place(Order order) {
        fraudScorers.ifAvailable(scorer -> scorer.score(order));
    }
}
```

| Method | Behaviour |
|---|---|
| `getObject()` | Like a normal injection. Throws if absent or ambiguous. |
| `getIfAvailable()` | Returns `null` if no bean exists. Throws if ambiguous. |
| `getIfAvailable(Supplier)` | Returns the supplied default if absent. |
| `getIfUnique()` | Returns `null` if absent **or** ambiguous. Never throws. |
| `ifAvailable(Consumer)` | Runs the consumer only if a bean exists. |
| `stream()` | All matching beans, in registration order. |
| `orderedStream()` | All matching beans, sorted by `@Order` / `Ordered`. |

`ObjectProvider` is injected as a **handle**, not a bean. Resolution happens when you
call it. That deferral is why it is one of the four fixes for the self-invocation trap
in Topic 40, and one of the fixes for a circular dependency below.

### Injecting all implementations

```java
// every PaymentGateway bean, ordered by @Order
public GatewayRouter(List<PaymentGateway> gateways) { ... }

// keyed by BEAN NAME
public GatewayRouter(Map<String, PaymentGateway> gatewaysByName) { ... }
```

An empty `List` injection **fails** by default (`NoSuchBeanDefinitionException`) —
Spring treats "no candidates" as an error, not an empty list. Use
`ObjectProvider<PaymentGateway>.stream()` if zero is legitimate.

### The `jakarta.*` alternatives

```java
import jakarta.inject.Inject;      // ≈ @Autowired(required = true)
import jakarta.inject.Named;       // ≈ @Qualifier / @Component with a name
import jakarta.annotation.Resource;
```

`@Resource` is the trap: it resolves **by name first**, falling back to type. So
`@Resource private PaymentGateway adyenGateway;` picks the bean named
`adyenGateway` — the field name is the lookup key. Surprising if you expect
`@Autowired` semantics. Use Spring's own annotations in `orderflow` and be able to
explain `@Resource` in an interview.

`[BOOT 3.x DELTA]` — none. `jakarta.*` replaced `javax.*` in Boot 3.0; Boot 4 does not
change it again. If you meet `javax.inject.Inject` in a codebase, that codebase is on
Boot 2.x.

---

## Example 1 — minimal

Three beans, one interface, two implementations, one ambiguity, one fix.

```java
package com.orderflow.payments;

public interface PaymentGateway {
    String name();
}

@org.springframework.stereotype.Component
class StripePaymentGateway implements PaymentGateway {
    public String name() { return "stripe"; }
}

@org.springframework.stereotype.Component
class AdyenPaymentGateway implements PaymentGateway {
    public String name() { return "adyen"; }
}

@org.springframework.stereotype.Service
class CheckoutService {

    private final PaymentGateway gateway;

    CheckoutService(PaymentGateway gateway) {   // ambiguous: TWO candidates
        this.gateway = gateway;
    }

    String which() { return gateway.name(); }
}
```

Start this and the context **fails**. Then apply exactly one of:

```java
// Option A — a default winner
@Primary @Component class AdyenPaymentGateway ...

// Option B — the caller decides
CheckoutService(@Qualifier("stripePaymentGateway") PaymentGateway gateway) { ... }

// Option C — rename the parameter to a bean name (works, and is a trap)
CheckoutService(PaymentGateway stripePaymentGateway) { ... }
```

Option C is the one to *know about and never use*. It works because Spring falls back
to matching the parameter name against the bean name. It means a rename in a
refactor silently changes which gateway processes payments. There is no warning.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` payments, sized from the Phase 7 gate targets:

- Peak **400 order placements/second** during a flash sale; each placement makes
  exactly one authorization call.
- SLO: **p99 of `authorize` under 250 ms**, error rate under 0.1%.
- Two acquirers for real commercial reasons: **Adyen** for EU cards (better
  interchange, local acquiring), **Stripe** for US cards and as the failover when
  Adyen's authorization error rate crosses 2%.
- A **stub** gateway used in the `local` and `test` profiles that authorizes
  everything instantly, so the Topic 65 load test does not hammer a real acquirer.

Two different selection problems live here, and they need different tools:

- A **fixed** dependency chosen at wiring time (settlement reconciliation always uses
  Adyen, because that is where the settlement files come from) → `@Qualifier`.
- A **dynamic** choice made per request (which acquirer for this card) → inject **all**
  implementations and route.

### The interface and a custom qualifier

```java
package com.orderflow.payments;

import java.lang.annotation.*;
import org.springframework.beans.factory.annotation.Qualifier;

@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.PARAMETER, ElementType.METHOD })
public @interface Acquirer {
    Network value();

    enum Network { ADYEN, STRIPE, STUB }
}
```

```java
package com.orderflow.payments;

public interface PaymentGateway {

    /** Non-blocking-free: this is a network call. Never call it inside a transaction — Topic 55. */
    AuthorizationResult authorize(AuthorizationRequest request);

    Acquirer.Network network();

    /** Which card networks/regions this gateway is allowed to serve. */
    boolean supports(CardRegion region);
}
```

### The two implementations

```java
package com.orderflow.payments.adyen;

import com.orderflow.payments.*;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Acquirer(Acquirer.Network.ADYEN)
@Order(10)                        // preferred first in the router's ordered stream
public class AdyenPaymentGateway implements PaymentGateway {

    private final AdyenHttpClient http;
    private final PaymentMetrics metrics;

    public AdyenPaymentGateway(AdyenHttpClient http, PaymentMetrics metrics) {
        this.http = http;
        this.metrics = metrics;
    }

    @Override
    public AuthorizationResult authorize(AuthorizationRequest request) {
        return metrics.timed("adyen", () -> http.authorize(request));
    }

    @Override public Acquirer.Network network() { return Acquirer.Network.ADYEN; }
    @Override public boolean supports(CardRegion region) { return region == CardRegion.EU; }
}
```

```java
package com.orderflow.payments.stripe;

@Component
@Acquirer(Acquirer.Network.STRIPE)
@Order(20)
public class StripePaymentGateway implements PaymentGateway {

    private final StripeHttpClient http;
    private final PaymentMetrics metrics;

    public StripePaymentGateway(StripeHttpClient http, PaymentMetrics metrics) {
        this.http = http;
        this.metrics = metrics;
    }

    @Override
    public AuthorizationResult authorize(AuthorizationRequest request) {
        return metrics.timed("stripe", () -> http.authorize(request));
    }

    @Override public Acquirer.Network network() { return Acquirer.Network.STRIPE; }
    @Override public boolean supports(CardRegion region) {
        return region == CardRegion.US || region == CardRegion.EU;   // can serve both
    }
}
```

### The stub, active only in non-production profiles

```java
package com.orderflow.payments.stub;

import org.springframework.context.annotation.Profile;

@Component
@Profile({ "local", "test" })
@Acquirer(Acquirer.Network.STUB)
@Order(1)
public class StubPaymentGateway implements PaymentGateway {

    @Override
    public AuthorizationResult authorize(AuthorizationRequest request) {
        return AuthorizationResult.approved("STUB-" + request.idempotencyKey());
    }

    @Override public Acquirer.Network network() { return Acquirer.Network.STUB; }
    @Override public boolean supports(CardRegion region) { return true; }
}
```

`@Profile` is *not* an injection concept — it decides whether the bean **definition**
is registered at all (Topic 35's two-phase startup makes this possible). In `local`
the stub exists and its `@Order(1)` puts it first. In `prod` it does not exist, so
nothing can accidentally select it.

### The fixed dependency — `@Qualifier`

```java
package com.orderflow.payments;

import org.springframework.stereotype.Service;

@Service
public class SettlementReconciliationService {

    private final PaymentGateway adyen;
    private final PaymentRepository payments;

    // Settlement files come from Adyen only. This is a wiring-time decision,
    // and the custom qualifier makes it compiler-checked.
    public SettlementReconciliationService(
            @Acquirer(Acquirer.Network.ADYEN) PaymentGateway adyen,
            PaymentRepository payments) {
        this.adyen = adyen;
        this.payments = payments;
    }
}
```

> **The custom qualifier carries an enum value**, and Spring matches annotation
> attribute values as part of qualifier resolution. So `@Acquirer(ADYEN)` at the
> injection point matches only the bean annotated `@Acquirer(ADYEN)`. No strings.

### The dynamic choice — inject them all

```java
package com.orderflow.payments;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class GatewayRouter {

    private final List<PaymentGateway> gateways;      // ordered by @Order
    private final GatewayHealth health;

    public GatewayRouter(List<PaymentGateway> gateways, GatewayHealth health) {
        this.gateways = List.copyOf(gateways);        // defensive copy — Topic 17
        this.health = health;
    }

    public PaymentGateway select(CardRegion region) {
        return gateways.stream()
                .filter(g -> g.supports(region))
                .filter(g -> health.errorRate(g.network()) < 0.02)   // 2% failover threshold
                .findFirst()
                .orElseThrow(() -> new NoAvailableGatewayException(region));
    }
}
```

Why this and not a `switch` over an enum, or nine `@Qualifier` fields:

- Adding a third acquirer is **one new `@Component`**. No existing file changes. That
  is the open/closed principle expressed through the container.
- The ordering is declared next to each implementation (`@Order`) rather than in a
  central list that drifts.
- `List.copyOf` in the constructor means `gateways` is genuinely immutable after
  construction. Combined with `final`, this bean is safely publishable to the 400
  request-handling threads that will share it — Topic 17's rule, applied.

### What the refactor to constructor injection actually changed

Before (the Topic 35–38 skeleton, written the quick way):

```java
@Service
public class OrderService {
    @Autowired private InventoryService inventory;
    @Autowired private WalletService wallet;
    @Autowired private PaymentService payments;
    @Autowired private OrderRepository orders;
    @Autowired private OrderEventPublisher events;
    @Autowired private PricingService pricing;
    @Autowired private FraudScorer fraud;
    @Autowired private NotificationService notifications;
    @Autowired private AuditLog audit;
}
```

After:

```java
@Service
public class OrderService {

    private final InventoryService inventory;
    private final WalletService wallet;
    private final PaymentService payments;
    private final OrderRepository orders;
    private final OrderEventPublisher events;
    private final PricingService pricing;
    private final FraudScorer fraud;
    private final NotificationService notifications;
    private final AuditLog audit;

    public OrderService(InventoryService inventory,
                        WalletService wallet,
                        PaymentService payments,
                        OrderRepository orders,
                        OrderEventPublisher events,
                        PricingService pricing,
                        FraudScorer fraud,
                        NotificationService notifications,
                        AuditLog audit) {
        this.inventory = inventory;
        ...
    }
}
```

Identical behaviour. But now the nine-parameter constructor is on screen, and the
next conversation is "should notification and audit be listeners on the order-placed
event instead of collaborators?" — which is the right conversation, and field
injection was preventing it from happening.

> **Do not reach for Lombok's `@RequiredArgsConstructor` to make this shorter.** It
> generates exactly this constructor, which is fine, but it also makes the nine
> dependencies invisible again — which was the point of the refactor. Use it once the
> class is genuinely small.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — field injection lets you construct an invalid object

**Wrong:**

```java
@Service
public class WalletService {
    @Autowired private WalletRepository wallets;

    public void debit(long walletId, long minorUnits) {
        wallets.debit(walletId, minorUnits);
    }
}
```

**Exact symptom:** a unit test written without Spring —

```java
var service = new WalletService();          // compiles, runs, returns an object
service.debit(1L, 500L);
```

— throws:

```
java.lang.NullPointerException: Cannot invoke
  "com.orderflow.wallet.WalletRepository.debit(long, long)"
  because "this.wallets" is null
```

The nastier production version: someone adds a `@Bean` method that calls
`new WalletService()` by hand. The context starts cleanly. The bean is registered.
Every wallet debit NPEs at request time, and the startup logs are spotless.

**Root cause:** the constructor does not require the dependency, so the type system
permits an object in an invalid state. Spring's reflection fills the field *only* for
beans it creates itself.

**Fix:** constructor injection. `new WalletService()` stops compiling. The invalid
state becomes unrepresentable — the same instinct as making an illegal state
untypable in TypeScript.

---

### Trap 2 — `@Autowired` on a field forces the field to be non-final

**Wrong:**

```java
@Service
public class InventoryService {
    @Autowired private final InventoryRepository repo;   // does not compile
}
```

**Exact symptom:** a *compile* error, not a runtime one:

```
error: variable repo not initialized in the default constructor
```

So you delete `final`:

```java
@Autowired private InventoryRepository repo;             // compiles
```

**Root cause:** a blank `final` instance field must be **definitely assigned by the
end of every constructor** — that is a JLS rule, checked by `javac`. Field injection
happens *after* the constructor returns, so the rule cannot be satisfied. Field
injection and `final` are mutually exclusive by language design, not by Spring's
choice.

The cost is the one from Topic 17: `final` fields get a **freeze at the end of
construction**, which guarantees that any thread seeing a reference to the object also
sees the correctly-initialised field, with no synchronisation. Drop `final` and you
lose that guarantee. You are then relying on Spring's singleton registry publishing
the bean through a `ConcurrentHashMap` to give you the happens-before edge instead.

**Honest scope of the harm:** in practice this rarely bites, because that
`ConcurrentHashMap` edge really is there for container-created singletons. The harm is
that you have swapped a guarantee you can point at in the JLS for an implementation
detail of a framework. At 400 rps across 200 request threads, "probably fine" is not a
design.

**Fix:**

```java
@Service
public class InventoryService {
    private final InventoryRepository repo;
    public InventoryService(InventoryRepository repo) { this.repo = repo; }
}
```

---

### Trap 3 — a circular dependency, and why it now fails loudly

**Wrong:**

```java
@Service
public class OrderService {
    private final PaymentService payments;
    public OrderService(PaymentService payments) { this.payments = payments; }
}

@Service
public class PaymentService {
    private final OrderService orders;             // <- the cycle
    public PaymentService(OrderService orders) { this.orders = orders; }
}
```

**Exact symptom:** the application **does not start**. The log ends with a Spring Boot
failure-analysis block whose shape is:

```
***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

   orderService defined in file [.../OrderService.class]
      |
   paymentService defined in file [.../PaymentService.class]
      ...
```

Underneath it, the underlying exception is
`BeanCurrentlyInCreationException: Error creating bean with name 'orderService':
Requested bean is currently in creation: Is there an unresolvable circular reference?`

**Root cause:** to construct `OrderService`, Spring must first construct
`PaymentService`; to construct `PaymentService`, it must first construct
`OrderService`, which is already in progress. `getSingleton` finds the bean name in
the "currently in creation" set and throws.

**Why setter/field cycles used to work.** Spring keeps a three-level singleton cache:
fully-created singletons, *early* singleton references, and object factories that can
produce an early reference. With setter or field injection Spring can instantiate A,
expose an early reference to the half-built A, instantiate B and hand it that early
reference, finish B, then finish populating A. With constructor injection there is no
half-built object to expose, so the trick is impossible.

**Why it now fails by default.** Spring Boot 2.6 flipped
`spring.main.allow-circular-references` to `false`. The reasoning was that the
early-reference trick hides a design problem *and* produces genuinely broken objects
in some orderings — most sharply when one side is proxied, because B may receive the
raw target while everyone else receives the proxy, so B's calls silently get no
advice (that is Topic 40's failure mode arriving through a different door). Boot 3
and Boot 4 keep the strict default.

**Fixes, worst to best:**

```properties
# 1. WORST — turn the check off. This is not a fix, it is a mute button.
spring.main.allow-circular-references=true
```

```java
// 2. Break it with @Lazy on ONE side.
@Service
public class PaymentService {
    private final OrderService orders;
    public PaymentService(@Lazy OrderService orders) { this.orders = orders; }
}
```

`@Lazy` injects a **proxy** that resolves the real `OrderService` on first method
call. The cycle disappears from startup because nothing is constructed eagerly. It
works, and it drags a proxy into your object graph — which means Topic 40's rules now
apply to that reference. Acceptable as a deliberate, commented decision. Not a
default.

```java
// 3. Defer with ObjectProvider — same idea, no proxy, explicit at the call site.
@Service
public class PaymentService {
    private final ObjectProvider<OrderService> orders;
    public PaymentService(ObjectProvider<OrderService> orders) { this.orders = orders; }

    void onAuthorized(long orderId) {
        orders.getObject().markPaid(orderId);
    }
}
```

```java
// 4. BEST — the cycle is telling you a third concept exists. Extract it.
@Service
public class OrderPaymentCoordinator {
    private final OrderService orders;
    private final PaymentService payments;
    public OrderPaymentCoordinator(OrderService orders, PaymentService payments) { ... }
}
```

Or invert the direction entirely: `PaymentService` publishes an
`ApplicationEvent`, and `OrderService` listens. That removes the compile-time
dependency instead of hiding it. For `orderflow` this is the right answer — payment
authorization *notifying* the order is a genuinely asynchronous relationship, and
modelling it as a synchronous back-reference was the original mistake.

---

### Trap 4 — `@Primary` and `@Qualifier` fighting, and the silent parameter-name match

**Wrong:**

```java
@Primary @Component class AdyenPaymentGateway implements PaymentGateway { }
@Component        class StripePaymentGateway implements PaymentGateway { }

@Service
public class RefundService {
    private final PaymentGateway gateway;
    public RefundService(PaymentGateway stripePaymentGateway) {   // parameter NAMED after a bean
        this.gateway = stripePaymentGateway;
    }
}
```

**Exact symptom:** none at startup. Refunds go to Stripe. Then someone runs a tidy-up
refactor and renames the parameter to `gateway` — still no startup error, still no
compile error — and every refund silently moves to Adyen because `@Primary` now wins.
You find out from a settlement mismatch report days later.

There is a second variant with a *different* symptom: build without `-parameters` and
parameter names are erased from the class file. Spring Boot's Maven/Gradle plugins add
`-parameters` for you, so this mostly bites in hand-rolled builds — where it appears as
a startup `NoUniqueBeanDefinitionException` on a build that worked in the IDE.

**Root cause:** parameter-name-to-bean-name matching is a *fallback* in Spring's
resolution order (type → `@Primary` → `@Qualifier` → parameter name). It is not
declared anywhere in your code, so nothing protects it from a rename.

**Fix:** never let selection depend on a name you did not write as a qualifier. Use
the custom qualifier annotation from Example 2. `@Acquirer(STRIPE)` survives every
rename, and deleting the Stripe bean becomes a startup failure rather than a
behaviour change.

---

### Trap 5 — setter injection on a mutable singleton

**Wrong:**

```java
@Service
public class PricingService {
    private DiscountPolicy policy;

    @Autowired
    public void setPolicy(DiscountPolicy policy) { this.policy = policy; }
}
```

**Exact symptom:** intermittent, unreproducible pricing differences, usually reported
as "a handful of orders got the wrong discount around 14:03". No exception, no log.
The mechanism is that `setPolicy` is `public`, so *anything* can call it — an admin
endpoint someone added, a test that leaked, a `@PostConstruct` in another bean. The
singleton is shared by every request thread, so one call changes pricing for everyone
mid-flight, and there is no memory barrier making that change visible in a defined
order.

**Root cause:** setter injection makes the dependency a mutable, publicly-writable
part of a shared singleton's API. You have added a reconfiguration back door and
called it wiring.

**Fix:** constructor injection with a `final` field. If the policy genuinely must
change at runtime, that is not injection — that is state, and it needs an explicit
thread-safe holder (`AtomicReference`, or `@RefreshScope` in a Spring Cloud setup),
with the mutability visible in the type.

**The one legitimate use of setter injection:** an *optional* dependency on a
framework or library class you do not control and cannot give a constructor to. That
is rare enough that meeting it in application code should prompt a question.

---

## Hands-on proof

Every item below is a command **you** run. I have no JVM here and will not print
output and call it real. What follows is exactly what to look for and how to read each
possible result.

### Setup

```bash
mkdir -p ~/java-lab/39 && cd ~/java-lab/39
java --version        # expect 21 or 25
mvn -v
```

Use the `orderflow` module you built in Topics 35–38, or generate a scratch app:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=di-lab \
  -d type=maven-project -o di-lab.zip && unzip di-lab.zip -d di-lab
```

> Do not pin Spring artifact versions by hand. The Boot BOM manages them; adding an
> explicit `<version>` is how you end up with the resolution problem from Topic 32.

### Proof 1 — ambiguity fails at startup, and the message names the candidates

Create the two `PaymentGateway` implementations and the ambiguous `CheckoutService`
from Example 1, then:

```bash
./mvnw spring-boot:run
```

**What to look for:** the `APPLICATION FAILED TO START` block, and the *Description*
line.

| What you see | What it means |
|---|---|
| `NoUniqueBeanDefinitionException: expected single matching bean but found 2: adyenPaymentGateway, stripePaymentGateway` | The expected result. Note it **names both candidates** — that list is your fix list. |
| `NoSuchBeanDefinitionException: No qualifying bean of type 'PaymentGateway'` | Your implementations were not scanned. Check they are in `com.orderflow` or below — Topic 36. |
| The app starts cleanly | You have exactly one candidate, or a parameter name accidentally matched a bean name. Rename the parameter to `x` and run again. |

**Why this matters:** both failures happen **at startup, before any traffic**. That is
the property field injection would still have given you, but constructor injection
gives it to you for hand-constructed objects too.

### Proof 2 — `@Qualifier` beats `@Primary`

Add `@Primary` to `AdyenPaymentGateway` and `@Qualifier("stripePaymentGateway")` to
the constructor parameter. Add a startup probe:

```java
@Component
class WhichGatewayProbe {
    WhichGatewayProbe(CheckoutService checkout) {
        System.out.println("SELECTED GATEWAY = " + checkout.which());
    }
}
```

```bash
./mvnw spring-boot:run | grep "SELECTED GATEWAY"
```

| What you see | What it means |
|---|---|
| `SELECTED GATEWAY = stripe` | Correct. `@Qualifier` at the injection point overrides `@Primary` at the definition. |
| `SELECTED GATEWAY = adyen` | Your `@Qualifier` is on the constructor rather than the parameter, or the string does not match a bean name. |
| Startup failure naming the qualifier | The string is wrong. Check the bean name — default is the class name, first letter lower-cased. |

### Proof 3 — prove the parameter-name fallback is real

Remove every `@Primary` and `@Qualifier`. Name the constructor parameter
`adyenPaymentGateway`. Run. Then rename it to `gateway` and run again.

| What you see | What it means |
|---|---|
| First run starts and prints `adyen`; second run fails with `NoUniqueBeanDefinitionException` | You have directly observed the parameter-name fallback. A variable name was load-bearing. |
| Both runs fail | Your build is not passing `-parameters` to `javac`. Confirm with the command below — and note this is exactly the environment difference that makes "works in my IDE" real. |

```bash
javap -p -c target/classes/com/orderflow/CheckoutService.class | grep -i MethodParameters
# or check the compiler args the Boot plugin passes:
./mvnw help:effective-pom | grep -A3 parameters
```

### Proof 4 — the circular dependency, and what the flag changes

Build the `OrderService` ↔ `PaymentService` cycle from Trap 3.

```bash
./mvnw spring-boot:run
```

| What you see | What it means |
|---|---|
| `APPLICATION FAILED TO START` with a bean-cycle diagram | Expected on Boot 2.6+. The diagram lists the cycle in order — read it as a directed graph. |
| Underlying `BeanCurrentlyInCreationException` | Confirms the mechanism: a bean was requested while it was already being created. |
| The app starts | You are on Boot ≤2.5, or someone set `spring.main.allow-circular-references=true`. Grep your `application.yml` and your `SpringApplication` setup. |

Now convert both sides to **field** injection and re-run:

```bash
./mvnw spring-boot:run
# then:
./mvnw spring-boot:run -Dspring-boot.run.arguments=--spring.main.allow-circular-references=true
```

| What you see | What it means |
|---|---|
| Field-injected cycle **still fails** by default | Correct on Boot 2.6+. The strict check applies to the early-reference trick too. |
| With the flag `=true`, the field-injected cycle **starts** | You have observed the three-level cache resolving the cycle via an early reference. The constructor-injected cycle will still fail even with the flag — there is no half-built object to expose. |

That last row is the whole lesson: the flag does not make cycles legal, it makes
*incompletely-initialised objects* legal.

### Proof 5 — verify the `@Fallback` annotation exists on your version

I flagged `@Fallback` (Spring Framework 6.2+) as something to check rather than
assume.

```bash
./mvnw dependency:tree | grep spring-context
unzip -l ~/.m2/repository/org/springframework/spring-beans/*/spring-beans-*.jar \
  | grep -i Fallback
```

| What you see | What it means |
|---|---|
| `org/springframework/context/annotation/Fallback.class` present | It exists. You can use it for the stub gateway instead of profiles. |
| Nothing | Your Framework version predates it. Use `@Profile` — which is the better tool here anyway, because a stub gateway should be *absent* in production, not merely deprioritised. |

### Proof 6 — see the whole resolved graph

```bash
# add spring-boot-starter-actuator, then:
./mvnw spring-boot:run &
curl -s localhost:8080/actuator/beans | jq '.contexts.application.beans
  | to_entries
  | map(select(.value.type | test("orderflow")))
  | map({bean: .key, type: .value.type, deps: .value.dependencies})'
```

**What to look for:** the `dependencies` array for `orderService`. That array is the
dependency graph Nest shows you in its module imports — except Spring computed it
from types and scanning rather than from anything you wrote down.

**How to read it:** if `orderService` lists nine dependencies, you have just produced
the evidence for the refactor conversation. This endpoint is the closest Spring gets
to Nest's explicit module graph, and it is worth knowing before you need it in an
incident.

---

## Practice exercises

### 1 — Easy: make the invalid object unconstructible

Take this class:

```java
@Service
public class InventoryService {
    @Autowired private InventoryRepository repo;
    @Autowired private InventoryMetrics metrics;

    public boolean reserve(String sku, int units) {
        return repo.decrementIfAvailable(sku, units);
    }
}
```

1. Write a plain JUnit test (no Spring) that constructs it with `new` and calls
   `reserve`. Capture the exact exception message.
2. Refactor to constructor injection with `final` fields.
3. Try to write the same test again. Describe precisely what stops you, and at which
   stage — compile or run.
4. Now try `@Autowired private final InventoryRepository repo;`. Record the exact
   `javac` error and explain, in one sentence, which language rule produced it.

### 2 — Medium: the audit (combines Topics 01–38)

This is a real-shaped `orderflow` class with **seven** distinct defects, spanning this
topic and earlier ones. Find them all. For each: name the topic, state the *observable*
symptom in production (not "bad practice"), and write the fix.

```java
package com.orderflow.orders;

@Service
public class OrderPricingService {

    @Autowired private ProductRepository products;
    @Autowired private DiscountPolicy policy;

    private Map<Long, Double> priceCache = new HashMap<>();
    private List<Long> processedOrderIds = new ArrayList<>();

    @Value("${orderflow.pricing.vat-rate}")
    private double vatRate;

    public Double priceOrder(Long orderId, List<OrderLine> lines) {

        for (Long processed : processedOrderIds) {
            if (processed == orderId) {
                return 0.0;
            }
        }

        double total = 0.0;
        for (OrderLine line : lines) {
            Double cached = priceCache.get(line.productId());
            if (cached == null) {
                cached = products.findById(line.productId()).getPrice();
                priceCache.put(line.productId(), cached);
            }
            total += cached * line.quantity();
        }

        Integer discount = policy.percentFor(orderId);
        double applied = (total > 100) ? discount : 0;

        processedOrderIds.add(orderId);
        return (total - (total * applied / 100)) * (1 + vatRate);
    }
}
```

Hints, in the order you should think about them: this bean is a **singleton shared by
every request thread** (Topic 38); two of the defects are from Topic 01; one is from
Topic 13; one is from Topic 17; two are from this topic. One of the defects is a
memory leak that only appears after days of uptime.

### 3 — Hard: production simulation — routed gateways under a failover

Advance the `orderflow` spine.

**Part A — build it.** Implement the full Example 2 structure: the `PaymentGateway`
interface, the `@Acquirer` custom qualifier with its enum attribute, `Adyen`, `Stripe`
and `Stub` implementations, and the `GatewayRouter`. Every bean uses constructor
injection with `final` fields. `StubPaymentGateway` is `@Profile("local")`.

**Part B — prove the wiring, three ways.**
1. A `@SpringBootTest` asserting that `SettlementReconciliationService` received the
   Adyen bean specifically. Assert on `gateway.network()`, not on the class name.
2. A `@SpringBootTest(properties = "spring.profiles.active=local")` asserting the
   router has **three** gateways and picks the stub first.
3. A plain JUnit test — **no Spring context at all** — that constructs
   `GatewayRouter` with a hand-made `List.of(fakeGateway)` and asserts the routing
   logic. This test must run in under 50 ms. If it needs a Spring context, your
   design has a problem; say what it is.

**Part C — the failover.** Make `GatewayHealth` report a rolling error rate.
Write a test that: routes 100 EU payments (expect all Adyen), pushes Adyen's error
rate to 5%, routes 100 more (expect all Stripe), then recovers Adyen and routes 100
more.

**Part D — break it deliberately.** Add a third gateway, `WorldpayPaymentGateway`,
with **no** `@Order` and **no** `@Acquirer`. Run the app. Record:
- what happens to `SettlementReconciliationService`;
- what happens to `GatewayRouter`'s list, and where the new gateway lands in the
  order;
- whether the app starts at all.

Then answer: which of those three outcomes would have been caught by a test, and
which only by reading the startup log? That gap is the argument for the custom
qualifier over the string one.

**Part E — the honest trade.** Argue the *other* side. `GatewayRouter` with an
injected `List` means adding a gateway changes routing behaviour with no change to any
existing file. State the circumstance under which that is a **liability** rather than a
feature, and what you would add to mitigate it. (A real answer exists and involves the
word "test".)

---

## Interview questions

### Q1 — "Why constructor injection over field injection?"

**Mid-level answer:** "Constructor injection is the recommended best practice. It
makes testing easier and Spring recommends it."

**Senior answer:** "Three concrete reasons, in order of how much they've cost me.
First, immutability: constructor injection lets the fields be `final`, which is
impossible with field injection because a blank final must be definitely assigned by
the end of the constructor — and `final` fields get the JMM freeze, so a singleton
shared across request threads is safely published without me thinking about it.
Second, testability without a container: I can `new` the class in a plain JUnit test
with fakes, and that test runs in milliseconds instead of loading a context. Third,
and the one that actually changes designs: a nine-parameter constructor is visibly
ugly, so the god class gets caught in review. Field injection makes nine dependencies
look like nine tidy lines. There's also a fourth — with constructor injection, a
missing dependency is a startup failure for *every* construction path, including a
hand-written `@Bean` method, whereas field injection only fills fields for beans
Spring itself creates."

**What separates them:** the mid answer cites authority. The senior answer gives a
language-level mechanism (definite assignment), a memory-model consequence (Topic 17),
and a **social** consequence — that the ugliness is the point. Senior engineers reason
about what a design makes easy to do wrong.

**Follow-up the interviewer asks:** "So is field injection ever acceptable?" The
honest answer is: in test classes, where `@Autowired` on a field is idiomatic and the
lifetime is a single test method, and occasionally in `@Configuration` classes. Anyone
who says "never, under any circumstances" is reciting.

---

### Q2 — "You have two `PaymentGateway` beans. Walk me through exactly how Spring picks one."

**Mid-level answer:** "It'll throw an error unless you add `@Primary` or
`@Qualifier`."

**Senior answer:** "Spring collects every bean assignable to `PaymentGateway`. If
there's exactly one, done. With more than one it narrows: first it filters to beans
matching any `@Qualifier` at the injection point — including custom qualifier
annotations and their attribute values. If no qualifier, it looks for exactly one
`@Primary`. If that doesn't resolve it, it falls back to matching the **parameter
name** against the bean name, which requires the class to have been compiled with
`-parameters`. Still ambiguous, and you get `NoUniqueBeanDefinitionException` at
startup — and the message names the candidates, which is the useful part. The
parameter-name fallback is the one I actively design against: it means a rename in a
refactor can change which acquirer processes payments, with no compile error and no
startup error. So I use a custom qualifier annotation carrying an enum, not a string,
so the compiler follows renames."

**What separates them:** naming the *order* of the resolution steps, knowing the
`-parameters` dependency, and — the real signal — treating the parameter-name fallback
as a hazard to be designed away rather than a feature.

**Follow-up:** "What if you want all of them?" They want `List<PaymentGateway>`
ordered by `@Order`, or `Map<String, PaymentGateway>` keyed by bean name, and ideally
that an empty `List` injection fails rather than injecting an empty list.

---

### Q3 — "Your app used to start with a circular dependency and now it doesn't. What changed and what do you do?"

**Mid-level answer:** "Spring Boot disabled circular references by default. You can
set `spring.main.allow-circular-references=true` to get the old behaviour back."

**Senior answer:** "Boot 2.6 flipped that property to `false`. Before that, Spring
resolved *setter and field* cycles using a three-level singleton cache: it would
instantiate A, expose an early reference to the half-built A, construct B with that
reference, then finish populating A. Constructor cycles were never resolvable, because
there's no partially-constructed object to hand out — you get
`BeanCurrentlyInCreationException`. The reason the default flipped is that the early
reference is genuinely dangerous: if A ends up proxied, B may hold the raw target
while every other collaborator holds the proxy, so B's calls silently get no
transaction or cache advice. Setting the flag back to `true` doesn't fix a cycle, it
re-enables handing out incompletely-initialised objects. What I actually do is treat
the cycle as a design signal: usually there's a third concept that wants extracting,
or the relationship should be an event rather than a synchronous back-reference.
`@Lazy` on one side works and I'd accept it with a comment explaining why, but it
inserts a proxy, so I'd want to know that reference isn't in a hot path."

**What separates them:** knowing the three-level cache exists, knowing *why* the
default changed (the proxy hazard, not tidiness), and refusing to treat the flag as a
fix.

**Follow-up:** "Show me the `@Lazy` fix and tell me what it costs." They want you to
say: a proxy per call, and every caveat from Topic 40.

---

### Q4 — "What does `@Autowired` actually do? When is it required?"

**Mid-level answer:** "It tells Spring to inject the dependency."

**Senior answer:** "It's an inert annotation — it doesn't *do* anything. It's metadata
read by `AutowiredAnnotationBeanPostProcessor`, which runs during the populate-
properties phase of the bean lifecycle and either invokes the marked constructor, calls
the marked setter, or reflectively writes the marked field. That distinction matters
coming from TypeScript, where a decorator is a function that executes at class
definition. In Java the annotation is just bytes in the class file until something
reads them. As for when it's required: since Spring 4.3, a class with exactly one
constructor uses it implicitly, so you almost never write `@Autowired` on a
constructor any more. You do need it when there's more than one constructor and you
must say which. And `required = false` exists but I prefer `ObjectProvider` for
optional dependencies, because `required = false` leaves you with a silent `null`."

**What separates them:** "annotations are inert metadata, a post-processor reads them"
is the sentence that shows you understand the container rather than memorising it. It's
also the exact mental model Topic 40 needs.

**Follow-up:** "Which lifecycle phase?" They want: after instantiation, during
populate-properties, before `@PostConstruct` — Topic 37's ordering.

---

### Q5 — "Constructor injection makes this class have eleven parameters. Is that a problem with constructor injection?"

**Mid-level answer:** "It's a bit verbose. You could use Lombok's
`@RequiredArgsConstructor`, or field injection to keep it tidy."

**Senior answer:** "No — it's a problem with the class, and constructor injection is
the thing that surfaced it. Eleven collaborators means the class is doing at least
three jobs. I'd look for the seams: which of those eleven are only used by one method,
which are notification-shaped and could become event listeners instead of
collaborators, and which cluster together and want extracting into a single
collaborator. In an `orderflow` order-placement service, audit logging and
notification are almost always listeners on an order-placed event rather than
injected dependencies — that alone typically removes two or three. `@RequiredArgs-`
`Constructor` generates exactly the same constructor, so it's harmless mechanically,
but it hides the parameter count again, which is the specific signal I wanted. I'd
apply it after the class is small, not to make a large class look small."

**What separates them:** refusing the premise. The interviewer is checking whether you
treat a code smell as a formatting problem. Naming a concrete decomposition — events
for notification and audit — is what makes it a senior answer rather than a slogan.

**Follow-up:** "What if they genuinely are all needed?" A good answer distinguishes an
orchestrator (which legitimately touches many things, and should then be *only* an
orchestrator with no logic of its own) from a service that has accreted
responsibilities.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Spring resolves by type, Nest resolves by token. Name one thing Spring's approach
   makes *easier* and one thing it makes *more dangerous*. Then say which you would
   choose for a 40-service codebase, and why.

2. The parameter-name fallback requires `-parameters` at compile time. Why does a
   language that erases generics (Topic 06) nevertheless bother to preserve parameter
   names in the class file, and what does that tell you about what erasure was
   actually for?

3. `@Lazy` breaks a circular dependency by injecting a proxy. Predict one thing that
   will behave differently about that reference compared to the real bean. (You have
   not read Topic 40 yet. Guess, write it down, and check your guess after Topic 40.)

4. Field injection cannot use `final` because of definite assignment. Suppose Java
   changed that rule tomorrow and let frameworks reflectively write final fields after
   construction. Would field injection then be acceptable? Give the strongest argument
   for yes, then defeat it.

5. Injecting `List<PaymentGateway>` fails at startup if there are zero candidates,
   rather than injecting an empty list. Argue that this is the wrong default. Then
   argue it is the right one. Which argument is stronger, and does your answer change
   between a library and an application?

6. A colleague proposes a lint rule banning `@Autowired` on fields, repo-wide, with no
   exceptions. What breaks? (Think about test classes and `@Configuration`.) Would you
   ship the rule anyway?

7. You have one `PaymentGateway` bean today. You add `@Primary` to it "for safety,
   before we add a second". Is that a good instinct or a bad one? Justify it as a code
   review comment.

---

## Quick reference card

### Resolution order when injecting by type

```
1. Collect all beans assignable to the declared type
2. If exactly one          -> inject it
3. Filter by @Qualifier at the injection point (incl. custom qualifier annotations)
4. Filter by @Primary at the bean definition
5. Filter by parameter name == bean name        (requires -parameters)
6. Zero candidates         -> NoSuchBeanDefinitionException      (startup)
   More than one           -> NoUniqueBeanDefinitionException    (startup)
```

### The three styles at a glance

| | Constructor | Setter | Field |
|---|---|---|---|
| Fields can be `final` | **yes** | no | no |
| Object valid on creation | **yes** | no | no |
| `new` in a test works | **yes** | partly | no |
| Missing dep fails at | **construction** | startup | startup (for Spring-made beans only) |
| God class visible | **yes** | no | no |
| Resolves a cycle | no | yes, with the flag | yes, with the flag |
| Use it for | everything | an optional dep on a class you do not control | test classes only |

### Annotations

```java
@Autowired                       // optional on a single constructor since Spring 4.3
@Autowired(required = false)     // skip if absent; leaves a null — prefer ObjectProvider
@Qualifier("beanName")           // narrow by name; string, not compiler-checked
@Primary                         // default winner; loses to @Qualifier
@Fallback                        // Framework 6.2+; chosen only if nothing else matches
@Value("${key:default}")         // a property, not a bean
@Value("#{expression}")          // SpEL — evaluated; almost always a mistake in app code
@Lazy                            // inject a resolve-on-first-use proxy; breaks cycles
@Profile("local")                // whether the definition is registered at all
jakarta.inject.Inject            // ≈ @Autowired
jakarta.annotation.Resource      // BY NAME first, then type — different semantics
```

### `ObjectProvider<T>` cheat sheet

```java
provider.getObject()                   // throws if absent or ambiguous
provider.getIfAvailable()              // null if absent; throws if ambiguous
provider.getIfAvailable(() -> fallback)
provider.getIfUnique()                 // null if absent OR ambiguous; never throws
provider.ifAvailable(bean -> ...)
provider.stream()                      // all candidates
provider.orderedStream()               // all candidates, sorted by @Order
```

### Multi-bean injection

```java
List<PaymentGateway> gateways          // all, ordered by @Order; FAILS if zero
Map<String, PaymentGateway> byName     // keyed by bean name; FAILS if zero
ObjectProvider<PaymentGateway>         // zero is fine; resolution deferred
```

### Circular dependency properties

```properties
spring.main.allow-circular-references=false   # the default since Boot 2.6. Leave it.
```

### Gotchas checklist

- [ ] One constructor → no `@Autowired` needed.
- [ ] `@Qualifier` goes on the **parameter**, not the constructor.
- [ ] `@Qualifier` at the injection point beats `@Primary` at the definition.
- [ ] Only one `@Primary` per type.
- [ ] Parameter names are a resolution fallback. Never let one be load-bearing.
- [ ] `@Resource` matches by **name** first. `@Autowired` matches by **type** first.
- [ ] A circular dependency is a design signal, not a configuration problem.
- [ ] `@Lazy` and `@Autowired(required=false)` both hide something. Comment them.
- [ ] Field injection in test classes is fine. In production code it is not.
- [ ] `List.copyOf` an injected `List` before storing it in a `final` field.

---

## When would I use this at work?

**1. Reviewing a pull request that adds a dependency.**
Someone adds a tenth `@Autowired` field to `OrderService`. With constructor injection
the diff shows a tenth constructor parameter, and the review comment writes itself:
"what job is this class doing now?" This is the single highest-frequency payoff of
the topic — it turns an architectural drift into a visible line in a diff.

**2. The morning after a config-driven incident.**
Payments started going to the wrong acquirer and nobody deployed a payment change.
You go straight to `/actuator/beans`, look at what `refundService` actually resolved,
and find the renamed parameter. Without knowing the resolution order you would be
reading payment code for two hours.

**3. Making a slow test suite fast.**
A team has 400 `@SpringBootTest` classes because every service needs a context to be
constructible. Converting the leaf services to constructor injection lets most of
those become plain JUnit tests with hand-made fakes — context loads drop from 400 to
a handful. This is Topic 60's argument, but the *enabling* change is made here, and
it is one of the most visible wins a new senior can deliver in their first month.

---

## Connected topics

**Prerequisites:**
- **17 — Immutability, `final`, safe publication**: why `final` fields are the point,
  not a style choice. The definite-assignment rule and the final-field freeze are both
  from there.
- **35 — `ApplicationContext`**: the two-phase startup. Definitions are registered
  before anything is instantiated, which is why `@Profile` can remove a bean entirely
  and why resolution failures are startup failures.
- **36 — Bean definition and component scanning**: where the candidate beans come
  from, and why the graph is not written down anywhere.
- **37 — Bean lifecycle**: `@Autowired` is processed during populate-properties, by a
  `BeanPostProcessor`, before `@PostConstruct`.
- **38 — Bean scopes**: why a singleton holding mutable injected state is a
  cross-request bug, and what a scoped proxy inserts.

**This unlocks:**
- **40 — Proxying mechanics**: `@Lazy` already handed you a proxy. Topic 40 explains
  what that object actually is, and why `this.method()` never touches it. Read it
  next; it is the highest-leverage document in this phase.
- **41 — AOP**: aspects are interceptors on the proxies from Topic 40, and `@Order` —
  which you met here for `List` injection ordering — is what sequences them.
- **43 — Configuration and `@ConfigurationProperties`**: the typed, validated
  replacement for scattered `@Value`.
- **45 — Bean Validation**: `@Validated` on a service is proxy-based, so it inherits
  every constraint from Topic 40 — and constructor validation is where injection and
  validation meet.
- **54–55 — `@Transactional`**: the transaction proxy sits on the beans you just
  wired. A cycle broken with `@Lazy` is a place where advice can go missing.
- **60 — Spring test slices**: constructor injection is what makes a service testable
  without a context. The fast test suite is bought here.
- **110 — Spring Cache**: `@Cacheable` is proxy-based; same inheritance.
- **111 — Resilience4j**: the `GatewayRouter` failover you built by hand in Exercise 3
  becomes a circuit breaker.
- **118 — Metrics**: `PaymentMetrics` in Example 2 becomes a real Micrometer timer,
  tagged by acquirer — and *not* by order ID.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0, `jakarta.*`
throughout. The mechanics in this topic have been stable since Spring 4.3 (2016) with
one behavioural change worth dating: circular references became a startup failure by
default in Boot 2.6 (2021), and Boot 3 and 4 keep that default. `@Fallback` is the
only genuinely new annotation here, from Framework 6.2 — verify it on your version
with the command in Proof 5 rather than taking my word for it.*
