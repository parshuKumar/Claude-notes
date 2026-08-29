# 35 — `ApplicationContext` as an IoC Container

## Phase: 4 — Spring Core
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: the `orderflow` skeleton is born here — `CatalogService`, `InventoryService`, `WalletService`, `PaymentGateway` and `OrderService`, wired by an `ApplicationContext`, backed by in-memory stubs. No HTTP, no database yet.

---

## ELI5 anchor

Imagine a restaurant kitchen.

**Without a container**, every chef fetches their own ingredients. The pastry chef
walks to the store for flour. The sauce chef walks to the store for cream. If the
store moves, every chef must be retrained.

**With a container**, there is one storeroom manager. Chefs do not shop. Each writes
a card: "I need flour and cream." The manager reads every card before the shift,
works out what must exist, buys it once, and hands each chef what their card asked
for.

That manager is the `ApplicationContext`.

Now the part specific to Spring, and the reason this document exists:

**The manager reads every card first, and only then goes shopping.**

Two separate passes. Pass one: collect every card, write a shopping list. Pass two:
buy things.

Because the list is complete before anything is bought, the manager can *edit the
list*. Cross out "cream", write "oat cream", because a card declared a dairy
allergy. Skip flour entirely because a card says "only if the oven works."

You cannot edit a shopping list that has already been spent. That split is the most
valuable idea here.

---

## The bridge from what you know

### Nest's DI container ≈ Spring's `ApplicationContext` — HONEST ANALOGUE

Let us be direct. You already know all of this:

- Inversion of control: a class declares what it needs, something else supplies it.
- A container that owns construction and lifetime.
- Singleton-by-default providers.
- Constructor injection.
- Testing by supplying a different implementation.

Nest's container and Spring's `ApplicationContext` do **the same job**. Same
concept, same defaults. This is an honest analogue, not a stretched metaphor.

```ts
// NestJS
@Injectable()
export class OrderService {
  constructor(
    private readonly inventory: InventoryService,
    private readonly wallet: WalletService,
  ) {}
}
```
```java
// Spring
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

If that were the whole story this topic would be one page.

### The real difference: TWO-PHASE STARTUP

The honest comparison is subtler than most write-ups admit. Nest **also** has two
passes: `NestFactory.create(AppModule)` runs a `DependenciesScanner` that records
every provider into a container, and only then does an `InstanceLoader` construct
instances. "Scan then instantiate" is not unique to Spring.

Three things *are* different, and they are what the rest of Phase 4 rests on.

**1. Spring's metadata phase is a documented, public extension point.**
The recorded metadata is an object you can hold: a `BeanDefinition`. There is an
official interface — `BeanFactoryPostProcessor` — whose entire purpose is "here is
the complete registry of definitions, before any bean object exists; modify it."
Nest's scan phase has no supported equivalent.

**2. Spring's graph is decided by conditions evaluated during the metadata phase.**
Boot's auto-configuration registers hundreds of candidate definitions, then asks of
each: is this class on the classpath? has the user already defined this bean? is
this property set? Failing candidates are dropped *before* instantiation. None of
that is expressible if you instantiate as you resolve.

**3. Spring has ONE flat namespace; Nest has module encapsulation.**
In Nest, a provider in `OrdersModule` is invisible to `PaymentsModule` unless
`OrdersModule` **exports** it and `PaymentsModule` **imports** the module. Real
encapsulation, enforced at wiring time. In Spring, one `ApplicationContext` is one
flat map of beans by name. There is no module boundary. Any bean can be injected
into any other bean in the same context. Java package-private visibility (Topic 03)
is the only barrier, and it is a compile-time one, not a container one.

| Concept | Nest | Spring | Verdict |
|---|---|---|---|
| IoC container | DI container | `ApplicationContext` | **HONEST ANALOGUE** |
| Singleton default | yes | yes | **HONEST ANALOGUE** |
| Constructor injection | yes | yes, preferred | **HONEST ANALOGUE** |
| Metadata pass before instantiation | yes, internal | yes, **public API** | **PARTIAL** |
| Graph source | explicit `imports` in a file | classpath scan + conditions | **PARTIAL** (Topic 36) |
| Provider visibility | module-scoped, needs `exports` | **flat — everything sees everything** | **NO ANALOGUE** |
| Startup validation | resolves the graph | eagerly builds every singleton, fails fast | **PARTIAL** |
| Runtime object identity | the class you wrote | possibly a **proxy subclass** | **NO ANALOGUE** (Topic 40) |

That last row will cost you a day of debugging at some point. Carry the fact today:
the object in the container may not be the object you wrote.

---

## What is this?

An `ApplicationContext` is a Java object that holds a registry of **bean
definitions**, creates the objects they describe in dependency order, keeps them for
the life of the application, runs their lifecycle callbacks (Topic 37), publishes
events, and exposes configuration.

Terms, defined properly, because Phase 4 uses them constantly:

**Bean** — an object whose construction and lifetime the container owns. That is the
whole definition. It is not a special kind of class. `OrderService` is a plain Java
class; it is "a bean" because the container made it.

**Bean definition** — the *metadata* describing how to produce a bean: which class,
which scope, which constructor arguments, which init method, whether it is lazy,
whether it is primary. It is data. It exists before any object does. In code:
`org.springframework.beans.factory.config.BeanDefinition`.

**`BeanFactory`** — the minimal container interface: `getBean(...)` and little else.

**`ApplicationContext`** — `BeanFactory` plus what an application needs: events,
resources, messages, `Environment`, and automatic detection of the post-processor
types below. In practice you always have one.

**`refresh()`** — the method that starts the container. Everything in this topic
happens inside one call to it.

### The two phases, precisely

`refresh()` is a fixed sequence. You need not memorise all of it, but you must be
able to place the boundary. Simplified, in order:

```
PHASE 1 — METADATA. None of your bean instances exist yet.
  1. prepareRefresh()                  reset state, validate required properties
  2. obtainFreshBeanFactory()          create the factory, LOAD BEAN DEFINITIONS
  3. prepareBeanFactory()              register built-in dependencies
  4. postProcessBeanFactory()          subclass hook
  5. invokeBeanFactoryPostProcessors() <-- DEFINITIONS CAN BE REWRITTEN HERE.
                                           Component scanning and @Configuration
                                           parsing actually run here too.
  6. registerBeanPostProcessors()      instantiate the processors that wrap beans

PHASE 1.5 — INFRASTRUCTURE
  7-10. message source, event multicaster, onRefresh (Boot starts the web
        server here), register listeners

PHASE 2 — INSTANTIATION. Now objects get created.
 11. finishBeanFactoryInitialization() <-- EVERY NON-LAZY SINGLETON IS CREATED HERE
 12. finishRefresh()                    publish ContextRefreshedEvent
