# 37 — Bean Lifecycle, Callbacks, and `BeanPostProcessor`

## Phase: 4 — Spring Core
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow` gains a catalogue warm-up. A `ProductCache` bean is preloaded at startup so the first customer request does not pay for a cold cache — built the wrong way first, then the right way, with the readiness consequences of each spelled out.

---

## ELI5 anchor

You hire someone. They do not start working the moment they walk in.

1. They arrive. (A body exists.)
2. HR gives them a laptop, a desk, a phone. (They now have their things.)
3. They are told their own name, their team's name, and where the building's
   facilities are. (They know their context.)
4. Security checks their badge on the way in, and may hand them a visitor escort.
5. They do their own first-day setup — read the runbook, warm up their tools.
6. Their manager runs a final onboarding checklist.
7. **Security wraps them in an escort.** From now on, anyone who wants to talk to
   them talks to the escort first, and the escort may add a step — logging the
   conversation, checking permission, starting a timer.
8. They work.
9. On their last day they hand back the laptop and close their accounts.

Step 7 is the whole point of this document.

The escort is a **proxy**. `@Transactional`, `@Cacheable`, `@Async`, `@Retryable`
and security checks are all the escort adding a step. And here is the consequence
that catches everybody: **during step 5, on their first day, the escort does not
exist yet.** If the new hire tries to invoke the escort's rules on themselves during
their own onboarding, nothing happens. No error. The rule just does not apply.

That is why a `@Transactional` method called from `@PostConstruct` is not
transactional. Everything else in this topic is detail; that is the payload.

---

## The bridge from what you know

### Nest lifecycle hooks ≈ Spring lifecycle callbacks — PARTIAL

Nest has lifecycle hooks and you have used them:

```ts
@Injectable()
export class ProductCache implements OnModuleInit, OnApplicationBootstrap, OnModuleDestroy {
  async onModuleInit()          { /* this module's providers are resolved */ }
  async onApplicationBootstrap(){ /* ALL modules are initialised */ }
  async onModuleDestroy()       { /* shutdown */ }
}
```

Spring's equivalents:

| Nest | Spring | Verdict |
|---|---|---|
| `onModuleInit` | `@PostConstruct` / `afterPropertiesSet()` | **HONEST ANALOGUE** — "this object's dependencies are in place" |
| `onApplicationBootstrap` | `SmartInitializingSingleton.afterSingletonsInstantiated()` or an `ApplicationReadyEvent` listener | **HONEST ANALOGUE** — "everything exists now" |
| `onModuleDestroy` | `@PreDestroy` / `destroy()` | **HONEST ANALOGUE** |
| `beforeApplicationShutdown` / `onApplicationShutdown` | `SmartLifecycle.stop()` with phases, plus graceful shutdown | **PARTIAL** — Spring's phasing is more explicit |
| **async** hooks — Nest awaits your promise | Spring's callbacks are **synchronous and blocking** | **NO ANALOGUE** — this matters a lot, see Trap 2 |
| — | `BeanPostProcessor` | **NO ANALOGUE** |

Two differences deserve real attention.

**1. Nest awaits your hooks; Spring blocks on yours.**
`async onModuleInit()` returns a promise Nest awaits, and Node's event loop stays
free for other work. Spring's `@PostConstruct` is an ordinary synchronous method call
on the thread performing `refresh()`. Whatever it does, startup waits for it, on one
thread. A 4-second catalogue load in `@PostConstruct` is 4 seconds added to every pod
start and every test context refresh. There is no "await" that lets startup proceed.

**2. `BeanPostProcessor` has no Nest counterpart at all, and it is how Spring works.**

In Nest, an interceptor or guard is attached at the **route boundary**. The provider
you wrote is the object that gets injected everywhere; the interception happens
outside it, in the request pipeline.

In Spring, a `BeanPostProcessor` is handed **every bean, one at a time, as it is
created**, and may return a *different object*. That returned object is what gets
stored in the container and injected everywhere. So `@Transactional` on
`OrderService` means: at creation time, a post-processor replaced your `OrderService`
with a generated subclass that begins a transaction and then calls yours.

Three consequences that follow directly, and that you should hold onto:

- Interception in Spring applies to **any** bean method reached through the
  container, not just HTTP entry points. That is more powerful than Nest's route
  interceptors.
- It only applies to calls that arrive **through the proxy**. A call from inside the
  bean to its own method bypasses it entirely. Nest has no equivalent trap, because
  the interception is not on the object. This is Topic 40.
- The replacement happens at a **specific step** in the lifecycle, and code running
  before that step does not see it. That is this topic's payload.

---

## What is this?

The bean lifecycle is the fixed sequence of things Spring does to each bean between
"the class exists" and "the object is discarded". You need it in order, because
almost every question about "why did my annotation not work?" is answered by knowing
which step runs before which.

### The full order — memorise this

```
CREATION (per bean, inside finishBeanFactoryInitialization)

 1.  Instantiate               constructor runs; constructor-injected deps supplied here
 2.  Populate properties       field and setter injection happen here (Topic 39)
 3.  *Aware callbacks          BeanNameAware.setBeanName
                               BeanClassLoaderAware.setBeanClassLoader
                               BeanFactoryAware.setBeanFactory
 4.  BeanPostProcessor.postProcessBeforeInitialization(bean, name)
                               every registered BPP, in order
 5.  @PostConstruct            your annotated method
 6.  InitializingBean.afterPropertiesSet()
 7.  custom init-method        @Bean(initMethod = "warm")
 8.  BeanPostProcessor.postProcessAfterInitialization(bean, name)
                               *** AOP PROXIES ARE CREATED HERE ***
                               the returned object replaces yours in the container

 9.  IN USE                    the container hands out the step-8 object

DESTRUCTION (singletons only, on context close)

10.  @PreDestroy
11.  DisposableBean.destroy()
12.  custom destroy-method     @Bean(destroyMethod = "close")
```

Read step 8 again. Steps 1 to 7 all run on **your** object. Step 8 may hand back a
**different** object, and that different object is what every other bean gets
injected with. Anything your code did in steps 1 to 7 was done by a thing that did
not yet know it would be wrapped.

### Three honest refinements

The list above is the canonical teaching order and it is the one to recite. Three
details make it accurate rather than merely useful:

**`@PostConstruct` is itself implemented by a `BeanPostProcessor`.**
`CommonAnnotationBeanPostProcessor` finds your `@PostConstruct` method and invokes it
from *its own* `postProcessBeforeInitialization`. So step 5 happens *inside* step 4,
performed by one particular post-processor. This does not change the ordering you
recite; it explains why `@PostConstruct` needs no interface and why it can be
disabled by removing that post-processor.

**Some `*Aware` callbacks also run inside step 4.**
`BeanNameAware`, `BeanClassLoaderAware` and `BeanFactoryAware` are invoked directly
by the bean factory at step 3. But `ApplicationContextAware`, `EnvironmentAware`,
`ResourceLoaderAware`, `ApplicationEventPublisherAware` and `MessageSourceAware` are
invoked by `ApplicationContextAwareProcessor` — a `BeanPostProcessor` — during step 4.
Practically: all of them are available by the time `@PostConstruct` runs, which is all
you need day to day.

**Prototype beans get steps 1 to 8 and then nothing.**
Spring does not track prototype instances after handing them out, so steps 10 to 12
**never run** for them. A prototype bean holding a file handle or a connection leaks
it. Topic 38.

### The application-level events, in order

Beyond individual beans, `SpringApplication` publishes events you can listen to:

```
ApplicationStartingEvent
ApplicationEnvironmentPreparedEvent
ApplicationContextInitializedEvent
ApplicationPreparedEvent
  --- refresh() runs: all beans created, all lifecycle callbacks fire ---
ContextRefreshedEvent
ApplicationStartedEvent
  --- ApplicationRunner and CommandLineRunner beans execute here ---
ApplicationReadyEvent          <- readiness flips to ACCEPTING_TRAFFIC after this
  (or ApplicationFailedEvent if startup threw)
