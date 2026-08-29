# 60 — The Spring Test Context — `@SpringBootTest` vs Slices, and Context Caching

## Phase: 6 — Testing
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow` slice tests for the three layers that have framework behaviour worth testing — the web layer (`@WebMvcTest` over `OrderController`), the persistence layer (`@DataJpaTest` over `OrderRepository`), and security (`@WebMvcTest` plus `spring-security-test` over the JWT rules from Topic 57).

---

## Mastery line (from the master plan)

> You can explain why a single `@MockBean` in one test class doubles your suite
> runtime (it forks the cached context), and you use `@WebMvcTest`/`@DataJpaTest`
> deliberately.

## Mid → Senior (from the master plan)

> "We use `@SpringBootTest` everywhere" → "the test context is cached by its full
> configuration key; any customisation — a `@MockBean`, a property override, an
> active profile — creates a new context and pays full startup. That's usually the
> entire explanation for a 12-minute suite."

---

## ELI5 anchor

Starting a Spring application is like opening a restaurant for the day: you unlock
the doors, light the ovens, staff every station, stock every fridge, and connect the
card machine. It takes a while. Once it is open, serving one customer is fast.

A test that needs Spring needs the restaurant open.

So Spring does the sensible thing: **it opens the restaurant once and leaves it
open** for every test that wants the same restaurant. That is the *context cache*,
and it is the single reason a Spring test suite is minutes rather than hours.

The catch is what counts as "the same restaurant". Spring is extremely literal about
it. If one test says "same restaurant, but with a different card machine" — that is
a `@MockitoBean` — Spring does not swap the card machine. **It builds a second,
complete restaurant from scratch**, ovens and all, and keeps both open.

Ask for four small variations across your test suite and you have five restaurants.
Your suite time is now five full startups, not one.

Everything else in this topic is detail. That is the mechanism.

---

## The bridge from what you know

### What you do in NestJS

```ts
const moduleRef = await Test.createTestingModule({
  imports: [OrdersModule],
})
  .overrideProvider(PaymentGateway)
  .useValue(fakeGateway)
  .compile();

const app = moduleRef.createNestApplication();
await app.init();
```

You build a testing module, override a provider, compile, and get an app. It is
explicit, it is per-test-file, and it is fast enough that nobody thinks about it.

Spring's equivalent looks superficially similar:

```java
@SpringBootTest
class OrderPlacementIT {

    @MockitoBean PaymentGateway paymentGateway;   // the override

    @Autowired OrderService orderService;
}
```

### Where the analogy breaks, and it breaks hard

| Nest | Spring | Verdict |
|---|---|---|
| `Test.createTestingModule({ imports: [...] })` | `@SpringBootTest` / a slice annotation | **PARTIAL** |
| `.overrideProvider(X).useValue(fake)` | `@MockitoBean X x;` | **PARTIAL** — same intent, wildly different cost |
| The testing module is rebuilt per test file | **The context is cached and shared across test classes** | **NO ANALOGUE** |
| An override affects only that file | **An override creates a whole new cached context** | **NO ANALOGUE — this is the topic** |
| Startup cost is small; you never think about it | Startup cost is seconds, and it is multiplied by the number of distinct configurations | **NO ANALOGUE** |
| `imports: [OrdersModule]` — explicit, narrow | `@SpringBootTest` loads **everything**; slices narrow it | **PARTIAL** |

Two habits you have that will hurt you here:

**Habit 1: "override whatever I need, it's cheap."** In Nest it is. In Spring, each
distinct set of overrides is a distinct cached context and a full startup.

**Habit 2: "one testing module per test file is normal."** In Spring, that pattern
produces a suite where every test class starts its own context, which is the
12-minute suite in the interview question.

The thing to internalise: **in Spring, the unit of cost is not the test class, it is
the distinct context configuration.** Fifty test classes sharing one configuration
cost one startup. Five test classes with five different configurations cost five.

---

## What is this?

### The layers

| Thing | What it does |
|---|---|
| `@ExtendWith(SpringExtension.class)` | The JUnit 5 extension (Topic 58) that ties JUnit's lifecycle to Spring's `TestContextManager`. Every Spring test annotation is a meta-annotation that includes it. |
| `TestContextManager` | Per test class. Asks for a context, then fires listener callbacks (dependency injection, transactions, SQL scripts) around each test. |
| **`MergedContextConfiguration`** | **The cache key.** A value object describing exactly which context this test class wants. |
| `DefaultContextCache` | A map from `MergedContextConfiguration` to a live `ApplicationContext`, with LRU eviction. Default max size **32**. |
| `TestExecutionListener`s | The hooks: `DependencyInjectionTestExecutionListener` (autowires your test), `TransactionalTestExecutionListener` (starts and rolls back the test transaction), `SqlScriptsTestExecutionListener`, and so on. |

### The context cache key — precisely

This is the most valuable thing in the document. Read it slowly.

The cached context is keyed by a `MergedContextConfiguration`, whose equality is
computed from **all** of the following. Change any one of them by any amount and you
have a different key, therefore a cache miss, therefore a full context startup.

| Part of the key | Changed by |
|---|---|
| **Configuration classes / locations** | `@SpringBootTest(classes = ...)`, a nested `@TestConfiguration`, `@Import(...)`, `@ContextConfiguration` |
| **Active profiles** | `@ActiveProfiles("test")` — and the *set* matters, so `{"test"}` and `{"test","kafka"}` are different keys |
| **Property sources** | `@TestPropertySource(locations = ...)` |
| **Inline properties** | `@SpringBootTest(properties = "orderflow.pricing.tier=5")` — **string-for-string**; a different value is a different key |
| **Context initializers** | `@ContextConfiguration(initializers = ...)` |
| **Context customizers** | This is the sneaky one. `@MockitoBean` / `@MockitoSpyBean` / `@MockBean` register a *bean override* customizer. So do `@DynamicPropertySource`, `webEnvironment = RANDOM_PORT`, and several slice annotations. |
| **Context loader** | Differs between `@SpringBootTest` and the various slices |
| **Parent context** | Rarely relevant in a Boot app |

So: **the set of mocked beans is part of the cache key.**

That is the whole explanation for the headline claim. `@MockitoBean PaymentGateway`
in one test class means that class's key differs from every other class's key, so
Spring builds a second complete context. Two mocked beans in two different classes,
in two different combinations, means three contexts: the plain one, the one with the
gateway mocked, and the one with something else mocked.

> **This is not a bug and it is not laziness.** Spring cannot swap a bean in a live
> context safely: other beans have already been injected with the real one, proxies
> have been created around it (Topic 40), `@PostConstruct` has run. The only sound
> way to produce a context where `PaymentGateway` is a mock is to build the context
> with the mock in place from the start. The cost is inherent to the guarantee.

### `@SpringBootTest` versus a slice

**`@SpringBootTest`** finds your `@SpringBootConfiguration` class (by walking up the
package tree from the test) and loads the **whole application**: every
auto-configuration, every `@Component`, the JPA entity manager, the security filter
chain, the connection pool, everything.

**A slice annotation** loads a deliberately narrow subset. Mechanically, each slice
annotation is a meta-annotation that does three things:

1. `@OverrideAutoConfiguration(enabled = false)` — turn off the normal
   "auto-configure everything" behaviour.
2. `@TypeExcludeFilters(...)` — restrict component scanning to a specific stereotype.
   `@WebMvcTest` scans `@Controller`, `@ControllerAdvice`, `@JsonComponent`,
   `WebMvcConfigurer`, filters and converters. It **does not** pick up `@Service`,
   `@Component` or `@Repository`.
3. A curated list of auto-configurations to switch back on, declared in resource
   files inside `spring-boot-test-autoconfigure`.

That third point is why slices are exact rather than approximate: the list of what
`@DataJpaTest` turns on is a file you can read, not a heuristic.

### The slices you will use on `orderflow`

| Annotation | Loads | Does **not** load | Use it for |
|---|---|---|---|
| `@WebMvcTest(OrderController.class)` | `DispatcherServlet`, MVC config, Jackson, `@ControllerAdvice`, validation, security filter chain, `MockMvc` | services, repositories, JPA, the datasource | routing, status codes, JSON shape, validation messages, `ProblemDetail` (Topic 46), security rules |
| `@DataJpaTest` | JPA, Hibernate, repositories, entities, a transaction per test | controllers, services, security, web | derived queries, mappings, constraints, query counts (Topic 50) |
| `@JdbcTest` / `@DataJdbcTest` | `JdbcTemplate` / Spring Data JDBC | as above | raw SQL paths |
| `@JsonTest` | Jackson only | everything else | serialization of `orderflow` DTOs; very fast |
| `@RestClientTest` | `RestClient`/`RestTemplate` builders + `MockRestServiceServer` | everything else | the `PaymentGateway` HTTP client, with recorded responses (Topic 59, Trap 4) |
| `@SpringBootTest` | everything | — | end-to-end behaviour, transactions, wiring |

**The rule:** use the narrowest thing that can observe the behaviour you are
testing. Not because narrow is virtuous, but because each distinct configuration is
a startup, and a narrow one starts far faster.

---

## Why does it matter?

**1. It is usually the entire explanation for a slow suite.**
Not "Java is slow", not "we have too many tests". A team with 400 tests and a
12-minute suite almost always has 25 to 60 distinct Spring contexts, each costing
3 to 8 seconds of startup that produces no assertions.

**2. It is invisible without one specific log setting.**
Nothing in the default output tells you a context was created. There is no warning,
no timing line attributing 6 seconds to "starting a second context". The DEBUG logger
in the Hands-on section is the instrument, and almost nobody knows it exists — which
is why this is such a reliable interview differentiator.

**3. A `@Transactional` test that rolls back never exercises commit.**
Constraint checks that are deferred to commit, the flush that Hibernate performs at
commit, optimistic-lock version checks (Topic 52) — none of it runs. Your test is
green and the code fails in production on the first real request. This trap is
independent of speed and it costs correctness, which is worse.

---

## Syntax breakdown

### `@SpringBootTest`

```java
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class OrderPlacementIT { }

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderApiIT { }

@SpringBootTest(properties = "orderflow.pricing.volume-discount.enabled=false")
class DiscountDisabledIT { }
```

| Bit | What it means |
|---|---|
| `@SpringBootTest` | Loads the full application context. Finds the config by walking *up* the package tree from the test class looking for `@SpringBootConfiguration`. Put your test in a matching package or it will not find it. |
| `webEnvironment = MOCK` | **The default.** A `WebApplicationContext` with no real server. Use with `MockMvc`. |
| `webEnvironment = RANDOM_PORT` | Starts a real embedded server on a random port. Inject the port with `@LocalServerPort`. Use with `TestRestTemplate` / `RestTestClient` / `WebTestClient`. **This is a different context key from `MOCK`.** |
| `webEnvironment = NONE` | No web environment at all. Fastest full-context option for pure service tests. |
| `properties = "..."` | Inline property overrides. **Part of the cache key, string for string.** Two classes overriding the same property to different values get two contexts. |

### Slices

```java
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;

@WebMvcTest(OrderController.class)
class OrderControllerTest { }

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryTest { }
```

| Bit | What it means |
|---|---|
| `@WebMvcTest(OrderController.class)` | Loads only that controller plus the MVC infrastructure. Without the argument it loads *all* controllers, which is slower and couples the test to unrelated controllers' dependencies. **Always name the controller.** |
| `@DataJpaTest` | JPA layer only. Each test method is wrapped in a transaction that is **rolled back** at the end (see Trap 4). |
| `@AutoConfigureTestDatabase(replace = NONE)` | **Critical.** By default `@DataJpaTest` replaces your `DataSource` with an embedded one — which is H2 if it is on the classpath. `replace = NONE` says "use the datasource I configured", which is how you point it at Testcontainers Postgres (Topic 61). |
| `@AutoConfigureMockMvc` | Adds `MockMvc` to a `@SpringBootTest`. Slices like `@WebMvcTest` already include it. |

### `@MockitoBean` — putting a mock *into the context*

```java
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;

@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;

    @MockitoBean OrderService orderService;      // replaces the bean in the context
    @MockitoSpyBean AuditLog auditLog;           // wraps the real bean
}
```

| Bit | What it means |
|---|---|
| `@MockitoBean` | Creates a Mockito mock (Topic 59) and **registers it in the application context in place of** the real bean. Different from `@Mock`, which creates a mock in your test class and does not touch any context. |
| `@MockitoSpyBean` | Wraps the *real* bean in a Mockito spy. |
| **cost** | Registers a bean-override context customizer, which becomes part of the cache key. **A new context.** |

**`@Mock` vs `@MockitoBean` — the distinction people get wrong:**

| | `@Mock` (Mockito) | `@MockitoBean` (Spring) |
|---|---|---|
| Where the mock lives | a field in your test class | the Spring `ApplicationContext` |
| Who uses it | only code you hand it to | every bean that autowires that type |
| Needs a Spring context | no | yes |
| Cost | microseconds | **a new cached context** |
| Use when | you construct the object under test yourself | a Spring-managed bean deep in the graph must be replaced |

If you can construct the object under test with `new`, use `@Mock` and skip Spring
entirely. That decision is worth several minutes of suite time.

> **`@MockBean` vs `@MockitoBean` — status, honestly.**
> `@MockitoBean` and `@MockitoSpyBean` live in
> `org.springframework.test.context.bean.override.mockito` and arrived in **Spring
> Framework 6.2** as part of a general bean-override mechanism. Boot's older
> `@MockBean` / `@SpyBean` (`org.springframework.boot.test.mock.mockito`) were
> **deprecated in favour of them**. For the target stack — Boot 4.1 / Framework 7.0
> — **`@MockitoBean` is the one to write.**
>
> **What I am not certain of:** whether Boot 4.1 still ships `@MockBean` at all, or
> whether it has been removed outright. Do not take my word either way — check your
> own build:
> ```bash
> mvn dependency:tree -Dincludes=org.springframework.boot:spring-boot-test
> javap -cp "$(find ~/.m2/repository/org/springframework/boot/spring-boot-test -name 'spring-boot-test-*.jar' | head -1)" \
>   org.springframework.boot.test.mock.mockito.MockBean
> ```
> If `javap` reports the class, it still exists (very likely deprecated). If it
> errors with "class not found", it has been removed on your version and
> `@MockitoBean` is your only option. Either way the *caching consequence* described
> in this document is identical for both annotations, because both register a
> bean-override context customizer.

### `MockMvc`

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

mockMvc.perform(post("/api/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"customerId":"c-1001","lines":[{"sku":"SKU-1001","quantity":2}]}
                """))
       .andExpect(status().isCreated())
       .andExpect(header().exists("Location"))
       .andExpect(jsonPath("$.totalMinor").value(3998))
       .andExpect(jsonPath("$.status").value("PENDING"));
```

| Bit | What it means |
|---|---|
| `MockMvc` | Drives the `DispatcherServlet` (Topic 44) **without a network socket or a servlet container**. Fast, and it exercises routing, argument resolvers, message converters, validation, exception handling and the security filter chain. |
| `perform(post(...))` | Builds and dispatches a mock request. |
| `andExpect(status().isCreated())` | A result matcher. Failures print the whole request and response, which is genuinely good diagnostics. |
| `jsonPath("$.totalMinor")` | JSONPath assertion on the body. |
| `andDo(print())` | Dumps the full exchange to the console. Your first move when a `MockMvc` test fails unexpectedly. |
| text block `"""` | Topic 30. Use it for JSON bodies rather than escaped one-liners. |

> **Boot 4 note:** Spring Framework 6.2 added `MockMvcTester`, an AssertJ-flavoured
> API over `MockMvc` (`assertThat(mockMvc.post().uri("/api/orders")...)`), and Boot 4
> adds `RestTestClient` for testing a running server. Both are worth adopting for new
> code. `MockMvc` as shown above still works and is what you will find in every
> existing codebase, so learn it first. Check what your version provides with
> `mvn dependency:tree -Dincludes=org.springframework:spring-test`.

### Security testing

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;
import org.springframework.security.test.context.support.WithMockUser;

@Test
@WithMockUser(username = "c-1001", roles = "CUSTOMER")
void aCustomerCanReadTheirOwnOrder() throws Exception { ... }

// JWT resource server (Topic 57)
mockMvc.perform(get("/api/orders/{id}", orderId)
            .with(jwt().jwt(j -> j.claim("sub", "c-1001").claim("scope", "orders:read"))))
       .andExpect(status().isOk());

// A state-changing request under cookie-session auth needs a CSRF token
mockMvc.perform(post("/api/orders").with(csrf()).content(body))
       .andExpect(status().isCreated());