```

Step 5 versus step 11 is the split. Between them, every definition in the
application is known, and not one of your objects has been constructed.

**What that split buys — all of it later in the curriculum:**

- A `@Bean` method in your configuration can override an auto-configured one,
  because both are definitions and one is registered later. (Topic 42)
- `@ConditionalOnMissingBean` can ask "did the user define this?" — answerable only
  if all definitions are known before any instantiation. (Topic 42)
- `BeanPostProcessor`s are instantiated at step 6, ahead of ordinary beans, so they
  are available to wrap every bean created at step 11. That wrapping is how
  `@Transactional` and `@Cacheable` work. (Topics 37, 40, 54)
- `${orderflow.catalog.page-size}` placeholders are resolved by a
  `BeanFactoryPostProcessor` at step 5, editing definitions before construction.

---

## Why does it matter?

**1. Startup failures become readable.** Spring's errors are long. Knowing the split
lets you classify any of them in seconds: phase 1 means "your metadata is wrong — a
bad scan, a missing class, a duplicate definition"; phase 2 means "your objects
cannot be built — a missing dependency, an ambiguous one, a throwing constructor."
That question alone halves the search space.

**2. Eager singleton instantiation is a feature you must not casually discard.**
By default Spring constructs every singleton at startup. If `OrderService` needs a
`PaymentGateway` and none exists, the application **refuses to start**. It does not
start and then fail on the first customer order at 3 a.m. When you later reach for
`spring.main.lazy-initialization=true` to speed up local development, understand you
are trading that safety away.

**3. It is the substrate for the next twenty topics.** Transactions, caching,
security, scheduling, retries, metrics and every starter you will ever add are beans
that wrap or observe other beans. If the container is a black box, all of that is
magic. If it is a two-phase registry, all of it is mechanism.

---

## Syntax breakdown

### `@Configuration`

```java
@Configuration
public class CatalogConfig { }
```

A **source of bean definitions**. Roughly Nest's `@Module`, with a crucial
difference: it declares beans; it does **not** declare visibility or imports. There
is nothing to export.

By default `@Configuration` classes are themselves subclassed at runtime by CGLIB
(`proxyBeanMethods = true`). That is what makes inter-bean calls work, below.

### `@Bean`

```java
@Configuration
public class CatalogConfig {

    @Bean
    public ProductRepository productRepository() {
        return new InMemoryProductRepository();
    }

    @Bean
    public CatalogService catalogService(ProductRepository repository) {
        return new CatalogService(repository);
    }
}
```

| Bit of syntax | What it means |
|---|---|
| `@Bean` on a method | calling this method produces a bean; the **return value** is the bean |
| method name `productRepository` | becomes the **bean name**. Rename the method, rename the bean. |
| return type `ProductRepository` | the bean's **type for injection**. Declare the interface and matching happens by interface. |
| parameter `ProductRepository repository` | the container resolves this by type and passes it in |
| body `new InMemoryProductRepository()` | ordinary Java. The escape hatch for classes you cannot annotate. |

**The inter-bean call rule.** Inside a `proxyBeanMethods = true` configuration class
you may write `new CatalogService(productRepository())` and `productRepository()`
will **not** run twice — the CGLIB subclass intercepts and returns the existing
singleton. Set `proxyBeanMethods = false` and that same line becomes a plain Java
call producing a second, unmanaged instance. Prefer the method-parameter form: it is
correct under both settings. (Trap 5.)

### `@Component`, `@Service`, `@Repository`

```java
@Service
public class InventoryService { }
```

"Find this class by scanning and register a definition." Topic 36 covers it
properly. For today: `@Service` and `@Repository` are `@Component` with a different
label. `@Repository` additionally opts into persistence exception translation
(Topic 47) — the only one of the three with extra runtime behaviour.

### `@SpringBootApplication`

```java
package com.orderflow;

@SpringBootApplication
public class OrderflowApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderflowApplication.class, args);
    }
}
```

Three annotations in one:

| Part | Effect |
|---|---|
| `@SpringBootConfiguration` | this class is itself a `@Configuration` |
| `@ComponentScan` | scan **this class's package and everything below it** |
| `@EnableAutoConfiguration` | register Boot's conditional auto-configurations (Topic 42) |

The scan root is the package of the annotated class. Getting this wrong is Trap 2,
and it is a genuine first-week failure.

### Creating a context by hand

```java
try (var context = new AnnotationConfigApplicationContext(CatalogConfig.class)) {
    CatalogService catalog = context.getBean(CatalogService.class);
    System.out.println(catalog.findBySku("SKU-1001"));
}
```

It implements `AutoCloseable`, so try-with-resources (Topic 08) closes it and runs
destruction callbacks (Topic 37).

---

## Example 1 — minimal

Two classes and one configuration. No Boot, no web server, no scanning. The point is
to see the container as an ordinary object you construct.

```java
package com.orderflow.catalog;

// Money as minor units, per Topic 01. Never double.
public record Product(String sku, String name, long priceMinor, boolean active) { }

public interface ProductRepository {
    java.util.Optional<Product> findBySku(String sku);
    int count();
}
```

```java
package com.orderflow.catalog;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryProductRepository implements ProductRepository {

    private final Map<String, Product> bySku = new ConcurrentHashMap<>();

    public InMemoryProductRepository() {
        bySku.put("SKU-1001", new Product("SKU-1001", "Cast iron skillet", 3499L, true));
        bySku.put("SKU-1002", new Product("SKU-1002", "Enamel stockpot",  5299L, true));
    }

    @Override public Optional<Product> findBySku(String sku) {
        return Optional.ofNullable(bySku.get(sku));   // Topic 26: absence in the signature
    }

    @Override public int count() { return bySku.size(); }
}

public class CatalogService {
    private final ProductRepository repository;      // final: Topic 17

    public CatalogService(ProductRepository repository) { this.repository = repository; }

    public Optional<Product> findBySku(String sku) { return repository.findBySku(sku); }
}
```

```java
package com.orderflow.catalog;

import org.springframework.context.annotation.*;

@Configuration
public class CatalogConfig {

    @Bean public ProductRepository productRepository() {
        return new InMemoryProductRepository();
    }

    @Bean public CatalogService catalogService(ProductRepository repository) {
        return new CatalogService(repository);
    }
}
```

```java
package com.orderflow;

import com.orderflow.catalog.*;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import java.util.Arrays;

public class ContainerDemo {
    public static void main(String[] args) {
        try (var context = new AnnotationConfigApplicationContext(CatalogConfig.class)) {

            Arrays.stream(context.getBeanDefinitionNames()).sorted().forEach(System.out::println);

            CatalogService catalog = context.getBean(CatalogService.class);
            System.out.println(catalog.findBySku("SKU-1001"));

            System.out.println(context.getBean(CatalogService.class)
                            == context.getBean(CatalogService.class));   // singleton?
        }
    }
}
```

Three things worth noticing about `CatalogService`:

1. It has **no Spring annotation at all**. It is a bean only because a `@Bean`
   method produced it. Framework coupling is optional, and pushing it to the edges
   is a real design lever.
2. Its field is `final`. Constructor injection makes that possible, which matters
   for safe publication (Topic 17) and is Topic 39's headline argument.
3. It depends on the **interface**, not the in-memory class. That is what lets
   Topic 47 swap in a Postgres repository without touching this file.

---

## Example 2 — production scenario (on the project spine)

### The situation and its constraints

`orderflow` is an order-and-payment service. First milestone: place an order end to
end — check stock, reserve it, debit the wallet, record a payment, produce an order.

Constraints, stated up front because they drive the design:

- **Traffic:** peak 400 order placements per second, plus 3 000 rps of catalogue reads.
- **Catalogue size:** 100 000 active products.
- **SLO:** p99 for `POST /orders` under 250 ms.
- **Deployment:** Kubernetes, rolling deploys, roughly 40 pod starts per day. A pod
  that starts broken must fail readiness rather than serve errors.

That last constraint is why eager instantiation matters here and is not academic. If
the container cannot build the graph, the pod dies at start and the rollout halts.
Make everything lazy and the pod starts healthy, passes readiness, takes traffic,
and *then* fails. The blast radius differs by orders of magnitude.

### The package layout — fixed now, used for the rest of the curriculum

```
com.orderflow
├── OrderflowApplication.java
├── catalog     Product, ProductRepository, CatalogService
├── inventory   InventoryService, InsufficientStockException
├── orders      Order, OrderLine, OrderStatus, OrderService
├── payments    PaymentGateway, PaymentResult, WalletPaymentGateway
└── wallet      WalletService, InsufficientFundsException
```

Topic 36 justifies this layout against the alternatives.

### Domain types

```java
package com.orderflow.inventory;

