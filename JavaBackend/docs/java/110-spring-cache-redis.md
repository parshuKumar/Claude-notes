# 110 — Spring Cache and Redis: Key Design, Stampede, Invalidation

## Phase: 11 — Distributed Systems & Production
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: cache the `orderflow` product catalogue — the 70% of Topic 65's load mix. Measure hit rate and the p99 change against the recorded baseline.

---

## ELI5 anchor

You work at a returns desk. Every time someone asks "what is the price of SKU-4471?",
you walk to the warehouse, find the shelf, read the label, walk back, and tell them.

So you get a notepad. The first person who asks costs you the walk. You write the
answer on the notepad. The next forty people get the answer instantly.

Three things go wrong with the notepad, and they are the whole topic:

**1. Someone changes the price and you do not rub out the note.** Now you confidently
tell forty people the wrong price. Worse: you sound certain.

**2. Your notes are written in pencil that fades after ten minutes.** At the exact
moment the note for SKU-4471 fades, forty people are standing at the desk asking about
SKU-4471. All forty send you to the warehouse. You make forty identical walks, at once,
for one answer. That is the **stampede**, and it is worse than having no notepad — with
no notepad you would have paced yourself.

**3. The notepad has no last page.** You keep writing. Every SKU anyone has ever asked
about, including the ones typed in wrong. Eventually the notepad fills the room.

Everything below is those three problems, in Java, with the specific mechanisms Spring
and Redis give you for each.

---

## The bridge from what you know

You know caching. You have set a TTL. You have argued about invalidation. You do not
need the concept explained.

Three Java-specific facts change how you write it, and only these three.

### Fact 1 — `@Cacheable` is a proxy. Everything from Topic 40 applies.

```ts
// Node — the shape you are used to. The wrapping is visible.
async function findProduct(sku: string): Promise<Product | null> {
  const cached = await redis.get(`product:${sku}`);
  if (cached) return JSON.parse(cached);

  const product = await db.query('select * from product where sku = $1', [sku]);
  await redis.setEx(`product:${sku}`, 300, JSON.stringify(product));
  return product;
}
```

Six lines. Every step is on screen: the lookup, the miss, the load, the write, the TTL,
the serialisation format. If you call `findProduct` from anywhere, including from
inside the same module, you get the caching, because the caching **is the function**.

```java
@Cacheable(cacheNames = "products", key = "#sku")
public Product findBySku(String sku) {
    return repository.findBySku(sku).orElse(null);
}
```

One line. Nothing on screen. And critically: the caching is **not** the method. It is a
`CacheInterceptor` in an advice chain, on a proxy object, that wraps your bean. A call
from `this` inside the same class goes straight to your method and skips the cache
entirely, silently, with no warning.

**This is not a new lesson — it is Topic 40's lesson, reappearing.** The reason it
belongs here is that a cache that silently does nothing looks *exactly* like a cache
that is working, right up until you look at the hit rate. A broken `@Transactional`
loses data loudly. A broken `@Cacheable` just... does nothing, forever.

### Fact 2 — the cache write can happen inside a transaction that later rolls back.

```java
@Transactional
public Product updatePrice(String sku, long newPriceMinor) {
    Product p = repository.findBySku(sku).orElseThrow();
    p.setPriceMinor(newPriceMinor);
    cachePut(p);                      // writes to Redis NOW
    auditService.record(sku);         // throws
    return p;                         // never reached; transaction rolls back
}
```

Postgres rolls back. Redis does not. Redis has no idea a transaction existed. The
database now holds the old price and the cache holds the new one, permanently, until
the TTL expires.

In Node you would have had the same problem — but you would have *seen* it, because
`await redis.setEx(...)` is a line in your function and the `BEGIN`/`COMMIT` are lines
too. In Spring both are annotations, invisible, and their relative ordering is
determined by advisor precedence you never configured.

Topic 41's drill was exactly this. This is where it costs money.

### Fact 3 — the default Redis value serializer is Java serialization.

`RedisCacheConfiguration.defaultCacheConfig()` serialises values with
`JdkSerializationRedisSerializer`. That means:

- Your cached values are opaque binary. `redis-cli GET products::SKU-4471` returns
  bytes you cannot read during an incident.
- Every cached class must implement `Serializable`, and adding a field changes the
  `serialVersionUID` unless you pinned it — so a rolling deploy has two versions of the
  service reading each other's cache entries and throwing.
- It is Java serialization, with all of Topic 19 attached. Anything that can write to
  your Redis can, in principle, put a gadget chain in a cache value that your service
  will deserialize.

You must change this. It is the first thing you configure and the thing most teams
never touch.

### What transfers cleanly

| Concept | Transfers? |
|---|---|
| TTL, eviction policy, hit rate | **Yes, exactly.** No re-learning. |
| Stampede / dogpile | **Yes as a concept**, but the Java mitigations are specific and one of them (`sync = true`) has a limitation you must know. |
| Cache-aside vs write-through | **Yes.** `@Cacheable` is cache-aside; `@CachePut` is write-through. |
| Redis data structures, `SCAN` over `KEYS`, `maxmemory-policy` | **Yes.** Identical. Your Redis knowledge is fully portable. |
| Where the caching code lives | **No.** It is on a proxy, not in the method. |
| Whether a cache write participates in a transaction | **No.** By default it does not, and the fix is a specific wrapper class. |
| The serialisation format | **No.** Java has a bad default you must override. |

---

## What is this?

**Spring Cache** is an abstraction. `@Cacheable`, `@CachePut` and `@CacheEvict` are
annotations that a `CacheInterceptor` reads at runtime and turns into calls against a
`Cache` interface. The interface has three relevant methods: `get`, `put`, `evict`.

**`CacheManager`** is the thing that produces `Cache` instances by name. Swapping
`ConcurrentMapCacheManager` (in-memory, for tests) for `RedisCacheManager`
(distributed, for production) changes no annotation on any method. That is the entire
value proposition of the abstraction, and it is a real one.

**Redis** you already know. What matters here is how Spring maps a cache name and a key
onto a Redis key, what bytes it writes, and what it does when you ask it to clear a
cache.

The three annotations, precisely:

| Annotation | Runs your method? | Reads cache? | Writes cache? |
|---|---|---|---|
| `@Cacheable` | Only on a miss | Yes | Yes, on a miss |
| `@CachePut` | **Always** | No | **Always** |
| `@CacheEvict` | Always | No | Removes, does not write |

The mistake people make is using `@Cacheable` where they meant `@CachePut` on an
update method — which then returns the *stale cached value* instead of running the
update at all.

---

## Why does it matter?

**1. It is the highest-leverage latency change available to `orderflow`.**
Seventy per cent of Topic 65's load mix is catalogue reads over 100k products with
realistic skew — a few hot products taking a disproportionate share. That is the
textbook cache-friendly workload. The p99 improvement is the largest single number you
will move in Phase 11.

**2. Every failure mode is silent.**
A cache that never hits looks like a working service, slightly slow. A cache serving
stale prices looks like a working service, until finance asks why orders were placed at
last month's price. A cache growing without bound looks fine until Redis evicts your
session data to make room, or `OOM command not allowed when used memory > 'maxmemory'`
appears and every write fails at once.

**3. The stampede turns a cache into an amplifier.**
At the moment a hot key expires, every concurrent request for it misses. Each one
issues the same query. Your database receives, in one instant, a burst equal to your
full concurrency for a single row. Under the Topic 65 baseline that burst lands on the
pool you just sized in Topic 109 — and a pool of 16 receiving 200 simultaneous
checkouts produces exactly the exhaustion you learned to diagnose there.

**4. Redis is now a dependency you can be down without.**
Or can you? If a Redis outage means every catalogue read goes to Postgres, does
Postgres survive it? That is a capacity question you have to answer *before* the
outage, and most teams discover the answer during one.

---

## Syntax breakdown

### Turning it on

```java
package com.orderflow;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class CacheConfig { }
```

| Bit of syntax | What it means |
|---|---|
| `@EnableCaching` | Registers `CacheInterceptor` and the advisor that creates caching proxies. Without it every cache annotation in your codebase is inert metadata — no error, no warning, no caching. |
| `@Configuration` | Ordinary Spring configuration class (Topic 35). Nothing special. |

`@EnableCaching` also accepts `order` and `mode`, which matter for Trap 3 below.

### `@Cacheable` — read-through

```java
@Cacheable(
    cacheNames = "products",       // which cache. REQUIRED. Also called value=.
    key = "#sku",                  // SpEL over the method parameters
    unless = "#result == null",    // evaluated AFTER the call; skip the put if true
    condition = "#sku != null",    // evaluated BEFORE the call; skip cache entirely
    sync = true                    // dedupe concurrent misses (see the limitation below)
)
public Product findBySku(String sku) {
    return repository.findBySku(sku).orElse(null);
}
```

| Attribute | When it is evaluated | What it does | The trap |
|---|---|---|---|
| `cacheNames` | startup | Names the cache region. | Two methods sharing a name and a key type collide. See Trap 1. |
| `key` | before the call | SpEL producing the key object. | Omit it and you get the default key generator, which is where Trap 1 lives. |
| `condition` | **before** the call | If false, no cache read and no cache write. `#result` is **not available**. | Using `#result` here silently never matches. |
| `unless` | **after** the call | If true, skip the write. The read already happened. `#result` **is** available. | The negation reads backwards. `unless = "#result == null"` means *do not cache nulls*. |
| `sync` | — | Dedupe concurrent loads for the same key. | **Per JVM only.** See below. |

SpEL expressions you will actually use:

```java
key = "#sku"                                  // a named parameter
key = "#p0"                                   // positional, if -parameters is off
key = "#product.sku"                          // a property of a parameter
key = "'catalogue:' + #categoryId + ':' + #page"   // a composed string
key = "#root.methodName + ':' + #sku"         // include the method name (see Trap 1)
condition = "#page < 5"                       // only cache the first few pages
unless = "#result == null or #result.discontinued"
```

> `#sku` resolves by parameter name only if the class was compiled with `-parameters`.
> Spring Boot's Maven and Gradle plugins set this for you. If you see
> `Property or field 'sku' cannot be found`, that flag is missing — check your build,
> do not switch to `#p0` and move on.

### `@CachePut` — write-through

```java
@CachePut(cacheNames = "products", key = "#result.sku")
public Product updatePrice(String sku, long newPriceMinor) {
    Product p = repository.findBySku(sku).orElseThrow();
    p.setPriceMinor(newPriceMinor);
    return repository.save(p);
}
```

Always runs the method. Always writes the returned value into the cache. Use it when
the method's return value *is* the new state. Note `key = "#result.sku"` — `@CachePut`
can use `#result` in the key because the method has already run.

### `@CacheEvict` — invalidate

```java
@CacheEvict(cacheNames = "products", key = "#sku")
public void discontinue(String sku) { ... }

@CacheEvict(cacheNames = "products", allEntries = true)
public void reimportCatalogue() { ... }

@CacheEvict(cacheNames = "products", key = "#sku", beforeInvocation = true)
public void deleteWithRiskyCleanup(String sku) { ... }
```

| Attribute | What it does | When you want it |
|---|---|---|
| `key` | Removes one entry. | The normal case. |
| `allEntries = true` | Clears the whole cache region. | Bulk reimport. **On Redis this is a `SCAN` + `DEL` sweep — see Trap 5.** |
| `beforeInvocation = true` | Evicts before the method runs, so a thrown exception still evicts. | When a *stale* entry is worse than an extra miss. Default is `false`: an exception leaves the stale entry in place. |

### `@Caching` — combining

```java
@Caching(evict = {
    @CacheEvict(cacheNames = "products",        key = "#sku"),
    @CacheEvict(cacheNames = "productsByBrand", key = "#result.brandId")
})
public Product discontinue(String sku) { ... }
```

One write invalidating two derived caches. This is where cache design starts to hurt,
and it is the honest argument for caching fewer, coarser things.

### `CacheManager` and Redis configuration

This is the block that matters most, because the defaults are wrong.

```java
package com.orderflow.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.boot.autoconfigure.cache.RedisCacheManagerBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext.SerializationPair;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

@Configuration
public class RedisCacheConfig {

    @Bean
    RedisCacheConfiguration defaultCacheConfiguration(ObjectMapper objectMapper) {
        return RedisCacheConfiguration.defaultCacheConfig()
                // 1. Keys as readable UTF-8 strings, so redis-cli is usable in an incident.
                .serializeKeysWith(SerializationPair.fromSerializer(
                        new StringRedisSerializer()))
                // 2. Values as JSON, NOT Java serialization. See Trap 2.
                .serializeValuesWith(SerializationPair.fromSerializer(
                        new GenericJackson2JsonRedisSerializer(objectMapper)))
                // 3. A default TTL. Never leave this unset. See Trap 4.
                .entryTtl(Duration.ofMinutes(10))
                // 4. Do not cache nulls by default; decide per cache. See Trap 6.
                .disableCachingNullValues()
                // 5. Explicit key prefix so you can SCAN for exactly this app's keys.
                .prefixCacheNameWith("orderflow:");
    }

    /** Per-cache overrides. A hot catalogue and a volatile price list want different TTLs. */
    @Bean
    RedisCacheManagerBuilderCustomizer perCacheTtl(RedisCacheConfiguration base) {
        return builder -> builder
                .withCacheConfiguration("products",
                        base.entryTtl(Duration.ofMinutes(30)))
                .withCacheConfiguration("productPrices",
                        base.entryTtl(Duration.ofSeconds(60)))
                .withCacheConfiguration("inventoryLevels",
                        base.entryTtl(Duration.ofSeconds(5)));
    }
}
```

Line by line, what each call buys you:

| Call | Default without it | Why you change it |
|---|---|---|
| `serializeKeysWith(StringRedisSerializer)` | Already `StringRedisSerializer` for keys | Set it explicitly anyway, so nobody has to check. |
| `serializeValuesWith(...Json...)` | **`JdkSerializationRedisSerializer`** | Java serialization: opaque, `Serializable`-only, version-fragile, Topic 19's attack surface. |
| `entryTtl(...)` | **No TTL at all — entries live forever** | Unbounded growth. Trap 4. |
| `disableCachingNullValues()` | Nulls **are** cached | Protects against cache penetration but grows on garbage keys. Decide deliberately. Trap 6. |
| `prefixCacheNameWith(...)` | Prefix is `cacheName::` | Add an app prefix when Redis is shared, so your `SCAN` patterns and your `allEntries` sweeps cannot touch another service's keys. |

> **Uncertainty, flagged:** `GenericJackson2JsonRedisSerializer` is the Jackson 2 class
> name. Spring Boot 4 makes **Jackson 3 standard** and deprecates Jackson 2 support, so
> the class you want on Boot 4.1 may be the Jackson 3 equivalent under a different name
> and package. I am not going to guess the identifier. Check the Spring Data Redis
> reference for the version resolved by your Boot BOM (`mvn dependency:tree | grep
> spring-data-redis`), or let your IDE complete
> `org.springframework.data.redis.serializer.` and read the list. The *concept* — do
> not use JDK serialization, use JSON — is version-independent and is the part that
> matters.

Boot properties for the Redis connection itself:

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 500ms                 # command timeout — keep it well under your SLO
      connect-timeout: 500ms
      lettuce:
        pool:
          enabled: true
          max-active: 16
          max-idle: 16
          min-idle: 4
  cache:
    type: redis
    # Declaring names up front creates the caches at startup so Micrometer can bind
    # metrics to them. Without this, lazily-created caches may not be instrumented.
    cache-names: products,productPrices,inventoryLevels
    redis:
      time-to-live: 10m
      cache-null-values: false
      key-prefix: "orderflow:"
      use-key-prefix: true
```

---

## Example 1 — minimal

The smallest complete thing, with the proof built in.

```java
package com.orderflow.catalog;

import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class ProductCatalogService {

    private final ProductRepository repository;

    public ProductCatalogService(ProductRepository repository) {
        this.repository = repository;
    }

    @Cacheable(cacheNames = "products", key = "#sku", unless = "#result == null")
    public Product findBySku(String sku) {
        System.out.println("DATABASE HIT for " + sku);      // temporary; see below
        return repository.findBySku(sku).orElse(null);
    }

    @CachePut(cacheNames = "products", key = "#result.sku")
    public Product updatePrice(String sku, long newPriceMinor) {
        Product p = repository.findBySku(sku).orElseThrow();
        p.setPriceMinor(newPriceMinor);
        return repository.save(p);
    }

    @CacheEvict(cacheNames = "products", key = "#sku")
    public void discontinue(String sku) {
        repository.markDiscontinued(sku);
    }
}
```

That `System.out.println` is deliberate and temporary. It is the crudest possible
cache-hit instrument and it is the right first move: call `findBySku("SKU-4471")` three
times and you should see the line **once**. If you see it three times, the cache is not
working and everything downstream of this point is a waste of your afternoon.

Replace it with the Micrometer metrics from Hands-on proof once you have confirmed the
basics. Do not ship the `println`.

### The version that is silently broken

```java
@Service
public class ProductCatalogService {

    @Cacheable(cacheNames = "products", key = "#sku")
    public Product findBySku(String sku) { ... }

    public List<Product> findMany(List<String> skus) {
        return skus.stream()
                   .map(this::findBySku)      // <-- SELF-INVOCATION. No cache. Ever.
                   .toList();
    }
}
```

`this::findBySku` is a method reference bound to `this` — the target object, not the
proxy. Every call goes straight to your method. The cache is never read and never
written by this path.

There is no exception. There is no log line. The hit rate for `products` will simply
sit at whatever the other callers produce, and `findMany` will hammer the database
forever. **This is Topic 40, and it is the single most common way a Spring cache does
nothing.**

The fix is the same as Topic 40's: extract the cached method to a separate bean, or
inject an `ObjectProvider` of yourself, or — best here — write a batch method that does
one query rather than N.

---

## Example 2 — production scenario (on the project spine)

### The constraints, stated up front

From the Topic 65 baseline:

- 100k products; the load profile has realistic skew, so a small set of SKUs takes a
  large share of catalogue traffic.
- Load mix: **70% catalogue read**, 20% order read, 10% order placement.
- Three `orderflow` replicas, `--cpus=2 --memory=2g`.
- HikariCP pool of 16 per replica (Topic 109's measured value).
- PostgreSQL 17 on 8 cores.
- Redis 7, single node, in the same `docker compose` stack.
- SLO under discussion: `GET /products/{sku}` p99 under 80 ms; `POST /orders` p99 under
  400 ms.
- Prices change through an admin path, roughly a few hundred edits a day, and must be
  visible to customers within one minute.

### The design decisions, and the reasoning for each

**Decision 1 — cache the product, not the response.**
Cache the domain object keyed by SKU, not the rendered JSON keyed by URL. The same
product appears in the detail endpoint, the search results, and the order-line
enrichment. One cache entry serves all three. Caching the response would triple the
memory and triple the invalidation work.

**Decision 2 — TTL of 30 minutes plus explicit eviction on write.**
The TTL is a safety net for changes that happen outside the service (a data migration,
a manual `UPDATE`, another service writing to the same table). Explicit eviction is
what meets the one-minute freshness requirement. **Neither alone is sufficient**: the
TTL alone is 30 times too slow, and eviction alone leaves stale entries forever when a
write path is missed.

This is the same lesson as Topic 51's second-level cache: a cache is only correct if
every writer goes through the invalidation path, and the TTL is your insurance against
the writer you did not know about.

**Decision 3 — `sync = true` on the hot read, with its limitation understood.**
Covered in the failure drill below.

**Decision 4 — a bounded Redis with an explicit eviction policy.**
100k products at a few hundred bytes each is small. But the cache key space is not
bounded by the product count — it is bounded by what clients ask for, including
nonsense. `maxmemory` plus `allkeys-lru` turns "Redis fills up and writes start
failing" into "Redis evicts the coldest entries", which is a degradation rather than an
outage.

### The code

```java
package com.orderflow.catalog;