```

That ordering is the basis of the right answer in Example 2: work that must happen
"once everything exists" belongs after `refresh()`, not inside a bean's own
`@PostConstruct`.

### `BeanPostProcessor` itself

```java
public interface BeanPostProcessor {
    default Object postProcessBeforeInitialization(Object bean, String beanName) { return bean; }
    default Object postProcessAfterInitialization(Object bean, String beanName)  { return bean; }
}
```

Two methods, both handed the bean and its name, both returning an object. Return the
same object to observe; return a different one to replace. Every bean in the context
passes through every registered post-processor.

This is how Spring implements, among other things:

| Feature | Post-processor (approximate) |
|---|---|
| `@Autowired`, `@Value` | `AutowiredAnnotationBeanPostProcessor` |
| `@PostConstruct`, `@PreDestroy`, `@Resource` | `CommonAnnotationBeanPostProcessor` |
| `ApplicationContextAware` and friends | `ApplicationContextAwareProcessor` |
| `@Transactional`, `@Async`, `@Cacheable`, AOP | `AnnotationAwareAspectJAutoProxyCreator` (an `AbstractAutoProxyCreator`) |
| `@ConfigurationProperties` binding | `ConfigurationPropertiesBindingPostProcessor` |

Nothing in that list is a language feature. All of it is objects being handed other
objects at step 4 or step 8.

---

## Why does it matter?

**1. It answers "why did my annotation do nothing?" deterministically.**
`@Transactional` in a method called from `@PostConstruct`: step 5 runs before step 8,
so no proxy exists from that bean's own point of view. `@Cacheable` the same.
`@Async` the same. One rule explains a whole family of silent failures.

**2. Startup work is on the critical path of every deploy.**
Your callbacks run synchronously on the startup thread. Forty pod starts a day times
four seconds of catalogue loading is a real cost, and if the load can fail, it is a
rollout-blocking failure. Choosing *which* hook to use is a production decision, not
a style one.

**3. Shutdown correctness is a data question.**
`@PreDestroy` runs on an orderly close. It does not run on `SIGKILL`, on an OOM kill,
or when the JVM's shutdown hook is skipped. Anything you "flush on shutdown" is
therefore best-effort. Knowing that changes how you design write buffers and
in-flight request draining.

---

## Syntax breakdown

### `@PostConstruct` and `@PreDestroy`

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Service
public class ProductCache {

    @PostConstruct
    void warm() { /* runs at step 5 */ }

    @PreDestroy
    void flush() { /* runs at step 10 */ }
}
```

| Detail | Rule |
|---|---|
| package | **`jakarta.annotation`**, not `javax.annotation`. See Trap 4 — this is the single most common Boot 2-to-3 migration failure and it fails **silently**. |
| return type | must be `void` |
| parameters | none |
| visibility | any — package-private is good practice; these are not part of your API |
| exceptions | may throw; a thrown exception fails bean creation and therefore startup |
| inheritance | a `@PostConstruct` method on a superclass runs too |
| dependency | comes from `jakarta.annotation-api`, which the Spring Boot starters bring in transitively |

**Why prefer these over the interfaces:** no Spring or Jakarta interface appears in
your class's signature, so the class stays testable as a plain object and portable.

### `InitializingBean` and `DisposableBean`

```java
import org.springframework.beans.factory.*;

@Service
public class ProductCache implements InitializingBean, DisposableBean {
    @Override public void afterPropertiesSet() { /* step 6 */ }
    @Override public void destroy()            { /* step 11 */ }
}
```

Marginally faster (no annotation lookup) and completely explicit. The cost is
coupling your class to Spring interfaces. Spring's own documentation recommends the
annotations for application code. Use these when you are writing framework-level
infrastructure and the coupling already exists.

### Custom init and destroy methods

```java
@Configuration
public class CatalogConfig {

    @Bean(initMethod = "warm", destroyMethod = "close")
    public ProductCache productCache(ProductRepository repository) {
        return new ProductCache(repository);
    }
}
```

Steps 7 and 12. The reason this exists: it works on classes you **cannot annotate**,
which is the whole third-party-library case.

Two behaviours worth knowing:

- For `@Bean`, `destroyMethod` defaults to a value meaning "infer": if the returned
  type has a public no-arg `close()` or `shutdown()` method, Spring calls it
  automatically. That is usually what you want, and it is occasionally a surprise —
  set `destroyMethod = ""` to switch it off.
- If a bean has all three of `@PreDestroy`, `DisposableBean` and a `destroyMethod`,
  all three run, in that order.

### `SmartInitializingSingleton`

```java
@Component
public class CatalogWarmer implements SmartInitializingSingleton {
    @Override public void afterSingletonsInstantiated() {
        // every singleton in the context now exists and is fully initialised
    }
}
```

Runs **after all** singletons are created, not just this bean's dependencies. This is
the true analogue of Nest's `onApplicationBootstrap`, and it is the correct hook when
your work needs other beans to be *ready* rather than merely *injected*.

Still synchronous, still on the startup thread, still inside `refresh()`. Compare
with `ApplicationRunner` below, which runs after `refresh()` completes.

### `ApplicationRunner` / `CommandLineRunner` and `ApplicationReadyEvent`

```java
@Bean
public ApplicationRunner catalogWarmup(ProductCache cache) {
    return args -> cache.warm();      // runs after refresh(), before ApplicationReadyEvent
}

@Component
public class WarmupOnReady {
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() { /* runs after runners; readiness flips after this */ }
}
```

Both run **outside** `refresh()`, so every proxy exists and every bean is fully
usable. This is where `@Transactional` and `@Cacheable` calls actually work.

### `BeanPostProcessor`

```java
package com.orderflow.platform;

import org.springframework.beans.factory.config.BeanPostProcessor;

public class TimingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;                       // observe only
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;                       // returning a different object REPLACES the bean
    }
}
```

Registered with a **`static`** `@Bean` method, for the reason in Topic 35:

```java
@Bean
public static TimingBeanPostProcessor timingBeanPostProcessor() {
    return new TimingBeanPostProcessor();
}
```

**Ordering:** implement `Ordered` or annotate `@Order(...)`; lower values run first.
Spring's own auto-proxy creator has a defined order, so if you write a post-processor
that must see the proxy, order it after; if it must see your raw object, order it
before.

**The one rule that matters:** a `BeanPostProcessor` must not depend on ordinary
application beans. Doing so forces those beans to be created before the
post-processor set is complete, so they miss post-processing. Trap 5.

---

## Example 1 — minimal

One bean that implements every callback, and prints. Run it and read the order rather
than trusting this document.

```java
package com.orderflow.lab;

import jakarta.annotation.*;
import org.springframework.beans.factory.*;
import org.springframework.context.*;

public class LifecycleWitness
        implements BeanNameAware, BeanFactoryAware, ApplicationContextAware,
                   InitializingBean, DisposableBean, SmartInitializingSingleton {

    public LifecycleWitness()                { say("1  constructor"); }

    @Override public void setBeanName(String name)                 { say("3a setBeanName"); }
    @Override public void setBeanFactory(BeanFactory f)            { say("3b setBeanFactory"); }
    @Override public void setApplicationContext(ApplicationContext c) { say("3c setApplicationContext"); }

    @PostConstruct  void init()              { say("5  @PostConstruct"); }
    @Override public void afterPropertiesSet(){ say("6  afterPropertiesSet"); }
    public void customInit()                 { say("7  custom initMethod"); }

    @Override public void afterSingletonsInstantiated() { say("8b afterSingletonsInstantiated"); }

    @PreDestroy     void cleanup()           { say("10 @PreDestroy"); }
    @Override public void destroy()          { say("11 destroy()"); }
    public void customDestroy()              { say("12 custom destroyMethod"); }

    private static void say(String s) { System.out.println("   " + s); }
}
```

```java
package com.orderflow.lab;

import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.context.annotation.*;

@Configuration
public class LifecycleDemo {

    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    public LifecycleWitness lifecycleWitness() { return new LifecycleWitness(); }

    @Bean
    public static BeanPostProcessor lifecycleSpy() {
        return new BeanPostProcessor() {
            @Override public Object postProcessBeforeInitialization(Object bean, String name) {
                if (bean instanceof LifecycleWitness) System.out.println("   4  BPP before  -> " + bean.getClass().getName());
                return bean;
            }
            @Override public Object postProcessAfterInitialization(Object bean, String name) {
                if (bean instanceof LifecycleWitness) System.out.println("   8  BPP after   -> " + bean.getClass().getName());
                return bean;
            }
        };
    }

    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(LifecycleDemo.class)) {
            System.out.println("   9  IN USE");
        }
    }
}
```