public class InsufficientStockException extends RuntimeException {
    public InsufficientStockException(String sku, int requested, int available) {
        super("Insufficient stock for %s: requested %d, available %d"
                .formatted(sku, requested, available));
    }
}
```
```java
package com.orderflow.wallet;

public class InsufficientFundsException extends RuntimeException {
    public InsufficientFundsException(String customerId, long requestedMinor, long balanceMinor) {
        super("Wallet %s has %d minor units, needs %d"
                .formatted(customerId, balanceMinor, requestedMinor));
    }
}
```

Both unchecked, deliberately — Topic 09's argument: the caller has no alternative
action beyond failing the order, and a checked exception would leak through every
lambda and signature between here and the controller.

```java
package com.orderflow.orders;

public enum OrderStatus { PENDING, PAID, REJECTED }

public record OrderLine(String sku, int quantity, long unitPriceMinor) {
    public long lineTotalMinor() { return unitPriceMinor * quantity; }
}

public record Order(String orderId, String customerId, java.util.List<OrderLine> lines,
                    long totalMinor, OrderStatus status, java.time.Instant placedAt) {
    public Order {
        // Compact constructor (Topic 27) + defensive copy (Topic 17): a record
        // wrapping a mutable List is not immutable without this.
        lines = java.util.List.copyOf(lines);
    }
}
```

### The services

```java
package com.orderflow.inventory;

import org.springframework.stereotype.Service;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class InventoryService {

    // In-memory stub. Topic 47 replaces this with a repository;
    // Topic 52 makes the decrement correct under contention.
    private final Map<String, Integer> availableBySku =
            new ConcurrentHashMap<>(Map.of("SKU-1001", 25, "SKU-1002", 8));

    public int available(String sku) {
        return availableBySku.getOrDefault(sku, 0);   // Topic 01 Trap 2: never .get() into an int
    }

    public void reserve(String sku, int quantity) {
        availableBySku.compute(sku, (key, current) -> {
            int have = current == null ? 0 : current;
            if (have < quantity) throw new InsufficientStockException(sku, quantity, have);
            return have - quantity;
        });
    }

    public void release(String sku, int quantity) {
        availableBySku.merge(sku, quantity, Integer::sum);
    }
}
```

```java
package com.orderflow.wallet;

import org.springframework.stereotype.Service;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class WalletService {

    private final Map<String, Long> balanceMinorByCustomer =
            new ConcurrentHashMap<>(Map.of("CUST-77", 100_000L));

    public long balanceMinor(String customerId) {
        return balanceMinorByCustomer.getOrDefault(customerId, 0L);
    }

    public void debit(String customerId, long amountMinor) {
        balanceMinorByCustomer.compute(customerId, (key, current) -> {
            long have = current == null ? 0L : current;
            if (have < amountMinor) throw new InsufficientFundsException(customerId, amountMinor, have);
            return have - amountMinor;
        });
    }

    public void credit(String customerId, long amountMinor) {
        balanceMinorByCustomer.merge(customerId, amountMinor, Long::sum);
    }
}
```

```java
package com.orderflow.payments;

public interface PaymentGateway {
    String name();
    PaymentResult charge(String customerId, long amountMinor, String idempotencyKey);
}

public record PaymentResult(boolean approved, String reference, String declineReason) {
    public static PaymentResult approved(String reference) {
        return new PaymentResult(true, reference, null);
    }
    public static PaymentResult declined(String reason) {
        return new PaymentResult(false, null, reason);
    }
}
```

> `PaymentResult` is a plain record today. Topic 28 would model it as a sealed
> interface over `Approved` and `Declined` records so the compiler forces you to
> handle both. That refactor happens in Topic 39 when a second gateway arrives.

```java
package com.orderflow.payments;

import com.orderflow.wallet.WalletService;
import org.springframework.stereotype.Component;
import java.util.UUID;

@Component
public class WalletPaymentGateway implements PaymentGateway {

    private final WalletService wallet;

    public WalletPaymentGateway(WalletService wallet) { this.wallet = wallet; }

    @Override public String name() { return "wallet"; }

    @Override
    public PaymentResult charge(String customerId, long amountMinor, String idempotencyKey) {
        try {
            wallet.debit(customerId, amountMinor);
            return PaymentResult.approved("WLT-" + UUID.randomUUID());
        } catch (RuntimeException e) {
            return PaymentResult.declined(e.getMessage());
        }
    }
}
```

```java
package com.orderflow.orders;

import com.orderflow.catalog.*;
import com.orderflow.inventory.InventoryService;
import com.orderflow.payments.*;
import org.springframework.stereotype.Service;
import java.time.Instant;
import java.util.*;

@Service
public class OrderService {

    private final CatalogService catalog;
    private final InventoryService inventory;
    private final PaymentGateway paymentGateway;

    public OrderService(CatalogService catalog, InventoryService inventory,
                        PaymentGateway paymentGateway) {
        this.catalog = catalog;
        this.inventory = inventory;
        this.paymentGateway = paymentGateway;
    }

    public Order place(String customerId, Map<String, Integer> quantityBySku) {

        List<OrderLine> lines = new ArrayList<>();
        for (var entry : quantityBySku.entrySet()) {
            Product product = catalog.findBySku(entry.getKey())
                    .orElseThrow(() -> new UnknownProductException(entry.getKey()));
            lines.add(new OrderLine(product.sku(), entry.getValue(), product.priceMinor()));
        }

        long totalMinor = lines.stream().mapToLong(OrderLine::lineTotalMinor).sum();
        lines.forEach(line -> inventory.reserve(line.sku(), line.quantity()));

        PaymentResult result =
                paymentGateway.charge(customerId, totalMinor, UUID.randomUUID().toString());

        if (!result.approved()) {
            lines.forEach(line -> inventory.release(line.sku(), line.quantity()));
            return new Order(UUID.randomUUID().toString(), customerId, lines,
                             totalMinor, OrderStatus.REJECTED, Instant.now());
        }
        return new Order(UUID.randomUUID().toString(), customerId, lines,
                         totalMinor, OrderStatus.PAID, Instant.now());
    }
}
```

> **This method is not correct yet and I will not pretend otherwise.** There is no
> atomicity: if `charge` throws rather than returning a decline, the reservation
> leaks. There is no idempotency across restarts. There is a race between two
> concurrent placements against the last unit of stock. Those are Topics 54, 116 and
> 52. Today's job is the *wiring*, and the wiring is real.

`CatalogService` gets `@Service` and `InMemoryProductRepository` gets `@Repository`
now that we are scanning. Topic 36 explains why the repository is annotated here
rather than produced by a `@Bean` method, and when you would choose the other way.

### The POM, and why Topics 31–32 are load-bearing

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <!-- Do not copy a version from any document, including this one.
       Generate a project at https://start.spring.io and read the version
       it puts here. That is authoritative for today. -->
  <version>CHECK-CURRENT</version>
</parent>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <!-- no <version>: the parent BOM's dependencyManagement supplies it -->
  </dependency>
</dependencies>
```