import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class ProductCatalogService {

    private final ProductRepository repository;

    public ProductCatalogService(ProductRepository repository) {
        this.repository = repository;
    }

    /**
     * The hot path. 70% of Topic 65's load mix arrives here.
     *
     * sync = true dedupes concurrent misses for the same key WITHIN THIS JVM.
     * With three replicas, a hot-key expiry produces at most 3 database queries
     * instead of one per concurrent request. That is a 3x floor, not a 1x floor,
     * and you must say so out loud when you propose this.
     *
     * Returns a projection, not an entity: nothing here should drag a persistence
     * context or a lazy proxy into a serializer (Topics 47 and 49). It also keeps
     * the cached payload small and stable across schema changes.
     */
    @Cacheable(cacheNames = "products", key = "#sku", sync = true)
    @Transactional(readOnly = true)
    public ProductView findBySku(String sku) {
        return repository.findViewBySku(sku).orElse(null);
    }

    /**
     * Batch path. ONE query, then one cache write per product.
     *
     * Deliberately NOT implemented as a loop over findBySku(): that would be
     * self-invocation and would bypass the cache entirely (Topic 40), and it
     * would also be an N+1 (Topic 50). Two bugs, one line.
     */
    @Transactional(readOnly = true)
    public List<ProductView> findAllBySkus(List<String> skus) {
        return repository.findViewsBySkuIn(skus);
    }

    /**
     * The write path. Eviction, not @CachePut.
     *
     * @CachePut would write the post-update object into the cache immediately —
     * which means writing a value that the transaction has not committed yet.
     * If the transaction rolls back, Redis keeps the uncommitted value forever.
     * Evicting is safe under rollback: the worst case is one extra database read.
     *
     * beforeInvocation = false (the default) is deliberate: we evict AFTER the
     * method returns normally. See the transaction-ordering note below, which is
     * the part that actually makes this correct.
     */
    @CacheEvict(cacheNames = "products", key = "#sku")
    @Transactional
    public void updatePrice(String sku, long newPriceMinor) {
        Product p = repository.findBySku(sku).orElseThrow(
                () -> new ProductNotFoundException(sku));
        p.setPriceMinor(newPriceMinor);
        // No save() call: dirty checking writes it at flush (Topic 48).
    }
}
```

### The ordering problem, and the one-line fix

Read `updatePrice` again. Two pieces of advice apply: the cache eviction and the
transaction. Which runs outermost?

If the **cache advice is outside** the transaction advice: evict happens after commit.
Correct.

If the **cache advice is inside** the transaction advice: evict happens before commit.
A concurrent reader can miss, load the *old* committed row from Postgres, and write it
back into the cache — after your eviction and before your commit. The cache is now
permanently stale with the old price, and no further eviction is coming.

That is a genuine race, it is rare, and it is exactly the kind of bug that gets
attributed to "Redis being flaky".

Both `@EnableTransactionManagement` and `@EnableCaching` default to
`Ordered.LOWEST_PRECEDENCE`, so the relative order is not something you should be
relying on. Do not fix this by tuning `order` attributes. Fix it by making the cache
transaction-aware:

```java
package com.orderflow.config;

import org.springframework.cache.CacheManager;
import org.springframework.cache.transaction.TransactionAwareCacheManagerProxy;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.redis.cache.RedisCacheManager;

@Configuration
public class TransactionAwareCacheConfig {

    /**
     * Defers every cache PUT and EVICT until after the surrounding transaction
     * commits. If the transaction rolls back, the cache operation never happens.
     *
     * Cost: reads are unaffected, but a write's cache effect is invisible to the
     * writing transaction itself. A method that evicts and then re-reads inside
     * the same transaction sees the OLD cached value. That is usually what you
     * want and occasionally a surprise — say it in the code review.
     */
    @Bean
    @Primary
    CacheManager transactionAwareCacheManager(RedisCacheManager delegate) {
        return new TransactionAwareCacheManagerProxy(delegate);
    }
}
```

This is the correct answer to Topic 41's drill, and it is worth more than any amount of
advisor-order tuning because it is declarative and it cannot be got wrong by the next
person.

### The freshness gap you have not closed

Eviction happens on the replica that handled the write. Redis is shared, so the entry
is gone for all three replicas. Good.

But if you later add an in-JVM L1 cache (Caffeine) in front of Redis — which is the
obvious next optimisation — that L1 lives in one JVM and the other two replicas keep
serving their own stale copies until their own TTL expires. Two-level caching requires
either a very short L1 TTL or a Redis pub/sub invalidation channel. Do not add the L1
without deciding which, and write the decision down.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the default key generator collides

**Wrong:**
```java
@Cacheable("catalogue")
public Product findBySku(String sku) { ... }

@Cacheable("catalogue")
public Brand findBrandByCode(String code) { ... }
```

Neither declares a `key`. Spring falls back to `SimpleKeyGenerator`, whose rule is:

- no parameters → `SimpleKey.EMPTY`
- exactly one parameter → **the parameter itself is the key**
- more than one → a `SimpleKey` wrapping all of them

So `findBySku("X-100")` and `findBrandByCode("X-100")` produce the **same Redis key**:
`catalogue::X-100`.

**Exact symptom:** a `ClassCastException` or a Jackson deserialisation error deep in
the cache interceptor, on a line that does not mention casting, intermittently, only
when the two methods happen to be called with the same string. Or — worse, if the types
are structurally similar enough for Jackson to accept — the wrong object is returned
with no error at all.

**Root cause:** the cache name is the namespace, and two unrelated methods were put in
the same namespace with the same key.

**Fix — pick one and apply it consistently across the codebase:**

```java
// Option A (preferred): one cache name per logical entity.
@Cacheable(cacheNames = "products", key = "#sku")
public Product findBySku(String sku) { ... }

@Cacheable(cacheNames = "brands", key = "#code")
public Brand findBrandByCode(String code) { ... }

// Option B: include the method name in the key. Verbose, but survives careless reuse.
@Cacheable(cacheNames = "catalogue", key = "#root.methodName + ':' + #sku")
public Product findBySku(String sku) { ... }
```

Never rely on the default generator for anything you will keep. Write the `key`
expression every time; it costs nine characters and removes a whole class of bug.

---

### Trap 2 — the default value serializer is Java serialization

**Wrong:** accepting `RedisCacheConfiguration.defaultCacheConfig()` unchanged.

**Exact symptom — four of them, and they arrive in this order:**

1. `java.io.NotSerializableException: com.orderflow.catalog.ProductView` the first time
   you cache a class that does not implement `Serializable`. Loud and easy. This one is
   a gift.
2. `redis-cli GET orderflow:products::SKU-4471` returns unreadable binary during an
   incident, so you cannot check whether a cached value is stale without writing a Java
   program.
3. During a rolling deploy that added a field: `java.io.InvalidClassException: local
   class incompatible: stream classdesc serialVersionUID = <n>, local class
   serialVersionUID = <n>`. Half your pods throw on every cache hit. The other half are
   fine. This looks like a Redis problem and is not.
4. A security review flags that your service deserialises Java objects from a Redis
   instance that several services can write to. They are right (Topic 19).

**Root cause:** `JdkSerializationRedisSerializer` is the default value serializer, and
Java serialization couples the wire format to the class definition.

**Fix:** JSON, configured explicitly, as in the `RedisCacheConfig` block above. Then
verify:

```bash
redis-cli --scan --pattern 'orderflow:products::*' | head -5
redis-cli GET "$(redis-cli --scan --pattern 'orderflow:products::*' | head -1)"
```

| What you see | What it means |
|---|---|
| Readable JSON | Correct. You can now diagnose staleness with one command. |
| Binary with `\xac\xed` near the start | Still Java serialization — `AC ED` is the Java serialization stream magic. Your configuration is not being applied. Check for a competing `CacheManager` bean. |
| `(nil)` | Nothing cached under that pattern. Check your `key-prefix` and cache name. |

A note on the JSON serializer: `GenericJackson2JsonRedisSerializer` embeds a type
field (`@class`) so it can reconstruct the exact class. That makes the payload larger
and couples the cache to your class *names* — renaming or moving a cached class
invalidates every entry, which fails on read. Either accept that and clear the cache on
such a deploy, or serialise a stable DTO whose fully-qualified name you commit to
never changing. Caching a purpose-built `ProductView` rather than the `Product` entity
is that decision, made early.

---

### Trap 3 — caching a value from a transaction that rolls back

**Wrong:**
```java
@Transactional
@CachePut(cacheNames = "products", key = "#result.sku")
public Product applyPromotion(String sku, int percentOff) {
    Product p = repository.findBySku(sku).orElseThrow();
    p.applyDiscount(percentOff);
    promotionAuditor.record(sku, percentOff);   // throws on the 1001st call today
    return p;
}
```

**Exact symptom:**

- The API returns a 500. The customer sees an error. Nothing was saved.
- The discounted price appears on the site anyway, for everyone, for the full TTL.
- The database, queried directly, shows the original price.
- Restarting the service does not fix it — the value is in Redis, not in the JVM.
- `redis-cli GET orderflow:products::SKU-4471` shows the discounted price while
  `select price_minor from product where sku='SKU-4471'` shows the original. Those two
  commands, side by side, are the whole diagnosis.

**Root cause:** the cache write executed before the transaction committed, and Redis has
no rollback. Two systems, one of which has transactions.

**Fix — three things, in order of value:**

1. **`TransactionAwareCacheManagerProxy`** (the block in Example 2). Defers puts and
   evicts to after commit. This is the general fix and it covers the annotations you
   have not written yet.
2. **Prefer `@CacheEvict` over `@CachePut` on write paths.** Under rollback, an
   unnecessary eviction costs one database read. An incorrect put costs a wrong price
   for the TTL. Asymmetric consequences, so pick the cheap failure.
3. **Keep the transaction narrow** so there is less inside it that can throw.

This is Topic 41's drill with money attached. Run that drill again if it has faded.

---

### Trap 4 — no TTL, and unbounded key growth

**Wrong:**
```java
// No entryTtl() anywhere in the configuration.
@Cacheable(cacheNames = "productSearch", key = "#query + ':' + #page")
public List<ProductView> search(String query, int page) { ... }
```

**Exact symptom:**

- `redis-cli DBSIZE` grows monotonically and never falls. Over days, then weeks.
- `redis-cli INFO memory` shows `used_memory` approaching `maxmemory`.
- If `maxmemory-policy` is `noeviction` (the Redis default): every write starts failing
  with `OOM command not allowed when used memory > 'maxmemory'`. Because your session
  store or rate limiter probably shares this Redis, **those** fail too — a cache
  configuration mistake takes down authentication.
- If `maxmemory-policy` is `allkeys-lru`: Redis silently evicts, including things you
  did not want evicted, and your hit rate quietly falls with no error anywhere.
- `redis-cli --bigkeys` shows the `productSearch` prefix dominating.

**Root cause:** two compounding mistakes. No TTL, so nothing expires. And a key derived
from **user input** (`#query`), so the key space is unbounded — every typo, every bot
scraping with random strings, is a permanent entry.