```

| Bit | What it means |
|---|---|
| `spring-security-test` | A separate test dependency. Without it none of these helpers exist. |
| `@WithMockUser` | Populates the `SecurityContext` before the test. Simple role-based cases. |
| `.with(jwt())` | A request post-processor that installs a `JwtAuthenticationToken`. The right tool for an OAuth2 resource server, because it exercises the actual authorization rules rather than a synthetic user. |
| `.with(csrf())` | Adds a valid CSRF token. If a `POST` unexpectedly returns 403, this is the first thing to check. |

### `@Transactional` and `@DirtiesContext`

```java
@SpringBootTest
@Transactional                       // roll back after each test. Read Trap 4 first.
class OrderServiceIT { }

@SpringBootTest
@DirtiesContext                      // close and evict the context. Read Trap 3 first.
class SomethingDrasticIT { }
```

| Bit | What it means |
|---|---|
| `@Transactional` on a test | `TransactionalTestExecutionListener` starts a transaction before the method and **rolls back** after. Your data never commits. |
| `@Commit` | Overrides that for one method: actually commit. |
| `@DirtiesContext` | Marks the context unusable; Spring closes it and removes it from the cache. The **next** test needing that configuration pays a full startup. |
| `@DirtiesContext(classMode = AFTER_EACH_TEST_METHOD)` | A full context rebuild per test method. Almost never justified. |

---

## Example 1 — minimal

Two test classes over `orderflow`, showing the cost difference for the same
behaviour.

### The expensive way

```java
package com.orderflow.orders;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.test.web.servlet.MockMvc;

@SpringBootTest                 // full app: JPA, security, datasource, everything
@AutoConfigureMockMvc
class OrderControllerFullContextTest {

    @Autowired MockMvc mockMvc;

    @Test
    void rejectsAnOrderWithNoLines() throws Exception {
        mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"customerId\":\"c-1001\",\"lines\":[]}"))
               .andExpect(status().isBadRequest());
    }
}
```

To assert that an empty `lines` array produces a 400, this starts Hibernate, a
connection pool, the entity manager and the full security chain. None of it
participates in the assertion.

### The slice

```java
package com.orderflow.orders;

import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;

    // OrderController's dependency. The slice does not scan @Service, so
    // without this the context fails to start with "no qualifying bean".
    @MockitoBean OrderService orderService;

    @Test
    void rejectsAnOrderWithNoLines() throws Exception {
        mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"customerId\":\"c-1001\",\"lines\":[]}"))
               .andExpect(status().isBadRequest());
    }
}
```

Same assertion, a fraction of the startup. Note the `@MockitoBean`: in a slice it is
**not optional**, because the slice deliberately does not load `@Service` beans. This
is the legitimate use of the annotation — and it still forks the cache, which is fine
here because every `@WebMvcTest(OrderController.class)` class with the same mocks
shares one context.

**How to verify the difference:** the Hands-on section shows you how to count
contexts and read startup timings. Do not take the word "faster" from me — measure
it on your own machine, because the number depends entirely on how much `orderflow`
has grown.

---

## Example 2 — production scenario (on the project spine)

Three slice test classes covering `orderflow`'s web, persistence and security
layers — the spine deliverable for this topic — written so that the whole set uses
as few distinct contexts as possible.

### 2a — the web layer

```java
package com.orderflow.orders.web;

import com.orderflow.orders.OrderService;
import com.orderflow.orders.PlaceOrderCommand;
import com.orderflow.inventory.InsufficientStockException;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(OrderController.class)
@DisplayName("POST /api/orders")
class OrderControllerTest {

    @Autowired MockMvc mockMvc;

    @MockitoBean OrderService orderService;

    private static final String VALID_BODY = """
        {"customerId":"c-1001","idempotencyKey":"key-1",
         "lines":[{"sku":"SKU-1001","quantity":2}]}
        """;

    @Test
    @WithMockUser(roles = "CUSTOMER")
    @DisplayName("returns 201 with a Location header on success")
    void createsAnOrder() throws Exception {
        when(orderService.place(any(PlaceOrderCommand.class)))
            .thenReturn(anOrder("o-9001", 3998L));

        mockMvc.perform(post("/api/orders")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(VALID_BODY))
               .andExpect(status().isCreated())
               .andExpect(header().string("Location", "/api/orders/o-9001"))
               .andExpect(jsonPath("$.totalMinor").value(3998))
               .andExpect(jsonPath("$.status").value("PENDING"));
    }

    @Test
    @WithMockUser(roles = "CUSTOMER")
    @DisplayName("maps InsufficientStockException to a 409 ProblemDetail")
    void mapsDomainExceptionToProblemDetail() throws Exception {
        when(orderService.place(any()))
            .thenThrow(new InsufficientStockException("SKU-1001", 2, 0));

        mockMvc.perform(post("/api/orders")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(VALID_BODY))
               .andExpect(status().isConflict())
               .andExpect(content().contentTypeCompatibleWith("application/problem+json"))
               .andExpect(jsonPath("$.type").value("https://orderflow.example/problems/insufficient-stock"))
               .andExpect(jsonPath("$.sku").value("SKU-1001"))
               // deliberately NOT asserting on $.detail — that is prose (Topic 58, Trap 2)
               .andExpect(jsonPath("$.stackTrace").doesNotExist());
    }