`spring-boot-starter-parent` is a **BOM** — a `dependencyManagement` block pinning a
consistent version set, which is why no `<version>` appears on the starters. Maven's
flat nearest-wins resolution puts exactly one version of each artifact on the
classpath, and **the classpath is precisely what Spring scans**. Component scanning
is not magic; it is directory and jar traversal over whatever Maven assembled. Get
the classpath wrong and beans vanish with no compile error.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — two implementations of `PaymentGateway`

A card gateway arrives alongside the wallet one:

```java
@Component
public class CardPaymentGateway implements PaymentGateway {
    @Override public String name() { return "card"; }
    @Override public PaymentResult charge(String c, long a, String k) { /* ... */ }
}
```

**Wrong:** leaving `OrderService`'s constructor parameter as bare `PaymentGateway`.

**Exact symptom:** the application refuses to start. The failure is an
`UnsatisfiedDependencyException` wrapping a `NoUniqueBeanDefinitionException`, the
message names the injection point and lists the candidates — of the form "expected
single matching bean but found 2: cardPaymentGateway, walletPaymentGateway". Boot
also prints a `Description` / `Action` block.

**Root cause:** injection is **by type**. Two definitions satisfy `PaymentGateway`
and the container will not guess. Nest does not have this failure mode because you
inject by *token*, not by structural type.

**Fix, in increasing order of quality:**

1. `@Primary` on one implementation — right when there genuinely is a default.
2. `@Qualifier("card")` at the injection point — explicit and greppable (Topic 39).
3. Inject `List<PaymentGateway>` and select at runtime by `name()`. Usually the
   right design for a payments router: you want the set, not one.
4. Inject `Map<String, PaymentGateway>` — Spring fills it with bean name to bean.
   Handy, slightly too clever: bean names become an implicit contract.

Note the good news: this failed at startup, loudly, listing both candidates. It
could not reach production. Eager instantiation doing its job.

---

### Trap 2 — a `@Component` outside the scan root

**Wrong:** `OrderflowApplication` sits in `com.orderflow.orders` because that is
where you were working when you created it, while services live in sibling packages.

**Exact symptom:** startup fails with `NoSuchBeanDefinitionException` naming the
missing type, wrapped in `UnsatisfiedDependencyException` naming the constructor
parameter that could not be satisfied.

A crueller variant: **nothing fails**. The bean was only optional — a
`MeterRegistry` customiser, an `ApplicationRunner`. It silently does not exist, and
months later you discover a metric was never recorded.

**Root cause — the scan root rule, stated precisely:**

> `@SpringBootApplication` implies `@ComponentScan` with **no** `basePackages`
> attribute. When `basePackages` is absent, Spring uses the package **of the class
> carrying the annotation** as the single base package, and scans that package and
> all sub-packages, recursively.

With `OrderflowApplication` in `com.orderflow.orders`:

| Class | Scanned? |
|---|---|
| `com.orderflow.orders.OrderService` | yes |
| `com.orderflow.orders.internal.PricingRules` | yes — sub-package |
| `com.orderflow.payments.CardPaymentGateway` | **no** — sibling, not below |
| `com.orderflow.catalog.CatalogService` | **no** |
| `com.example.shared.ClockProvider` | **no** — different root entirely |

Sibling packages are not below. This catches everyone once.

**Fix (best first):**

1. Put the application class in the **root** package, `com.orderflow`. That is the
   convention for a reason; everything else follows.
2. If a legacy layout forbids that, be explicit — and prefer
   `@SpringBootApplication(scanBasePackageClasses = {OrderflowApplication.class, SharedMarker.class})`
   over the `scanBasePackages` string form. A class reference survives a package
   rename; a string does not.
3. For beans from a **third-party jar**, scanning is the wrong tool entirely. You do
   not own those classes and they are not in your package. Write a `@Bean` method,
   or rely on the library's auto-configuration (Topic 42).

**Ten-second diagnosis:** print sorted bean definition names (Hands-on proof) and
grep for the class. "Present under a surprising name" is a different problem from
"absent".

---

### Trap 3 — enabling lazy initialisation to make startup faster

**Wrong:** `spring.main.lazy-initialization=true` in `application.yml` because
startup takes 9 seconds and that feels slow.

**Exact symptom:** the application starts fine, readiness passes, and then the first
request to a rarely-used endpoint returns HTTP 500 with a `BeanCreationException` in
the log — a container error, at request time, on a request thread. A customer is
holding a broken checkout page while the pod looks perfectly healthy to monitoring.

Worse: a misconfiguration that every pod would have caught at startup is now
discovered only on the first request touching that specific bean — possibly a
nightly batch path exercised once a day.

**Root cause:** lazy initialisation converts a definition error from a *startup*
failure into a *first-use* failure. It does not make the graph correct; it makes the
graph unverified.

**Fix:**
- Never set it in a deployed profile.
- Setting it in a **local development** profile is legitimate and is what it is for.
  Keep it in `application-local.yml`.
- For slow test suites the right lever is Topic 60's slice tests and context
  caching, not laziness.
- If production startup is genuinely too slow, measure first: run with `--debug` and
  read the condition-evaluation report to see how many auto-configurations you are
  paying for.

---

### Trap 4 — treating `@Configuration` as a Nest module

**Wrong:** assuming that because `OrderConfig` declares `orderService`, only
`orders` code can use it — and structuring the app as if that were true.

**Exact symptom:** no error. Ever. That is the problem. Six months later
`CatalogService` has a `WalletService` constructor parameter, because someone needed
a balance for a "you can afford this" badge and the container happily supplied it.
Your module boundary was never real, and `mvn dependency:analyze` will not tell you
either, because it is all one artifact.

**Root cause:** one `ApplicationContext` is one **flat** namespace.
`@Configuration` groups definitions for *your* readability. It creates no visibility
boundary. There is no `exports`.

**Fix — pick a real enforcement mechanism, because the container is not one:**

1. **Package-private classes** (Topic 03). Make `InMemoryProductRepository`
   package-private; only the `catalog` package can name the type, while Spring can
   still instantiate it reflectively. Cheapest real boundary in plain Java, and
   badly under-used.
2. **ArchUnit** tests asserting `com.orderflow.catalog..` may not depend on
   `com.orderflow.wallet..`. A failing build is a boundary.
3. **Spring Modulith**, which formalises exactly this: named application modules
   with allowed dependencies, verified by a test. It exists because the flat
   namespace is a recognised weakness.
4. **Separate Maven modules** (Topic 31). Strongest boundary, most expensive to change.

Be honest about which one you have. "We have `@Configuration` classes per feature"
is a naming convention, not an architecture.

---

### Trap 5 — expecting `@Bean` method calls to behave like Java

