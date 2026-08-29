# 36 — Bean Definition, Registration, and Component Scanning vs Nest Modules

## Phase: 4 — Spring Core
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: the `orderflow` package structure is fixed here — `catalog`, `inventory`, `orders`, `payments`, `wallet`, plus a `platform` package for shared infrastructure. One `PaymentGateway` bean is registered per configured provider, programmatically, from a `BeanDefinitionRegistryPostProcessor`.

---

## ELI5 anchor

A library has two ways to know what books it owns.

**Way one — a written index.** Someone typed every book into a list. To know what
the library has, you read the list. If a book is missing from the list, it does not
officially exist, even if it is sitting on a shelf. Adding a book means two actions:
put it on the shelf, and write it in the index.

**Way two — walk the shelves.** No index. To know what the library has, a librarian
walks every aisle and writes down every spine that has a coloured sticker on it.
Adding a book means one action: put it on the shelf with a sticker.

Way two is faster to add to and easier to get right day-to-day. But there is a cost,
and it is not obvious until you need it: **you can never read the index, because
there is no index.** If someone asks "why do we have three copies of this book?",
way one has an answer in a file. Way two requires you to walk the shelves again.

Nest is way one: the `imports` array is the index. Spring's component scanning is
way two: `@Component` is the coloured sticker, and the librarian is
`ClassPathBeanDefinitionScanner`.

Spring supports way one too — that is what `@Bean` in a `@Configuration` class is.
Knowing when to pay for the index is the skill this topic teaches.

---

## The bridge from what you know

### Component scanning vs Nest modules — PARTIAL

```ts
// NestJS: the graph is a data structure you can read.
@Module({
  imports:   [CatalogModule, InventoryModule, WalletModule],
  providers: [OrderService, WalletPaymentGateway],
  exports:   [OrderService],
})
export class OrdersModule {}
```

Everything is in that file. Which providers exist. Which are visible outside. Which
other modules this one depends on. You can open it and answer "what does `orders`
need?" without a debugger.

```java
// Spring: the graph is emergent.
@Service
public class OrderService {
    public OrderService(CatalogService c, InventoryService i, PaymentGateway p) { ... }
}
```

There is no `orders` module file. There is no list of what `orders` provides. There
is no declaration that `orders` depends on `catalog`. All of it is inferred at
startup from (a) which classes happen to be on the classpath, (b) which of those
carry a stereotype annotation, and (c) which constructor parameter types happen to
match other beans.

**What transfers:**

| Concept | Nest | Spring | Verdict |
|---|---|---|---|
| Marking a class as available for injection | `@Injectable()` | `@Component` / `@Service` / `@Repository` | **HONEST ANALOGUE** |
| Declaring a bean explicitly with construction logic | a provider with `useFactory` | `@Bean` method in `@Configuration` | **HONEST ANALOGUE** |
| Injecting a token instead of a type | `@Inject(TOKEN)` | `@Qualifier("name")` | **PARTIAL** — Spring still resolves by type first |
| Grouping related providers | `@Module` | `@Configuration` (grouping only, no boundary) | **PARTIAL** |
| The dependency graph written down | `imports` array | **nowhere** | **NO ANALOGUE** |
| Provider visibility control | `exports` | **none in the container** | **NO ANALOGUE** |
| Registering providers at runtime from config | dynamic modules (`forRoot`/`forRootAsync`) | `BeanDefinitionRegistryPostProcessor`, `ImportBeanDefinitionRegistrar` | **PARTIAL** — Spring's runs earlier and is more powerful |

### Be honest about the trade

Scanning is genuinely more convenient. Adding a service in Spring is one
annotation; in Nest it is an annotation plus an entry in a `providers` array plus,
if another module needs it, an entry in `exports` and an entry in that module's
`imports`. Four edits versus one. Over a year that adds up, and Nest developers
regularly forget the `exports` and get a runtime resolution error.

The cost lands later, and it lands on the person who did not write the code:

- **"Why does this bean exist?"** In Nest you grep the `providers` arrays. In Spring
  the answer may be: your `@Component`, or a `@Bean` in a configuration class, or an
  auto-configuration in a jar you did not know you depended on, or a
  `BeanDefinitionRegistryPostProcessor` in a shared internal library. There are at
  least six registration paths and none of them is a single file.
- **"What would break if I deleted this?"** Nest tells you at compile time when you
  remove something from `providers`. Spring tells you at startup, which is still
  early, but only if something eagerly depends on it.
- **"Which module owns this?"** Spring has no answer. There are no modules.

**The rule that comes out of this:** use scanning for the ordinary case, and use an
explicit `@Bean` in a named `@Configuration` whenever a future reader will ask *why*
this bean exists or *why it is configured that way*. Verbosity is the price of
traceability, and you should pay it deliberately rather than never or always.

---

## What is this?

### A bean definition is a data structure

`BeanDefinition` is an interface. An instance of it describes how to make one bean.
The fields you will actually touch:

| Property | Meaning |
|---|---|
| `beanClassName` | the class to instantiate |
| `scope` | `singleton` (default), `prototype`, `request`, ... (Topic 38) |
| `lazyInit` | create at startup or on first `getBean` |
| `primary` | wins when several candidates match a type |
| `autowireCandidate` | if `false`, this bean is never considered for injection by type |
| `dependsOn` | bean names that must be created first, even with no injection relationship |
| `factoryBeanName` + `factoryMethodName` | for `@Bean`: "call this method on this configuration bean" |
| `constructorArgumentValues` | explicit constructor arguments |
| `initMethodName` / `destroyMethodName` | lifecycle hooks (Topic 37) |
| `role` | `ROLE_APPLICATION` (yours), `ROLE_SUPPORT`, `ROLE_INFRASTRUCTURE` (Spring's own) |

That last one is genuinely useful for reading a bean dump: filter to
`ROLE_APPLICATION` and you see your beans instead of Spring's several hundred.

The registry that holds them is `BeanDefinitionRegistry`, with
`registerBeanDefinition(String name, BeanDefinition definition)` and
`removeBeanDefinition(String name)`. Both are available to you in phase 1.

### The six ways a definition gets registered

Learn these as a checklist. When a bean mysteriously exists, it came from one of
them.

**1. Component scanning.** A class carrying `@Component` or a meta-annotated
stereotype, found in a scanned package.

**2. A `@Bean` method** in a `@Configuration` class (or, less commonly, in a plain
`@Component` — see the syntax section).

**3. `@Import`.** A `@Configuration` class can pull in another configuration class,
an `ImportSelector` (returns class names to register, decided at runtime), or an
`ImportBeanDefinitionRegistrar` (registers definitions directly).

**4. Auto-configuration.** Boot reads
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
from **every jar on the classpath** and registers each listed class as a conditional
configuration. This is the source of most beans in a Boot app. Topic 42.

**5. Programmatic registration** via a `BeanDefinitionRegistryPostProcessor`. You are
handed the registry in phase 1 and may add or remove definitions. This is Example 2.

**6. `@ComponentScan` on a `@Configuration` class** pointing at additional packages —
including packages inside a jar you depend on.

### How scanning actually works

This part is worth knowing precisely, because it explains several surprising
behaviours.

`ClassPathBeanDefinitionScanner` resolves the base package to a classpath resource
pattern — roughly `classpath*:com/orderflow/**/*.class` — and enumerates every
matching `.class` resource across directories and jars. For each one it reads the
class file's **annotation metadata using ASM**, a bytecode reader. It does **not**
call `Class.forName`.

Three consequences:

1. **Scanning does not load your classes.** A class can be a scan candidate,
   be rejected by a filter, and never be loaded by the JVM at all. This is what
   makes `@ConditionalOnClass` safe: Boot can ask "is this class present?" without
   risking a `NoClassDefFoundError` from loading something whose dependencies are
   missing.
2. **Scanning cost is proportional to the number of class files under the base
   package**, across every jar. `@ComponentScan("com")` in a service with 400
   dependencies means reading a very large number of class files at every startup
   and in every test context (Topic 60).
3. **Only annotations are visible.** The scanner sees `@Component` on the class. It
   cannot see anything about method bodies, and it does not run your code.

Candidate detection then applies filters. By default the include filter is
"annotated with `@Component`" (which, via meta-annotation, covers `@Service`,
`@Repository`, `@Controller`, `@RestController`, `@Configuration`). Boot adds an
exclude filter — `TypeExcludeFilter` — which is the mechanism test slices use to
scan only the layer they care about (Topic 60).

### Bean names

You need the naming rule because collisions are a real failure mode.

- **Scanned classes:** the simple class name, decapitalised.
  `InventoryService` becomes `inventoryService`. The decapitalisation follows
  `java.beans.Introspector.decapitalize`, which has one special case: **if the first
  two characters are both uppercase, the name is left unchanged.** So a class named
  `SLAMonitor` gets the bean name `SLAMonitor`, not `sLAMonitor`. This surprises
  people who then write `@Qualifier("slaMonitor")` and get a
  `NoSuchBeanDefinitionException`.
- **Explicit override:** `@Component("legacyGateway")` or `@Service("orders")`.
- **`@Bean` methods:** the method name, or `@Bean("explicitName")`.
- **Auto-configuration classes:** Boot registers them under their **fully-qualified**
  class names, which is why they never collide with your short-named beans. You can
  see this immediately in a sorted definition dump — the long dotted names are Boot's.

**Names must be unique within a context.** Two definitions with the same name is
`BeanDefinitionOverrideException` at startup, since Boot 2.1. Trap 2.

---

## Why does it matter?

**1. "Why does this bean exist?" is a question you will be asked, and the answer is
not in one file.** Knowing the six registration paths turns a mystery into a
checklist you can walk in two minutes.

**2. Scan scope is a startup-time and test-time cost.** Every test context refresh
re-scans. A too-broad `@ComponentScan` in a service with a large dependency tree
shows up as a slow test suite, and it is not obvious that scanning is the cause.
Topic 60 is where this bites hardest.

**3. Programmatic registration is a real capability, not a curiosity.** "Register one
bean per entry in a configuration list" is a genuinely common requirement — one
payment gateway per configured provider, one Kafka consumer per configured topic, one
data source per tenant. Doing it with definitions is clean; faking it with a map
inside one bean is what people do instead, and it loses per-bean lifecycle,
per-bean configuration properties and per-bean metrics.

---

## Syntax breakdown

### `@Component` and its stereotypes

```java
@Component                  // generic
@Service                    // business logic — semantic label only
@Repository                 // data access — semantic label PLUS exception translation
@Controller                 // web handler — MVC treats it specially
@RestController             // @Controller + @ResponseBody
@Configuration              // a source of @Bean definitions
```

All of these are **meta-annotated** with `@Component`. Meta-annotation means: the
annotation itself carries `@Component`, and Spring's scanner treats "annotated with
something that is annotated with `@Component`" as a match. That is why the include
filter for `@Component` picks up all of them.

`@Repository` is the only one of `@Service`/`@Repository`/`@Component` with runtime
behaviour beyond the label: it marks the class for **persistence exception
translation**, which converts vendor-specific database exceptions into Spring's
unchecked `DataAccessException` hierarchy (Topic 47). Use it on data access classes
for that reason, not for decoration.

### Custom stereotypes

Because meta-annotation composes, you can define your own:

```java
package com.orderflow.platform;

import org.springframework.stereotype.Service;
import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Service                                  // meta-annotated => still a component
public @interface UseCase { }
```

Now `@UseCase` on `OrderService` registers it exactly as `@Service` would, and you
have a marker you can point a pointcut at (Topic 41), an ArchUnit rule at, or a
metrics aspect at. This is the closest Spring gets to Nest's "these providers belong
together", and it is worth reaching for once a codebase has real layers.

`@Retention(RUNTIME)` is required — scanning reads the class file, and a
`SOURCE`- or `CLASS`-retained annotation is not in the constant pool metadata the
scanner examines.

### `@ComponentScan`

```java
@Configuration
@ComponentScan(
    basePackageClasses = { OrderflowApplication.class },   // prefer this over strings
    includeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION, classes = UseCase.class),
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.REGEX, pattern = "com\\.orderflow\\..*\\.internal\\..*"),
    lazyInit = false
)
public class ScanConfig { }
```

| Attribute | What it does |
|---|---|
| `basePackages` | package names as strings — breaks silently on rename |
| `basePackageClasses` | classes whose packages are the roots — survives renames, refactor-safe |
| `includeFilters` | add extra matchers on top of the default `@Component` one |
| `useDefaultFilters = false` | scan **only** what `includeFilters` says. Combine with `includeFilters` for a very narrow scan. |
| `excludeFilters` | remove matches. `FilterType.ANNOTATION`, `ASSIGNABLE_TYPE`, `REGEX`, `ASPECTJ`, `CUSTOM`. |
| `lazyInit` | mark every scanned definition lazy |
| `nameGenerator` | supply a `BeanNameGenerator` — e.g. fully-qualified names to avoid collisions |

### `@Bean` in `@Configuration` — the explicit index

```java
@Configuration
public class CatalogConfig {

    @Bean(name = "productRepository", initMethod = "warm", destroyMethod = "close")
    @Primary
    public ProductRepository productRepository(CatalogProperties properties) {
        return new InMemoryProductRepository(properties.seedSize());
    }
}
```

| Attribute | Effect |
|---|---|
| `name` / `value` | override the method-name-derived bean name; accepts aliases |
| `initMethod` / `destroyMethod` | lifecycle callbacks by name (Topic 37) — how you hook a third-party class you cannot annotate |
| `autowireCandidate = false` | exists in the context but is never picked by type matching |

### `@Import`

```java
@Configuration
@Import({ PaymentsConfig.class, ObservabilityConfig.class })
public class PlatformConfig { }
```

Pulls another configuration class in explicitly, without scanning. This is how you
compose configuration in a **library**: you cannot rely on the consumer's scan root
reaching your packages, so you expose one configuration class and let them
`@Import` it — or you register it as an auto-configuration (Topic 42).

### `BeanDefinitionRegistryPostProcessor`

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry);
}
```

It extends `BeanFactoryPostProcessor` and runs **even earlier**: registry
post-processors run before ordinary factory post-processors, because adding
definitions must happen before anyone inspects the complete set. This is the hook
for "one bean per configured item".

---

## Example 1 — minimal

Three classes, one explicit scan, and a definition dump filtered to your own beans.

```java
package com.orderflow.lab;

import org.springframework.stereotype.*;

@Repository class SkuLookup      { }
@Service    class PriceCalculator { }
@Component  class SLAMonitor      { }     // note the name this gets
```

```java
package com.orderflow.lab;

import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.context.annotation.*;
import java.util.Arrays;