**Fix:**

```java
// 1. TTL on every cache, no exceptions. Shorter for a large, low-value key space.
.withCacheConfiguration("productSearch", base.entryTtl(Duration.ofMinutes(2)))

// 2. Do not cache unbounded key spaces at all. Cache only what has bounded cardinality.
@Cacheable(cacheNames = "productSearch",
           key = "#query.toLowerCase() + ':' + #page",
           condition = "#page < 3 and #query.length() <= 40")
public List<ProductView> search(String query, int page) { ... }
```

```conf
# 3. Bound Redis itself. This is not optional for a cache.
maxmemory 512mb
maxmemory-policy allkeys-lru
```

The `condition` is the interesting part: caching only the first three pages of short
queries. Deep pagination is rare and low-value, so caching it costs memory and buys
nothing. Making that a `condition` rather than a code branch keeps it visible.

**And say this out loud, because it is the general rule:** any cache key derived from
user input needs both a TTL and a cardinality bound. Without both, the cache is a
memory leak an attacker can drive.

---

### Trap 5 — `@CacheEvict(allEntries = true)` on a large Redis cache

**Wrong:**
```java
@CacheEvict(cacheNames = "products", allEntries = true)
public void reimportCatalogue(List<ProductImport> batch) { ... }
```
called once per import batch, a few hundred times during a nightly reimport.

**Exact symptom:**

- Redis latency spikes during the reimport window. `redis-cli --latency` shows it.
- Every `orderflow` replica's cache hit rate drops to near zero for the duration,
  which drives the full catalogue read load onto Postgres.
- That load lands on the connection pool you sized in Topic 109 for a *cached*
  workload, so `hikaricp.connections.pending` climbs and p99 on `POST /orders` — an
  endpoint with nothing to do with the catalogue — degrades too.
- The reimport job itself appears fine. The damage is entirely on the read path.

**Root cause:** Redis has no "delete these keys by prefix" primitive. `RedisCache.clear()`
sweeps with `SCAN` over the prefix pattern and deletes what it finds. That is
proportional to the number of keys, it is not atomic, and doing it hundreds of times in
one window means the cache never gets a chance to refill.

Second-order cause: evicting everything is a cache-wide outage that you inflicted on
yourself, and it is almost never what the requirement asked for.

**Fix:**

```java
// Evict exactly the SKUs the batch touched. Bounded, targeted, cheap.
public void reimportCatalogue(List<ProductImport> batch) {
    repository.upsertAll(batch);
    Cache products = cacheManager.getCache("products");
    if (products != null) {
        batch.forEach(item -> products.evict(item.sku()));
    }
}
```

If you genuinely must clear everything — a schema change to the cached DTO, say — do it
once, out of band, with a versioned key prefix rather than a sweep:

```java
// Bump the version on deploy. Old keys age out via TTL; no SCAN, no latency spike.
.prefixCacheNameWith("orderflow:v3:")
```

That trick is worth remembering. It turns "clear the cache" from an O(n) operation into
a configuration change, and it makes the cache safe to change shape during a rolling
deploy, because v2 and v3 pods do not read each other's entries.

---

### Trap 6 — caching `null`, or not caching it, without deciding

**Wrong:** either default, applied without thought.

**Exact symptom — if nulls are NOT cached (`disableCachingNullValues()`):**

Requests for SKUs that do not exist always miss and always hit Postgres. A scraper
enumerating SKU patterns, or a broken client retrying a deleted product, produces a
sustained database read load that the cache does nothing about. This is **cache
penetration**. It looks like a database problem and the cache dashboard looks healthy —
hit rate is fine, because misses on nonexistent keys still count as misses but nobody
is looking at the miss *rate for 404s*.

**Exact symptom — if nulls ARE cached (the Redis default):**

Every nonexistent SKU anyone asks about becomes a Redis entry. Combined with Trap 4's
unbounded key space, an attacker can fill Redis by requesting random SKUs. And a
newly-created product is invisible for a full TTL, because the "does not exist" answer
was cached before it was created — which produces a support ticket that reads "I added
the product but it does not show up".

**Root cause:** it is a real trade-off with no default answer, and both defaults are
wrong for someone.

**Fix — decide per cache, and write the reason in the code:**

```java
// products: cache the negative result briefly. Nonexistent SKUs are common
// (old links, scrapers) and a 30-second window of "not found" is acceptable
// because product creation is an admin action that can tolerate 30 seconds.
.withCacheConfiguration("products",
        base.entryTtl(Duration.ofMinutes(30)))          // null values allowed

// inventoryLevels: never cache a negative. "No stock record" means a data
// problem we want to see immediately, not a cached answer.
.withCacheConfiguration("inventoryLevels",
        base.entryTtl(Duration.ofSeconds(5)).disableCachingNullValues())
```

And on the product-creation path, evict the negative entry explicitly:

```java
@CacheEvict(cacheNames = "products", key = "#product.sku")
@Transactional
public Product create(NewProduct product) { ... }
```

Without that eviction, caching nulls guarantees the "I created it and it is not there"
ticket.

---

## Hands-on proof

Every command below is one **you** run. I have no Redis, no JVM and no metrics
endpoint, and I am not going to print numbers and call them measured.

### Setup

```bash
mkdir -p ~/java-lab/110 && cd ~/java-lab/110
java --version                # expect 21 or 25

docker run -d --name redis110 -p 6379:6379 redis:7 \
  redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

docker run -d --name pg110 \
  -e POSTGRES_USER=orderflow -e POSTGRES_PASSWORD=orderflow -e POSTGRES_DB=orderflow \
  -p 5432:5432 postgres:17

curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql,data-redis,cache,actuator \
  -d javaVersion=21 -d groupId=com.orderflow -d artifactId=cache-lab \
  -d type=maven-project -o cache-lab.zip && unzip cache-lab.zip -d cache-lab
```

Add the Prometheus registry so cache metrics are exported:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
  <scope>runtime</scope>
</dependency>
```

`application.yml`:

```yaml
spring:
  cache:
    type: redis
    cache-names: products,productPrices
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 500ms
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,caches
logging:
  level:
    org.springframework.cache: TRACE