Before you run it, write down your predicted order. Then check three specific things:

1. Does `4 BPP before` print above or below `5 @PostConstruct`? (This tells you
   whether you believed the "`@PostConstruct` is implemented by a post-processor"
   refinement above.)
2. Does `8b afterSingletonsInstantiated` print before or after `8 BPP after`?
3. Do the destruction lines print at all if you delete the try-with-resources?

---

## Example 2 — production scenario (on the project spine)

### The requirement and the constraints

`orderflow` serves 3 000 catalogue reads per second across 100 000 active products.
Product data is read constantly and changes rarely — a few hundred price or
availability updates an hour. An in-process cache is the obvious win.

The constraints that decide the design:

- **Cold-cache cost.** A miss goes to the product store. Under 3 000 rps, a pod
  starting with a completely cold cache takes sustained miss traffic for its first
  seconds. Across 40 pod starts a day, that is a recurring latency spike, and it
  shows up as p99 regressions correlated with deploys.
- **SLO:** `GET /products/{sku}` p99 under 40 ms.
- **Startup budget:** Kubernetes readiness probe has a 30-second `failureThreshold`
  window. A pod that takes longer than that to become ready is killed and restarted,
  which under a rolling deploy is a crash loop.
- **The product store can be slow or briefly unavailable.** It is a dependency, and
  dependencies fail.

Those last two are in tension, and that tension is the whole design problem.

### The cache bean

```java
package com.orderflow.catalog;

import org.springframework.stereotype.Component;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.LongAdder;

@Component
public class ProductCache {

    private final Map<String, Product> bySku = new ConcurrentHashMap<>();
    private final LongAdder hits   = new LongAdder();
    private final LongAdder misses = new LongAdder();
    private volatile boolean warm = false;

    public Optional<Product> get(String sku) {
        Product product = bySku.get(sku);
        if (product == null) { misses.increment(); return Optional.empty(); }
        hits.increment();
        return Optional.of(product);
    }

    public void put(Product product) { bySku.put(product.sku(), product); }

    public void putAll(Collection<Product> products) {
        products.forEach(this::put);
    }

    public void markWarm()      { this.warm = true; }
    public boolean isWarm()     { return warm; }
    public int size()           { return bySku.size(); }
    public long hitCount()      { return hits.sum(); }
    public long missCount()     { return misses.sum(); }
}
```

> A real deployment uses **Caffeine** here, for size bounds, TTL, refresh-after-write
> and built-in statistics (Topic 15 made this argument). A `ConcurrentHashMap` that
> only ever grows is a memory leak with 100 000 products and no eviction. The
> lifecycle lesson is identical either way, so the map keeps the example readable.
> `volatile` on `warm` matters: it is written by the startup thread and read by
> request threads, and without it there is no visibility guarantee (Topic 17,
> properly in Topic 88).

### Attempt 1 — the obvious wrong version

```java
@Service
public class CatalogService {

    private final ProductRepository repository;
    private final ProductCache cache;

    public CatalogService(ProductRepository repository, ProductCache cache) {
        this.repository = repository;
        this.cache = cache;
    }

    @PostConstruct                       // <-- the mistake
    void warmCache() {
        cache.putAll(repository.findAllActive());   // 100k rows, synchronous
        cache.markWarm();
    }

    public Optional<Product> findBySku(String sku) {
        return cache.get(sku).or(() -> repository.findBySku(sku));
    }
}
```

Four separate problems, and only one of them is obvious:

1. **Startup blocks.** `findAllActive()` runs on the `refresh()` thread. If it takes
   6 seconds, every pod start takes 6 seconds longer and every test context refresh
   pays it too (Topic 60).
2. **A slow dependency becomes a crash loop.** If the product store is degraded and
   the query takes 40 seconds, the readiness probe's window expires, Kubernetes kills
   the pod, and the restart runs the same query again. You have converted "the
   catalogue is slow" into "the service will not start".
3. **No retry, no partial success.** A single exception fails bean creation, which
   fails `refresh()`, which fails startup. There is nowhere to catch and degrade.
4. **The subtle one:** if you later add `@Transactional` or `@Cacheable` to
   `warmCache` or to anything it calls on `this`, it silently will not apply. Step 5
   runs before step 8. Trap 1.

### Attempt 2 — the version that ships

Three changes: move the work out of `refresh()`, make it non-blocking, and make
readiness honest about it.

```java
package com.orderflow.catalog;

import org.springframework.boot.availability.*;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.event.EventListener;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.stereotype.Component;
import jakarta.annotation.PreDestroy;

import java.time.Duration;
import java.util.List;
import java.util.concurrent.*;

@Component
public class CatalogWarmer {

    private final ProductRepository repository;
    private final ProductCache cache;
    private final ApplicationEventPublisher events;

    private final ExecutorService warmupExecutor =
            Executors.newSingleThreadExecutor(r -> {
                Thread t = new Thread(r, "catalog-warmup");
                t.setDaemon(true);          // must never keep the JVM alive
                return t;
            });

    public CatalogWarmer(ProductRepository repository, ProductCache cache,
                         ApplicationEventPublisher events) {
        this.repository = repository;
        this.cache = cache;
        this.events = events;
    }

    // AFTER refresh(). Every proxy exists. Nothing here blocks the container.
    @EventListener(ApplicationReadyEvent.class)
    public void startWarmup() {
        warmupExecutor.submit(this::warmInPages);
    }

    private void warmInPages() {
        int page = 0;
        int pageSize = 2_000;
        try {
            List<Product> batch;
            do {
                batch = repository.findActivePage(page++, pageSize);
                cache.putAll(batch);
            } while (!batch.isEmpty());

            cache.markWarm();
            events.publishEvent(new AvailabilityChangeEvent<>(this, ReadinessState.ACCEPTING_TRAFFIC));
        } catch (RuntimeException e) {
            // Degrade, do not die. Every lookup still works; it just misses.
            events.publishEvent(new AvailabilityChangeEvent<>(this, ReadinessState.REFUSING_TRAFFIC));
        }
    }

    @PreDestroy
    void stop() {
        warmupExecutor.shutdownNow();
        try {
            warmupExecutor.awaitTermination(Duration.ofSeconds(5).toMillis(), TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();   // Topic 89: never swallow an interrupt
        }
    }
}
```

`CatalogService` no longer warms anything; it just reads through:

```java
public Optional<Product> findBySku(String sku) {
    Optional<Product> cached = cache.get(sku);
    if (cached.isPresent()) return cached;
    Optional<Product> loaded = repository.findBySku(sku);
    loaded.ifPresent(cache::put);          // populate on miss; cold traffic still works
    return loaded;
}
```

**What each change bought:**

| Change | Consequence |
|---|---|
| `ApplicationReadyEvent` instead of `@PostConstruct` | startup is not blocked; the pod becomes ready in its normal time |
| dedicated daemon executor | the warm-up runs off the startup thread; a slow store cannot cause a crash loop |
| paged loading | bounded memory during load, and partial progress is useful |
| catch and publish `REFUSING_TRAFFIC` | a warm-up failure is *visible* instead of silent, and a degraded pod can be taken out of rotation |
| read-through on miss | correctness never depends on the warm-up having finished |
| `@PreDestroy` shutting the executor down | shutdown does not hang waiting for a background load |

**The honest caveat:** publishing `REFUSING_TRAFFIC` means a pod that cannot warm its
cache stops taking traffic. That is right only if serving uncached is genuinely
unacceptable. On a read-through cache it usually is not — you would rather serve
slowly than not at all. Decide deliberately: the code above is one policy, and the
alternative (stay ready, emit a metric, alert) is often the better one. What is *not*
acceptable is the Attempt 1 behaviour, where the decision is made for you by a
readiness probe timing out.

### A `BeanPostProcessor` that earns its place

Requirement: know how long every `orderflow` service's initialisation takes, without
adding timing code to each service.