@Configuration
@ComponentScan(basePackageClasses = SkuLookup.class)
public class ScanDemo {

    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(ScanDemo.class)) {
            Arrays.stream(ctx.getBeanDefinitionNames())
                  .filter(name -> ctx.getBeanDefinition(name).getRole()
                                   == BeanDefinition.ROLE_APPLICATION)
                  .sorted()
                  .forEach(name -> System.out.println(
                          name + "  ->  " + ctx.getBeanDefinition(name).getBeanClassName()));
        }
    }
}
```

Two things to predict before you run it:

1. What bean name does `SLAMonitor` get? (Two leading capitals — see the naming rule.)
2. Does `ScanDemo` itself appear in the list? It is a `@Configuration`, which is
   meta-annotated `@Component`, and it is in the scanned package. So it is registered
   twice — once because it was passed to the constructor, once because it was
   scanned. Spring recognises this as the same definition rather than a collision.
   Change the scan to a *different* package and watch the list change.

The `ROLE_APPLICATION` filter is the important line. Without it you get Spring's
internal post-processors mixed in with yours.

---

## Example 2 — production scenario (on the project spine)

### Deciding the package structure

`orderflow` needs a layout that will still be right at 80 000 lines. The two
candidates:

**Package by layer**
```
com.orderflow.controller     OrderController, ProductController, WalletController
com.orderflow.service        OrderService, CatalogService, InventoryService
com.orderflow.repository     OrderRepository, ProductRepository
com.orderflow.model          Order, Product, Wallet
```

**Package by feature**
```
com.orderflow.orders         OrderController, OrderService, Order, OrderRepository
com.orderflow.catalog        ProductController, CatalogService, Product, ProductRepository
com.orderflow.wallet         WalletService, Wallet, WalletRepository
```

Package by feature wins for `orderflow`, for three specific reasons:

1. **Package-private actually works.** In a layered layout, `OrderService` and
   `OrderRepository` are in different packages, so the repository must be `public`
   and every package in the application can use it. In a feature layout the
   repository can be package-private and only `orders` can reach it. That is the only
   free encapsulation Java gives you (Topic 03), and layering throws it away.
2. **A change lands in one directory.** "Add a discount code to orders" touches
   `com.orderflow.orders` and nothing else.
3. **It gives ArchUnit and Spring Modulith something to enforce.** "`catalog` must
   not depend on `wallet`" is expressible. "`service` must not depend on `service`"
   is not.

The cost is honest: you lose the at-a-glance "where are all the controllers" view,
and a genuinely cross-cutting class has no obvious home. That is why `platform`
exists below.

### The fixed layout

```
com.orderflow
├── OrderflowApplication.java        @SpringBootApplication — root package, so it scans everything
├── platform/                        cross-cutting infrastructure, no domain logic
│   ├── UseCase.java                 custom stereotype
│   ├── PaymentProviderProperties.java
│   └── PaymentGatewayRegistrar.java BeanDefinitionRegistryPostProcessor
├── catalog/                         Product, ProductRepository, CatalogService
├── inventory/                       InventoryService, InsufficientStockException
├── orders/                          Order, OrderLine, OrderStatus, OrderService
├── payments/                        PaymentGateway, PaymentResult, gateway impls
└── wallet/                          WalletService, InsufficientFundsException
```

Classes are from Topic 35 and carry forward unchanged.

### The requirement: one gateway bean per configured provider

Production constraint: `orderflow` routes payments across several providers. Which
providers are enabled differs per environment — two in staging, five in production,
and operations adds a sixth without a code change. Each provider needs its own
endpoint, its own timeout, and its own metrics and health indicator so a single
provider's outage is visible on its own dashboard.

**The tempting wrong design:** one `PaymentGatewayService` bean holding a
`Map<String, ProviderConfig>` and switching internally. It works, and it costs you:
every provider shares one set of metrics, one health status, one circuit breaker
(Topic 110) and one timeout configuration. When provider C degrades, your dashboard
shows "payments are slow" and nothing more.

**The right design:** one bean per provider, registered from configuration.

`application.yml`:
```yaml
orderflow:
  payments:
    providers:
      - id: wallet
        endpoint: internal://wallet
        timeout: 200ms
      - id: acquirer-eu
        endpoint: https://eu.acquirer.example/v2
        timeout: 900ms
      - id: acquirer-us
        endpoint: https://us.acquirer.example/v2
        timeout: 1200ms
```

`platform/PaymentProviderProperties.java`:
```java
package com.orderflow.platform;

import java.time.Duration;
import java.util.List;

public record PaymentProviderProperties(List<Provider> providers) {

    public record Provider(String id, String endpoint, Duration timeout) { }
}
```

`payments/HttpPaymentGateway.java` — the class we will register N times:
```java
package com.orderflow.payments;

import java.time.Duration;

public class HttpPaymentGateway implements PaymentGateway {

    private final String id;
    private final String endpoint;
    private final Duration timeout;

    public HttpPaymentGateway(String id, String endpoint, Duration timeout) {
        this.id = id;
        this.endpoint = endpoint;
        this.timeout = timeout;
    }

    @Override public String name() { return id; }

    @Override
    public PaymentResult charge(String customerId, long amountMinor, String idempotencyKey) {
        // Topic 111 replaces this with a real HTTP client, a timeout and a retry budget.
        return PaymentResult.approved(id.toUpperCase() + "-STUB");
    }
}
```

Note: **no annotation on the class.** It cannot be scanned, because scanning
registers one definition per class and we need N. Registration is programmatic:

`platform/PaymentGatewayRegistrar.java`:
```java
package com.orderflow.platform;

import com.orderflow.payments.HttpPaymentGateway;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.*;
import org.springframework.boot.context.properties.bind.Binder;
import org.springframework.core.env.Environment;
import org.springframework.context.EnvironmentAware;

import java.time.Duration;
import java.util.List;