```

### Proof 1 — the cache is actually wired

```bash
curl -s localhost:8080/actuator/caches | jq
```

**What to look for:** every cache name you declared, with its `cacheManager`.

| What you see | What it means |
|---|---|
| `products` listed with `cacheManager: "cacheManager"` | Wired. Proceed. |
| Empty list | `spring.cache.cache-names` is unset **and** no cache has been used yet — Redis caches are created lazily. Either declare the names or call a cached method first. |
| `cacheManager` is `ConcurrentMapCacheManager` | Redis auto-configuration did not apply. Check `spring.cache.type=redis` and that `spring-boot-starter-data-redis` is on the classpath. **You are caching in local heap and will not notice until a second replica exists.** |
| Endpoint 404s | `caches` not in `management.endpoints.web.exposure.include`. |

### Proof 2 — the annotation is on the proxy, and it fires

```properties
logging.level.org.springframework.cache=TRACE
```

**What to look for** when you call a `@Cacheable` method: TRACE lines from
`CacheInterceptor` naming the cache and the computed key, one pair per invocation
(a lookup, then possibly a put).

| What you see | What it means |
|---|---|
| A cache lookup logged, then no put, on the second identical call | Working. Miss then hit. |
| **No cache log lines at all** for a call you expected to be cached | The interceptor never ran. Either self-invocation (Topic 40) or `@EnableCaching` is missing. Print `service.getClass().getName()` and look for `$$SpringCGLIB$$`. |
| A lookup and a put on **every** call | Every call is missing. Check whether your key is stable — a key containing a timestamp, a `Date`, or an object without `equals`/`hashCode` (Topic 13) produces a new key every time. |
| `condition` never satisfied | Your SpEL is silently false. Test the expression by temporarily hardcoding `condition = "true"`. |

Also print the identity once, as Topic 40 taught:

```java
System.out.println(catalog.getClass().getName());
System.out.println(AopUtils.isAopProxy(catalog));
```

### Proof 3 — look at the actual keys and bytes

```bash
# NEVER use KEYS on a production Redis. SCAN is incremental and non-blocking.
redis-cli --scan --pattern 'products::*' | head -20
redis-cli DBSIZE
redis-cli TTL "products::SKU-4471"
redis-cli GET  "products::SKU-4471"
redis-cli MEMORY USAGE "products::SKU-4471"
```

| What you see | What it means |
|---|---|
| Keys shaped `products::SKU-4471` | Default prefixing: `cacheName::key`. |
| Keys shaped `orderflow:products::SKU-4471` | Your `prefixCacheNameWith` is applied. |
| `TTL` returns `-1` | **No expiry set.** Trap 4. Every entry is permanent. Fix the configuration now. |
| `TTL` returns `-2` | The key does not exist. Either it expired or you are looking at the wrong key. |
| `GET` returns readable JSON | Your serializer configuration is applied. |
| `GET` returns binary starting with `\xac\xed` | Java serialization. Trap 2. |
| `DBSIZE` far larger than your product count | Something with an unbounded key space is being cached. Run `redis-cli --bigkeys`. |

### Proof 4 — Micrometer cache metrics

```bash
curl -s localhost:8080/actuator/metrics/cache.gets | jq
curl -s 'localhost:8080/actuator/metrics/cache.gets?tag=result:hit'  | jq '.measurements'
curl -s 'localhost:8080/actuator/metrics/cache.gets?tag=result:miss' | jq '.measurements'
curl -s localhost:8080/actuator/metrics/cache.puts | jq '.measurements'
curl -s localhost:8080/actuator/prometheus | grep '^cache_'
```

Hit rate is `hit / (hit + miss)`. Compute it; do not eyeball it.

| What you see | What it means |
|---|---|
| Both `hit` and `miss` present and hits growing faster | Working cache. Track the ratio over time, not at a point. |
| `cache.gets` missing entirely | The cache was not registered with Micrometer at startup. Declare `spring.cache.cache-names` — lazily-created Redis caches may never get bound. This is the most common reason "cache metrics do not work". |
| Hit rate near zero under steady load | Either the TTL is far shorter than the inter-arrival time for a key, the key is unstable, or self-invocation is bypassing the cache. Check Proof 2 first. |
| Hit rate near 100% and p99 unchanged | The cache is working and the database was never the bottleneck. Honest and useful: stop optimising here and go profile (Topic 78). |
| `cache.puts` far exceeding `cache.gets` misses | Something is calling `@CachePut` in a loop, or eviction is running so often that entries never get read. |

### Proof 5 — watch Redis under load

```bash
redis-cli --stat            # one line per second: keys, memory, clients, ops/sec
redis-cli INFO stats | grep -E 'keyspace_hits|keyspace_misses|evicted_keys|expired_keys'
redis-cli INFO memory | grep -E 'used_memory_human|maxmemory_human|maxmemory_policy'
redis-cli --latency
```

| What to look for | What it means |
|---|---|
| `evicted_keys` climbing | Redis is at `maxmemory` and evicting. Your effective TTL is now shorter than you configured, and you cannot reason about hit rate. Raise memory or shrink the key space. |
| `expired_keys` climbing steadily | Normal TTL expiry. Healthy. |
| `keyspace_hits` / `keyspace_misses` disagreeing with Micrometer's ratio | Something other than Spring Cache is using this Redis instance. Find out what. |
| `--latency` showing spikes correlated with your reimport job | Trap 5's `allEntries` sweep. |
| `maxmemory_policy: noeviction` | The Redis default, and dangerous for a cache. Writes will fail rather than evict. |

`redis-cli MONITOR` prints every command the server receives. It is the best possible
tool for "is my application sending the commands I think it is" and it is genuinely
dangerous — it degrades the server measurably under load. Use it for a few seconds on a
development instance, never on production, never left running:

```bash
timeout 5 redis-cli MONITOR | grep -i 'products' | head -40
```

**What to look for:** `GET` then `SETEX` on a miss; `GET` alone on a hit; `DEL` on an
eviction. If you see `SETEX` on every call, the cache is never hitting.

### Proof 6 — the spine measurement

Against the Topic 65 baseline, with the cache disabled and then enabled:

```bash
# Disable without changing code:
java -jar orderflow.jar --spring.cache.type=none
```

Run the identical k6 scenario both ways, at a fixed arrival rate. Record for each:

| Source | Record |
|---|---|
| k6 | p50 / p95 / p99 for `GET /products/{sku}` |
| k6 | p50 / p95 / p99 for `POST /orders` (should be unaffected — check that it is) |
| Micrometer | cache hit rate at the end of the run |
| Micrometer | `hikaricp.connections.pending` maximum, both runs |
| Micrometer | `hikaricp.connections.acquire` p99, both runs |
| Redis | `INFO stats` hits/misses, `used_memory_human` |

**How to read the comparison:**

| What you see | What it means |
|---|---|
| Catalogue p99 falls substantially, order p99 also falls | Expected and the best outcome: the cache removed catalogue load from the shared connection pool, so orders got faster as a side effect. Say this explicitly in the write-up — it is the most valuable finding. |
| Catalogue p99 falls, `hikaricp.connections.pending` max falls to zero | The database was the constraint and you removed most of the demand. |
| Catalogue p99 barely moves at a high hit rate | The database read was never the expensive part. Profile where the time actually goes (Topic 78) rather than tuning the cache further. |
| Catalogue p50 improves but p99 gets **worse** | Look for the stampede. p99 is where the misses live, and a synchronised expiry makes the tail worse even while the median improves. This is the drill below. |
| Hit rate low despite obvious skew in the load profile | The TTL is shorter than the inter-arrival time for all but the hottest keys, or the k6 script requests SKUs uniformly rather than with the recorded skew. Check the load script before blaming the cache. |

Commit the comparison to `/docs/java/baselines/` alongside Topic 65's numbers.

---

## Failure drill

**Mandatory for this topic even though it is CORE**, because the stampede is the trap
that actually matters and reading about it does not produce the reflex.

### The scenario

One hot product. A TTL. Two hundred concurrent readers. At the instant the entry
expires, all two hundred miss simultaneously and all two hundred query Postgres for the
same row.

### Setup

```java
package com.orderflow.lab;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.concurrent.atomic.AtomicInteger;

@Service
public class HotProductService {

    /** Counts how many times the loader ACTUALLY ran. This is the whole experiment. */
    private final AtomicInteger databaseLoads = new AtomicInteger();

    private final ProductRepository repository;

    public HotProductService(ProductRepository repository) {
        this.repository = repository;
    }

    // sync deliberately ABSENT for run A.
    @Cacheable(cacheNames = "hotProducts", key = "#sku")
    public ProductView load(String sku) {
        databaseLoads.incrementAndGet();
        // A realistic query cost. Without this the race window is too small to observe.
        try { Thread.sleep(50); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return repository.findViewBySku(sku).orElseThrow();
    }

    public int databaseLoadCount() { return databaseLoads.get(); }
    public void resetCounter()     { databaseLoads.set(0); }
}
```

```java
package com.orderflow.lab;

import org.springframework.web.bind.annotation.*;

@RestController
public class StampedeController {

    private final HotProductService hot;

    public StampedeController(HotProductService hot) { this.hot = hot; }

    @GetMapping("/drill/hot/{sku}")
    public ProductView get(@PathVariable String sku) { return hot.load(sku); }

    @GetMapping("/drill/loads")
    public int loads() { return hot.databaseLoadCount(); }