```java
package com.orderflow.platform;

import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.core.Ordered;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class InitTimingBeanPostProcessor implements BeanPostProcessor, Ordered {

    private final Map<String, Long> startNanos = new ConcurrentHashMap<>();
    private final Map<String, Long> durationMillis = new ConcurrentHashMap<>();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (isOrderflowBean(bean)) startNanos.put(beanName, System.nanoTime());
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Long started = startNanos.remove(beanName);
        if (started != null) {
            durationMillis.put(beanName, (System.nanoTime() - started) / 1_000_000);
            // The proxy question, answered for free:
            System.out.printf("init %-28s %4d ms   runtime class = %s%n",
                    beanName, durationMillis.get(beanName), bean.getClass().getName());
        }
        return bean;                       // observe only; never replace
    }

    private boolean isOrderflowBean(Object bean) {
        return bean.getClass().getName().startsWith("com.orderflow");
    }

    @Override public int getOrder() { return Ordered.LOWEST_PRECEDENCE; }
}
```

`LOWEST_PRECEDENCE` is deliberate: running last in `postProcessAfterInitialization`
means the auto-proxy creator has already run, so `bean.getClass().getName()` shows you
the **proxy** class where one was created. That single printed line tells you which of
your beans are proxied and which are not — the fastest possible preview of Topic 40.

Registered, `static`, in `PlatformConfig`:

```java
@Bean
public static InitTimingBeanPostProcessor initTimingBeanPostProcessor() {
    return new InitTimingBeanPostProcessor();
}
```

Note what this post-processor does **not** do: it does not inject `CatalogService`,
`MeterRegistry` or any other application bean. It uses `System.nanoTime` and prints.
That restraint is the whole of Trap 5.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `@Transactional` or `@Cacheable` called from `@PostConstruct`

**Wrong:**
```java
@Service
public class InventorySeeder {

    private final InventoryRepository repository;

    public InventorySeeder(InventoryRepository repository) { this.repository = repository; }

    @PostConstruct
    void seed() {
        loadOpeningStock();          // a call on 'this'
    }

    @Transactional
    public void loadOpeningStock() {
        repository.deleteAll();
        repository.saveAll(readSeedFile());
        throw new IllegalStateException("seed file line 402 is malformed");
    }
}
```

**Exact symptom:** the exception propagates and startup fails — but **the deletes and
the partial inserts are still in the database.** Nothing rolled back. On a restart the
table is in a state no code path intended: partially deleted, partially seeded. With
`@Cacheable` instead of `@Transactional` the symptom is quieter — the method executes
its body every single time and the cache hit rate for that key stays at zero, with no
error anywhere.

**Root cause — two facts, both from the ordering:**

1. `@Transactional` is applied by a `BeanPostProcessor` at **step 8**. `@PostConstruct`
   runs at **step 5**. At step 5 the proxy has not been created yet.
2. Even after step 8, `loadOpeningStock()` here is a call on `this` — the raw object —
   not on the proxy. So it would bypass the advice even if the proxy existed. That is
   the self-invocation trap, Topic 40.

Both mean the same thing: no transaction manager was ever involved.

**Fix, in order of preference:**

1. **Move the work out of `refresh()` entirely.** An `ApplicationRunner` or an
   `ApplicationReadyEvent` listener runs after all proxies exist, and calls the
   service *through the container*, so the proxy is used:
   ```java
   @Bean
   public ApplicationRunner seedInventory(InventorySeeder seeder) {
       return args -> seeder.loadOpeningStock();   // 'seeder' IS the proxy
   }
   ```
   This is the right answer almost every time. It also gives you somewhere to catch,
   log and decide.
2. **Extract the transactional work into a separate collaborator bean** and inject it.
   Calling `other.loadOpeningStock()` goes through `other`'s proxy.
3. **Use `TransactionTemplate` explicitly** instead of the annotation. No proxy is
   involved, so ordering is irrelevant. Verbose but unambiguous.

**Do not** "fix" it by self-injecting the bean into itself. It works and it is a
signal that the design is wrong; Topic 40 covers why and when it is nonetheless the
pragmatic choice.

---

### Trap 2 — heavy or fallible I/O inside `@PostConstruct`

**Wrong:** Attempt 1 above — `repository.findAllActive()` for 100 000 products inside
`@PostConstruct`.

**Exact symptom, in escalating order:**

- Startup time rises. The "Started OrderflowApplication in X seconds" line grows.
- The test suite slows disproportionately: every non-cached `@SpringBootTest` context
  pays the same cost (Topic 60).
- Under a degraded product store, `refresh()` blocks past the readiness probe's
  `failureThreshold * periodSeconds` window. Kubernetes kills the pod. The replacement
  does the same. **A slow dependency has become a crash loop**, and because no pod
  reaches ready, a rolling deploy stalls with the old version still serving — which is
  lucky. If it happens on a scale-up rather than a deploy, you simply cannot add
  capacity during the exact incident where you need it.
- If the store throws, `refresh()` throws, and the log shows `BeanCreationException`
  naming your bean with the store's exception as the cause.

**Root cause:** lifecycle callbacks are synchronous and run on the thread performing
`refresh()`. Unlike Nest's `async onModuleInit`, there is no awaiting. `refresh()` is
also what the readiness state is gated on.

**Fix:**
- Move it to `ApplicationRunner` or `@EventListener(ApplicationReadyEvent.class)`.
- Run it on a dedicated daemon executor so it does not block even there.
- Make the cache read-through so correctness never depends on warm-up completing.
- Load in pages so partial progress survives a failure.
- Decide explicitly what a warm-up failure should do to readiness, and write it down.
- **Bound it.** Whatever you do at startup, give it a timeout. Unbounded startup work
  is unbounded startup time.

**Legitimate uses of `@PostConstruct`** — keep them, they are fine: validating that
injected configuration is coherent, computing a derived immutable value, registering a
listener, building an in-memory lookup from already-injected data. All cheap, all
local, all incapable of hanging.

---

### Trap 3 — assuming `@PreDestroy` always runs

**Wrong:**
```java
@Component
public class OrderEventBuffer {

    private final List<OrderPlaced> pending = new CopyOnWriteArrayList<>();

    public void record(OrderPlaced event) { pending.add(event); }

    @PreDestroy
    void flush() { publisher.publishAll(pending); }   // "we flush on shutdown"
}
```

**Exact symptom:** most of the time it works, which is what makes it dangerous. Then,
after an incident where pods were OOM-killed, your order-event stream is missing
events for the affected pods — and the count of missing events correlates exactly with
the pods that died hard. Nothing in any log says "we dropped these".

**Root cause:** `@PreDestroy` runs when the context is **closed in an orderly way**.
It does not run when:

| Situation | Does `@PreDestroy` run? |
|---|---|
| `SIGTERM` with a JVM shutdown hook registered (Boot's default) | yes |
| `SIGKILL` (`kill -9`) | **no** |
| Kubernetes killing after `terminationGracePeriodSeconds` expires | **no** — that is a `SIGKILL` |
| container OOM kill | **no** |
| `System.exit` inside a shutdown hook | typically **no** for the remainder |
| the JVM crashing | **no** |
| a **prototype**-scoped bean, ever | **no** (Topic 38) |
| the callback itself throwing | that bean's later callbacks do not run |

**Fix — treat shutdown flush as best-effort and design so losing it is survivable:**

1. **Durability first.** If an event must not be lost, write it durably when it is
   produced, not when the process ends. That is the transactional outbox pattern
   (Topic 116), and it is the actual answer.
2. **Enable graceful shutdown** so in-flight requests drain rather than being cut off:
   ```yaml
   server:
     shutdown: graceful
   spring:
     lifecycle:
       timeout-per-shutdown-phase: 20s
   ```
   and set the Kubernetes `terminationGracePeriodSeconds` **higher** than that
   timeout. If the grace period is shorter, Kubernetes `SIGKILL`s you mid-drain and
   the setting achieves nothing.
3. **Use `SmartLifecycle` with phases** when shutdown order matters — stop accepting
   work before you stop the thing that processes it. `@PreDestroy` gives you no
   ordering control across beans; `SmartLifecycle.getPhase()` does.
4. **Bound every `@PreDestroy`.** A callback that waits indefinitely for a queue to
   drain converts a graceful shutdown into a `SIGKILL` once the grace period expires,
   which loses more than doing nothing would have.

---

### Trap 4 — `javax.annotation.PostConstruct` on Boot 3 or 4

**Wrong:**
```java
import javax.annotation.PostConstruct;    // wrong package for Jakarta EE 9+

@Service
public class ProductCache {
    @PostConstruct
    void warm() { cache.putAll(repository.findAllActive()); }
}
```

**Exact symptom:** **nothing happens, and nothing is logged.** No exception, no
warning. The application starts perfectly. The cache is empty. Every catalogue lookup
misses and falls through to the store. Your p99 for `GET /products/{sku}` sits at the
store's latency instead of the cache's, and if the code has no read-through fallback,
lookups return empty and customers see "product not found" for products that exist.

If the old `javax.annotation-api` jar is *not* on the classpath at all, you get a
compile error instead, which is the lucky version. The dangerous case is a
transitively-inherited `javax.annotation-api` that lets it compile and be ignored at
runtime.

**Root cause:** Jakarta EE 9 renamed the `javax.*` namespace to `jakarta.*`. Spring
Boot 3.0 moved to Jakarta EE 9+, and Boot 4 is on Jakarta EE 11.
`CommonAnnotationBeanPostProcessor` looks for `jakarta.annotation.PostConstruct`. An
annotation from a different package is just an unrecognised annotation, and Java does
not warn about those. This is the defining failure of the Boot 2-to-3 migration and
it is Topic 127.

**Fix:**
- Change the import to `jakarta.annotation.PostConstruct`.
- Search the whole codebase: `grep -rn "javax\." src/` and check every hit. The same
  rename hits `javax.persistence` (JPA), `javax.validation` (Bean Validation),
  `javax.servlet` and `javax.transaction`.
- Add an enforcement rule so it cannot come back — a Checkstyle `IllegalImport` on
  `javax.annotation`, `javax.persistence` and `javax.validation`, or an ArchUnit
  `noClasses().should().dependOnClassesThat().resideInAPackage("javax.persistence..")`.
- Remove any transitive `javax.annotation-api` from the dependency tree with a Maven
  `<exclusion>` (Topic 32) so the wrong import cannot compile.

**The general lesson worth more than the fix:** an annotation Spring does not
recognise is silently ignored. There is no "unknown annotation" error in Java. Any
annotation-driven behaviour you rely on should have a test that proves the behaviour,
not the annotation's presence.

---

### Trap 5 — a `BeanPostProcessor` that injects application beans

**Wrong:**
```java
@Component
public class AuditingBeanPostProcessor implements BeanPostProcessor {

    private final AuditLogRepository auditLog;         // an application bean
    private final MeterRegistry meters;                // another one

    public AuditingBeanPostProcessor(AuditLogRepository auditLog, MeterRegistry meters) {
        this.auditLog = auditLog;
        this.meters = meters;
    }

    @Override public Object postProcessAfterInitialization(Object bean, String name) {
        auditLog.record("bean-created", name);
        return bean;
    }
}
```

**Exact symptom:** the application starts, and the log contains warnings of the shape
"Bean 'auditLogRepository' of type [...] is not eligible for getting processed by all
BeanPostProcessors (for example: not eligible for auto-proxying)". People ignore these
warnings for years.

The consequence is not a warning. `AuditLogRepository`, `MeterRegistry`, and
everything they transitively depend on, were created **before the post-processor set
was complete**. So the auto-proxy creator never saw them. Any `@Transactional`,
`@Cacheable`, `@Async`, `@Retryable`, `@Validated` or security annotation on those
beans **silently does nothing**. In `orderflow` terms: if `AuditLogRepository`
transitively pulls in a `@Transactional` service, that service now runs without
transactions, in production, with no error — writes that were supposed to be atomic
are not.

**Root cause:** a `BeanPostProcessor` must exist before ordinary beans are created,
because it processes them. If it has dependencies, those dependencies must be created
first — which necessarily happens before all post-processors are registered. Spring
cannot both honour your dependency and process it, so it honours the dependency and
warns.

**Fix:**

1. **Do not depend on application beans.** Most post-processors need nothing —
   `InitTimingBeanPostProcessor` above uses only `System.nanoTime`.
2. **If you must, take an `ObjectProvider` and resolve lazily**, at use time rather
   than construction time:
   ```java
   private final ObjectProvider<AuditLogRepository> auditLog;

   public AuditingBeanPostProcessor(ObjectProvider<AuditLogRepository> auditLog) {
       this.auditLog = auditLog;                 // nothing resolved yet
   }

   @Override public Object postProcessAfterInitialization(Object bean, String name) {
       auditLog.ifAvailable(repo -> repo.record("bean-created", name));
       return bean;
   }
   ```
   Be careful: resolving it during post-processing of an early bean re-creates the
   problem. Only resolve after startup.
3. **Do the work later.** "Record what beans exist" does not need to happen during
   creation. Collect names into a field during post-processing, then write them out
   from an `ApplicationRunner`.
4. Always declare the post-processor from a **`static` `@Bean` method** so its
   enclosing configuration class is not instantiated early either.

**Take the warnings seriously.** Search your logs for "not eligible for getting
processed by all BeanPostProcessors" today. Every bean named there is a bean whose
annotations may be silently inert.

---

## Hands-on proof

Commands **you** run. I have no JVM and no running `orderflow`; nothing below is
captured output. Each proof gives the file, the command, what to look for, and how to
read every plausible result.

### Proof 1 — establish the order empirically

Add `LifecycleWitness` and `LifecycleDemo` from Example 1 to your project and run the
`main`, or make `LifecycleDemo` a `@Configuration` in the Boot app and run
`./mvnw spring-boot:run`.

**What to look for:** the numeric prefixes, in order.

| What you see | What it means |
|---|---|
| `1, 3a, 3b, 3c, 4, 5, 6, 7, 8, 8b, 9, 10, 11, 12` | the canonical order confirmed on your JDK and Spring version |
| `4 BPP before` appears **before** `5 @PostConstruct` | correct, and consistent with `@PostConstruct` being invoked by a post-processor inside the before-initialization phase |
| `3c setApplicationContext` appears after `3a`/`3b` and near `4` | correct — `ApplicationContextAware` is applied by `ApplicationContextAwareProcessor`, itself a BPP |
| `8b afterSingletonsInstantiated` appears **after** every bean's `8` | correct — it runs once all singletons are done, not per bean |
| `10, 11, 12` do not print | you removed the try-with-resources, or the process was killed. This is Trap 3 demonstrated. |
| an ordering differs from the list | trust your output over this document, and note the Spring version. Then work out which post-processor is responsible. |

### Proof 2 — reveal which beans got proxied

Add `InitTimingBeanPostProcessor` from Example 2, registered `static`.

```bash
./mvnw spring-boot:run
```

**What to look for:** the `runtime class =` column for each `com.orderflow` bean.

| What you see | What it means |
|---|---|
| `com.orderflow.orders.OrderService` — plain name | no proxy. No `@Transactional`, `@Cacheable`, `@Async` or security advice applies to this bean. |
| a name containing `$$SpringCGLIB$$` | a CGLIB subclass proxy was created at step 8. Class-based proxying. |
| a name containing `$Proxy` and a number | a JDK dynamic proxy — interface-based. Topic 40 explains which you get and why. |
| a bean you expected to be proxied is **not** | the annotation is on a `final` class, a `final`/`private`/`static` method, or the feature is not enabled (`@EnableCaching`, `@EnableAsync`, or the transaction auto-configuration). |
| the init timings are all near zero except one | that one is your startup cost. Now you know which bean to move to `ApplicationReadyEvent`. |

This one printed line is the single highest-value diagnostic in Phase 4. Keep the
post-processor behind a profile (`@Profile("diagnostics")`) rather than deleting it.

### Proof 3 — measure the cost of `@PostConstruct` warm-up

Implement Attempt 1 (`@PostConstruct` warming) and Attempt 2 (`ApplicationReadyEvent`
warming) behind two profiles.

```bash
for i in 1 2 3 4 5; do ./mvnw spring-boot:run -Dspring-boot.run.profiles=eagerwarm  ; done
for i in 1 2 3 4 5; do ./mvnw spring-boot:run -Dspring-boot.run.profiles=readywarm  ; done
```

**What to look for:** the value of X in Boot's `Started OrderflowApplication in X
seconds` line, five runs each.

| What you see | What it means |
|---|---|
| `eagerwarm` X is consistently larger | expected — the warm-up is inside `refresh()`, on the critical path |
| the two are the same | your seed data is too small to show it. Increase the stub repository to 100 000 products and re-run. |
| `readywarm` X is small but the cache takes a while to fill | correct — the work moved off the startup path. Print `cache.size()` from a scheduled task to watch it fill. |
| `readywarm` fails to reach `Started` at all | the executor thread is not a daemon, or `ApplicationReadyEvent` is never published because startup failed earlier. Read the log above the failure. |

Then simulate a slow store: add `Thread.sleep(45_000)` to the repository's load
method. Run both profiles.

| What you see | What it means |
|---|---|
| `eagerwarm` never prints `Started`; the process just sits there | **Trap 2 reproduced.** In Kubernetes the readiness probe would fail and the pod would be killed and restarted. |
| `readywarm` prints `Started` in its normal time and warms up 45 seconds later | the fix, demonstrated. The pod is ready and serving, read-through, throughout. |

### Proof 4 — prove `@PreDestroy` is best-effort

With `LifecycleWitness` running inside a Boot app:

```bash
./mvnw spring-boot:run &
APP_PID=$!
# wait until you see "Started OrderflowApplication" in the output, then:
kill -TERM $APP_PID          # graceful
```

Restart and repeat with `kill -KILL $APP_PID`.

| What you see | What it means |
|---|---|
| `SIGTERM`: `10 @PreDestroy`, `11 destroy()`, `12 custom destroyMethod` print | Boot's shutdown hook ran the orderly close. This is the happy path. |
| `SIGKILL`: none of them print | **Trap 3 confirmed.** Anything you flush on shutdown is lost in this scenario. |
| `SIGTERM` and still nothing prints | check you are signalling the JVM and not the Maven wrapper process. Find the real PID with `jps -l`. |
| `SIGTERM` hangs for a long time | some `@PreDestroy` or `SmartLifecycle.stop()` is waiting. That is exactly what turns into a `SIGKILL` when Kubernetes' grace period expires. |

Then set `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase: 20s`,
start a long-running request with `curl`, and `SIGTERM` mid-request.

| What you see | What it means |
|---|---|
| the in-flight `curl` completes, then shutdown proceeds | graceful shutdown working: the connector stops accepting new requests and drains existing ones |
| the `curl` is cut off | graceful shutdown is not enabled, or the phase timeout is shorter than the request |

### Proof 5 — find beans that missed post-processing

```bash
./mvnw spring-boot:run 2>&1 | grep -i "not eligible for getting processed"
```

| What you see | What it means |
|---|---|
| no output | no bean was created too early. Good. |
| one or more bean names | **Trap 5.** Each named bean may have inert `@Transactional`, `@Cacheable`, `@Async` or security annotations. Cross-check each against Proof 2's proxy column. |
| a bean you wrote appears | find what forced it early: a `BeanPostProcessor` or `BeanFactoryPostProcessor` depending on it, or a non-`static` `@Bean` method for one |
| only Spring's own beans appear | usually benign, but note which — if one of them is something you annotated, it is not benign |

For a deeper view, enable Boot's startup tracking:

```java
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(OrderflowApplication.class);
    app.setApplicationStartup(new BufferingApplicationStartup(4096));
    app.run(args);
}
```
with `management.endpoints.web.exposure.include=startup` and then:
```bash
curl -s localhost:8080/actuator/startup | head -c 2000
```

**What to look for:** timed steps such as `spring.beans.instantiate` with a bean name
and a duration.

| What you see | What it means |
|---|---|
| a JSON document with timed events | the buffer captured startup; find the longest `spring.beans.instantiate` entries |
| an empty or 404 response | the endpoint is not exposed, or `setApplicationStartup` was not called before `run` |
| the endpoint returns once and is empty afterwards | expected — the buffer is drained on read |

---

## Practice exercises

### 1 — easy: prove the order, then break it

1. Build `LifecycleWitness` and predict the output order **before** running it. Write
   the prediction down.
2. Run it. List every place your prediction was wrong and explain why.
3. Add a second bean, `SecondWitness`, that depends on `LifecycleWitness`. Predict
   whether `SecondWitness`'s constructor runs before or after `LifecycleWitness`'s
   `@PostConstruct`. Verify.
4. Throw a `RuntimeException` from `@PostConstruct`. Record the exception class that
   reaches the console, which bean it names, and whether the `@PreDestroy` of
   *already-created* beans runs.
5. Change the bean's scope to `prototype`, get it twice from the context, close the
   context, and record which callbacks fired. Explain the result in one sentence.

### 2 — medium: combining earlier topics

Build the `ProductCache` and prove three properties from earlier phases.

1. **Topic 17:** the `warm` flag is written by the startup thread and read by request
   threads. Explain why it is `volatile` and what could be observed without it. Then
   replace `ConcurrentHashMap` with a plain `HashMap` and describe — do not guess, be
   specific — what could go wrong under 3 000 rps.
2. **Topic 12:** the cache is keyed by `String` SKU. Given 100 000 SKUs of the form
   `SKU-100001`, argue whether `String.hashCode` distributes them well across
   `HashMap` buckets. Then check: write the SKUs into a `HashMap`, and count distinct
   `(n-1) & spread(hash)` bucket indices for a table sized to 100 000 entries.
3. **Topic 15:** replace the map with a bounded LRU using
   `LinkedHashMap` + `removeEldestEntry`, capped at 20 000 entries. Then state the two
   reasons you would use Caffeine instead in production.
4. **Topic 26:** `ProductCache.get` returns `Optional<Product>`, and `CatalogService`
   composes it with `.or(...)`. Rewrite the read-through path using `orElseGet` and
   say which reads better and why.
5. **Topics 31–32:** confirm `jakarta.annotation-api` is on your classpath and find
   which dependency brings it:
   `mvn dependency:tree -Dincludes=jakarta.annotation:*`. Then run
   `mvn dependency:tree | grep -i javax` and report whether any `javax.*` API jar is
   present. If one is, identify which dependency drags it in and write the
   `<exclusion>` that removes it.

### 3 — hard: production simulation — warm-up under a degraded dependency

**Part A — build both versions.** Implement the `@PostConstruct` warm-up
(profile `eagerwarm`) and the `ApplicationReadyEvent` warm-up (profile `readywarm`).
Seed the stub repository with 100 000 products.

**Part B — inject latency.** Give the repository a configurable delay,
`orderflow.catalog.load-delay`. For each profile, run at 0 s, 5 s and 45 s and record:

- the `Started OrderflowApplication in X seconds` value,
- whether `GET /products/SKU-1001` returns correctly at 2 seconds after start,
- `cache.size()` at 2 seconds and at 60 seconds after start.

**Part C — the readiness argument.** Add an Actuator readiness probe
(`management.endpoint.health.probes.enabled=true`) and a Kubernetes-style probe
simulation: a script that polls `/actuator/health/readiness` every 2 seconds and
declares failure after 15 consecutive non-`UP` responses. Run it against both profiles
at a 45-second delay. Record which profile "gets killed".

Then write a short recommendation answering:

- Which profile survives a degraded product store, and why exactly?
- Should a warm-up failure make the pod refuse traffic? Argue **both** sides using
  your read-through fallback as evidence, then pick one and state the condition under
  which you would switch.
- What is the correct relationship between
  `spring.lifecycle.timeout-per-shutdown-phase` and Kubernetes
  `terminationGracePeriodSeconds`, and what breaks if you get it backwards?

**Part D — the proxy discovery.** Add `InitTimingBeanPostProcessor`. Then add
`@Transactional` to `CatalogService.findBySku` and re-run.

1. Record `CatalogService`'s runtime class before and after adding the annotation.
2. Now add a `@PostConstruct` to `CatalogService` that calls its own
   `findBySku("SKU-1001")`. Add SQL logging (`spring.jpa.show-sql=true`, or a print in
   the stub repository) and determine whether a transaction was started for that call.
3. Move the same call into an `ApplicationRunner` that takes `CatalogService` as a
   parameter, and determine the same thing again.
4. Write three sentences explaining the difference **in terms of the lifecycle step
   numbers**. This is the answer to the most common Spring interview question you will
   face, and Topic 40 will formalise it.

---

## Interview questions

### Q1 — "Give me the bean lifecycle in order."

**Mid-level answer:** "Spring constructs the bean, injects dependencies, calls
`@PostConstruct`, then the bean is used, and on shutdown it calls `@PreDestroy`."

**Senior answer:** "Instantiate; populate properties, which is where field and setter
injection happen; the `Aware` callbacks — `BeanNameAware`, `BeanClassLoaderAware`,
`BeanFactoryAware`; then every `BeanPostProcessor`'s `postProcessBeforeInitialization`;
then `@PostConstruct`; then `InitializingBean.afterPropertiesSet`; then any custom
`initMethod`; then every `BeanPostProcessor`'s `postProcessAfterInitialization` — and
**that is where AOP proxies are created**, which is the step that actually matters.
Then the bean is in use. On an orderly context close: `@PreDestroy`,
`DisposableBean.destroy`, custom `destroyMethod`. Two refinements: `@PostConstruct` is
itself invoked by `CommonAnnotationBeanPostProcessor` during the before-initialization
phase, and `ApplicationContextAware` and its relatives are applied by
`ApplicationContextAwareProcessor`, also a post-processor. And prototype beans get the
creation half and none of the destruction half, which is why a prototype holding a
resource leaks it."

**What separates them:** naming step 8 as the proxy-creation point and flagging it as
the load-bearing one, rather than reciting a list. The prototype-destruction detail is
a strong extra signal.

**Follow-up:** "So what does that mean for a `@Transactional` method called from
`@PostConstruct`?" This is always the follow-up. Q2.

---

### Q2 — "A `@Transactional` method is called from `@PostConstruct` and nothing rolls back. Why?"

**Mid-level answer:** "Because the proxy is not ready yet during `@PostConstruct`."

**Senior answer:** "Two independent reasons, and it is worth separating them because
they have different fixes. First, ordering: `@Transactional` is applied by an
auto-proxy-creating `BeanPostProcessor` in `postProcessAfterInitialization`, and
`@PostConstruct` runs before that, so from this bean's own point of view the proxy does
not exist yet. Second, self-invocation: even after the proxy exists, calling
`this.loadOpeningStock()` invokes the raw method directly and never touches the proxy,
because the advice lives on the proxy, not on my object. The observable damage is that
the writes still happen and simply are not atomic — with `@Cacheable` the equivalent
symptom is a hit rate stuck at zero with no error. My fix is to move the work out of
`refresh()` entirely, into an `ApplicationRunner` or an `ApplicationReadyEvent`
listener that takes the service as an injected parameter — that parameter *is* the
proxy, so the advice applies, and I also get somewhere to catch and degrade. If it
genuinely must run during initialisation I would use `TransactionTemplate`
programmatically, because that involves no proxy at all."

**What separates them:** separating the two causes, naming the *silent* symptom shape
for both transactions and caching, and choosing a fix that solves the real problem
(startup work in the wrong place) rather than patching the symptom with self-injection.

**Follow-up:** "What if you cannot move it?" They want `TransactionTemplate`, or
extracting a collaborator bean. If the candidate reaches for self-injection first
without acknowledging it is a smell, that is the signal they are missing.

---

### Q3 — "What is a `BeanPostProcessor` and what would you use one for?"

**Mid-level answer:** "It lets you run code before and after a bean's initialisation.
Spring uses it internally for things like `@Autowired`."

**Senior answer:** "It is an interface with two methods, both handed every bean and
its name, and — crucially — both **returning an object**. Returning a different object
replaces the bean in the container, so every injection point receives the replacement.
That is not a hook, it is a substitution mechanism, and it is how most of Spring
works: `@Transactional`, `@Async`, `@Cacheable` and AOP all come from an auto-proxy
creator running in `postProcessAfterInitialization`; `@Autowired` and `@Value` come
from `AutowiredAnnotationBeanPostProcessor`; `@PostConstruct` comes from
`CommonAnnotationBeanPostProcessor`. In my own code I use them rarely and only for
genuinely cross-cutting infrastructure — my most common use is a diagnostic one that
prints each bean's runtime class after initialisation, which immediately tells me which
beans are proxied. The hard rule is that a `BeanPostProcessor` must not depend on
application beans: doing so forces those beans to be created before the post-processor
set is complete, Spring logs 'not eligible for getting processed by all
BeanPostProcessors', and every annotation on those beans becomes silently inert. Also
declare it from a `static` `@Bean` method so its configuration class is not
instantiated early either."

**What separates them:** "returns an object, so it substitutes" is the insight — most
candidates describe it as a callback. Then naming the dependency rule and the exact
warning text, which is knowledge that only comes from having debugged it.

**Follow-up:** "Nest has interceptors. Are they the same thing?" No: Nest attaches
interception at the route boundary, so the injected provider is always your object and
self-calls are unaffected. A `BeanPostProcessor` replaces the object itself, which is
more general and is precisely why the self-invocation trap exists in Spring and not in
Nest.

---

### Q4 — "Where would you put code that must run once at startup?"

**Mid-level answer:** "In a `@PostConstruct` method, or in a `CommandLineRunner`."

**Senior answer:** "It depends on what the code needs, and there are four rungs.
`@PostConstruct` if it only needs this bean's own injected dependencies and is cheap
and cannot fail slowly — validating configuration, computing a derived value.
`SmartInitializingSingleton.afterSingletonsInstantiated` if it needs *all* singletons
to exist but must still be inside `refresh()`. `ApplicationRunner` or
`@EventListener(ApplicationReadyEvent.class)` for anything that touches a network, a
database or a proxy-dependent annotation — those run after `refresh()`, so every proxy
exists and nothing blocks the container's startup. And for anything genuinely slow, I
would additionally push it onto a dedicated daemon executor from the ready listener, so
even that does not delay readiness. The deciding constraint is usually operational
rather than technical: any work inside `refresh()` is on the critical path of every pod
start and every uncached test context, and if it can hang, it can turn a slow
dependency into a readiness-probe crash loop. So my default is the ready event, and I
justify moving anything earlier."

**What separates them:** four options with an explicit selection criterion, and naming
the crash-loop failure mode. "Slow startup" is a mid-level concern; "slow startup
becomes a rollout-blocking crash loop" is the senior one.

**Follow-up:** "Your warm-up fails. Should the pod become unready?" There is no single
right answer — they are testing whether you reason about it. Good answers hinge on
whether there is a read-through fallback: with one, stay ready and alert; without one,
refuse traffic.

---

### Q5 — "Can you rely on `@PreDestroy` to flush a buffer?"

**Mid-level answer:** "Yes, `@PreDestroy` runs when the application shuts down."

**Senior answer:** "Only for an orderly shutdown, so no — not for anything that must
not be lost. It runs from the JVM shutdown hook on `SIGTERM`. It does not run on
`SIGKILL`, on a container OOM kill, on a JVM crash, or when Kubernetes' grace period
expires mid-drain and escalates to `SIGKILL`. It also never runs for prototype-scoped
beans, because Spring does not retain references to them. So I treat it as
best-effort cleanup — closing a client, stopping an executor — and never as a
durability mechanism. If an event must survive, it gets written durably at the moment
it is produced; that is the transactional outbox pattern. Operationally I would also
enable `server.shutdown=graceful` with a
`spring.lifecycle.timeout-per-shutdown-phase`, and make sure Kubernetes'
`terminationGracePeriodSeconds` is *larger* than that timeout — if it is smaller you
get killed mid-drain and the graceful setting achieves nothing. And every `@PreDestroy`
must be bounded, because one that waits forever converts a graceful shutdown into a
hard kill that loses more than doing nothing would have."

**What separates them:** the complete list of cases where it does not run, the
prototype detail, and the grace-period ordering relationship — which is a real
production configuration that is very commonly backwards.

**Follow-up:** "How would you verify graceful shutdown actually works in your
service?" They want an actual experiment: start a long request, send `SIGTERM`,
observe whether the request completes — not "we set the property".

---

## Mental model checkpoint

1. Step 8 can return a different object than the one you wrote. Name three things
   that become possible because of that, and one thing that becomes impossible or
   surprising.

2. `@PostConstruct` runs at step 5 and proxies appear at step 8. Suppose Spring
   reversed those two steps. What would work that does not work today, and what would
   break? Is there a reason the current order is the right one?

3. Nest awaits an `async onModuleInit`; Spring blocks on `@PostConstruct`. That is a
   consequence of a deeper difference between the two runtimes. What is it, and what
   would Spring have to change to offer the Nest behaviour?

4. A `BeanPostProcessor` that depends on an application bean makes that bean miss
   post-processing. Explain why Spring cannot simply fix this by creating
   post-processors last. What is the actual constraint?

5. Prototype beans get creation callbacks but no destruction callbacks. Argue that
   this is the correct design. Then say what you would do about a prototype bean that
   holds a database connection.

6. You have a read-through cache and a warm-up that failed. Make the case for keeping
   the pod ready, then the case for making it unready. Which do you choose for
   `orderflow`'s catalogue, and what specific fact about the catalogue decides it?

7. An annotation Spring does not recognise is silently ignored, with no compile error
   and no warning. Design a practice — not a tool, a practice — that makes your team
   immune to Trap 4's class of failure. Then say what it costs.

---

## Quick reference card

### The order

```
CREATION
 1  instantiate (constructor injection here)
 2  populate properties (field/setter injection here)
 3  BeanNameAware / BeanClassLoaderAware / BeanFactoryAware
 4  BeanPostProcessor.postProcessBeforeInitialization
 5  @PostConstruct                      (invoked by CommonAnnotationBeanPostProcessor)
 6  InitializingBean.afterPropertiesSet
 7  custom @Bean(initMethod = "...")
 8  BeanPostProcessor.postProcessAfterInitialization   <-- AOP PROXIES CREATED HERE
 8b SmartInitializingSingleton.afterSingletonsInstantiated   (once, after ALL singletons)
 9  in use