**Wrong:**
```java
@Configuration(proxyBeanMethods = false)
public class CatalogConfig {
    @Bean public ProductRepository productRepository() { return new InMemoryProductRepository(); }
    @Bean public CatalogService catalogService() { return new CatalogService(productRepository()); }
    @Bean public CatalogWarmer  catalogWarmer()  { return new CatalogWarmer(productRepository()); }
}
```

**Exact symptom:** no exception. Three separate `InMemoryProductRepository`
instances exist — one managed, two private. A write through `CatalogService` is
invisible to `CatalogWarmer`. In `orderflow` terms: you preload the catalogue cache
in the warmer (Topic 37) and the service still misses on every lookup — a cache hit
rate pinned at 0% with no error anywhere. The signal is a **metric**, not a stack
trace. That is what makes it nasty.

**Root cause:** `proxyBeanMethods = false` removes the CGLIB subclass that
intercepts inter-bean calls, so `productRepository()` becomes a plain Java call that
runs the body every time.

**Fix:** take the dependency as a **method parameter**, never as a direct call:

```java
@Bean public CatalogService catalogService(ProductRepository repository) { ... }
@Bean public CatalogWarmer  catalogWarmer(ProductRepository repository)  { ... }
```

Correct under both settings, and it makes the dependency visible in the signature.
Make it your default and you never have to remember which mode you are in.

> **Confirming which mode you are in:** print
> `context.getBean(CatalogConfig.class).getClass().getName()`. A CGLIB-enhanced name
> contains `$$SpringCGLIB$$`. The same diagnostic you will use constantly in Topic 40.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM and no running `orderflow`,
so I will not print output and call it captured. What follows is the exact file, the
exact command, what to look for, and how to read every result you might get.

### Setup

```bash
mkdir -p ~/java-lab && cd ~/java-lab
java --version && mvn --version

curl https://start.spring.io/starter.zip \
  -d dependencies=web -d type=maven-project -d language=java -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=orderflow -d packageName=com.orderflow \
  -o orderflow.zip
unzip orderflow.zip -d orderflow && cd orderflow
```

**What to look for:** the `spring-boot-starter-parent` version in the generated
`pom.xml`. That is your authoritative current version — use it, not a number copied
from any document.

| What you see | What it means |
|---|---|
| parent version starting `4.` | current Boot 4 line; this doc applies as written |
| parent version starting `3.` | Boot 3.x, out of OSS support since June 2026 — read every `[BOOT 3.x DELTA]` box |
| the download fails | start.spring.io parameters may have changed; use the web UI |

### Proof 1 — see the two phases, and see every definition

`src/main/java/com/orderflow/platform/ContainerInspector.java`:
```java
package com.orderflow.platform;

import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.*;
import java.util.Arrays;

@Configuration
public class ContainerInspector {

    // PHASE 1. Runs before any of your beans are instantiated.
    @Bean
    public static BeanFactoryPostProcessor definitionReporter() {
        return beanFactory -> {
            System.out.println("=== PHASE 1: definitions registered = "
                    + beanFactory.getBeanDefinitionCount());
            Arrays.stream(beanFactory.getBeanDefinitionNames()).sorted()
                  .filter(n -> n.contains("Service") || n.contains("Gateway") || n.contains("Repository"))
                  .forEach(n -> System.out.println("  DEF  " + n + "  ->  "
                          + beanFactory.getBeanDefinition(n).getBeanClassName()));
        };
    }

    // PHASE 2 is over by the time this runs.
    @Bean
    public ApplicationRunner instanceReporter(ApplicationContext context) {
        return args -> {
            System.out.println("=== PHASE 2: singletons instantiated");
            Arrays.stream(context.getBeanDefinitionNames()).sorted()
                  .filter(n -> n.contains("Service") || n.contains("Gateway") || n.contains("Repository"))
                  .forEach(n -> System.out.println("  BEAN " + n + "  ->  "
                          + context.getBean(n).getClass().getName()));
        };
    }
}
```

> The `BeanFactoryPostProcessor` `@Bean` method is `static`. That is required, not
> stylistic: a non-static one forces its configuration class to be instantiated
> early, which can make other beans in that class initialise before post-processors
> are ready. Spring logs a warning about it. Make every `BeanFactoryPostProcessor`
> and `BeanPostProcessor` `@Bean` method `static`.

```bash
./mvnw spring-boot:run
```

**What to look for:** the `=== PHASE 1` block must appear **before** `=== PHASE 2`,
and before any constructor logging you add to your services.

| What you see | What it means |
|---|---|
| PHASE 1 lists `orderService`, `catalogService` with class names, then PHASE 2 lists them with instances | Expected. Definitions existed before objects. You have observed the split directly. |
| PHASE 1 prints a count in the hundreds | Correct — most are Boot's auto-configuration definitions (Topic 42), not yours. |
| PHASE 1 does not list a bean you expect | It was never registered. Scan root (Trap 2), or missing stereotype annotation. It is not a bean at all. |
| PHASE 1 lists it but PHASE 2 fails | Definition fine, construction not: missing or ambiguous dependency, or a throwing constructor. |
| a PHASE 2 class name contains `$$SpringCGLIB$$` or `$Proxy` | The container handed you a proxy, not your object. Expected for `@Configuration` classes. Topic 40. |

That table is the diagnostic you will use for the rest of Phase 4. Learn the table,
not the code that produces it.

### Proof 2 — prove eager instantiation, then take it away

Add to `OrderService`'s constructor:
```java
System.out.println(">>> constructing OrderService on " + Thread.currentThread());
```

```bash
./mvnw spring-boot:run
./mvnw spring-boot:run -Dspring-boot.run.jvmArguments="-Dspring.main.lazy-initialization=true"
```

**What to look for:** whether that line appears before "Started OrderflowApplication".

| What you see | What it means |
|---|---|
| normal run: appears during startup, before "Started" | eager singleton instantiation, the default; your graph was validated at boot |
| lazy run: does **not** appear at startup | confirmed — laziness defers construction, and nothing has been validated |
| lazy run: appears anyway | something forced it — an `ApplicationRunner` injecting it, or an eager bean depending on it. Laziness is not absolute. |
| lazy run is much faster and you are tempted | re-read Trap 3; keep it in the local profile only |

### Proof 3 — cause the ambiguity failure on purpose

Add `CardPaymentGateway` from Trap 1, then `./mvnw spring-boot:run`.

| What you see | What it means |
|---|---|
| `NoUniqueBeanDefinitionException` naming both gateways | expected: injection by type, two matching definitions |
| `UnsatisfiedDependencyException` wrapping it, naming `orderService` and the parameter | read the wrapper for *where*, the cause for *why*. Always read the innermost cause first. |
| Boot's `Description` / `Action` block | Boot's failure analysers — read this *before* the stack trace; it is usually the actual answer |
| no failure at all | the second bean was never registered. Trap 2 in disguise — check the package and the annotation. |

Now fix it three ways in turn and re-run each: `@Primary`, `@Qualifier`, then change
the parameter to `List<PaymentGateway> gateways` and print
`gateways.stream().map(PaymentGateway::name).toList()`. The third is the one worth keeping.

### Proof 4 — prove the flat namespace

Temporarily add a `com.orderflow.wallet.WalletService` parameter to
`CatalogService`'s constructor and print the balance from it. Run.