    @PostMapping("/drill/reset")
    public void reset() { hot.resetCounter(); }
}
```

```yaml
spring:
  cache:
    type: redis
    cache-names: hotProducts
    redis:
      time-to-live: 10s          # short, so the drill runs in seconds not minutes
```

### Run A — no protection

```bash
# 1. Warm the cache and clear the counter.
curl -s localhost:8080/drill/hot/SKU-4471 > /dev/null
curl -X POST localhost:8080/drill/reset

# 2. Confirm the key is there and note its remaining TTL.
redis-cli TTL "hotProducts::SKU-4471"

# 3. Wait for it to expire, then fire 200 concurrent requests at the instant after.
redis-cli DEL "hotProducts::SKU-4471"      # deterministic expiry — better than sleeping
for i in $(seq 1 200); do
  curl -s "localhost:8080/drill/hot/SKU-4471" > /dev/null &
done
wait

# 4. How many times did the database loader actually run?
curl -s localhost:8080/drill/loads
```

Watch, in another terminal, while step 3 runs:

```bash
docker exec pg110 psql -U orderflow -d orderflow -c \
  "select state, count(*) from pg_stat_activity where datname='orderflow' group by state;"

curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending | jq '.measurements'
```

### What to capture

Write these down before reading further:

1. The load count reported by `/drill/loads`.
2. The maximum `hikaricp.connections.pending` you saw during the burst.
3. The `pg_stat_activity` state breakdown during the burst.
4. Whether any request returned an error, and which.
5. Wall-clock time for the whole `for` loop to finish.

### How to read it

| What you see | What it means |
|---|---|
| Load count far above 1 — tens or more | **The drill has fired.** One expired key produced many identical database queries. That is the stampede, and it is a multiplier you built yourself. |
| Load count exactly 1 | Your requests were not concurrent — `&` backgrounding in a shell is not always fast enough. Use a real load generator (`k6`, `hey -c 200 -n 200`) and retry. |
| `hikaricp.connections.pending` spiking to a large number | The burst saturated the pool you sized in Topic 109. **A cache miss storm is a connection-pool event.** This link is the reason this drill exists. |
| Some requests returning `SQLTransientConnectionException` | The burst exceeded pool capacity for longer than `connection-timeout`. A cache designed to reduce load caused an outage. |
| `pg_stat_activity` showing many `active` backends running the same query | The database is doing the same work over and over. Look at the query text — they are identical. |

### Run B — `sync = true`

Change one attribute:

```java
@Cacheable(cacheNames = "hotProducts", key = "#sku", sync = true)
public ProductView load(String sku) { ... }
```

Re-run the identical steps.

| What you see | What it means |
|---|---|
| Load count drops to 1 (single JVM) | `sync = true` made the interceptor call `Cache.get(key, Callable)`, which serialises concurrent loaders for the same key. One loads; the rest wait for its result. |
| Total wall time roughly unchanged | Correct and expected. The 199 waiting requests still wait for the one slow load. You removed the database amplification, not the latency. Say this precisely. |
| Load count still high | Check that the cache manager actually supports `sync`. Not every `Cache` implementation does, and an unsupported one may fall back silently. Verify with the `caches` actuator endpoint which implementation you have. |

### Run C — the limitation that matters

Start **three** instances on different ports against the same Redis:

```bash
java -jar target/cache-lab.jar --server.port=8081 &
java -jar target/cache-lab.jar --server.port=8082 &
java -jar target/cache-lab.jar --server.port=8083 &

redis-cli DEL "hotProducts::SKU-4471"
for p in 8081 8082 8083; do
  for i in $(seq 1 70); do curl -s "localhost:$p/drill/hot/SKU-4471" > /dev/null & done
done
wait

for p in 8081 8082 8083; do echo -n "port $p loads: "; curl -s "localhost:$p/drill/loads"; echo; done
```

| What you see | What it means |
|---|---|
| Each instance reports a load count of 1; three loads total | **The limitation, demonstrated.** `sync = true` deduplicates within one JVM. It does not coordinate across replicas. With N replicas your stampede floor is N, not 1. |
| Some instance reports more than 1 | Its own `sync` is not engaging — recheck that instance's configuration. |

> **Uncertainty, flagged honestly:** how `RedisCache` implements
> `get(Object, Callable)` — a JVM-local per-key lock versus any Redis-side coordination —
> has changed across Spring Data Redis versions, and there is also a separate,
> unrelated `RedisCacheWriter` locking mode that concerns `clear()` rather than
> per-key loads. **Do not take my word for the mechanism.** Run Run C on your version:
> the three-instance experiment answers the question empirically and takes five
> minutes. Then read `RedisCache.get(Object, Callable)` for the version your Boot BOM
> resolves.

### Run D — the fix that scales across replicas

`sync = true` gets you from N-per-replica to 1-per-replica. To get to 1 overall you
need a distributed lock, and you should reach for it only when the arithmetic justifies
the complexity.

```java
package com.orderflow.lab;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.UUID;
import java.util.function.Supplier;

@Service
public class SingleFlightLoader {

    private final StringRedisTemplate redis;

    public SingleFlightLoader(StringRedisTemplate redis) { this.redis = redis; }