DESTRUCTION (singletons only, orderly close only)
10  @PreDestroy
11  DisposableBean.destroy()
12  custom @Bean(destroyMethod = "...")
```

### Application events

```
ApplicationStarting -> EnvironmentPrepared -> ContextInitialized -> ApplicationPrepared
  -> [refresh(): all bean lifecycles] -> ContextRefreshed -> ApplicationStarted
  -> ApplicationRunner / CommandLineRunner -> ApplicationReady   (readiness flips here)
  (ApplicationFailed on error)
```

### Which hook to use

| Need | Use |
|---|---|
| validate injected config, compute a derived value | `@PostConstruct` |
| all singletons must exist, still inside `refresh()` | `SmartInitializingSingleton` |
| touches network/DB, or calls a `@Transactional`/`@Cacheable` method | `ApplicationRunner` or `@EventListener(ApplicationReadyEvent.class)` |
| slow, and must not delay readiness | ready event + dedicated daemon executor |
| ordered start/stop across beans | `SmartLifecycle` with phases |
| lifecycle on a class you cannot annotate | `@Bean(initMethod=, destroyMethod=)` |

### Imports and properties

```java
import jakarta.annotation.PostConstruct;   // NOT javax.annotation
import jakarta.annotation.PreDestroy;      // NOT javax.annotation
```
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s     # keep < Kubernetes terminationGracePeriodSeconds
management:
  endpoint:
    health:
      probes:
        enabled: true                   # /actuator/health/readiness and /liveness
```