| What you see | What it means |
|---|---|
| it starts and prints the balance | confirmed: there is **no** module boundary. Anything injects anything. This is the honest difference from Nest. |
| it fails | something else is wrong — check the scan root. It should succeed. |

Revert it. Then, if you want the boundary to be real, add the ArchUnit test from
Trap 4 and watch this same change fail the build instead.

### Proof 5 — count what Boot registered for you

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug
```

**What to look for:** a section headed "CONDITIONS EVALUATION REPORT", with
"Positive matches" and "Negative matches".

| What you see | What it means |
|---|---|
| long lists under both headings | normal. Each entry is an auto-configuration whose conditions were evaluated in **phase 1** — proof that conditions run against definitions, before instantiation. |
| an auto-configuration you expected under "Negative matches" | read the stated reason: usually `@ConditionalOnClass did not find required class` (missing dependency) or `@ConditionalOnMissingBean found bean X` (you already defined one) |
| nothing printed | `--debug` did not reach the application; try setting `debug: true` in `application.yml` instead |

You are not expected to understand this report today — it is Topic 42. Look at it
now purely as evidence that the metadata phase is where decisions get made.

---

## Practice exercises

### 1 — easy: build a context by hand and interrogate it

No Spring Boot. One `main`, one `@Configuration`, `AnnotationConfigApplicationContext`.

1. Declare `ProductRepository`, `CatalogService`, `InventoryService` and
   `WalletService` as `@Bean` methods. **None of these classes may carry a Spring
   annotation** — prove to yourself that framework coupling is optional.
2. Print `getBeanDefinitionNames()` sorted, and `getBeanDefinitionCount()`. Compare
   the count to a Boot app's. Explain the difference in one sentence.
3. Call `getBean(CatalogService.class)` twice; print whether the references are `==`.
   State what you expected *before* running it.
4. Add a print to each constructor. From the output order, state the order the
   container chose and **why** it chose that order.
5. Wrap the context in try-with-resources with a `close()`-time print. Remove the
   try-with-resources and observe what no longer happens.

Deliverable: code, output, and three sentences answering "in what order were beans
created, and what decided that order?"

### 2 — medium: combining earlier topics

Extend the minimal context with an order-total calculator, demonstrating four
earlier topics inside a Spring container.

1. **Topics 27 + 17:** prove the compact constructor's defensive copy works — build
   the list, construct the `Order`, mutate the original list, assert the order's
   lines are unchanged.
2. **Topic 01:** all money is `long` minor units. Assert a 3-line order of 19.99,
   0.10 and 0.20 totals exactly 2029. Then write the `double` version and show the
   discrepancy.
3. **Topic 26:** `findBySku` returns `Optional<Product>`; `OrderService` must handle
   absence without ever calling `get()`.
4. **Topic 32:** run `mvn dependency:tree`. Find `spring-core`, `spring-beans`,
   `spring-context`, `spring-aop`. Explain why `spring-aop` is on the classpath
   though you have written no aspect, and what would happen to component scanning if
   `spring-context` were excluded.
5. Register a `BeanFactoryPostProcessor` that prints, for every definition whose
   class name starts with `com.orderflow`, its scope, lazy flag and bean class name.
   Then set one bean's scope to prototype **from inside the post-processor** —
   `definition.setScope("prototype")` — and prove it took effect by comparing
   `getBean` results for identity.

Item 5 is the exercise. It is the thing Nest cannot do.

### 3 — hard: production simulation — the startup safety net

Your `orderflow` fleet does 40 rolling restarts a day. A pod that starts and then
fails on first request is a customer-visible incident; a pod that refuses to start
is a halted rollout with no customer impact. Build and verify that safety net.

**Part A — build the graph.** Implement the full Example 2 wiring. Add an
`ApplicationRunner` that places one order for `CUST-77` of 2 x `SKU-1001` and prints
the resulting `Order`, remaining stock, and wallet balance.

**Part B — break it four ways, one at a time.** For each record: the exception class
name, which bean the message names, whether the failure was phase 1 or phase 2, and
how long diagnosis took from the message alone.

1. Delete `@Repository` from `InMemoryProductRepository`.
2. Add `CardPaymentGateway` so two gateways exist.
3. Move `OrderflowApplication` into `com.orderflow.orders`, changing nothing else.
4. Make `WalletService`'s constructor throw `IllegalStateException("wallet ledger unreachable")`.

**Part C — the argument.** Enable `spring.main.lazy-initialization=true` and repeat
all four. Record whether each failure still happens at startup or moves to request
time. Then write a short recommendation to your team answering: which failures are
still caught at startup with laziness on? which become customer-visible? under what
**specific** circumstance would you accept lazy initialisation in a deployed
environment? Give a condition, not an opinion.

**Part D — make the boundary real.** Add an ArchUnit test asserting
`com.orderflow.catalog..` must not depend on `com.orderflow.wallet..`. Add the
forbidden dependency from Proof 4 and confirm the test fails. Then answer: does this
replace, or merely supplement, the container's flat namespace? What can it still not
catch?

---

## Interview questions

### Q1 — "What is the difference between a `BeanFactory` and an `ApplicationContext`?"

**Mid-level answer:** "`BeanFactory` is the basic DI container. `ApplicationContext`
extends it and adds events, internationalisation and resource loading, and it
creates beans eagerly while `BeanFactory` is lazy."

**Senior answer:** "`BeanFactory` is the minimal `getBean` contract.
`ApplicationContext` adds the application-level services, but the operationally
important addition is that it **automatically detects and applies**
`BeanFactoryPostProcessor`s and `BeanPostProcessor`s during `refresh()`, and eagerly
instantiates non-lazy singletons at the end of it. With a raw `BeanFactory` you must
register those by hand, and without them there is no annotation configuration, no
property-placeholder resolution and no AOP proxying — so you lose essentially all of
Spring. In practice the distinction matters in two places: reading Spring's own
source, and understanding that eager instantiation is a fail-fast safety property
lazy initialisation trades away."

**What separates them:** the mid answer lists features; the senior names the
mechanism (`refresh()` applying post-processors) and draws an operational conclusion.
The senior answer also says where the distinction *does not* matter, which signals
judgment.

**Follow-up:** "So when would you actually use a plain `BeanFactory`?" Honest answer:
essentially never in an application. A candidate who invents a use case is guessing.

---

### Q2 — "Walk me through what happens between `SpringApplication.run` and your first bean's constructor."

**Mid-level answer:** "Spring scans for components, creates the context, injects
dependencies, and starts the embedded server."

**Senior answer:** "`run` prepares the `Environment`, chooses a context type,
applies `ApplicationContextInitializer`s, then calls `refresh()`. Inside `refresh()`
there are two phases. Phase one loads bean **definitions** — `@Configuration`
parsing and component scanning actually run here, driven by
`ConfigurationClassPostProcessor`, which is itself a `BeanFactoryPostProcessor`.
Auto-configuration conditions are evaluated here too, against the definitions and
the classpath. At the end of phase one every definition is known and **not one of my
objects exists**. Then `registerBeanPostProcessors` instantiates the wrappers, and
`finishBeanFactoryInitialization` constructs every non-lazy singleton in dependency
order — my constructor runs there. I care about the split because it is the only
reason `@ConditionalOnMissingBean`, placeholder resolution and `@Bean` overriding
work at all: you cannot conditionally skip an object you have already created."

**What separates them:** naming the phases, naming at least one concrete
post-processor, and explaining *what the split buys*. Anyone can memorise step names;
explaining why the ordering is load-bearing is the signal.

**Follow-up:** "Where exactly does an AOP proxy get created in that sequence?" They
want `BeanPostProcessor.postProcessAfterInitialization` — Topic 37, and the setup for
the `@Transactional` self-invocation question they are heading towards.

---

### Q3 — "You are coming from NestJS. What is genuinely different about Spring's container?"

**Mid-level answer:** "They are pretty similar — both do DI with singletons by
default and constructor injection. Spring uses `@Service` where Nest uses
`@Injectable`."

**Senior answer:** "The core IoC job is the same and I would not pretend otherwise.
Three real differences. **Visibility**: a Nest provider is module-scoped and must be
exported to be visible outside its module; one Spring `ApplicationContext` is a
single flat namespace where every bean can inject every other. `@Configuration` is
not a module — a real boundary needs package-private types, ArchUnit, Spring
Modulith or separate build modules. **Graph discovery**: Nest's graph is written
down in `imports` arrays so I can read it; Spring's comes from classpath scanning
plus conditional auto-configuration, so it is written down nowhere and I need the
bean-definition list or the condition report to see it. **Identity**: Spring may
hand callers a proxy subclass rather than my object — that is how `@Transactional`
works and why a self-call inside a bean bypasses it. Nest applies interceptors at
the route boundary, so that trap does not exist there."

**What separates them:** refusing the false dichotomy of "totally different" versus
"basically the same", and naming a specific consequence for each difference. The
proxy-identity point especially — most candidates from a Node background have never
considered that the injected object might not be their class.

**Follow-up:** "You said the graph is not written down. How do you find out why a
particular bean exists in a service you just inherited?" They want
`getBeanDefinitionNames`, the `--debug` condition report, and `getBeanDefinition` to
see the source. "Read the code" means the candidate has never had to do it.

---

### Q4 — "Why does Spring instantiate all singletons at startup? Is that not wasteful?"

**Mid-level answer:** "So beans are ready and the first request is not slow. You can
turn it off with lazy initialisation if startup is too slow."

**Senior answer:** "The point is not warm-up, it is **validation**. Constructing the
whole graph at startup is the only way to prove the graph is satisfiable — a missing
dependency, an ambiguous one, a constructor that throws on bad configuration all
become a startup failure. In a Kubernetes rollout that means the pod never passes
readiness and the rollout halts, with zero customer impact. With lazy
initialisation, the same defect produces a healthy-looking pod that serves 500s on
the first request touching the broken bean, possibly hours later on a rarely-used
path. So the cost is startup seconds and the benefit is a whole class of production
incident eliminated. I use laziness in a local dev profile, never in a deployed one.
If deployed startup is genuinely too slow I would look at the condition report to see
what auto-configurations I am paying for, or at AOT and native images, before trading
away fail-fast."

**What separates them:** reframing from performance to correctness, a concrete
deployment consequence, and naming where laziness *is* appropriate rather than
declaring it universally bad.

**Follow-up:** "Does lazy initialisation catch a missing bean at all?" A missing
definition referenced by an eagerly-created bean may still fail; a purely lazy
subtree will not be touched. The honest answer is "it depends what else pulls it in,
which is exactly the unpredictability that makes it unsuitable for production."

---

### Q5 — "Two classes implement `PaymentGateway`. What happens, and how do you resolve it?"

**Mid-level answer:** "`NoUniqueBeanDefinitionException`. Add `@Primary`, or use
`@Qualifier`."

**Senior answer:** "Startup fails with `NoUniqueBeanDefinitionException` wrapped in
`UnsatisfiedDependencyException`, and the message lists both candidates — genuinely
good failure design, because it fails before traffic and names the candidates. How I
resolve it depends on what I meant. `@Primary` if there is a real default.
`@Qualifier` if the choice is per-injection-point and I want it greppable. But for a
payments router the honest answer is usually that I do not want one gateway — I want
the set, so I inject `List<PaymentGateway>` and select by `name()` or a
`supports(...)` method at call time. That turns a wiring problem into a strategy
pattern, and adding a third gateway then requires no change to `OrderService`. Worth
naming the underlying cause too: Spring resolves by **type**, so two beans of one
type is inherently ambiguous. Nest injects by **token**, so it has no such failure
mode — a different trade-off, not a better one."

**What separates them:** treating the exception as a design signal rather than a
configuration nuisance, offering `List` injection as the usually-correct answer, and
identifying by-type versus by-token resolution as the root cause.

**Follow-up:** "What if the two implementations are in a library you do not control?"
They want `@Qualifier` at the injection point, or an explicit `@Bean` method — which
brings you back to phase one.

---

## Mental model checkpoint

1. Spring registers all definitions before instantiating any bean. Name two Spring
   features **impossible** without that split, and say which step of `refresh()` each
   depends on.

2. Nest also scans before instantiating. So why is `BeanFactoryPostProcessor` a
   distinctive Spring capability? What would Nest need to add to offer the same
   thing, and what would that cost in a codebase where the module graph is currently
   readable in one file?

3. One context is a flat namespace with no module boundaries. Argue that this is a
   **design flaw**. Then argue it is a **deliberate simplification** buying something
   real. Which do you believe, and what evidence would change your mind?

4. `spring.main.lazy-initialization=true` makes startup faster and failures later.
   Describe a service *and a deployment model* where that trade is genuinely correct.
   Be specific about the deployment model.

5. `CatalogService` has no Spring annotation and comes from a `@Bean` method;
   `InventoryService` is `@Service`. Both are beans. What has each choice cost and
   bought? When would you convert one to the other?

6. The container hands `OrderService` an object typed `PaymentGateway`. Given only
   what you know today, what could go wrong if that object is a runtime-generated
   subclass of the class you wrote rather than an instance of it? Write your guesses
   down before Topic 40 — you will want to check them.

7. A colleague proposes deleting all `@Configuration` classes and annotating every
   class `@Component`, "because scanning does it automatically." Give the strongest
   case **for**, then against, then say which classes you would insist stay in
   explicit `@Bean` methods, and why.

---

## Quick reference card

| Type | What it is |
|---|---|
| `BeanFactory` | minimal container: `getBean` |
| `ApplicationContext` | `BeanFactory` + events, resources, messages, `Environment`, auto-applied post-processors |
| `ConfigurableApplicationContext` | adds `refresh()` and `close()`; what `SpringApplication.run` returns |
| `AnnotationConfigApplicationContext` | standalone context built from `@Configuration` classes |
| `BeanDefinition` | the metadata recipe: class, scope, constructor args, init method, lazy, primary |
| `BeanFactoryPostProcessor` | phase 1 hook — rewrite definitions before any object exists |
| `BeanPostProcessor` | phase 2 hook — wrap or modify each bean instance (Topic 37) |

```
refresh(), one line each:
  prepareRefresh                     validate required properties
  obtainFreshBeanFactory             LOAD BEAN DEFINITIONS
  prepareBeanFactory                 register built-in dependencies
  postProcessBeanFactory             subclass hook
  invokeBeanFactoryPostProcessors    scan, parse @Configuration, evaluate conditions, EDIT DEFINITIONS
  registerBeanPostProcessors         instantiate the wrappers
  (message source, event multicaster, onRefresh, listeners)
  finishBeanFactoryInitialization    CREATE EVERY NON-LAZY SINGLETON
  finishRefresh                      publish ContextRefreshedEvent