    @Test
    @WithMockUser(roles = "CUSTOMER")
    @DisplayName("returns 400 with field errors for an empty line list")
    void rejectsEmptyLines() throws Exception {
        mockMvc.perform(post("/api/orders")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                        {"customerId":"c-1001","idempotencyKey":"key-1","lines":[]}
                        """))
               .andExpect(status().isBadRequest())
               .andExpect(jsonPath("$.errors[0].field").value("lines"));
    }
}
```

**What this slice is genuinely proving**, none of which a plain unit test could:

- routing and HTTP method binding (Topic 44)
- `@Valid` bean-validation wiring, including that the constraint is actually applied
  at the controller boundary (Topic 45)
- `@ControllerAdvice` exception mapping and the `ProblemDetail` contract (Topic 46)
- that no stack trace leaks into a response body
- Jackson serialization of the response DTO
- that the security filter chain admits an authenticated customer (Topic 56)

**And what it deliberately does not prove:** anything about `OrderService`. That is
mocked, on purpose — the controller's contract is "call the service, translate the
result". Testing the service here would be testing two things at once.

### 2b — the persistence layer

```java
package com.orderflow.orders;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.dao.DataIntegrityViolationException;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)  // real Postgres, Topic 61
@DisplayName("OrderRepository")
class OrderRepositoryTest {

    @Autowired OrderRepository orders;
    @Autowired TestEntityManager em;

    @Test
    @DisplayName("finds orders by customer, newest first")
    void findsByCustomerNewestFirst() {
        em.persist(order("c-1001", INSTANT_MONDAY));
        em.persist(order("c-1001", INSTANT_TUESDAY));
        em.persist(order("c-2002", INSTANT_TUESDAY));
        em.flush();

        assertThat(orders.findByCustomerIdOrderByPlacedAtDesc("c-1001"))
            .extracting(Order::placedAt)
            .containsExactly(INSTANT_TUESDAY, INSTANT_MONDAY);
    }

    @Test
    @DisplayName("rejects a duplicate idempotency key at FLUSH, not at save")
    void duplicateIdempotencyKeyIsRejected() {
        orders.save(order("c-1001", "key-1"));
        orders.save(order("c-1001", "key-1"));

        // The save() calls do not throw. The constraint fires when Hibernate
        // flushes — which is Topic 48's flush timing, and is exactly the thing
        // a mocked repository (Topic 59, Trap 2) cannot show you.
        assertThatThrownBy(() -> em.flush())
            .isInstanceOf(DataIntegrityViolationException.class);
    }

    @Test
    @DisplayName("loading an order with lines issues a bounded number of queries")
    void loadingOrdersDoesNotNPlusOne() {
        // The full query-counting version is Topic 61's exercise; the point here
        // is that this assertion is only possible against a real persistence layer.
    }
}
```

Two things to notice:

1. **`replace = NONE`.** Without it, `@DataJpaTest` swaps in an embedded database.
   If H2 is on the classpath you are now testing against H2 and every conclusion in
   this file is about H2's semantics, not Postgres's. Topic 61 is entirely about why
   that is a false-confidence machine.

2. **`duplicateIdempotencyKeyIsRejected` asserts on `flush()`, not on `save()`.**
   That distinction *is* the test. Anyone who has only ever mocked a repository
   believes `save` throws. It does not.

### 2c — security

```java
package com.orderflow.orders.web;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(OrderController.class)     // SAME slice + SAME mocks as 2a -> SAME cached context
@DisplayName("Order API authorization")
class OrderControllerSecurityTest {

    @Autowired MockMvc mockMvc;

    @MockitoBean OrderService orderService;   // identical override set to 2a: cache hit

    @Test
    @DisplayName("an unauthenticated request is rejected")
    void unauthenticatedIsRejected() throws Exception {
        mockMvc.perform(get("/api/orders/o-9001"))
               .andExpect(status().isUnauthorized());
    }

    @Test
    @DisplayName("a token without the orders:read scope is forbidden")
    void wrongScopeIsForbidden() throws Exception {
        mockMvc.perform(get("/api/orders/o-9001")
                    .with(jwt().jwt(j -> j.claim("sub", "c-1001").claim("scope", "wallet:read"))))
               .andExpect(status().isForbidden());
    }

    @Test
    @DisplayName("a customer cannot read another customer's order")
    void cannotReadAnotherCustomersOrder() throws Exception {
        when(orderService.findFor("c-1001", "o-9001"))
            .thenThrow(new OrderNotVisibleException("o-9001"));

        mockMvc.perform(get("/api/orders/o-9001")
                    .with(jwt().jwt(j -> j.claim("sub", "c-1001").claim("scope", "orders:read"))))
               .andExpect(status().isNotFound());   // 404, not 403 — do not leak existence
    }

    @Test
    @DisplayName("an admin token can read any order")
    void adminCanReadAnyOrder() throws Exception {
        mockMvc.perform(get("/api/orders/o-9001")
                    .with(jwt().jwt(j -> j.claim("sub", "admin-1")
                                          .claim("scope", "orders:read orders:admin"))))
               .andExpect(status().isOk());
    }
}
```

**The context-caching point, made concrete.** `OrderControllerSecurityTest` and
`OrderControllerTest` have:

- the same slice annotation with the same argument
- the same set of overridden beans (`OrderService` only)
- no profiles, no property overrides, no initializers

Therefore **the same `MergedContextConfiguration`, therefore one context for both
classes.** You paid one startup for two test classes and eight tests.

Now imagine someone adds `@MockitoBean WalletService` to the security test because
one test needed it. The override set is now `{OrderService, WalletService}`, the key
differs, and you have two contexts. **One annotation, one extra full startup, every
build, forever.** That is the mechanism behind the interview question, and you can
now see it in ten lines of code.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — one `@MockitoBean` forks the cache and doubles the suite

**Wrong:**

```java
// Twelve test classes, all @SpringBootTest, no overrides. One shared context. Fine.

@SpringBootTest
class WalletTopUpIT {

    @MockitoBean NotificationSender notificationSender;   // "just so it doesn't send emails"

    @Autowired WalletService walletService;
}
```

**Exact symptom:** the CI suite goes from 4 minutes to 8. The PR that caused it
changed no production code and added four tests. Nobody connects the two, because
the timing report attributes the extra time evenly across all test classes — the
second context's startup is charged to whichever class happened to trigger it. Six
months later the suite is 12 minutes and the team's explanation is "we have a lot of
tests now".

**Root cause:** the set of bean overrides is part of the cache key. The other eleven
classes share context A; this one requires context B, which is a complete, separate
application startup — every auto-configuration, every bean, the connection pool, the
entity manager, the security chain.

**Fix, in order of preference:**

1. **Do not use a Spring context at all.** If `WalletService` can be constructed with
   `new WalletService(repo, clock)`, use Topic 59's plain `@Mock` and skip Spring.
   This is the biggest win and it is available more often than people think.
2. **Standardise the override set.** Put the mocks that *every* integration test
   needs into one shared base class or a test slice annotation, so all of them share
   one key:
   ```java
   @SpringBootTest
   @ActiveProfiles("test")
   public abstract class AbstractOrderflowIT {
       @MockitoBean protected NotificationSender notificationSender;
       @MockitoBean protected PaymentGateway paymentGateway;
   }
   ```
   Every subclass now has the identical key. **Two contexts total for the whole
   suite: the plain one and the standard-overrides one.** This is the single most
   effective structural fix available.
3. **Replace the mock with a test-profile bean.** A `@Profile("test")
   NoOpNotificationSender` is part of the *configuration*, so it does not fork
   anything; every test on the `test` profile shares one context.
4. If you genuinely need a per-class override, accept the cost knowingly and write a
   comment saying so.

**Prove it in 30 seconds:** the Hands-on section's DEBUG logger prints the cache
size. Add the annotation, run, watch `size` go from 1 to 2.

---

### Trap 2 — `@SpringBootTest` where a slice would do

**Wrong:**

```java
@SpringBootTest
@AutoConfigureMockMvc
class ProductControllerTest { }         // asserts on JSON field names

@SpringBootTest
class OrderDtoMappingTest { }           // asserts a mapper converts an entity to a DTO

@SpringBootTest
class SkuValidatorTest { }              // asserts a regex
```

**Exact symptom:** `mvn test` takes 12 minutes for 380 tests, and
`target/surefire-reports/*.txt` shows most classes with an elapsed time between 3 and
9 seconds — for tests whose assertions take microseconds. The wall-clock time is
almost entirely context startup and per-class overhead.

**Root cause:** `@SpringBootTest` loads the entire application. The regex test needed
zero Spring beans; the DTO mapping test needed Jackson; the controller test needed
MVC. Each got a datasource, an entity manager, a connection pool and a security
filter chain instead.

**Fix — a decision table you can apply in review:**

| What am I asserting on? | Use |
|---|---|
| A pure function, a regex, a calculation | **No Spring at all.** Plain JUnit (Topic 58). |
| JSON serialization of a DTO | `@JsonTest` |
| Routing, status codes, validation, error mapping, security rules | `@WebMvcTest(TheController.class)` |
| A query, a mapping, a constraint, a query count | `@DataJpaTest` + real Postgres (Topic 61) |
| An outbound HTTP client | `@RestClientTest` |
| Wiring, transactions, the whole flow end to end | `@SpringBootTest` — and have few of them |

The healthy shape is a small number of `@SpringBootTest` classes (they can all share
one context), a moderate number of slice tests, and a large majority of tests with no
Spring at all.

---

### Trap 3 — `@DirtiesContext` sprinkled defensively

**Wrong:**

```java
@SpringBootTest
@DirtiesContext            // "to be safe" / "the other test was interfering"
class InventoryReservationIT { }
```

**Exact symptom:** suite time grows in steps that correlate with nothing in the code.
Adding an *unrelated* test class makes the suite slower than adding two. With the
cache DEBUG logger on, you see `hitCount` barely rising while `missCount` climbs
steadily, and contexts being closed and rebuilt repeatedly.

**Root cause:** `@DirtiesContext` closes the context and evicts it from the cache. The
next test class needing that configuration pays a full startup. If several classes
are annotated, you can end up rebuilding the same context five or six times in one
run. `@DirtiesContext(classMode = AFTER_EACH_TEST_METHOD)` rebuilds it *per test
method* — a genuinely catastrophic setting that people apply casually.

Worse, it is usually treating a symptom. "The other test was interfering" means
something in the context has mutable state that survives across tests. Evicting the
context hides that; it does not fix it.

**Fix:**

1. **Find the actual leaked state.** It is almost always one of: a cache
   (`@Cacheable`, Topic 41/51), a static field, a scheduler that keeps running, an
   in-memory collection in a singleton bean, or rows left in the database.
2. **Clean it explicitly.** `@BeforeEach` clearing the cache, `@Sql` scripts
   resetting tables, `TRUNCATE` between tests — all cheap and targeted.
3. **Reserve `@DirtiesContext` for cases where the context is genuinely unusable
   afterwards** — you replaced a bean at run time, closed a connection pool, or
   mutated a `@ConfigurationProperties` object. Those are rare and you should be able
   to name the mechanism.
4. If you must use it, **`@DirtiesContext(methodMode = AFTER_METHOD)` on one method**
   is far cheaper than class-level, and class-level is far cheaper than
   `AFTER_EACH_TEST_METHOD`.

---

### Trap 4 — a `@Transactional` test rolls back, so the commit path is never tested

**Wrong:**

```java
@SpringBootTest
@Transactional          // the default advice everyone gives: "so tests clean up"
class OrderPlacementIT {

    @Test
    void placesAnOrder() {
        Order order = orderService.place(command);
        assertThat(order.status()).isEqualTo(OrderStatus.PENDING);
    }
}
```

**Exact symptom:** green. Then, on the first production request, a
`DataIntegrityViolationException` from a deferred constraint, or an
`OptimisticLockException` (Topic 52), or a `ConstraintViolationException` at flush
that no test ever saw. The bug is 100% reproducible in production and 0%
reproducible in the suite.

**Root cause:** `@Transactional` on a test class makes
`TransactionalTestExecutionListener` open a transaction before each test and **roll
it back afterwards**. Consequences, in increasing order of nastiness:

- **Nothing commits.** Any check that happens at COMMIT — deferred constraints,
  `SET CONSTRAINTS DEFERRED`, some trigger behaviour — never runs.
- **The flush may never happen either.** If your test only calls `save()` and then
  asserts, Hibernate may not have issued the INSERT at all (Topic 48). A unique
  constraint cannot fire against a statement that was never sent.
- **The test and the code under test share one transaction.** So `@Transactional`
  boundaries *inside* your service are joined into the test's transaction rather than
  being real boundaries. A `REQUIRES_NEW` behaves differently. A rollback that your
  code expects to be partial is not.
- **The persistence context is shared**, so an entity you "saved" is returned from
  the first-level cache on the next `findById` — even if the SQL would have returned
  something different (Topic 48/51).

**Fix — pick per test, deliberately:**

1. **Drop `@Transactional` and clean up explicitly.** `@Sql(scripts =
   "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)`, or a `TRUNCATE` in
   `@AfterEach`, or a Testcontainers database that is recreated. This is the honest
   default for anything asserting on persistence behaviour.
2. **Force a flush inside the test** when you want to observe a constraint:
   `entityManager.flush()` — as in Example 2b. This catches flush-time violations
   even inside a rolled-back transaction.
3. **Use `@Commit` on the specific test** that must exercise the commit path, and
   clean up after it.
4. **Use `TransactionTemplate` explicitly** when you need multiple real transactions
   in one test — which is what any concurrency or optimistic-locking test requires:
   ```java
   transactionTemplate.executeWithoutResult(status -> orderService.place(command));
   // now the transaction is genuinely committed; assert on a fresh read
   ```

> **This trap interacts with Topic 54.** The default rollback rule is "unchecked
> exceptions only" — a checked exception commits. A `@Transactional` test can never
> observe that, because the test's own rollback masks whether your service's
> transaction would have committed. If you want to prove Topic 54's checked-exception
> behaviour, the test must not be `@Transactional`.

---

### Trap 5 — inline property overrides fragmenting the cache

**Wrong:**

```java
@SpringBootTest(properties = "orderflow.pricing.tier-threshold=10")
class TierTenIT { }

@SpringBootTest(properties = "orderflow.pricing.tier-threshold=50")
class TierFiftyIT { }

@SpringBootTest(properties = "orderflow.pricing.tier-threshold = 10")   // note the spaces
class TierTenAgainIT { }
```

**Exact symptom:** three contexts for three classes, including two that were
*intended* to be identical. Suite time grows by one full startup for each new
threshold anyone wants to test.

**Root cause:** inline properties are part of the cache key **as strings**. Not as
parsed key-value pairs — as the literal strings you wrote. `"a=10"` and `"a = 10"`
are different keys. And even semantically distinct values each get their own context.

**Fix:**

1. **Make the value injectable at run time instead of at context-build time.** If
   the threshold is bound via `@ConfigurationProperties` to a mutable object, the
   test can set it in `@BeforeEach` and no context is forked. If it is bound to a
   record or a final field, it cannot — which is a real trade-off worth knowing about
   when you design your config classes (Topic 43).
2. **Use one profile with one set of test properties** for the whole suite, in
   `src/test/resources/application-test.yml`, and `@ActiveProfiles("test")`
   everywhere. One key, one context.
3. **Test the threshold logic without Spring.** `DiscountPolicy.basisPointsFor` from
   Topic 58 is a static method taking the threshold as a parameter — parameterize the
   *function*, not the *context*.

That last point generalises: **if you find yourself varying configuration to test
behaviour, the behaviour probably wants to be a parameter rather than a property.**

---

## Hands-on proof

Every command is one **you** run. I have no JVM, no Spring, and no test runner, so
nothing below is captured output. What follows is the exact instrument, what to look
for, and how to read each outcome.

### Proof 1 — the context cache, made visible (the single best instrument)

Create `src/test/resources/logback-test.xml`, or simply add to
`src/test/resources/application.yml`:

```yaml
logging:
  level:
    org.springframework.test.context.cache: DEBUG
```

Then:

```bash
mvn test
```

**What to look for:** lines from `DefaultContextCache` reporting cache statistics
after each test class. The shape of that line — **shown as an illustration of the
format, not as captured output** — is:

```
DEBUG o.s.t.c.cache.DefaultContextCache : Spring test ApplicationContext cache statistics:
  [DefaultContextCache@1a2b3c size = 4, maxSize = 32, parentContextCount = 0,
   hitCount = 37, missCount = 4]
```

Read the numbers like this:

| What you see | What it means |
|---|---|
| `size = 1` and a high `hitCount` at the end of the run | The ideal. Every test class shares one context. One startup for the whole suite. |
| `size = 2` or `3` | Normal and healthy: typically the plain context, the standard-overrides context, and maybe a web-environment one. |
| `size` in the teens or twenties | **This is your suite-time problem.** Each unit of `size` is one full application startup. Multiply by your app's startup time to get the wasted seconds. |
| `size = 32` (equal to `maxSize`) | You have hit the LRU limit and contexts are being **evicted and rebuilt**. Your `missCount` will be much higher than `size`, and the same configuration is starting repeatedly. This is the worst case. |
| `missCount` far exceeding `size` | Eviction (see above) or `@DirtiesContext`. Both mean the same context is being built more than once. |
| No such log lines at all | The logger name is wrong, or your logging config is not being picked up in tests. Try `-Dlogging.level.org.springframework.test.context.cache=DEBUG` on the command line instead. |

**This one setting answers the interview question.** "Your suite takes 12 minutes —
where do you look first?" The answer is: turn this on and read `size`.

### Proof 2 — find *which* class forks the context

```bash
mvn test -Dlogging.level.org.springframework.test.context.cache=DEBUG \
         -Dsurefire.useFile=false 2>&1 | grep -n "size = "
```

**What to look for:** the point in the run where `size` increments, and which test
class was executing immediately before it.

| What you see | What it means |
|---|---|
| `size` increments right after a class with a `@MockitoBean` | Confirmed: that override set forks the context. Apply Trap 1's fixes. |
| `size` increments after a class with `@ActiveProfiles` or `properties = ...` | The profile set or the property string is a distinct key. Trap 5. |
| `size` increments after a `@WebMvcTest` following `@SpringBootTest` classes | Expected — a slice uses a different context loader and a different configuration. Slices *always* have their own contexts, and that is fine because they are cheap. |
| `size` stays flat but time is still bad | Context startup is not your problem. Look at per-class elapsed times in `target/surefire-reports/`, and at whether tests are doing real I/O. |

### Proof 3 — force eviction and see the cost

```bash
mvn test -Dspring.test.context.cache.maxSize=1 \
         -Dlogging.level.org.springframework.test.context.cache=DEBUG
```

**What to look for:** the total suite time compared with a normal run, and the
`missCount`.

| What you see | What it means |
|---|---|
| Suite time increases dramatically; `missCount` roughly equals the number of test classes | You have just measured what a full startup costs, multiplied by your class count. That number is your budget for how much context fragmentation costs. |
| Barely any change | Either you already have very few Spring tests, or almost every class already had its own context — in which case you had already lost the caching benefit entirely. |

Then run it the other way to confirm the ceiling matters:

```bash
mvn test -Dspring.test.context.cache.maxSize=64 \
         -Dlogging.level.org.springframework.test.context.cache=DEBUG
```

| What you see | What it means |
|---|---|
| Suite gets noticeably faster and `missCount` drops | You were hitting the default limit of 32 and thrashing. Raising the limit is a **painkiller, not a cure** — you still have far too many distinct configurations, and now you are also holding 64 application contexts in memory at once, which will eventually cause an OOM in CI. |
| No change | You were not evicting. Good; leave `maxSize` at its default. |

### Proof 4 — per-class timings

```bash
mvn surefire:test -Dsurefire.reportFormat=plain
grep -H "Time elapsed" target/surefire-reports/*.txt | sort -t: -k3 -rn | head -20
```

**What to look for:** the classes with the largest elapsed time, and whether their
test *count* justifies it.

| What you see | What it means |
|---|---|
| A class with 2 tests and 8 seconds elapsed | Almost all of that is context startup charged to the first test. Candidate for a slice, or for no Spring at all. |
| A class with 40 tests and 8 seconds | Fine. The startup is amortised. This is what you want. |
| Many classes at 3–6 seconds each | Context fragmentation. Cross-reference with Proof 2. |
| One class at 60 seconds | Not a context problem — something in it is doing real I/O, sleeping, or waiting on a timeout. Look at the test, not the config. |

### Proof 5 — see what a slice actually loaded

```bash
mvn -Dtest=OrderControllerTest test -Dsurefire.useFile=false \
    -Dlogging.level.org.springframework.boot.autoconfigure=DEBUG
```

Or run any Spring test with `--debug` semantics by adding
`-Dspring-boot.run.arguments=--debug`; for tests the reliable route is the property
above, which produces the condition-evaluation report from Topic 42.

**What to look for:** the "Positive matches" and "Negative matches" sections.

| What you see | What it means |
|---|---|
| `DataSourceAutoConfiguration` in **Negative matches** under `@WebMvcTest` | Correct. The slice excluded it. No connection pool was created. |
| `DataSourceAutoConfiguration` in **Positive matches** under `@WebMvcTest` | Something re-enabled it — usually a `@TestConfiguration` or an `@Import` that drags it in. That is why your "fast" slice is not fast. |
| A `@Service` bean you did not expect | Something is `@Import`ing a configuration class, or a `@Component` is being picked up by a custom filter. Slices only exclude by *type filter*; an explicit `@Import` bypasses that. |

### Proof 6 — prove the rollback trap

Write two versions of the same test — one `@Transactional`, one not — that save two
orders with the same idempotency key.

```bash
mvn -Dtest=IdempotencyRollbackProofTest test
```

**What to look for:** whether the constraint violation is observed.

| What you see | What it means |
|---|---|
| The `@Transactional` version passes and the non-transactional version fails with `DataIntegrityViolationException` | Trap 4 reproduced exactly. The transactional test never flushed, so the constraint never ran. |
| Both fail | Your code flushes explicitly somewhere, or `FlushMode` is forcing it (Topic 48). Good — but do not rely on it; make the flush explicit in the test. |
| Both pass | The unique constraint does not exist in your schema. Check your Flyway migration. This is itself a valuable finding. |

---

## Practice exercises

### 1 — Easy: count your contexts and cut one

**Part A.** Turn on `logging.level.org.springframework.test.context.cache=DEBUG` and
run the whole `orderflow` suite. Record the final `size`, `hitCount` and `missCount`.

**Part B.** For each unit of `size`, identify which test class or classes are
responsible and *what specifically* about their configuration differs — name the
part of the cache key (profiles, overrides, inline properties, slice type).

**Part C.** Eliminate exactly one context. Options: move a `@MockitoBean` into a
shared base class, replace a mock with a `@Profile("test")` bean, or convert a
`@SpringBootTest` into a plain JUnit test. Re-run and confirm `size` dropped by one.
Record the suite-time difference honestly — if it is 400 ms, say 400 ms.

**Part D.** Write one paragraph: at what suite size does this stop being worth doing
manually, and what would you automate instead?

### 2 — Medium: the three-slice suite, combining earlier topics

Build the spine deliverable properly.

**Part A — web.** `@WebMvcTest(OrderController.class)` covering: 201 with a
`Location` header, 400 with field errors from Bean Validation (Topic 45), 409 mapped
from `InsufficientStockException` to a `ProblemDetail` (Topics 09 and 46), and a
check that no stack trace appears in any error body. Assert on the `type` URI and on
structured fields — **never** on the `detail` prose (Topic 58, Trap 2).

**Part B — persistence.** `@DataJpaTest` with `replace = NONE` covering: a derived
query with ordering (Topic 47), a unique-constraint violation observed at `flush()`
(Topic 48), and a `@Version` optimistic-lock failure on `Inventory` (Topic 52) using
two separate `EntityManager`s.

**Part C — security.** A second `@WebMvcTest` class with the **identical override
set** as Part A, covering: unauthenticated 401, wrong-scope 403, cross-customer
access returning 404 rather than 403, and admin access succeeding (Topic 57).

**Part D — the caching check.** Run with the cache logger on. Parts A and C **must**
share one context. If they do not, find the difference in their keys and fix it. Then
deliberately break it: add one extra `@MockitoBean` to Part C, re-run, observe `size`
increment, and record the time cost. Revert.

### 3 — Hard: production simulation — cut the suite in half, with evidence

Your `orderflow` suite has grown. Treat this as a real optimisation task with a
written result.

**Part A — measure.** Record: total `mvn test` wall time, final cache `size`,
`hitCount`, `missCount`, and the ten slowest test classes with their test counts.
This is your baseline. **Do not skip it** — an optimisation without a baseline is a
story.

**Part B — classify.** Put every Spring-using test class into one of four buckets:

1. needs no Spring at all (pure logic that got a context by accident)
2. needs a slice
3. needs a full context and can share the standard configuration
4. genuinely needs its own configuration — and you can name why

**Part C — restructure.** Introduce `AbstractOrderflowIT` with a single standardised
override set. Move bucket-3 classes onto it. Convert bucket-1 classes to plain JUnit
(Topic 58) with fakes and `@Mock` (Topic 59). Convert bucket-2 to slices. For every
class remaining in bucket 4, add a code comment stating the specific cache-key
difference and why it is unavoidable.

**Part D — a test that keeps you honest.** Add a check that fails the build if the
context count exceeds a threshold. Two ways to do it — pick one and justify:

- an ArchUnit rule forbidding `@SpringBootTest` outside a package or a base class
- a CI step that greps the cache-statistics output for `size = N` and fails above a
  ceiling

**Part E — the write-up.** Report the before/after numbers. Then answer:

- Which change gave the largest reduction per unit of effort?
- Which tests did you *lose confidence in* by converting them, and is that trade
  acceptable? Name at least one.
- One of your `@Transactional` integration tests is now not `@Transactional`. What
  did that reveal? (If it revealed nothing, say so — a negative result honestly
  reported is worth more than an invented one.)
- What in this suite is still lying to you because it runs against something other
  than production Postgres? That question is Topic 61.

---

## Interview questions

### Q1 — "Your test suite takes 12 minutes. Where do you look first?"

**Mid-level answer:** "I'd look for slow tests — probably the integration tests or
anything hitting a database — and try to parallelise the build or mock more things
out."

**Senior answer:** "First I'd turn on
`logging.level.org.springframework.test.context.cache=DEBUG` and read the cache
statistics line. It reports the number of distinct Spring contexts, plus hit and miss
counts. If `size` is 20-something, that is 20 full application startups, and it is
almost always the entire explanation — the test context is cached by its **full
configuration**: the configuration classes, the active profiles, the inline property
overrides string-for-string, the context initializers and the set of overridden
beans. One `@MockitoBean` in one class produces a different key, so Spring builds a
second complete context rather than swapping the bean, because it cannot safely swap
a bean other beans have already been injected with.

If `size` is near 32 I'd also check for eviction — the default cache max is 32, so
past that you thrash and rebuild the same contexts. Then I'd cross-reference the
Surefire per-class elapsed times: a class with two tests and eight seconds is paying
startup, a class with forty tests and eight seconds is fine.

The fixes in order: delete Spring from tests that never needed it, standardise the
override set behind one abstract base class so every integration test shares one key,
move mocks into `@Profile("test")` beans where possible, and use slices where a
slice can observe the behaviour. Parallelising comes last — it hides the waste rather
than removing it, and parallel Spring tests multiply the memory held by cached
contexts."

**What separates them:** naming the exact log property, knowing the cache key's
components, explaining *why* Spring rebuilds rather than swapping, and ordering the
fixes by leverage rather than reaching for parallelism.

**Interviewer's follow-up:** "You raised `spring.test.context.cache.maxSize` and it
got faster. Ship it?" The answer: no — that is a painkiller. You still have too many
configurations and now you hold more contexts in memory, which becomes an OOM in CI.
Use it as a *diagnostic* to confirm you were evicting.

---

### Q2 — "What exactly is the Spring test context cache keyed on?"

**Mid-level answer:** "The test configuration — like which classes you load and
whether you use `@SpringBootTest` or a slice."

**Senior answer:** "It is keyed on a `MergedContextConfiguration`, whose equality
covers: the configuration classes and locations, the active profile set, the property
source locations, the inline properties as literal strings, the context initializer
classes, the context customizers, the context loader and the parent context. The two
that surprise people are the last two of those first six. Inline properties are
compared as strings, so `properties = "a=10"` and `properties = "a = 10"` are
different contexts. And bean overrides — `@MockitoBean`, `@MockitoSpyBean`, and Boot's
older `@MockBean` — register a context customizer, so the *set of mocked beans* is
part of the key.

The practical consequence is that context reuse is an emergent property of how
uniformly your test classes are configured, and it is worth designing for
deliberately: one abstract base class with one override set gives you one context for
the whole integration suite."

**What separates them:** listing the key's actual components, and knowing the
string-comparison detail on inline properties — which is the kind of thing you only
know from having debugged it.

**Interviewer's follow-up:** "How would you verify that two test classes share a
context?" The cache DEBUG logger, and checking that `size` does not increment between
them.

---

### Q3 — "When would you use `@SpringBootTest` rather than a slice?"

**Mid-level answer:** "`@SpringBootTest` when you want an integration test and slices
when you want to test one layer. Slices are faster."

**Senior answer:** "I pick the narrowest thing that can *observe the behaviour I am
asserting on*. For routing, validation, error mapping and security rules, a
`@WebMvcTest` naming the specific controller sees all of that and skips JPA and the
datasource. For a query, a mapping or a constraint, `@DataJpaTest` with
`replace = NONE` so it hits real Postgres. For a regex or a calculation, no Spring at
all.

`@SpringBootTest` earns its cost when the behaviour under test **is** the wiring: a
transaction spanning several services, `@Transactional` propagation semantics —
which need the real proxy, so they cannot be tested with a plain Mockito test —
auto-configuration back-off, or an end-to-end flow. I'd keep those few and make them
all share one context configuration.

There is one trap I'd call out specifically: `@DataJpaTest` replaces your DataSource
with an embedded database by default. If H2 is on the classpath, you have silently
started testing H2's SQL semantics instead of Postgres's, which is a different topic
and a serious one."

**What separates them:** the rule ("narrowest that can observe the behaviour"),
naming `@Transactional` proxies as a genuine reason to need a full context, and
volunteering the `@AutoConfigureTestDatabase` trap unprompted.

**Interviewer's follow-up:** "Why can't you test `@Transactional` behaviour with a
plain Mockito test?" Because `@Transactional` is a Spring proxy (Topic 40). With
`new OrderService(...)` there is no proxy, so there is no transaction — the method
runs, but none of the transactional semantics apply.

---

### Q4 — "Someone adds `@Transactional` to every integration test class 'so the database stays clean'. Your review comment?"

**Mid-level answer:** "That's fine, it's the standard way to clean up after tests."

**Senior answer:** "It is the standard advice, and it silently removes several
classes of test. The test transaction rolls back, so nothing commits: deferred
constraints never fire, and if the test only calls `save()` without flushing,
Hibernate may never even issue the INSERT — so a unique-constraint violation is
literally impossible to observe. The test also *shares* its transaction with the code
under test, so inner `@Transactional` boundaries are joined rather than being real
boundaries, `REQUIRES_NEW` behaves differently, and the shared persistence context
returns entities from the first-level cache rather than from a real read.

So my comment would be: keep it for tests that only need arrange-act-assert on
queries, but for anything asserting on persistence *behaviour* — constraints,
locking, rollback semantics — drop it and clean up with `@Sql` or a truncate, or use
`TransactionTemplate` to get real committed transactions. And add an explicit
`entityManager.flush()` in any test that expects a constraint to fire.

The specific one I'd want covered without `@Transactional` is Topic 54's default:
rollback is on unchecked exceptions only, so a checked exception commits. A
transactional test can never show you that, because its own rollback masks it."

**What separates them:** listing four distinct consequences of the shared
transaction, and naming the specific production bug class it hides.

**Interviewer's follow-up:** "How do you keep the database clean without it?"
`@Sql` cleanup scripts, truncation in `@AfterEach`, or a fresh Testcontainers
database — which is Topic 61, and costs seconds.

---

### Q5 — "What's the difference between `@Mock` and `@MockitoBean`?"

**Mid-level answer:** "`@Mock` is Mockito's and `@MockitoBean` is Spring's. They both
create mocks; `@MockitoBean` works with Spring."

**Senior answer:** "`@Mock` creates a mock object in your test class and does nothing
else — you have to hand it to the object under test yourself, normally through the
constructor. It costs microseconds and needs no Spring at all. `@MockitoBean`
replaces a bean **inside the application context**, so every Spring-managed bean that
autowires that type gets the mock. That is the right tool when the thing you need to
replace is buried deep in a bean graph you are not constructing yourself.

The cost difference is the important part: `@MockitoBean` registers a bean-override
context customizer, which is part of the context cache key, so a test class with a
different override set gets a whole new application context and a full startup. So
my default is: if I can construct the object under test with `new`, I use `@Mock` and
skip Spring; I reach for `@MockitoBean` only inside slices, where the slice
deliberately did not load the collaborator, or when the bean is genuinely
unreachable otherwise. And I standardise the override set across integration tests so
they share one context.

On naming: `@MockitoBean` is the current annotation, from Spring Framework 6.2, in
`org.springframework.test.context.bean.override.mockito`. Boot's older `@MockBean`
was deprecated in favour of it. I'd check the exact availability against the version
in the project rather than assume."

**What separates them:** the *cost* answer, the decision rule, and the honest note
about checking the annotation's status rather than asserting it.

**Interviewer's follow-up:** "Give me a case where `@MockitoBean` is unavoidable."
A `@WebMvcTest` where the controller's `@Service` is not scanned by the slice; or
replacing a bean that a Spring-managed proxy already wraps.

---

## Mental model checkpoint

Reason these out without looking anything up.

1. Spring cannot swap a bean in a live cached context, so it builds a new one. Name
   three specific things that have already happened to the beans in a running context
   that make a safe swap impossible.

2. Inline properties are compared as literal strings, not as parsed key-value pairs.
   Why might the Spring team have chosen that? What would it cost to compare them
   semantically, and would it be worth it?

3. The default cache `maxSize` is 32. Why a bound at all — what fails if the cache is
   unbounded? And what does *hitting* that bound do to your suite time?

4. A `@Transactional` test shares its transaction with the code under test. List four
   distinct kinds of bug that this makes undetectable, and for each, say which topic
   in Phase 5 taught you the underlying mechanism.

5. Slices work by excluding component types and re-enabling a curated list of
   auto-configurations. Given that, predict what happens if a `@WebMvcTest` class also
   carries `@Import(PersistenceConfig.class)` — and whether the slice's exclusions
   protect you.

6. Your suite has 400 tests, `size = 1`, and takes 11 minutes. Context caching is
   working perfectly. Where is the time going, and what is your next instrument?

7. `@MockitoBean` is convenient and expensive; a `@Profile("test")` bean is
   inconvenient and cheap. Construct the argument for making the expensive one the
   team default anyway, then say why you would still not.

---

## Quick reference card

### Annotations

```java
@SpringBootTest                                   // full context
@SpringBootTest(webEnvironment = RANDOM_PORT)     // real server; DIFFERENT cache key
@SpringBootTest(properties = "a=b")               // inline props; part of the key, as a string
@ActiveProfiles("test")                           // profile SET is part of the key
@TestPropertySource(locations = "/test.properties")

@WebMvcTest(OrderController.class)                // MVC + that controller only
@DataJpaTest                                      // JPA only; @Transactional + rollback
@AutoConfigureTestDatabase(replace = Replace.NONE)// use MY datasource (Topic 61)
@JsonTest                                         // Jackson only
@RestClientTest(PaymentGatewayClient.class)       // outbound HTTP + MockRestServiceServer
@AutoConfigureMockMvc                             // add MockMvc to @SpringBootTest

@MockitoBean    OrderService orderService;        // mock IN the context; forks the cache
@MockitoSpyBean AuditLog auditLog;                // spy on the real bean; forks the cache
@TestConfiguration                                // nested config; also part of the key

@Transactional                                    // rolls back. Read Trap 4.
@Commit                                           // this one commits
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
@DirtiesContext                                   // evicts the context. Read Trap 3.
```

### The cache key, in one list

```
configuration classes / locations
+ active profiles (as a set)
+ property source locations
+ inline properties (as literal strings)
+ context initializer classes
+ context customizers  <-- @MockitoBean / @MockBean / @DynamicPropertySource live here
+ context loader
+ parent context
```

Any difference in any of these = a different context = a full startup.

### MockMvc

```java
mockMvc.perform(post("/api/orders")
            .with(csrf())
            .with(jwt().jwt(j -> j.claim("scope", "orders:write")))
            .contentType(MediaType.APPLICATION_JSON)
            .content(body))
       .andDo(print())                                   // dump the exchange when confused
       .andExpect(status().isCreated())
       .andExpect(header().exists("Location"))
       .andExpect(jsonPath("$.totalMinor").value(3998))
       .andExpect(jsonPath("$.stackTrace").doesNotExist());
```

### Diagnostic commands

```bash
# THE instrument for context caching
mvn test -Dlogging.level.org.springframework.test.context.cache=DEBUG

# Which class forked it
mvn test -Dlogging.level.org.springframework.test.context.cache=DEBUG \
         -Dsurefire.useFile=false 2>&1 | grep -n "size = "

# Measure what a startup costs (diagnostic only, never a fix)
mvn test -Dspring.test.context.cache.maxSize=1

# Were you evicting?
mvn test -Dspring.test.context.cache.maxSize=64

# Per-class timings
mvn surefire:test -Dsurefire.reportFormat=plain
grep -H "Time elapsed" target/surefire-reports/*.txt | sort -t: -k3 -rn | head

# What did the slice actually load?
mvn -Dtest=OrderControllerTest test \
    -Dlogging.level.org.springframework.boot.autoconfigure=DEBUG

# Is @MockBean still present on your version?
mvn dependency:tree -Dincludes=org.springframework.boot:spring-boot-test
```

### Gotchas checklist

- [ ] The context cache key includes the **set of mocked beans**. One `@MockitoBean` = one new context.
- [ ] Inline `properties = "..."` are compared **as strings**. Whitespace matters.
- [ ] The profile **set** is part of the key. `{test}` and `{test,kafka}` differ.
- [ ] `@DataJpaTest` replaces your DataSource unless you add `replace = NONE`.
- [ ] `@WebMvcTest` without an argument loads **all** controllers.
- [ ] `@WebMvcTest` does not scan `@Service`/`@Component`/`@Repository`.
- [ ] `@Transactional` on a test rolls back — the commit path is never exercised.
- [ ] `@DirtiesContext` evicts. Use it only when you can name what made the context unusable.
- [ ] Default cache `maxSize` is 32. Above that you thrash.
- [ ] `@Mock` needs no context. `@MockitoBean` costs a context. Prefer `@Mock`.
- [ ] `@SpringBootTest` finds config by walking **up** the package tree from the test.
- [ ] A 403 on a `POST` under cookie auth is usually a missing `.with(csrf())`.

---

## [BOOT 3.x DELTA]

1. **`@MockBean` / `@SpyBean` versus `@MockitoBean` / `@MockitoSpyBean`.** On Boot
   3.x up to 3.3 you will find `@MockBean` (`org.springframework.boot.test.mock.mockito`)
   everywhere; it is the annotation the whole ecosystem's documentation was written
   against. `@MockitoBean` and `@MockitoSpyBean`
   (`org.springframework.test.context.bean.override.mockito`) arrived with Spring
   Framework 6.2 and Boot deprecated its own versions in favour of them.
   **For the target stack write `@MockitoBean`.** As stated in the Syntax section, I
   am not certain whether Boot 4.1 still ships `@MockBean` or has removed it —
   verify with the `javap` command given there rather than trusting either answer.
   The context-caching consequence is identical for both: both register a bean-override
   context customizer and therefore fork the cache.

2. **`MockMvcTester` and `RestTestClient`.** Framework 6.2 added `MockMvcTester`, an
   AssertJ-style API over `MockMvc`; Boot 4 adds `RestTestClient` for a running
   server. On 3.x you use `MockMvc` and `TestRestTemplate` / `WebTestClient`. Nothing
   about context caching changes.

3. **Slice annotations are stable across 3.x and 4.x** — `@WebMvcTest`,
   `@DataJpaTest`, `@JsonTest`, `@RestClientTest` all behave the same way, as does
   `@AutoConfigureTestDatabase(replace = NONE)`. What moved in Boot 4 is which *jar*
   some auto-configurations live in (the modularisation from the master plan), which
   can surface as a missing class after a migration rather than as a behaviour change.

4. **Spring Security 7 changes filter-chain configuration idioms** (Topics 56–57), so
   a 3.x security test's *production* configuration will differ. The test-side
   helpers — `@WithMockUser`, `jwt()`, `csrf()` from `spring-security-test` — are the
   same shape on both lines.

5. **Everything about the cache key is unchanged between 3.x and 4.x.** The
   `MergedContextConfiguration` mechanism, the default `maxSize` of 32, the
   `org.springframework.test.context.cache` DEBUG logger — all identical. Which means
   the diagnostic in this document works on any Spring Boot codebase you will meet.

---

## When would I use this at work?

**1. Week one at a new job, with a slow suite.**
You turn on the cache logger, read `size`, and within an hour you have a specific,
measurable finding — "we have 26 Spring contexts; 19 of them differ only by which
beans are mocked" — plus a concrete proposal. This is one of the highest
credibility-per-hour moves available to a new senior engineer, and almost nobody
does it, because the instrument is not well known.

**2. Reviewing a PR that adds `@SpringBootTest` to a test of a pure calculation.**
Your comment is a question: "what does the Spring context contribute to these
assertions?" Usually the answer is nothing, and the test becomes a plain JUnit test
that runs in a millisecond. Applied consistently over a year, this is the difference
between a 3-minute and a 15-minute suite.

**3. Debugging a bug that "the tests should have caught".**
You look at the test, see `@Transactional` on the class, and know immediately that
the commit path was never exercised — so a deferred constraint, an optimistic lock
failure, or a flush-time violation was invisible to the entire suite. That is a
five-second diagnosis that otherwise takes half a day.

---

## Connected topics

**Prerequisites:**
- **35 — `ApplicationContext`**: the thing being cached. The two-phase startup is why
  a bean cannot be swapped in a live context.
- **37 — Bean lifecycle**: `@PostConstruct` has already run in a cached context.
- **39 — Constructor injection**: makes `@Mock` viable and `@MockitoBean` avoidable.
- **40 — Proxying**: `@Transactional` needs the proxy, so a plain Mockito test cannot
  exercise it. This is the strongest single reason `@SpringBootTest` sometimes earns
  its cost.
- **42 — Auto-configuration**: slices work by disabling it and re-enabling a curated
  list. The condition-evaluation report is how you see what a slice loaded.
- **43 — Profiles and properties**: the profile set and inline properties are part of
  the cache key.
- **44/45/46 — Controllers, validation, ProblemDetail**: what `@WebMvcTest` proves.
- **47–53 — Spring Data and Hibernate**: what `@DataJpaTest` proves, and what
  `@Transactional` rollback hides.
- **54 — `@Transactional`**: Trap 4 is a direct consequence of that topic.
- **56/57 — Security**: what the security slice tests exercise.
- **58 — JUnit 5**: `SpringExtension` is an `@ExtendWith`.
- **59 — Mockito**: `@MockitoBean` is a Mockito mock placed in the context.

**This unlocks:**
- **61 — Testcontainers**: `@AutoConfigureTestDatabase(replace = NONE)` plus
  `@ServiceConnection` is how a slice test points at a real Postgres. Also: a
  container's connection properties become part of the cache key unless you use a
  singleton, which is that topic's central trap.
- **62 — Contract testing**: generated provider tests typically run as
  `@SpringBootTest` or `@WebMvcTest`, so everything here applies to their cost.
- **63 — Mutation testing**: PIT re-runs covering tests many times. A suite with 26
  contexts is unusable for mutation testing; fixing this topic is a prerequisite.
- **65 — The load gate**: the `load` profile from Topic 43 is a distinct context key.
  Know that before you wonder why the load-test setup is slow.
- **109 — Connection pools**: each cached context holds its own pool. Twenty-six
  contexts means twenty-six pools, which is a real memory and connection cost in CI.

---

*Java baseline 21, running on JDK 25, targeting Spring Boot 4.1 / Framework 7.0. The
context cache mechanism, its key components, the default `maxSize` of 32 and the
`org.springframework.test.context.cache` DEBUG logger are stable across Boot 3.x and
4.x. The one genuinely uncertain item in this document is whether Boot 4.1 still
ships the deprecated `@MockBean` alongside `@MockitoBean` — the `javap` command in
the Syntax section settles it against your own build, and `@MockitoBean` is the
correct choice either way.*