### Diagnostics

```java
bean.getClass().getName()                             // $$SpringCGLIB$$ or $Proxy => proxied
new BufferingApplicationStartup(4096)                 // + /actuator/startup
```
```bash
./mvnw spring-boot:run 2>&1 | grep -i "not eligible for getting processed"
kill -TERM <pid>   # @PreDestroy runs
kill -KILL <pid>   # @PreDestroy does NOT run
```

### Gotchas checklist

- [ ] `jakarta.annotation`, never `javax.annotation`. Failure is silent.
- [ ] No network or database I/O in `@PostConstruct`.
- [ ] `@Transactional` / `@Cacheable` do not work from `@PostConstruct` or on self-calls.
- [ ] `BeanPostProcessor` must not depend on application beans; declare it `static`.
- [ ] `@PreDestroy` is best-effort. Never a durability mechanism.
- [ ] Prototype beans get no destruction callbacks.
- [ ] Kubernetes grace period **longer** than the Spring shutdown phase timeout.
- [ ] Every startup task and every `@PreDestroy` must be bounded.

---

## When would I use this at work?

**1. Diagnosing "my annotation does nothing".** Someone reports that `@Cacheable`
never caches or `@Transactional` never rolls back. You ask two questions — is it being
called from `@PostConstruct`, and is it a self-call — and you are usually done. Adding
the one-line proxy-class print from Proof 2 settles it in under a minute.