```

```java
// Annotations introduced here
@Configuration                              // a source of bean definitions
@Configuration(proxyBeanMethods = false)    // no CGLIB; inter-bean calls are plain Java
@Bean                                       // return value is a bean; method name = bean name
@Component / @Service / @Repository         // register by scanning (Topic 36)
@SpringBootApplication                      // = @SpringBootConfiguration + @ComponentScan + @EnableAutoConfiguration
@SpringBootApplication(scanBasePackageClasses = Marker.class)
@Primary / @Qualifier("name")               // disambiguate (Topic 39)

// Diagnostics
Arrays.stream(context.getBeanDefinitionNames()).sorted().forEach(System.out::println);
context.getBeanDefinition("orderService").getBeanClassName();
context.getBean("orderService").getClass().getName();   // reveals $$SpringCGLIB$$
context.getBeansOfType(PaymentGateway.class).keySet();  // all candidates, name -> bean
```

```bash
./mvnw spring-boot:run
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug      # condition report
./mvnw spring-boot:run -Dspring-boot.run.jvmArguments="-Dspring.main.lazy-initialization=true"
mvn dependency:tree                                             # the classpath that gets scanned
```

| Exception | Phase | Usual cause |
|---|---|---|
| `NoSuchBeanDefinitionException` | 2 | not scanned, not annotated, or wrong type |
| `NoUniqueBeanDefinitionException` | 2 | two beans of one type; `@Primary`/`@Qualifier`/`List` |
| `UnsatisfiedDependencyException` | 2 | wrapper naming bean and parameter; read the *cause* |
| `BeanCurrentlyInCreationException` | 2 | circular constructor dependency (Topic 39) |
| `BeanDefinitionOverrideException` | 1 | two definitions with the same name |
| `BeanCreationException` at request time | 2, deferred | you enabled lazy initialisation |

**Gotchas checklist**
- [ ] Application class in the **root** package. Siblings are not scanned.
- [ ] `@Bean` dependencies as **method parameters**, never direct method calls.
- [ ] `BeanFactoryPostProcessor` / `BeanPostProcessor` `@Bean` methods must be `static`.
- [ ] One context = one flat namespace. `@Configuration` is not a module.
- [ ] Eager singleton creation is a safety feature. Not for production disabling.
- [ ] Money is `long` minor units everywhere in `orderflow` (Topic 01).
- [ ] Constructor injection so fields can be `final` (Topics 17, 39).

---

## When would I use this at work?

**1. Diagnosing a startup failure in a service you did not write.** A 300-line stack
trace lands in CI. Read Boot's `Description`/`Action` block first, then the innermost
cause, then classify: phase 1 (metadata — scan or duplicate definition) or phase 2
(construction — missing or ambiguous dependency). That classification halves the
search space and is available in ten seconds. The most frequent payoff of this topic.

**2. Deciding whether a class should be a `@Bean` or a `@Component`.** This comes up
in almost every review of new Spring code. The rule you now have: classes you own and
that are unambiguous get a stereotype; third-party classes, classes needing
construction logic, and anything where a reader will later ask "why does this exist?"
get an explicit `@Bean`. You are trading conciseness for traceability, and you can now
say which one the situation needs.

**3. Arguing about startup time in a Kubernetes rollout.** Someone proposes lazy
initialisation to cut pod start from 9 seconds to 3. You can state exactly what is
traded: a class of misconfiguration moves from "rollout halts, nobody notices" to
"pod serves 500s on a cold path". That is a conversation about risk, and you are the
person able to frame it that way rather than as a preference.

---

## Connected topics

**Prerequisites:**
- **03 — Access modifiers and packages**: package-private is the only real
  encapsulation left by Spring's flat namespace.
- **08 — Exceptions**: try-with-resources on the context; reading nested causes.
- **17 — Immutability and safe publication**: constructor injection gives `final`
  fields, and `final` fields have memory-model guarantees. This is the technical
  reason constructor injection is not merely a style preference.
- **26 — Optional**, **27 — Records**: `findBySku`, `Product`, `Order`, `OrderLine`.
- **31–32 — Maven and dependency resolution**: **the classpath is what gets
  scanned**. A missing or conflicting artifact is a missing bean. The most direct
  dependency in this whole phase.

**This unlocks:**
- **36 — Bean definitions and component scanning**: how definitions get into the
  registry, and the honest cost of implicit scanning versus Nest's `imports`.
- **37 — Bean lifecycle and `BeanPostProcessor`**: what happens to each bean between
  construction and use, and exactly where proxies are created.
- **38 — Bean scopes**: what "singleton by default" means, and what breaks when it is not.
- **39 — Injection styles**: `@Qualifier`, `@Primary`, circular dependencies.
- **40 — Proxying**: why `getClass().getName()` sometimes contains `$$SpringCGLIB$$`.
- **42 — Auto-configuration**: the payoff of the phase split; conditions evaluated
  against definitions before instantiation is the entire mechanism.
- **54 — `@Transactional`**: a `BeanPostProcessor` plus a proxy plus a transaction manager.
- **60 — Test slices and context caching**: the test framework caches contexts by
  configuration key because `refresh()` is expensive. Knowing what `refresh()` does
  is what makes that cache comprehensible rather than mysterious.

---

## `[BOOT 3.x DELTA]`

**Container core: unchanged.** `refresh()`, the two-phase split, `BeanDefinition`,
`BeanFactoryPostProcessor`, `BeanPostProcessor`, `@Configuration`/`@Bean` semantics
and `proxyBeanMethods` behave the same on Boot 3.x / Framework 6.x as described here.
I am saying this explicitly rather than inventing a difference.

Differences that are real:

| Area | Boot 3.x | Boot 4.x |
|---|---|---|
| OSS support | **ended June 2026** — no free CVE patches | current |
| Artifact layout | fewer, larger starter jars | modularised into smaller focused jars; some artifact ids and packages moved, so a `<dependency>` copied from a 3.x tutorial may not resolve |
| JSON | Jackson 2 | Jackson 3 standard; Jackson 2 deprecated |
| Null safety | Spring's own `@Nullable` | JSpecify annotations portfolio-wide |

Two older deltas worth knowing, because you will meet 2.x codebases:

- **Bean definition overriding** has defaulted to **disabled** since Boot 2.1. On 2.0
  and earlier a duplicate bean name silently replaced the earlier definition; since
  2.1 you get `BeanDefinitionOverrideException` at startup. Inheriting a service with
  `spring.main.allow-bean-definition-overriding=true` set means something is being
  silently replaced and nobody knows what. Treat it as debt.
- **Circular references** have defaulted to **disabled** since Boot 2.6. Earlier
  versions silently resolved singleton cycles with early references. Topic 39.

---

*Java baseline 21, running on JDK 25. Spring Framework 7.0 / Spring Boot 4.1,
Jakarta EE 11 — `jakarta.*` imports throughout, never `javax.*`. No specific Spring
artifact versions appear in this document by design: the Boot BOM supplies them, and
the authoritative current version is whatever a freshly generated project from
start.spring.io puts in its parent POM.*