public class PaymentGatewayRegistrar
        implements BeanDefinitionRegistryPostProcessor, EnvironmentAware {

    private Environment environment;

    @Override public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {

        // Binder reads typed configuration in PHASE 1, before @ConfigurationProperties
        // beans exist. That is deliberate: we need the values before any bean is made.
        List<PaymentProviderProperties.Provider> providers =
                Binder.get(environment)
                      .bind("orderflow.payments", PaymentProviderProperties.class)
                      .map(PaymentProviderProperties::providers)
                      .orElse(List.of());

        for (var provider : providers) {
            String beanName = "paymentGateway-" + provider.id();

            if (registry.containsBeanDefinition(beanName)) {
                throw new IllegalStateException(
                        "Duplicate payment provider id in configuration: " + provider.id());
            }

            AbstractBeanDefinition definition = BeanDefinitionBuilder
                    .genericBeanDefinition(HttpPaymentGateway.class)
                    .addConstructorArgValue(provider.id())
                    .addConstructorArgValue(provider.endpoint())
                    .addConstructorArgValue(provider.timeout())
                    .setPrimary("wallet".equals(provider.id()))
                    .getBeanDefinition();

            registry.registerBeanDefinition(beanName, definition);
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // Nothing to do. Definitions were added in the registry callback above.
    }
}
```

Registered — deliberately **not** by scanning, because a post-processor must exist
before scanning results are consumed:

```java
package com.orderflow;

import com.orderflow.platform.PaymentGatewayRegistrar;
import org.springframework.context.annotation.*;

@Configuration
public class PlatformConfig {

    // static: a BeanDefinitionRegistryPostProcessor must be created without
    // instantiating its enclosing configuration class first.
    @Bean
    public static PaymentGatewayRegistrar paymentGatewayRegistrar() {
        return new PaymentGatewayRegistrar();
    }
}
```

### Consuming the registered beans

`OrderService` from Topic 35 took a single `PaymentGateway`. Now there are several,
so it takes the set — the Trap 1 fix from Topic 35, now the natural design:

```java
package com.orderflow.payments;

import org.springframework.stereotype.Service;
import java.util.*;

@Service
public class PaymentRouter {

    private final Map<String, PaymentGateway> gatewaysById;

    // Spring injects every PaymentGateway bean, keyed by bean name.
    public PaymentRouter(List<PaymentGateway> gateways) {
        Map<String, PaymentGateway> byId = new HashMap<>();
        for (PaymentGateway gateway : gateways) {
            byId.put(gateway.name(), gateway);
        }
        this.gatewaysById = Map.copyOf(byId);   // Topic 17: immutable after construction
    }

    public Set<String> availableProviders() { return gatewaysById.keySet(); }

    public PaymentGateway route(String providerId) {
        PaymentGateway gateway = gatewaysById.get(providerId);
        if (gateway == null) {
            throw new UnknownPaymentProviderException(providerId, gatewaysById.keySet());
        }
        return gateway;
    }
}
```

`OrderService`'s constructor parameter becomes `PaymentRouter router`, and its call
site becomes `router.route(providerId).charge(...)`.

**What this bought:** adding `acquirer-apac` is a YAML edit and a redeploy. Each
gateway is a separate bean, so Topic 118's metrics aspect tags per bean name,
Topic 110's circuit breaker wraps each independently, and a per-provider health
indicator is a small addition. None of that is available from a map inside one bean.

**What it cost, stated honestly:** the graph is now *even less* visible in a file
than ordinary scanning — a reader grepping for `HttpPaymentGateway` finds a class
nobody obviously instantiates. That is why the registrar lives in `platform` with an
explicit name, why the bean names have a readable prefix, and why the next section's
hands-on step prints them. If you use this pattern, you owe the codebase that
discoverability.

### Making the module boundary real

Because the container will not enforce it (Topic 35, Trap 4):

```java
package com.orderflow;

import com.tngtech.archunit.junit.*;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;

@AnalyzeClasses(packages = "com.orderflow")
class ModuleBoundaryTest {

    @ArchTest
    static final ArchRule catalogIsIndependent =
            layeredArchitecture().consideringOnlyDependenciesInLayers()
                .layer("catalog").definedBy("com.orderflow.catalog..")
                .layer("wallet").definedBy("com.orderflow.wallet..")
                .layer("orders").definedBy("com.orderflow.orders..")
                .whereLayer("wallet").mayOnlyBeAccessedByLayers("orders", "payments")
                .whereLayer("catalog").mayOnlyBeAccessedByLayers("orders");
}
```

This is the `exports` list Nest gave you for free, reconstructed as a test. It is
worth the twenty lines.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a `@Component` inside a shared library jar

Topic 35's Trap 2 covered a component in a sibling package of your own code. This is
the version that catches teams rather than individuals.

**Wrong:** an internal library `com.acme.observability` publishes
`@Component class TraceEnricher`. `orderflow` adds the dependency and expects it to
work.

**Exact symptom:** nothing happens. No exception. Traces are simply missing the
enrichment. The dependency is on the classpath — `mvn dependency:tree` proves it —
and the class file is in the jar. Weeks later someone notices a field is absent from
every span.

**Root cause:** `com.acme.observability` is not under `com.orderflow`, so the scan
root never reaches it. Classpath presence and scan coverage are different things, and
this is exactly where people conflate them.

**Fix — and the choice matters:**

1. **The library should provide an auto-configuration** (Topic 42): an
   `@AutoConfiguration` class listed in
   `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
   Boot reads that file from every jar, so no scan-root coordination is needed. This
   is the correct answer for a library and it is what every Spring starter does.
2. **The library exposes one `@Configuration` class** the consumer `@Import`s.
   Explicit, greppable, one line in the consumer. Good for internal libraries where
   you want the dependency visible.
3. **The consumer adds the package to its scan:**
   `@SpringBootApplication(scanBasePackages = {"com.orderflow", "com.acme.observability"})`.
   Works, and it is the worst option: it is the consumer compensating for the
   library's packaging, it silently picks up every future `@Component` the library
   adds, and it makes the library's internals part of the consumer's bean graph.

**Diagnostic:** if a bean from a jar is missing, ask "is the class on the classpath?"
and "was its package scanned or auto-configured?" as two separate questions. The
answers are frequently yes and no.

---

### Trap 2 — two classes with the same simple name

**Wrong:** `com.orderflow.orders.EventPublisher` and
`com.orderflow.payments.EventPublisher`, both `@Component`. Reasonable naming in each
package. Fatal together.

**Exact symptom:** startup fails with `BeanDefinitionOverrideException` — a message
of the form "Invalid bean definition with name 'eventPublisher' ... There is already
a bean definition bound." Boot's action block suggests
`spring.main.allow-bean-definition-overriding=true`.

**Do not take that suggestion.** Setting it makes the error disappear and makes one
of your two classes silently never instantiated. Which one wins depends on scan
order, which depends on filesystem order. You now have a bug that behaves differently
on your laptop and in CI.

**Root cause:** the default bean name generator uses the **simple** class name,
decapitalised. Packages do not disambiguate. Two `EventPublisher` classes are one
bean name.

**Fix (best first):**

1. **Rename the classes** so they are distinct: `OrderEventPublisher`,
   `PaymentEventPublisher`. Usually the right answer, because if two classes have the
   same name a reader is going to confuse them too.
2. **Name the beans explicitly:** `@Component("orderEventPublisher")`. Fixes the
   collision, leaves the readability problem.
3. **Use a fully-qualified name generator** on your scan:
   `@ComponentScan(nameGenerator = FullyQualifiedAnnotationBeanNameGenerator.class)`.
   Collisions become impossible. Cost: every `@Qualifier` and every
   `getBean("...")` string in the codebase must now use the full class name, so this
   is a decision for a new codebase, not a retrofit.

**Related surprise, same root cause:** the two-leading-capitals rule.
`@Component class SLAMonitor` is registered as `SLAMonitor`, not `sLAMonitor` and not
`slaMonitor`. `@Qualifier("slaMonitor")` then fails with
`NoSuchBeanDefinitionException` naming a bean that visibly exists in your dump. Print
the names rather than deriving them in your head.

---

### Trap 3 — scanning far too broadly

**Wrong:** `@ComponentScan("com")` or `@SpringBootApplication(scanBasePackages = "com")`,
usually added to make Trap 1 go away.

**Exact symptom:** several, all indirect.

- Application startup goes from about 3 seconds to noticeably longer, and the time is
  invisible in any profiler you point at your own code, because it is spent reading
  class files.
- The test suite slows down disproportionately, because every non-cached context
  refresh re-scans (Topic 60).
- Beans appear that you never intended: a `@Component` inside a third-party jar under
  `com.` starts being instantiated, and its configuration expectations are not met.
  You may get a `BeanCreationException` for a class you have never heard of.
- Worst variant: a `@Configuration` class in a dependency starts contributing beans
  that override yours, and behaviour changes on a dependency bump with no code change.

**Root cause:** the base package becomes a classpath resource pattern applied across
**every** jar. `com` matches essentially everything in the Java ecosystem.

**Fix:**
- Scan only your own root, from the root package. `com.orderflow`.
- Pull library beans in through auto-configuration or `@Import` (Trap 1), never by
  widening the scan.
- If you genuinely must scan two roots, name them both explicitly with
  `basePackageClasses`, and never a prefix shorter than your organisation's full
  package root.

**Measure it:** compare startup with your current scan against a scan narrowed to
`basePackageClasses = OrderflowApplication.class`, using the "Started ... in X
seconds" line, run five times each. If the difference is under a few hundred
milliseconds, say so — do not claim a win you did not measure.

---

### Trap 4 — `@Component` on something that cannot be instantiated

**Wrong:**
```java
@Component
public interface PaymentGateway { }          // an interface

@Component
public abstract class AbstractGateway { }     // an abstract class
```

**Exact symptom:** silence. No bean, no error, no warning. Then, later, a
`NoSuchBeanDefinitionException` for `PaymentGateway` at some unrelated injection
point, and you stare at the annotation on the interface wondering why it did nothing.

**Root cause:** the scanner's candidate check requires an **independent, concrete**
class. Interfaces and abstract classes are rejected as candidates, quietly, by
design. So are non-static inner classes, because they cannot be instantiated without
an enclosing instance.

**Fix:** annotate the concrete implementations. `@Component` belongs on
`WalletPaymentGateway`, never on `PaymentGateway`.

**The related mistake worth naming:** annotating the interface hoping "all
implementations become beans." Spring has no such feature. Every implementation is
registered individually, or produced by a `@Bean` method, or registered
programmatically as in Example 2.

---

### Trap 5 — the same class registered twice, by two different paths

**Wrong:**
```java
@Service                                   // path 1: scanning
public class CatalogService { ... }

@Configuration
public class CatalogConfig {
    @Bean                                  // path 2: explicit
    public CatalogService catalogService(ProductRepository repository) {
        return new CatalogService(repository);
    }
}
```

**Exact symptom:** it depends, and both outcomes are bad.

If the derived names collide — scanned `catalogService` and `@Bean` method
`catalogService` — you get `BeanDefinitionOverrideException` at startup, which is the
*lucky* case because it is loud.

If the names differ — say the `@Bean` method is named `productCatalogService` — then
**two `CatalogService` beans exist**. Injection by type becomes ambiguous, giving
`NoUniqueBeanDefinitionException`; or, if one is `@Primary`, injection silently picks
one and half the application uses an instance whose cache nothing ever warms
(Topic 37). The observable signal is a cache hit rate of zero on one code path and
normal on another, with no exception anywhere.

**Root cause:** the six registration paths are independent. Nothing deduplicates
across them. Annotating a class *and* declaring it in a configuration is registering
it twice.

**Fix:** pick one path per class, and make it a reviewable rule:

| Situation | Registration path |
|---|---|
| your class, no construction logic | `@Component` / `@Service` / `@Repository` |
| your class, needs construction arguments not injectable by type | `@Bean` method |
| third-party class | `@Bean` method (you cannot annotate it) |
| N instances from configuration | `BeanDefinitionRegistryPostProcessor` |
| a class a future reader will ask "why does this exist?" about | `@Bean`, in a named configuration, with a comment |

**Diagnostic:** `context.getBeansOfType(CatalogService.class).keySet()` returns every
bean of that type with its name. If it has more than one entry and you expected one,
you have found this.

---

## Hands-on proof

Commands **you** run. I have no JVM and no running `orderflow`, so nothing below is
captured output — it is what to run, what to look for, and how to read each result.

### Proof 1 — dump only *your* definitions, with their source

`platform/DefinitionDump.java`:
```java
package com.orderflow.platform;

import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.annotation.*;
import java.util.Arrays;

@Configuration
public class DefinitionDump {

    @Bean
    public ApplicationRunner dump(ConfigurableApplicationContext ctx) {
        return args -> {
            var factory = ctx.getBeanFactory();
            System.out.println("=== APPLICATION-ROLE DEFINITIONS ===");
            Arrays.stream(factory.getBeanDefinitionNames()).sorted().forEach(name -> {
                BeanDefinition d = factory.getBeanDefinition(name);
                if (d.getRole() != BeanDefinition.ROLE_APPLICATION) return;
                System.out.printf("%-40s class=%-55s scope=%-10s lazy=%-5s primary=%s factoryMethod=%s%n",
                        name, d.getBeanClassName(), d.getScope(), d.isLazyInit(),
                        d.isPrimary(), d.getFactoryMethodName());
            });
        };
    }
}
```

```bash
./mvnw spring-boot:run
```

**What to look for:** the `factoryMethod` column. It is the registration-path tell.

| What you see | What it means |
|---|---|
| `factoryMethod=null`, class is one of yours | registered by **component scanning** |
| `factoryMethod=<a method name>` | registered by a **`@Bean` method** of that name |
| `class=null` on some rows | the definition was registered with a factory rather than a class name; look at `getFactoryBeanName()` |
| long fully-qualified names in the list | Boot's auto-configuration classes (Topic 42), registered under their FQCN so they cannot collide with yours |
| `scope=` blank or empty string | means singleton — Spring stores the default as an empty string, not the literal `"singleton"` |
| a bean you did not expect | walk the six registration paths. It came from one of them. |

Keep this class. It is the fastest answer to "why does this bean exist?" that you
will have until you learn the Actuator `beans` endpoint (Topic 118).

### Proof 2 — see the scanner's candidates without starting an application

`ScanProbe.java`:
```java
package com.orderflow.platform;

import org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider;
import org.springframework.core.type.filter.AnnotationTypeFilter;
import org.springframework.stereotype.Component;

public class ScanProbe {
    public static void main(String[] args) {
        var provider = new ClassPathScanningCandidateComponentProvider(false);
        provider.addIncludeFilter(new AnnotationTypeFilter(Component.class));

        String base = args.length > 0 ? args[0] : "com.orderflow";
        long start = System.nanoTime();
        var candidates = provider.findCandidateComponents(base);
        long millis = (System.nanoTime() - start) / 1_000_000;

        candidates.stream()
                  .map(d -> d.getBeanClassName())
                  .sorted()
                  .forEach(System.out::println);
        System.out.println("--- " + candidates.size() + " candidates in " + millis + " ms for " + base);
    }
}
```

Run it with different base packages:

```bash
./mvnw -q compile exec:java -Dexec.mainClass=com.orderflow.platform.ScanProbe -Dexec.args="com.orderflow"
./mvnw -q compile exec:java -Dexec.mainClass=com.orderflow.platform.ScanProbe -Dexec.args="com.orderflow.payments"
./mvnw -q compile exec:java -Dexec.mainClass=com.orderflow.platform.ScanProbe -Dexec.args="com"
```

(If `exec:java` is not configured, run the class from your IDE instead — the point is
the comparison, not the launcher.)

**What to look for:** the candidate count and the elapsed milliseconds for each base
package.

| What you see | What it means |
|---|---|
| `com.orderflow` lists your components, small count, fast | the expected baseline |
| `com.orderflow.payments` lists a strict subset | confirms scanning is package-prefix based and recursive |
| `com` lists far more, and takes noticeably longer | **this is Trap 3, measured.** Note both numbers before you ever widen a scan. |
| `com` throws or hangs | it is reading every class file in every jar. That is the point of the demonstration. |
| an abstract class or interface you annotated is **absent** | Trap 4 confirmed — the candidate check rejects non-concrete types silently |

The `false` argument to the constructor means "do not apply default filters", so the
filter you add is the only one. That is what lets you probe for your `@UseCase`
stereotype specifically: swap the filter to
`new AnnotationTypeFilter(UseCase.class)` and re-run.

### Proof 3 — prove programmatic registration produced N beans

With the Example 2 registrar in place:

```bash
./mvnw spring-boot:run
```

and add to `PaymentRouter`'s constructor:
```java
System.out.println(">>> gateways wired: " + byId.keySet());
```

| What you see | What it means |
|---|---|
| the set contains one entry per `providers:` entry in your YAML | working as designed |
| the set is empty | `Binder` found nothing. Check the property prefix matches `orderflow.payments` exactly, and that the YAML list indentation is right. |
| fewer entries than YAML rows | two providers share an `id`, so `name()` collided in the map. The registrar's duplicate check should have caught it — verify it ran. |
| `BeanDefinitionOverrideException` on `paymentGateway-<id>` | two configuration sources define the same provider id; check for a profile-specific YAML also contributing providers |
| the registrar never runs at all | the `@Bean` method is probably not `static`, or `PlatformConfig` is outside the scan root |

Then change the YAML — add a fourth provider — restart, and confirm the set grows
with no code change. That is the capability this pattern bought.

### Proof 4 — prove the scanner reads bytecode rather than loading classes

Add a component whose *method body* references a class you have not depended on:

```java
package com.orderflow.lab;

import org.springframework.stereotype.Component;

@Component
public class OptionalIntegration {
    public String describe() {
        // com.example.absent.Thing is NOT on the classpath.
        // This will not compile unless you provide a stub at compile time
        // and then remove the jar at runtime — see the note below.
        return "unused";
    }
}
```

The honest version of this experiment, without fighting the compiler: instead of
removing a dependency, use `ScanProbe` from Proof 2 with a filter that matches a
class, then check whether a `static { System.out.println("LOADED"); }` initialiser in
that class prints.

```java
@Component
public class LoadWitness {
    static { System.out.println(">>> LoadWitness class was LOADED"); }
}
```

| What you see | What it means |
|---|---|
| `ScanProbe` lists `LoadWitness` but `>>> LoadWitness class was LOADED` does **not** print | confirmed: the scanner read the class file with ASM and never initialised the class. This is why `@ConditionalOnClass` is safe. |
| the `LOADED` line prints during the probe | something did load it — check you are not calling `Class.forName` or referencing the type in your probe code |
| the `LOADED` line prints during a full `spring-boot:run` | expected: at that point the container is actually instantiating the bean, which requires loading |

### Proof 5 — find where a mystery bean came from, end to end

Pick any bean from Proof 1's dump that you did not write — a `DataSource`, an
`ObjectMapper`, a `MeterRegistry`.

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug
```

**What to look for:** search the condition-evaluation report's "Positive matches" for
the bean's type or the auto-configuration class name.

| What you see | What it means |
|---|---|
| a matched auto-configuration naming a `@Bean` method for it | registration path 4. The jar is on your classpath and Boot's imports file listed it. |
| nothing in the report, but the bean is in the dump with a `factoryMethod` | registration path 2 or 3 — a `@Bean` method in your own or an imported configuration |
| nothing in the report and `factoryMethod=null` | registration path 1 or 5 — scanned, or registered programmatically. Grep for the class name and for `registerBeanDefinition`. |

Then run `mvn dependency:tree` and locate the jar that contributed it. The full chain
— YAML or annotation, to registration path, to jar, to Maven coordinate — is the
skill. Practise it on three beans.

---

## Practice exercises

### 1 — easy: map every registration path

In a fresh `orderflow` project, deliberately create one bean by each of the first
three registration paths, and one by the fifth:

1. `InventoryService` by component scanning.
2. A `java.time.Clock` by a `@Bean` method (a third-party class you cannot annotate —
   use `Clock.systemUTC()`).
3. A configuration class pulled in with `@Import` rather than scanned. Prove it is
   not being scanned by putting it in a package *outside* your scan root.
4. Two `HttpPaymentGateway` beans registered from a
   `BeanDefinitionRegistryPostProcessor`.

Then run the Proof 1 dump and, for each of the beans, state which column of the
output identifies its registration path. Where two paths are indistinguishable in the
dump, say so and explain what you would do instead to tell them apart.

### 2 — medium: combining earlier topics

Build a `@UseCase` custom stereotype and use it to enforce a real rule.

1. **Topic 03:** make `InMemoryProductRepository` **package-private**. Confirm the
   application still starts — Spring instantiates it reflectively. Then try to
   reference the type from `com.orderflow.orders` and observe the compile error.
   Write two sentences on why this is a stronger boundary than any Spring annotation.
2. **Topic 04:** define `@UseCase` as a meta-annotated `@Service`. Explain, in terms
   of annotation retention, why `@Retention(RUNTIME)` is mandatory here and what
   would happen with `CLASS`.
3. **Topic 27 + 26:** annotate `OrderService` with `@UseCase` and confirm it still
   registers. Print its bean name — predict it before you look.
4. **Topics 31–32:** run `mvn dependency:tree -Dincludes=org.springframework:*`. For
   each of `spring-core`, `spring-beans`, `spring-context`, say which one contains
   the scanner, which contains `BeanDefinition`, and which contains
   `AnnotationConfigApplicationContext`. Verify with
   `jar tf ~/.m2/repository/.../spring-context-*.jar | grep ClassPathBeanDefinitionScanner`.
   Then answer: if `spring-context` were excluded by a `dependencyManagement`
   mistake, at what point would you find out, and what would the error be?
5. Write an ArchUnit test asserting that every class annotated `@UseCase` lives in a
   package ending in `.orders`, `.catalog` or `.payments`, and that no `@UseCase`
   class is annotated `@Repository`. Break it deliberately and confirm it fails.

### 3 — hard: production simulation — provider onboarding without a deploy

Operations needs to onboard a new payment provider by changing configuration only.
Build it, then prove it under the constraints.

**Part A — implement Example 2 fully.** `PaymentProviderProperties`,
`PaymentGatewayRegistrar`, `HttpPaymentGateway`, `PaymentRouter`. Wire `OrderService`
to route through `PaymentRouter`.

**Part B — three profiles.** Create `application-local.yml` with one provider,
`application-staging.yml` with three, and `application-load.yml` with five. Start the
app under each profile and print `router.availableProviders()`. Record the set for
each. Then answer: what happens if `application.yml` *and* a profile YAML both define
`orderflow.payments.providers`? Do the lists merge or does one replace the other?
Verify empirically rather than guessing — this is a genuine Topic 43 trap and getting
it wrong in production means silently losing providers.

**Part C — make it fail well.** Introduce each of these and record the exact
exception type, the message, and whether the failure is at startup or at request time:

1. Two providers sharing `id: acquirer-eu`.
2. A provider with a missing `endpoint`.
3. A provider whose `timeout` is written `900` instead of `900ms`.
4. Two providers both marked primary (modify the registrar to set `setPrimary(true)`
   for all).

For each, decide whether the failure should be at startup, and if it currently is
not, change the registrar so it is. Write down the principle you are applying.

**Part D — discoverability.** You have made the bean graph less visible in exchange
for configuration-driven onboarding. Write the mitigation:

- A startup log line listing every registered gateway bean name, its provider id and
  its endpoint (mask any credentials).
- A test that fails if the set of registered providers differs from a checked-in
  expected list for the `load` profile, so an accidental YAML change is caught in CI.

Then write three sentences for your team's README explaining where payment gateway
beans come from, aimed at someone who greps for `HttpPaymentGateway` and finds no
`new`.

---

## Interview questions

### Q1 — "How does component scanning decide what to register?"

**Mid-level answer:** "Spring scans the package of the `@SpringBootApplication` class
and all sub-packages, and registers any class annotated with `@Component` or a
stereotype like `@Service` or `@Repository`."

**Senior answer:** "The base package is turned into a classpath resource pattern —
effectively `classpath*:com/orderflow/**/*.class` — and every matching class file is
enumerated across directories and jars. For each one Spring reads the annotation
metadata with **ASM**, not by loading the class, which is important: a candidate can
be examined and rejected without ever being initialised. That is what makes
`@ConditionalOnClass` safe. Then the filters apply: by default, include anything
meta-annotated `@Component`, and Boot adds a `TypeExcludeFilter` that the test slices
use. Only independent concrete classes become candidates, so an interface or abstract
class annotated `@Component` is silently skipped. The bean name is the simple class
name decapitalised via `Introspector.decapitalize`, which leaves names starting with
two capitals unchanged. The operational consequence I care about is that scan cost is
proportional to the number of class files under the base package across every jar —
which is why a base package of `com` is a startup and test-suite problem."

**What separates them:** ASM versus classloading, the concrete-class requirement, and
a cost model for scan breadth. The mid answer describes the rule; the senior answer
describes the mechanism and what it costs.

**Follow-up:** "So why can `@ConditionalOnClass` reference a class that is not on the
classpath without throwing?" They want: the condition is evaluated from metadata, and
the annotation's value is read as a string from the constant pool rather than as a
resolved `Class`.

---

### Q2 — "Coming from Nest, what do you lose by using component scanning instead of explicit module imports?"

**Mid-level answer:** "It is less explicit. You cannot see the dependencies in one
file, but it is much less boilerplate."

**Senior answer:** "Two distinct things. First, **the graph is not written down**.
Nest's `imports` array is the graph; in Spring there are six independent registration
paths — scanning, `@Bean`, `@Import`, auto-configuration imports files,
`ImportBeanDefinitionRegistrar`, and programmatic registry post-processors — and no
single file lists them. So 'why does this bean exist?' becomes a six-step
investigation instead of a grep. Second, **there is no visibility control**: Nest's
`exports` genuinely prevents a provider being injected outside its module; one Spring
`ApplicationContext` is a flat namespace where anything can inject anything. If I
want that boundary I have to build it from package-private types, ArchUnit rules,
Spring Modulith or separate Maven modules. What I gain is real too — adding a service
is one annotation instead of four edits, and Nest teams genuinely do forget the
`exports`. So my rule is: scan the ordinary case, and use an explicit `@Bean` in a
named configuration whenever a future reader will ask why this bean exists or why it
is configured this way. Traceability is worth verbosity at exactly those points."

**What separates them:** separating "not written down" from "no visibility control" —
they are different losses with different remedies. And naming a decision rule rather
than a preference.

**Follow-up:** "You inherited a service. A bean of type `TaskExecutor` exists and you
did not create it. How do you find out where it came from?" They want: dump
definitions with role and factory method, run `--debug` for the condition report,
then `mvn dependency:tree` to the jar.

---

### Q3 — "You need one bean per entry in a configuration list. How?"

**Mid-level answer:** "I would have one service bean that holds a map of
configurations and looks up the right one at call time."

**Senior answer:** "That works and it is what most codebases do, but it collapses N
things into one bean and you lose everything that is per-bean: metrics tagged by
provider, an independent circuit breaker, an independent health indicator, per-bean
configuration properties and per-bean lifecycle. When one provider degrades, your
dashboard says 'payments are slow' and stops there. The container-native answer is a
`BeanDefinitionRegistryPostProcessor`: it runs in phase one, before any bean exists,
so I can read the configuration with `Binder` straight off the `Environment`, loop
the entries, and call `registry.registerBeanDefinition` once per provider with
`BeanDefinitionBuilder` supplying the constructor arguments. Consumers then inject
`List<PaymentGateway>` or `Map<String, PaymentGateway>` and get all of them. The
honest cost is discoverability — a reader greps for the class and finds nothing that
constructs it — so I would pay that back with an explicitly named registrar class,
readable bean names, and a startup log line listing what was registered."

**What separates them:** knowing the hook exists *and* naming what per-bean identity
buys operationally, then volunteering the cost and its mitigation. Candidates who
only know `@Bean` and `@Component` cannot reach this answer.

**Follow-up:** "Why must the `@Bean` method returning that post-processor be
`static`?" Because a non-static `@Bean` method requires its configuration class to be
instantiated to call it, and a `BeanDefinitionRegistryPostProcessor` must exist before
ordinary configuration classes are processed. Spring logs a warning when you get this
wrong.

---

### Q4 — "Startup fails with `BeanDefinitionOverrideException`. What do you do?"

**Mid-level answer:** "Set `spring.main.allow-bean-definition-overriding=true`, or
rename one of the beans."

**Senior answer:** "I would not enable overriding. That flag does not fix anything —
it makes one of the two definitions silently win, and which one wins depends on
registration order, which for scanned beans depends on filesystem enumeration order.
That is a bug that behaves differently on a laptop and in CI. The exception message
names the bean name and both definitions, so first I identify the two sources. The
usual cause is two classes with the same simple name in different packages, since the
default name generator uses the decapitalised simple name and ignores the package.
The fix in order of preference: rename the classes so they are distinct — if two
classes share a name, humans will confuse them too; or give explicit bean names; or,
for a new codebase, use `FullyQualifiedAnnotationBeanNameGenerator` so collisions are
structurally impossible, accepting that every qualifier string then uses the full
class name. The other case is a class registered twice by two different paths — a
`@Service` annotation *and* a `@Bean` method — and there the fix is to delete one of
them, because you never wanted two."

**What separates them:** refusing the flag and explaining precisely why it is
dangerous (order-dependent, environment-dependent), plus naming both root causes.

**Follow-up:** "When would enabling overriding be legitimate?" The honest answer is
"in a test, to replace a bean deliberately" — and even then `@MockitoBean` or a test
configuration is better, because it is scoped to the test rather than global
(Topic 60).

---

### Q5 — "What is the difference between `@Component` and `@Bean`, and when do you use each?"

**Mid-level answer:** "`@Component` goes on a class and is picked up by scanning;
`@Bean` goes on a method inside a `@Configuration` class. Use `@Bean` for
third-party classes you cannot annotate."

**Senior answer:** "Mechanically: `@Component` produces a definition with a bean
class name that Spring instantiates reflectively; `@Bean` produces a definition with
a factory bean name and factory method name, so Spring calls your method and takes
the return value. That difference is visible in a definition dump and is how I tell
which path registered a bean. Practically, I use scanning for my own classes with no
construction logic, and `@Bean` in four cases: a third-party class I cannot annotate;
a class needing constructor arguments that are not injectable by type, like a
timeout or an endpoint; where I want `initMethod`/`destroyMethod` on a class I do not
own; and — the one people miss — where a future reader will ask *why* this bean
exists. Scanning is one annotation and no explanation. An explicit `@Bean` method in a
named configuration is a place to put the reason. I treat the verbosity as the price
of traceability and pay it deliberately. The failure mode to avoid is doing both for
the same class, which either collides at startup or produces two instances of which
only one is ever warmed."

**What separates them:** the factory-method-versus-class-name mechanic, four concrete
criteria rather than one, and naming the double-registration failure with its silent
symptom.

**Follow-up:** "If two `@Bean` methods in different configuration classes return the
same type, what happens?" Both are registered under different names; injection by
type is then ambiguous unless one is `@Primary` or the injection point qualifies —
which is Topic 39.

---

## Mental model checkpoint

1. Scanning reads class files with ASM rather than loading classes. Name two Spring
   features that would be impossible, or unsafe, if it loaded them instead.

2. Nest requires four edits to add a shared provider; Spring requires one annotation.
   Describe a codebase size and team shape where Nest's extra three edits are
   *worth it*, and one where they are not. What changes between the two?

3. There is no `exports` in Spring. You have four ways to build a boundary
   (package-private, ArchUnit, Spring Modulith, separate Maven modules). Rank them by
   strength and by cost of adoption, and say which you would introduce first into a
   200 000-line service that currently has none.

4. `@Component` on an interface silently does nothing. Argue that this should be a
   compile error or at least a startup warning. Then argue why Spring cannot easily
   make it one.

5. A `BeanDefinitionRegistryPostProcessor` can register beans that appear in no
   source file. Under what circumstances is that power worth its discoverability
   cost? Write the rule you would put in a team style guide.

6. Auto-configuration classes are registered under fully-qualified names while your
   scanned beans get short names. What problem does that asymmetry solve, and what
   would break if Boot used short names for auto-configurations?

7. You are told a service's startup takes 14 seconds and someone blames "Spring being
   slow". Given only this topic, list three specific, measurable hypotheses and the
   command you would run to test each.

---

## Quick reference card

### The six registration paths

| # | Path | Tell in a definition dump |
|---|---|---|
| 1 | component scanning | `factoryMethod=null`, class name is yours |
| 2 | `@Bean` method | `factoryMethod=<method name>` |
| 3 | `@Import` of a configuration | as 2, but the config class is not in a scanned package |
| 4 | auto-configuration imports file | fully-qualified bean names; appears in the `--debug` report |
| 5 | `BeanDefinitionRegistryPostProcessor` | `factoryMethod=null` and no source annotation anywhere |
| 6 | `@ComponentScan` on extra packages | as 1, but the class is from another root |

### Key APIs

```java
BeanDefinitionRegistry registry;                   // add/remove definitions (phase 1)
registry.registerBeanDefinition(name, definition);
registry.containsBeanDefinition(name);

BeanDefinitionBuilder.genericBeanDefinition(HttpPaymentGateway.class)
        .addConstructorArgValue(id)
        .setPrimary(true)
        .setLazyInit(false)
        .setInitMethodName("warm")
        .getBeanDefinition();

Binder.get(environment).bind("orderflow.payments", Props.class);   // typed config in phase 1

context.getBeansOfType(PaymentGateway.class);      // name -> bean, every candidate
context.getBeanDefinition(name).getRole();         // ROLE_APPLICATION filters out Spring's own
context.getBeanDefinition(name).getFactoryMethodName();

new ClassPathScanningCandidateComponentProvider(false)   // probe scanning standalone
        .findCandidateComponents("com.orderflow");
```

### Annotations

```java
@Component @Service @Repository @Controller @RestController @Configuration
@ComponentScan(basePackageClasses = Marker.class,
               includeFilters = @Filter(type = ANNOTATION, classes = UseCase.class),
               excludeFilters = @Filter(type = REGEX, pattern = ".*\\.internal\\..*"),
               useDefaultFilters = false,
               nameGenerator = FullyQualifiedAnnotationBeanNameGenerator.class,
               lazyInit = true)
@Bean(name = "x", initMethod = "warm", destroyMethod = "close")
@Import({ PaymentsConfig.class })
@Primary
```

### Naming rules

- scanned class -> simple name decapitalised (`InventoryService` -> `inventoryService`)
- **two leading capitals -> unchanged** (`SLAMonitor` -> `SLAMonitor`)
- `@Component("name")` / `@Bean("name")` override it
- auto-configurations -> fully-qualified class names
- names must be unique per context, or `BeanDefinitionOverrideException`

### Gotchas checklist

- [ ] Application class in the **root** package; never widen the scan to fix a jar.
- [ ] `basePackageClasses`, not `basePackages` strings.
- [ ] `@Component` on interfaces and abstract classes does nothing, silently.
- [ ] One registration path per class. Never `@Service` *and* a `@Bean` method.
- [ ] Never enable `spring.main.allow-bean-definition-overriding` to make an error stop.
- [ ] `BeanDefinitionRegistryPostProcessor` `@Bean` methods must be `static`.
- [ ] Custom stereotypes need `@Retention(RUNTIME)`.
- [ ] Programmatic registration owes the codebase a startup log line and a README paragraph.

---

## When would I use this at work?

**1. Onboarding into an unfamiliar Spring service.** The first question is always
"where do these beans come from?" With the six-path checklist and a definition dump
you can answer it in minutes for any bean, including ones from jars. Without it you
read source files and guess.

**2. Designing the package layout for a new service.** Package-by-feature versus
package-by-layer is a decision you make once and live with for years. Knowing that
package-private is the only free encapsulation Java gives you — and that a layered
layout forfeits it entirely — turns a taste argument into a technical one.

**3. Building configuration-driven infrastructure.** Any time the requirement is "one
of these per configured X" — payment providers, tenants, Kafka topics, outbound
webhooks — you now have the container-native answer instead of a map inside a god
service, along with a clear account of the discoverability you owe back.

---

## Connected topics

**Prerequisites:**
- **03 — Access modifiers and packages**: package-private classes are the boundary
  that package-by-feature makes possible and package-by-layer destroys.
- **04 — Interfaces and abstract classes**: why `@Component` on either does nothing.
- **31–32 — Maven and dependency resolution**: scanning enumerates class files across
  **the classpath Maven assembled**. A jar being present and its package being scanned
  are two different facts, and conflating them is Trap 1.
- **35 — `ApplicationContext`**: the two-phase split. Everything here happens in
  phase 1.

**This unlocks:**
- **37 — Bean lifecycle**: what happens to each definition once it becomes an object,
  and where `initMethod` fits.
- **38 — Bean scopes**: `BeanDefinition.setScope` is the property this topic
  introduced; Topic 38 is what the values mean.
- **39 — Injection styles**: `@Qualifier` matches on the bean names this topic
  generates, which is why the two-leading-capitals rule matters.
- **42 — Auto-configuration**: registration path 4 in full — the imports file,
  conditions, and ordering.
- **43 — Configuration and `@ConfigurationProperties`**: `Binder` used in phase 1,
  and the profile-merge question in Exercise 3 Part B.
- **60 — Test slices and context caching**: slices work by swapping the scan's
  include and exclude filters, which is why `@WebMvcTest` starts far fewer beans than
  `@SpringBootTest`.
- **118 — Metrics**: per-bean gateway registration is what makes per-provider metrics
  possible.

---

## `[BOOT 3.x DELTA]`

**Scanning and registration: unchanged.** `ClassPathBeanDefinitionScanner`, ASM-based
metadata reading, filters, the naming rules, `BeanDefinitionRegistryPostProcessor`
and `@Import` behave identically on Boot 3.x / Framework 6.x. Stated explicitly
rather than manufacturing a difference.

Real differences relevant to this topic:

| Area | Boot 2.x | Boot 3.x | Boot 4.x |
|---|---|---|---|
| Auto-configuration registration file | `META-INF/spring.factories` (deprecated from 2.7) | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` | same as 3.x |
| Auto-config annotation | `@Configuration` + factories entry | `@AutoConfiguration` | `@AutoConfiguration` |
| Bean definition overriding default | allowed before 2.1, disabled from 2.1 | disabled | disabled |
| Starter artifact layout | monolithic starters | monolithic starters | **modularised** — some artifact ids and package names moved, so a dependency coordinate copied from a 3.x article may not resolve |
| Test bean replacement | `@MockBean` | `@MockBean` (deprecated late in the line) | `@MockitoBean` |

If you are reading a Boot 2.x codebase and cannot find why a library's beans exist,
look for `META-INF/spring.factories` inside its jar — that is the 2.x equivalent of
the imports file and it is easy to miss.

---

*Java baseline 21, running on JDK 25. Spring Framework 7.0 / Spring Boot 4.1,
Jakarta EE 11 — `jakarta.*` imports, never `javax.*`. No Spring artifact versions are
quoted here: the Boot BOM supplies them, and the authoritative current version is
whatever a freshly generated start.spring.io project puts in its parent POM.*