**2. Reviewing a PR that adds startup work.** A `@PostConstruct` that calls a
repository is now something you flag by reflex, and you can explain precisely what it
costs: every pod start, every uncached test context, and a readiness-probe crash loop
the first time that dependency is slow. That is a much stronger review comment than
"prefer a runner".

**3. Getting shutdown right before an incident teaches you.** Setting
`server.shutdown=graceful`, making the Kubernetes grace period longer than the phase
timeout, and moving must-not-lose writes out of `@PreDestroy` into an outbox — these
are ten minutes of work that prevent a category of data-loss incident that is
extremely hard to diagnose afterwards, because nothing logs it.

---

## Connected topics

**Prerequisites:**
- **08 — Exceptions**: a throwing `@PostConstruct` fails bean creation; reading the
  nested `BeanCreationException` cause.
- **15 — LinkedHashMap and LRU**: the `ProductCache` bounded-cache exercise, and why
  Caffeine is the production answer.
- **17 — Immutability and safe publication**: `volatile` on the `warm` flag, and why
  final fields from constructor injection are safely published.
- **35 — `ApplicationContext`**: all bean lifecycles happen inside
  `finishBeanFactoryInitialization`, which is phase 2 of `refresh()`.
- **36 — Bean definition and scanning**: `initMethod` / `destroyMethod` are
  `BeanDefinition` properties, set by `@Bean` attributes.

**This unlocks:**
- **38 — Bean scopes**: why prototype beans get no destruction callbacks, and how
  scoped proxies are created — also at step 8.
- **39 — Injection styles**: why constructor injection is complete at step 1 while
  field injection completes at step 2, and what that means for a `@PostConstruct` that
  reads an injected field.
- **40 — Proxying**: step 8 in full — JDK versus CGLIB, what cannot be advised, and
  the self-invocation trap this topic previewed.
- **41 — AOP**: the interceptor chain that the step-8 proxy walks.
- **42 — Auto-configuration**: auto-configured beans go through exactly this
  lifecycle; that is why `@ConditionalOnMissingBean` back-off is a definition-time
  decision and proxying is an instantiation-time one.
- **54 — `@Transactional`**: the payoff. Trap 1 is the single most common Spring bug.
- **60 — Test slices and context caching**: every uncached context refresh re-runs
  every `@PostConstruct` in the application. This is often the entire explanation for
  a slow suite.
- **116 — Transactional outbox**: the real answer to "flush on shutdown".
- **127 — `javax` to `jakarta` migration**: Trap 4 in full.

---

## `[BOOT 3.x DELTA]`

**The lifecycle itself is unchanged** between Framework 6.x/Boot 3.x and Framework
7.0/Boot 4.x. The step order, `BeanPostProcessor` semantics, proxy creation at
`postProcessAfterInitialization`, `SmartInitializingSingleton`, `SmartLifecycle`
phases and graceful shutdown all behave identically. Stated explicitly rather than
inventing a difference.

The real deltas that touch this topic:

| Area | Boot 2.x | Boot 3.x and 4.x |
|---|---|---|
| `@PostConstruct` / `@PreDestroy` package | `javax.annotation` | **`jakarta.annotation`** |
| the annotation-api jar | `javax.annotation-api` | `jakarta.annotation-api` |
| behaviour if you use the wrong package | worked on 2.x | **silently ignored** — Trap 4 |

Other version notes:

- **Graceful shutdown** (`server.shutdown=graceful`) has existed since Boot 2.3 and
  behaves the same on 3.x and 4.x. It is **not** the default — you must set it.
- **Readiness and liveness probes** (`management.endpoint.health.probes.enabled`) have
  existed since Boot 2.3, unchanged on 3.x and 4.x. On Kubernetes, Boot enables the
  probes automatically when it detects the environment; setting the property
  explicitly is still the safer habit.
- **`BufferingApplicationStartup` and `/actuator/startup`** have existed since Boot
  2.4 and are unchanged.

Nothing in this topic requires Boot 4 specifically. If you are working on a 3.x
codebase, everything here applies as written — with the `jakarta` rule already in
force, since that landed in 3.0.

---

*Java baseline 21, running on JDK 25. Spring Framework 7.0 / Spring Boot 4.1,
Jakarta EE 11 — `jakarta.*` imports throughout, never `javax.*`. No Spring artifact
versions are quoted: the Boot BOM supplies them, and the authoritative current version
is whatever a freshly generated start.spring.io project puts in its parent POM.*