    /**
     * One loader per key across the whole fleet, best-effort.
     *
     * This is NOT a correct distributed lock and must not be used where
     * correctness depends on mutual exclusion. It is a load-shedding
     * optimisation: a lost lock costs a duplicate query, which is harmless.
     * A correct distributed lock needs fencing tokens and a much longer
     * conversation about failure modes.
     */
    public <T> T loadOnce(String lockKey, Duration lockTtl,
                          Supplier<T> loader, Supplier<T> fallback) {
        String token = UUID.randomUUID().toString();
        Boolean acquired = redis.opsForValue()
                .setIfAbsent("lock:" + lockKey, token, lockTtl);

        if (Boolean.TRUE.equals(acquired)) {
            try {
                return loader.get();
            } finally {
                // Best-effort release. A crash here leaves the lock to expire by TTL,
                // which is why the TTL must exceed the worst-case load time.
                redis.delete("lock:" + lockKey);
            }
        }
        // Lost the race. Do NOT queue behind the winner; serve something now.
        return fallback.get();
    }
}
```

The `fallback` is the design decision, and it is where most teams get it wrong. Options,
in order of preference for a product catalogue:

1. **Serve the stale value.** Keep two TTLs — a soft one and a hard one — and serve the
   soft-expired value while one loader refreshes. This is stale-while-revalidate, and
   for a product catalogue it is almost always right: a 30-second-old price shown to a
   customer is fine; a 500 is not.
2. **Wait briefly with a timeout, then fall through to the database.** Bounded damage.
3. **Fail fast with a 503.** Only if stale data is genuinely unacceptable.

**Never** queue unboundedly behind the lock holder. That converts a cache miss into a
thread-exhaustion event, which is Topic 109 again.

### What the drill proves

Three things, and you should be able to state each in one sentence:

1. A TTL is not a stampede mitigation. Expiry is precisely when the stampede happens.
2. `sync = true` is a per-JVM mitigation. Its floor is your replica count, and you must
   say the replica count out loud when you propose it.
3. A cache miss storm is a connection-pool event. The two topics are one system, and
   the instrument that shows it is `hikaricp.connections.pending` during the burst.

---

## Practice exercises

### 1 — Easy: prove the annotations do what you think

**Part A.** Build a service with all three annotations on the same cache: `@Cacheable`
on a read, `@CachePut` on an update, `@CacheEvict` on a delete. Call them in this
sequence, checking Redis with `redis-cli` after each step:

```
read(SKU-1)  ->  read(SKU-1)  ->  update(SKU-1)  ->  read(SKU-1)  ->  delete(SKU-1)  ->  read(SKU-1)
```

For each of the six steps, predict **before running**: does the method body execute?
Does Redis change? What is the value in Redis afterwards? Then check, and explain every
prediction you got wrong.

**Part B.** Replace `@CachePut` with `@Cacheable` on the update method. Run the same
sequence. Explain precisely why the update appears not to happen, in terms of what the
interceptor does.

**Part C.** Set `unless = "#result == null"` on the read and request a SKU that does not
exist, twice. Then remove `unless` and repeat. Show the difference with
`redis-cli --scan` and explain which behaviour you want for `orderflow`, and why.

### 2 — Medium: the audit (combines Topics 01–108)

Below is a caching layer from a Spring Boot 4.1 service. It contains **seven** distinct
defects. For each: name the exact symptom an on-call engineer observes, name the topic
it comes from, and write the fix.

```java
package com.orderflow.catalog;

import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class CatalogService {

    private final ProductRepository products;
    private final PriceClient priceClient;

    public CatalogService(ProductRepository products, PriceClient priceClient) {
        this.products = products;
        this.priceClient = priceClient;
    }

    @Cacheable("catalog")
    public Product findBySku(String sku) {
        return products.findBySku(sku).orElse(null);
    }

    @Cacheable("catalog")
    public Category findCategory(String code) {
        return products.findCategoryByCode(code).orElse(null);
    }

    public List<Product> findAll(List<String> skus) {
        return skus.stream().map(this::findBySku).collect(Collectors.toList());
    }

    @Cacheable(value = "prices", key = "#sku + java.time.Instant.now().toString()")
    public Long livePrice(String sku) {
        return priceClient.fetch(sku);
    }

    @Transactional
    @CacheEvict(value = "catalog", allEntries = true)
    public void bulkUpdatePrices(List<PriceUpdate> updates) {
        for (PriceUpdate u : updates) {
            Product p = products.findBySku(u.sku()).orElseThrow();
            p.setPriceMinor(u.priceMinor());
            priceClient.notifyPartner(u.sku(), u.priceMinor());
        }
    }

    @Cacheable(value = "searchResults", key = "#q")
    public List<Product> search(String q) {
        return products.search(q);
    }
}
```

Hints, one per defect, unordered: Topic 40, Topic 41, Topic 50, Topic 55, and three from
this topic (key collision, unbounded key space, and one where the key means the cache
can never hit).

**Part B.** Rewrite it. Then state, in three sentences, what your rewrite gave up
relative to the original — because a correct rewrite here does give something up, and
naming it is the point.

### 3 — Hard: production simulation on `orderflow` under load

Budget a full session. This is the Spine deliverable.

**Part A — baseline.** Re-run Topic 65's load with `spring.cache.type=none`. Confirm
you are within ±10% of the committed baseline. Record catalogue p50/p95/p99, order
p50/p95/p99, `hikaricp.connections.pending` max and `acquire` p99.

**Part B — cache the catalogue.** Add `@Cacheable` to the product read path with a
30-minute TTL and explicit eviction on write. Re-run the identical load. Produce a
before/after table with every metric from Part A plus the cache hit rate.

**Part C — measure the warm-up.** Restart with a cold Redis and run the load
immediately. Plot hit rate and catalogue p99 over the first five minutes. Answer: how
long until the cache is providing its steady-state benefit, and what does that mean for
a rolling deploy or a Redis restart?

**Part D — the stampede at scale.** With the cache warm and the load running, run
`redis-cli FLUSHDB`. Capture, over the following sixty seconds: catalogue p99,
`hikaricp.connections.pending`, `hikaricp.connections.timeout`, the order-placement
error rate, and Postgres CPU. Write one paragraph explaining why an endpoint that does
not read the catalogue degraded.

**Part E — mitigate, three ways.** Implement and measure each independently, repeating
Part D each time:
1. `sync = true`.
2. A jittered TTL — add a random 0–20% to each entry's TTL so hot keys do not expire
   together. (This needs a custom `Cache` wrapper or a `RedisCacheWriter`; describe the
   approach even if you only prototype it.)
3. Stale-while-revalidate with the `SingleFlightLoader` from the drill.

Produce one table comparing all three plus the unmitigated case, on: catalogue p99
during the storm, peak `hikaricp.connections.pending`, and order error rate.

**Part F — the Redis outage.** Stop Redis entirely while the load runs. Capture what
happens. Then decide and implement: should a Redis failure fail the request, or fall
through to Postgres? Implement your choice — a `CacheErrorHandler` is the hook —
re-run, and show that a Redis outage now degrades rather than breaks. State the
capacity condition under which "fall through to Postgres" is the wrong answer, using
your own numbers from Part A.

---

## Interview questions

### Q1 — "We added `@Cacheable` to our product lookup and the hit rate is zero. Where do you look?"

**Mid-level answer:** "I would check the TTL and make sure Redis is actually connected.
Maybe the key is different each time."

**Senior answer:** "Four things, in this order, because they are ordered by how often
they are the cause.

First, self-invocation. `@Cacheable` is implemented by a proxy, so a call from inside
the same bean — including `this::findBySku` in a stream — never touches the
interceptor. I would print `service.getClass().getName()` and look for
`$$SpringCGLIB$$`, and turn on `org.springframework.cache` at TRACE. If there are no
interceptor log lines at all for that call path, that is the answer and it takes thirty
seconds to establish.

Second, key stability. If the key expression includes anything time-derived, or an
object without `equals` and `hashCode`, every call generates a new key. TRACE logging
shows the computed key, so you can see it changing.

Third, `condition` and `unless`. `condition` is evaluated before the call and cannot
see `#result`; people put `#result` there and it silently never matches.

Fourth — and this one is a real gotcha — the metrics themselves. Spring registers cache
metrics for caches that exist at startup. Redis caches are created lazily, so if you
have not declared `spring.cache.cache-names`, `cache.gets` may be absent entirely and
'zero hit rate' actually means 'no metric'. I would confirm the cache is working with
`redis-cli --scan` before trusting the dashboard."

**What separates them:** starting with the proxy, having an ordered checklist rather
than a list, and knowing that the measurement instrument itself can be the thing that
is broken.

**Follow-up:** "How would you stop this recurring?" Good answers: a test that asserts
the loader ran once across two calls, running against a real cache manager — an
assertion on behaviour, not on the annotation being present.

---

### Q2 — "Explain the cache stampede and how you would prevent it."

**Mid-level answer:** "When a cache entry expires, lots of requests miss at once and
all hit the database. You prevent it with a lock so only one request loads the value."

**Senior answer:** "Right shape, and the Java specifics are where it gets interesting.

The mechanism: a hot key expires, and every concurrent request for it misses in the
same instant. The database receives a burst equal to your full concurrency for one row.
The important second-order effect is that this burst lands on the connection pool — so
a cache miss storm presents as connection-pool exhaustion, and the endpoints that
degrade include ones with nothing to do with the cache.

Mitigations, in the order I would apply them.

`@Cacheable(sync = true)` first, because it is one attribute. It makes the interceptor
use `Cache.get(key, Callable)`, which serialises concurrent loaders for the same key.
The limitation people miss is that this is **per JVM**. With five replicas your floor is
five simultaneous loads, not one. I would say that number out loud when proposing it.

Jittered TTLs second, because they attack the synchronisation itself. If everything was
warmed in one batch, everything expires in one batch. Adding a random percentage to
each entry's TTL spreads the expiries out and it costs nothing.

Stale-while-revalidate third, and it is the one that actually solves it for a catalogue.
Keep a soft expiry and a hard expiry: past soft, serve the stale value and trigger one
background refresh; past hard, block. For product data, a 30-second-old price is
acceptable and a 500 is not, so this is usually the right answer.

A distributed lock last, because it is the most complex and the failure modes are
subtle. And whatever the losers of that lock do, they must not queue unboundedly — that
converts a cache miss into thread exhaustion.

The thing I would check before any of this: whether the stampede is even happening. p50
improving while p99 gets worse after adding a cache is the signature."

**What separates them:** naming the per-JVM limitation of `sync`, connecting the storm
to the connection pool, treating jitter as attacking synchronisation rather than as a
trick, and refusing to queue behind the lock.

**Follow-up:** "How would you prove `sync = true` is per-JVM?" They want the
three-instance experiment, not a documentation citation.

---

### Q3 — "Your service caches a value, then the transaction rolls back. What happens?"

**Mid-level answer:** "The cache would have the wrong value. You would need to evict it
in a catch block or something."

**Senior answer:** "The cache keeps the uncommitted value for the full TTL, because
Redis has no idea a transaction existed. The database says one thing and the cache says
another, permanently, and restarting does not fix it because the value is not in the
JVM.

Three fixes, and only one of them is general.

The general one is `TransactionAwareCacheManagerProxy`. Wrap the cache manager in it
and every put and evict is deferred until after commit; a rollback means the cache
operation never happens. It is declarative, so it covers the annotations someone adds
next year.

The design-level one is preferring `@CacheEvict` over `@CachePut` on write paths.
Under rollback, an unnecessary eviction costs one database read; an incorrect put costs
a wrong price for the TTL. The failures are asymmetric, so pick the cheap one.

The one I would not do is a catch block, because it only covers the exceptions you
thought of and it is invisible to the next person.

Worth noting the cost of the general fix: with transaction-aware caching, a method that
evicts and then reads within the same transaction sees the old cached value, because
the eviction has not happened yet. That is usually what you want and occasionally
surprising, so it belongs in a code comment."

**What separates them:** naming the specific class, the asymmetric-failure argument for
evict over put, and volunteering the downside of their own recommendation.

**Follow-up:** "Does this also apply to Kafka publishes inside a transaction?" They are
checking whether you connect it to the dual-write problem — Topic 115.

---

### Q4 — "How do you decide what the TTL should be?"

**Mid-level answer:** "It depends on how often the data changes. Five minutes is
usually a reasonable default."

**Senior answer:** "The TTL is not the freshness mechanism. It is the *backstop*, and
conflating those is where most cache designs go wrong.

Freshness comes from explicit invalidation on the write path. If prices must be visible
within one minute and prices change through our admin API, then the admin API evicts,
and freshness is bounded by network time, not by the TTL.

The TTL then answers a different question: how long am I willing to serve stale data
when invalidation *fails*? And it does fail — a data migration, a manual `UPDATE`, a
batch job that writes with native SQL, another service on the same tables. That is
Topic 51's second-level-cache lesson exactly. So the TTL is set by 'what is the worst
staleness I can defend in an incident review', which for a product catalogue might be
thirty minutes and for an inventory level might be five seconds.

Two things I would add. First, jitter — if everything is warmed together it expires
together, and that is a self-inflicted stampede. Second, the TTL interacts with hit
rate through the request inter-arrival time: if a key is requested every ten minutes
and the TTL is five, the hit rate for that key is zero and you are paying for Redis to
achieve nothing. Which means the TTL is partly a memory-versus-hit-rate decision and
should be set per cache, not globally."

**What separates them:** separating freshness from staleness-tolerance, naming the
writers that bypass invalidation, and connecting TTL to inter-arrival time.

**Follow-up:** "What if you cannot invalidate — a third party writes to the database?"
Good answers: then the TTL *is* the freshness mechanism and must be set to the
requirement, or you use change-data-capture to drive invalidation (Topic 115's
territory).

---

### Q5 — "Redis goes down. What happens to your service?"

**Mid-level answer:** "Requests would fail until it comes back. We would need a circuit
breaker around it."

**Senior answer:** "By default, an exception from the cache propagates and the request
fails — which means a cache outage becomes a service outage, and that is backwards. A
cache is supposed to be an optimisation.

The hook is `CacheErrorHandler`. Override it to log and fall through to the method, and
now a Redis failure degrades to 'slower' instead of 'broken'.

But that is only correct if the answer to one question is yes: **can Postgres absorb the
full uncached read load?** For `orderflow`, 70% of traffic is catalogue reads against a
pool of 16. If the cache is doing most of that work, falling through means the
connection pool saturates, `pending` climbs, and every endpoint including order
placement starts timing out. So a Redis outage becomes a Postgres outage. That is worse
than failing catalogue reads and keeping orders working.

The honest answer is that it is a measured decision, not a principle. I would run the
load with `spring.cache.type=none` — Part A of our own baseline — and see whether the
database survives. If it does, fall through. If it does not, then either fall through
only for a bounded fraction of requests using a bulkhead, or fail catalogue reads fast
and protect the order path, which is the higher-value traffic.

And I would add a timeout on the Redis client well below the request SLO — the default
Lettuce command timeout is far too long for a cache lookup. A Redis that is *slow* is
more dangerous than a Redis that is *down*, because slow does not trip anything."

**What separates them:** treating fall-through as a capacity question with a
measurement attached, prioritising order traffic over catalogue traffic, and the
observation that slow is worse than down.

**Follow-up:** "How would you find out whether Postgres can absorb it, without an
outage?" They want the `spring.cache.type=none` load run — a cheap experiment that
answers it exactly.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `@Cacheable` reads the cache and may skip your method. `@CachePut` always runs your
   method and always writes. Construct a scenario where using `@Cacheable` where you
   meant `@CachePut` produces a bug that no test catches, and say why the test misses it.

2. The default `SimpleKeyGenerator` uses the single argument itself as the key. That is
   a reasonable default in isolation. Explain what it assumes about how cache names are
   allocated, and why that assumption fails in a codebase with more than one developer.

3. A TTL of 30 minutes with explicit eviction on write gives you one-second freshness in
   the normal case. What exactly is the 30 minutes protecting you against? Name three
   concrete writers that would bypass your eviction path in `orderflow`.

4. `sync = true` deduplicates loads within one JVM. Suppose you run 50 replicas.
   Compute the stampede reduction factor, then argue whether `sync = true` is still
   worth enabling. Now do the same for 2 replicas.

5. Caching nulls prevents cache penetration but enables a memory-exhaustion attack, and
   not caching them does the reverse. Design a scheme that gets most of both, and name
   what it costs in complexity.

6. Your cache hit rate is 99%. Your p50 improved a lot and your p99 got slightly worse.
   Give the most likely explanation, and name the one metric that would confirm it.

7. Someone proposes putting a Caffeine cache in front of the Redis cache for the hottest
   1000 products. State the correctness problem this introduces across three replicas,
   and the two ways to address it. Then say which you would choose for `orderflow` and
   why.

---

## Quick reference card

### The three annotations

| | Runs method | Reads cache | Writes cache | Use for |
|---|---|---|---|---|
| `@Cacheable` | on miss only | yes | on miss | reads |
| `@CachePut` | always | no | always | writes where the return value is the new state |
| `@CacheEvict` | always | no | removes | writes, deletes, and anything you are unsure about |

### Attribute semantics

```java
@Cacheable(
  cacheNames = "products",         // the namespace. Always name it explicitly.
  key        = "#sku",             // always write this. Never trust the default.
  condition  = "#sku != null",     // BEFORE the call. #result NOT available.
  unless     = "#result == null",  // AFTER the call. #result available. Reads backwards.
  sync       = true                // per-JVM dedupe of concurrent misses.
)
```

### Redis configuration that is not optional

```java
RedisCacheConfiguration.defaultCacheConfig()
    .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer()))
    .serializeValuesWith(SerializationPair.fromSerializer(jsonSerializer))  // NOT JDK
    .entryTtl(Duration.ofMinutes(10))          // never leave unset
    .disableCachingNullValues()                // decide deliberately, per cache
    .prefixCacheNameWith("orderflow:v1:");     // bump the version to "clear" for free
```

```conf
maxmemory 512mb
maxmemory-policy allkeys-lru     # the Redis default is noeviction: writes FAIL at the limit
```

```java
@Bean @Primary
CacheManager cacheManager(RedisCacheManager delegate) {
    return new TransactionAwareCacheManagerProxy(delegate);   // defers puts/evicts to commit
}
```

### Diagnostic commands

```bash
# Spring side
curl -s localhost:8080/actuator/caches | jq
curl -s 'localhost:8080/actuator/metrics/cache.gets?tag=result:hit'  | jq '.measurements'
curl -s 'localhost:8080/actuator/metrics/cache.gets?tag=result:miss' | jq '.measurements'
curl -s localhost:8080/actuator/metrics/cache.puts | jq '.measurements'
curl -s localhost:8080/actuator/prometheus | grep '^cache_'

# Redis side — SCAN, never KEYS
redis-cli --scan --pattern 'orderflow:products::*' | head
redis-cli DBSIZE
redis-cli TTL  "orderflow:products::SKU-4471"      # -1 means NO EXPIRY
redis-cli GET  "orderflow:products::SKU-4471"      # should be readable JSON
redis-cli MEMORY USAGE "orderflow:products::SKU-4471"
redis-cli --bigkeys
redis-cli --stat
redis-cli --latency
redis-cli INFO stats  | grep -E 'keyspace_hits|keyspace_misses|evicted_keys'
redis-cli INFO memory | grep -E 'used_memory_human|maxmemory'
timeout 5 redis-cli MONITOR | grep products   # DEV ONLY. Degrades the server.

# Turn caching off without a code change, to measure its value
java -jar orderflow.jar --spring.cache.type=none
```

```properties
logging.level.org.springframework.cache=TRACE
```

### Gotchas checklist

- [ ] `@EnableCaching` is present. Without it, every annotation is inert.
- [ ] No `@Cacheable` method is called from inside its own bean (Topic 40).
- [ ] Every `@Cacheable` declares an explicit `key`.
- [ ] Value serializer is JSON, not `JdkSerializationRedisSerializer`.
- [ ] Every cache has a TTL. Verify with `redis-cli TTL` — `-1` means no expiry.
- [ ] `maxmemory` and `maxmemory-policy` are set on the Redis instance.
- [ ] `CacheManager` is wrapped in `TransactionAwareCacheManagerProxy`.
- [ ] Write paths use `@CacheEvict`, not `@CachePut`, unless there is a reason.
- [ ] No cache key is derived from unbounded user input without a `condition`.
- [ ] `@CacheEvict(allEntries = true)` is not on a frequently-called method.
- [ ] `spring.cache.cache-names` is declared, so Micrometer binds metrics at startup.
- [ ] Redis command timeout is well below the request SLO.
- [ ] `CacheErrorHandler` behaviour on a Redis outage is a decision, backed by a
      `spring.cache.type=none` load run.

---

## When would I use this at work?

**1. A read-heavy endpoint whose p99 is dominated by a database round trip.**
This is the common case and the cache is often the right answer. What this topic gives
you is the discipline to measure the hit rate and the p99 *both* before and after, so
you can tell the difference between "the cache is working" and "the database was never
the bottleneck". The second outcome is common and is worth discovering in a day rather
than a quarter.

**2. Reviewing a pull request that adds `@Cacheable`.**
Four questions, every time: is it called from inside its own class; is there a TTL; is
there an eviction path on every writer; and what is the key's cardinality. Those four
catch most cache bugs before they ship, and asking them takes a minute.

**3. An incident where the database is saturated and nothing changed.**
Check whether Redis evicted, restarted, or was flushed. A cache that quietly stopped
working transfers its entire load to the database with no deploy and no code change,
and it is invisible unless you are watching hit rate. This is the incident that gets
misdiagnosed as "the database got slower".

---

## Connected topics

**Prerequisites:**

- **13 — equals/hashCode**: cache keys are map keys. An unstable key means a cache that
  never hits.
- **19 — Serialization**: why the default Redis value serializer is the wrong default,
  and why "it's just a cache" does not exempt it from a security review.
- **40 — Proxying**: `@Cacheable` lives on the proxy. Self-invocation silently disables
  it, and it is the most common cause of a zero hit rate.
- **41 — AOP ordering**: the cache-versus-transaction ordering problem, and why
  `TransactionAwareCacheManagerProxy` is the answer rather than tuning `@Order`.
- **51 — Hibernate caching**: the same invalidation lesson one layer down. A cache is
  only correct if every writer goes through its invalidation path.
- **55 — Transactions and the pool**: why a cache miss storm is a connection-pool event.
- **65 — The load baseline**: the hit-rate and p99 measurements are meaningless without
  it.
- **109 — HikariCP**: the stampede lands on the pool. `hikaricp.connections.pending`
  during a miss burst is the metric that links the two topics.

**This unlocks:**

- **111 — Resilience4j**: a circuit breaker around Redis, and the bulkhead that bounds
  how many requests fall through to the database during a cache outage.
- **115 — The outbox**: cache invalidation driven by change-data-capture, for the
  writers that bypass your service entirely.
- **118 — Micrometer**: `cache.gets`, `cache.puts` and `cache.evictions` as first-class
  dashboard metrics, and why a cache key in a metric tag is a cardinality disaster.
- **119 — Tracing**: a span around the cache lookup turns "the request was slow" into
  "the cache missed and the load took 900 ms".
- **129 — Capacity modelling**: the cache hit rate is an input. Your database capacity
  requirement is `(1 - hit_rate) x request_rate`, and that multiplication is why hit
  rate deserves an SLO of its own.

---

## `[BOOT 3.x DELTA]`

The Spring Cache abstraction — `@Cacheable`, `@CachePut`, `@CacheEvict`, `@EnableCaching`,
`CacheManager`, SpEL key expressions, `sync`, `condition`, `unless` — is unchanged
between Boot 3.5 and Boot 4.1. Everything in the Syntax breakdown compiles on both.

Three real differences:

1. **Jackson 3 is standard in Boot 4; Jackson 2 support is deprecated.** This is the
   one thing in this document that may need a different class name on your version. The
   Jackson-2-era serializer identifier is `GenericJackson2JsonRedisSerializer`; on Boot
   4.1 with Jackson 3 there may be a differently-named equivalent. **Check the Spring
   Data Redis reference for the version your Boot BOM resolves rather than copying an
   identifier from anywhere, including this document.** The principle — JSON, not JDK
   serialization — is version-independent.

2. **Module and package locations moved** in Boot 4's codebase modularisation. If you
   are reading Spring's own source to understand how the Redis cache auto-configuration
   wires up, the package you remember from 3.x may have moved. Find it via `--debug` and
   the condition-evaluation report (Topic 42).

3. **Redis master/replica auto-configuration** is new in Boot 4. It does not change
   anything in this document, but it is the reason a Boot 4 service may connect
   differently to a replicated Redis than the 3.x service it replaced — worth knowing
   during a migration.

Boot 3.5 left OSS support in June 2026. Nothing in this topic is a reason to stay on
it.

---

*Java baseline 21. The Spring Cache abstraction has been stable since Spring 3.1 and
the annotations have not changed meaningfully in a decade, so this knowledge has a long
half-life. The parts most likely to move under you are the Redis serializer class names
during the Jackson 2 to Jackson 3 transition, and the internals of
`RedisCache.get(Object, Callable)` that back `sync = true` — both of which this document
tells you to verify rather than assume.*
