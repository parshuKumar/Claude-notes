# 44 — `@RestController`, Request Mapping, and Content Negotiation

## Phase: 5 — Spring Boot & Persistence
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: the `orderflow` REST API — products, inventory, orders, payments, wallet

---

## ELI5 anchor

Picture a large office building with **one** front desk.

Every visitor walks through the same door and speaks to the same receptionist. That
receptionist does not do any of the actual work. Her job is:

1. Look at what the visitor wants and find the right department. *(routing)*
2. Take the visitor's paperwork and translate it into the form that department uses.
   *(deserialization and argument binding)*
3. Send the visitor through. *(invoking your method)*
4. Take whatever the department hands back and turn it into a document the visitor can
   read — in the language the visitor asked for. *(return-value handling and content
   negotiation)*
5. If the department throws its hands up, hand the visitor a standard apology form rather
   than the department's internal notes. *(exception handling — Topic 46)*

That receptionist is `DispatcherServlet`. Spring calls it a **front controller** — one
entry point that dispatches to many handlers.

The reason this matters to you: you have been writing the *departments*. Everything
interesting that goes wrong — a 415, a 406, a parameter that binds to `null`, a JSON field
that vanishes — happens at the **front desk**, not in your department. If you do not know
the desk's five steps, those failures look like magic. If you do, each one has an obvious
place to look.

---

## The bridge from what you know

### `@RestController` ≈ a Nest controller: **HONEST ANALOGUE** in role

This is the most direct mapping in the entire curriculum. Put them side by side.

```ts
// NestJS
@Controller('products')
export class ProductController {

  constructor(private readonly products: ProductService) {}

  @Get(':id')
  findOne(@Param('id') id: string): ProductDto {
    return this.products.findOne(id);
  }

  @Get()
  list(@Query('page') page = 0, @Query('size') size = 50): PageDto<ProductDto> {
    return this.products.list(page, size);
  }

  @Post()
  @HttpCode(201)
  create(@Body() dto: CreateProductDto): ProductDto {
    return this.products.create(dto);
  }
}
```

```java
// Spring
@RestController
@RequestMapping("/products")
public class ProductController {

    private final ProductService products;

    public ProductController(ProductService products) {   // Topic 39
        this.products = products;
    }

    @GetMapping("/{id}")
    public ProductResponse findOne(@PathVariable String id) {
        return products.findOne(id);
    }

    @GetMapping
    public PageResponse<ProductResponse> list(
            @RequestParam(defaultValue = "0")  int page,
            @RequestParam(defaultValue = "50") int size) {
        return products.list(page, size);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@RequestBody @Valid CreateProductRequest request) {
        return products.create(request);
    }
}
```

The translation table is almost mechanical:

| NestJS | Spring MVC | Verdict |
|---|---|---|
| `@Controller('products')` | `@RestController` + `@RequestMapping("/products")` | **HONEST** |
| `@Get(':id')` | `@GetMapping("/{id}")` | **HONEST** |
| `@Param('id')` | `@PathVariable` | **HONEST** |
| `@Query('page')` | `@RequestParam` | **HONEST** |
| `@Body()` | `@RequestBody` | **HONEST** |
| `@Headers('x')` | `@RequestHeader("x")` | **HONEST** |
| `@HttpCode(201)` | `@ResponseStatus(HttpStatus.CREATED)` | **HONEST** |
| returning a plain object → JSON | returning a plain object → JSON | **HONEST** |
| `@Res()` for full control | `ResponseEntity<T>` | **PARTIAL** — `ResponseEntity` is a return value, not an injected mutable response |
| a custom `ParameterDecorator` | a `HandlerMethodArgumentResolver` | **PARTIAL** — same idea, different registration |
| interceptors / guards / pipes | `Filter`s, `HandlerInterceptor`s, argument resolvers, `@ControllerAdvice` | **PARTIAL** — Spring splits the responsibilities differently |

**So what is actually new?** Not the annotations. The machinery underneath.

### What does not transfer: the machinery

In Nest, the request pipeline is defined by the framework in TypeScript you can read, and
the extension points are named after what they do — guards, interceptors, pipes, filters.
The pipeline is *linear and documented as a diagram*.

In Spring MVC, the pipeline is a servlet, a list of `HandlerMapping`s, a list of
`HandlerAdapter`s, a list of `HandlerMethodArgumentResolver`s, a list of
`HandlerMethodReturnValueHandler`s, and a list of `HttpMessageConverter`s — **all of them
beans in the context, all of them auto-configured (Topic 42), all of them replaceable**.

That is more powerful and much less discoverable. The payoff for learning it:

> **Knowing where the seams are is what lets you add a custom argument resolver instead of
> parsing JSON by hand in a controller.**

Every codebase with a helper called `extractTenantFromRequest(HttpServletRequest req)`
called at the top of forty controller methods is a codebase where someone did not know
`HandlerMethodArgumentResolver` existed.

### One structural difference worth naming early

Nest's guards and interceptors run **inside** the framework's request pipeline, so they see
the resolved handler and its metadata. Spring Security is a **servlet filter chain**
(Topic 56) that runs *before* `DispatcherServlet` even starts — so at security time there
is no controller method yet, only a URL and an HTTP method. That is why Spring Security's
URL-based rules and its method-level `@PreAuthorize` are two different mechanisms operating
at two different points in the request.

---

## What is this?

### `@RestController` is two annotations

```java
@Controller
@ResponseBody
public @interface RestController { }
```

- `@Controller` makes the class a bean **and** marks it as a source of request-handler
  methods for `RequestMappingHandlerMapping`.
- `@ResponseBody` says: whatever the method returns is the **response body**, written by an
  `HttpMessageConverter`.

Without `@ResponseBody`, a method returning `String` returns a **view name** to be resolved
by a template engine. That is Trap 4 and it produces a genuinely confusing 404.

### The request path, in order

This is the section to know cold. Every arrow is a place a request can go wrong.

```
   TCP socket
        |
   [ Tomcat / Jetty / Undertow ]  accepts, parses HTTP, picks a worker thread
        |
   [ jakarta.servlet Filter chain ]        <- Spring Security lives here (Topic 56)
        |                                     also: CharacterEncodingFilter, CORS,
        |                                     your correlation-ID filter (Topic 120)
        v
   [ DispatcherServlet.doDispatch ]        <- THE FRONT CONTROLLER
        |
        |-- 1. getHandler(request)
        |      asks each HandlerMapping in order.
        |      RequestMappingHandlerMapping matches @RequestMapping metadata
        |      -> returns a HandlerExecutionChain: the HandlerMethod + interceptors
        |      -> no match: 404
        |
        |-- 2. HandlerInterceptor.preHandle()   for each interceptor
        |
        |-- 3. getHandlerAdapter(handler)
        |      RequestMappingHandlerAdapter handles @RequestMapping methods
        |
        |-- 4. ARGUMENT RESOLVERS
        |      for each parameter of your method, find the first
        |      HandlerMethodArgumentResolver whose supportsParameter() returns true
        |        @PathVariable   -> PathVariableMethodArgumentResolver
        |        @RequestParam   -> RequestParamMethodArgumentResolver
        |        @RequestBody    -> RequestResponseBodyMethodProcessor
        |                             -> picks an HttpMessageConverter by Content-Type
        |                             -> JSON bytes become your DTO here      <== deserialization
        |                             -> then @Valid runs (Topic 45)
        |
        |-- 5. YOUR METHOD RUNS                                               <== your code
        |
        |-- 6. RETURN VALUE HANDLERS
        |      first HandlerMethodReturnValueHandler that supports the return type
        |        @ResponseBody / ResponseEntity -> RequestResponseBodyMethodProcessor
        |                             -> content negotiation picks a media type
        |                             -> picks an HttpMessageConverter
        |                             -> your object becomes JSON bytes       <== serialization
        |
        |-- 7. HandlerInterceptor.postHandle(), then afterCompletion()
        |
        |-- on exception at any point:
        |      HandlerExceptionResolver chain
        |        ExceptionHandlerExceptionResolver -> your @ControllerAdvice   (Topic 46)
        |        ResponseStatusExceptionResolver   -> @ResponseStatus
        |        DefaultHandlerExceptionResolver   -> the standard 400/405/406/415 mapping
        v
   response written back through the filter chain
```

**Four facts to extract from that diagram:**

1. **Deserialization happens in an argument resolver, before your method is entered.**
   A malformed request body never reaches your code. It becomes a 400 in step 4.
2. **Serialization happens in a return-value handler, after your method returns.**
   A `LazyInitializationException` thrown during serialization is thrown *outside* your
   method and often outside your transaction. That is Trap 1 and it is why the stack trace
   looks like a Jackson bug.
3. **The `HttpMessageConverter` list is the single point where objects and bytes meet.**
   Both directions. Add a converter and you support a new media type everywhere at once.
4. **Everything in that diagram is a bean you can list.** `/actuator/beans`, or the
   condition evaluation report from Topic 42, will show you `RequestMappingHandlerMapping`
   arriving from `WebMvcAutoConfiguration`.

### Content negotiation

**Content negotiation** is how Spring decides what media type to *write*.

The default strategy is the `Accept` header. `ContentNegotiationManager` holds an ordered
list of strategies:

| Strategy | Default | How it decides |
|---|---|---|
| `HeaderContentNegotiationStrategy` | **on** | reads the `Accept` header |
| `ParameterContentNegotiationStrategy` | off | `?format=json` (`spring.mvc.contentnegotiation.favor-parameter=true`) |
| `FixedContentNegotiationStrategy` | off | always one type |
| path-extension (`/products.json`) | **removed** | dropped in Spring 6 for security reasons |

Then Spring intersects three things:

```
  what the client asked for   (Accept: application/json)
     ∩
  what the method declares    (@GetMapping(produces = "application/json"))
     ∩
  what a converter can write  (a Jackson converter can write application/json)
     =
  the chosen media type
```

Empty intersection → **406 Not Acceptable**.

The mirror image, for reading:

```
  what the client sent        (Content-Type: application/xml)
     ∩
  what the method accepts     (@PostMapping(consumes = "application/json"))
     =
  can we read it?
```

Empty → **415 Unsupported Media Type**.

**Memorise this pair.** They are the two status codes people confuse, and the difference is
which direction failed:

| Status | Direction | Meaning |
|---|---|---|
| **415 Unsupported Media Type** | request | "I cannot read what you sent me." Check `Content-Type` and `consumes`. |
| **406 Not Acceptable** | response | "I cannot write what you asked for." Check `Accept` and `produces`. |

---

## Why does it matter?

**1. Because the seams are where the leverage is.**
Correlation IDs, tenant resolution, idempotency keys, pagination parameters, ETag handling
— every one of these is either forty lines repeated in every controller, or one argument
resolver / one interceptor / one converter. The second version is a senior engineer's
output.

**2. Because the confusing failures all live in the front desk, not your code.**
415, 406, 405, a `@RequestParam` that is silently `null`, a DTO field that is silently
ignored, a 404 on a URL that obviously exists. None of these have anything to do with your
method body. Knowing the dispatch order turns each one into a single place to look.

**3. Because the controller is the boundary of your system.**
Everything past it is your domain; everything before it is the internet. The two rules that
follow — never accept an entity, never return an entity — are the difference between a
service you can refactor and one where a column rename is an API-breaking change.

**4. Because `orderflow` has to serve 1,800 rps at a 40 ms p99.**
That budget is spent partly in this layer: JSON serialization of a 50-item page, the
thread model, whether you hold a database connection through serialization. Topic 65
measures it; this topic is where the decisions get made.

---

## Syntax breakdown

### The class

```java
@RestController                                  // @Controller + @ResponseBody
@RequestMapping("/api/products")                 // class-level prefix for every method
public class ProductController { }
```

### The mapping annotations

```java
@GetMapping                      // @RequestMapping(method = GET)
@PostMapping
@PutMapping
@PatchMapping
@DeleteMapping
@RequestMapping(method = RequestMethod.HEAD)     // the general form
```

Full attribute set, all optional:

```java
@GetMapping(
    path     = "/{id}",                          // URI template
    produces = MediaType.APPLICATION_JSON_VALUE, // narrows content negotiation
    consumes = MediaType.APPLICATION_JSON_VALUE, // narrows what bodies are accepted
    params   = "!draft",                         // only match when ?draft is absent
    headers  = "X-Api-Version=2"                 // only match with that header
)
```

`params` and `headers` participate in **matching**, not validation. A request that fails
them does not get a 400 — it gets a 404 (or a 405), because no handler matched. That
surprises people.

### Path variables

```java
@GetMapping("/{orderId}/lines/{lineId}")
public LineResponse line(@PathVariable String orderId,
                         @PathVariable("lineId") String id) { }
```

| Form | Notes |
|---|---|
| `@PathVariable String orderId` | name inferred from the parameter name — **only works if compiled with `-parameters`**. Trap 2. |
| `@PathVariable("lineId") String id` | explicit name. Always safe. Use this in library code. |
| `@PathVariable Map<String,String> all` | every variable at once |
| `{id:[0-9]+}` | a regex constraint in the template; a non-matching URL is a 404, not a 400 |

Path variables are **required**. A URL that does not supply one simply does not match.

### Request parameters

```java
public PageResponse<ProductResponse> list(
        @RequestParam(defaultValue = "0")   int page,
        @RequestParam(defaultValue = "50")  int size,
        @RequestParam(required = false)     String category,
        @RequestParam(name = "q")           Optional<String> query,
        @RequestParam                       List<String> tags,      // ?tags=a&tags=b
        @RequestParam                       MultiValueMap<String,String> all) { }
```

| Form | Missing-parameter behaviour |
|---|---|
| `@RequestParam int page` | **400** `MissingServletRequestParameterException` — required by default |
| `@RequestParam(defaultValue = "0") int page` | `0`. Note: specifying a default implies `required = false`. |
| `@RequestParam(required = false) String category` | `null` |
| `@RequestParam Optional<String> query` | `Optional.empty()` — nicer than `null` at the call site (Topic 26) |
| `@RequestParam(required = false) int page` | **danger**: `int` cannot be `null`, so a missing value throws. Use `Integer` or a default. |

That last row is a real bug and it is Topic 01 all over again: unboxing `null`.

### Request body

```java
@PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
public OrderResponse place(@RequestBody @Valid PlaceOrderRequest request) { }
```

- `@RequestBody` → an `HttpMessageConverter` reads the body into your type. Chosen by the
  request's `Content-Type`.
- `@Valid` → Bean Validation runs on the bound object **before** your method body.
  Violations become a `MethodArgumentNotValidException` → 400. Topic 45.
- `@RequestBody(required = false)` allows an empty body → `null`.

### Other resolvers you get for free

```java
@RequestHeader("X-Correlation-Id") String correlationId
@RequestHeader HttpHeaders headers
@CookieValue("session") String session
@RequestPart("file") MultipartFile file            // multipart/form-data
@ModelAttribute ProductFilter filter               // binds query params into an object
HttpServletRequest request                          // the raw servlet request
UriComponentsBuilder uriBuilder                     // for building Location headers
Principal principal                                 // the authenticated user (Topic 56)
```

### Return values

```java
// 1. a plain object -> 200 + serialized body
public ProductResponse get(...) { return product; }

// 2. a status annotation
@ResponseStatus(HttpStatus.CREATED)
public ProductResponse create(...) { return product; }

// 3. ResponseEntity: full control over status, headers and body
public ResponseEntity<ProductResponse> create(...) {
    URI location = uriBuilder.path("/api/products/{id}").buildAndExpand(p.id()).toUri();
    return ResponseEntity.created(location)          // 201 + Location header
                         .eTag("\"" + p.version() + "\"")
                         .body(p);
}

// 4. no content
public ResponseEntity<Void> delete(...) { return ResponseEntity.noContent().build(); }

// 5. streaming, for a large export
public ResponseEntity<StreamingResponseBody> export(...) { }
```

**`ResponseEntity` vs `@ResponseStatus`:** use `@ResponseStatus` when the status is a
constant property of the endpoint. Use `ResponseEntity` when the status or the headers
depend on what happened — 200 vs 201, a `Location`, an `ETag`, a `Retry-After`. Do not use
both on the same method; `ResponseEntity` wins and the annotation becomes a lie for the
next reader.

### `[BOOT 3.x DELTA]` and Boot 4 notes

| Thing | Boot 2.x / Spring 5 | Boot 3.x / 4.x, Spring 6 / 7 |
|---|---|---|
| Servlet package | `javax.servlet.*` | **`jakarta.servlet.*`** |
| Trailing-slash matching | `/products/` matched `/products` by default | **removed.** `/products/` is now a **404**. Trap 3. |
| Path-extension negotiation | `/products.json` worked | removed (security) |
| Jackson | Jackson 2, `com.fasterxml.jackson.databind` | **Jackson 3 is standard; Jackson 2 deprecated.** Boot auto-configures whichever converter matches what is on the classpath — which is Topic 42's `@ConditionalOnClass` doing its job. *Flagged: verify the Jackson 3 package names against your resolved version; I am not quoting them from memory.* |
| API versioning | do it yourself with `headers=` or a path prefix | Framework 7 / Boot 4 add **built-in API versioning** configured under `spring.mvc.apiversion.*`, with a version attribute on the mapping annotations. *Flagged: I am not going to invent the exact attribute names — check the Boot 4.1 reference docs before using it.* |
| HTTP clients | `RestTemplate`, `WebClient`, Feign | Boot 4 auto-configures **HTTP Service Clients** — annotated Java interfaces as clients. Phase 11. |
| Artifact coordinates | stable | Boot 4 modularised into many smaller jars; an artifact may have moved. Resolve with `dependency:tree`, never type a version — the BOM supplies it (Topic 32). |

---

## Example 1 — minimal

One resource, three endpoints, no cleverness.

```java
package com.orderflow.catalog.web;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService products;

    public ProductController(ProductService products) {
        this.products = products;
    }

    @GetMapping("/{sku}")
    public ProductResponse findBySku(@PathVariable String sku) {
        return products.findBySku(sku);
    }

    @GetMapping
    public List<ProductResponse> list(@RequestParam(defaultValue = "0")  int page,
                                      @RequestParam(defaultValue = "50") int size) {
        return products.list(page, size);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@RequestBody CreateProductRequest request) {
        return products.create(request);
    }
}
```

```java
package com.orderflow.catalog.web;

/** A DTO. Note: NOT the JPA entity. See Trap 1. */
public record ProductResponse(String sku, String name, long priceMinor, String currency) {}

public record CreateProductRequest(String sku, String name, long priceMinor, String currency) {}
```

**Predict these before running anything:**

| Request | Response |
|---|---|
| `GET /api/products/SKU-1001` | 200, JSON body |
| `GET /api/products` | 200, JSON array; `page=0`, `size=50` |
| `GET /api/products?size=abc` | **400** — type conversion failed, `MethodArgumentTypeMismatchException` |
| `GET /api/products/` (trailing slash) | **404** on Spring 6+. Trap 3. |
| `POST /api/products` with `Content-Type: text/plain` | **415** — no converter can read `text/plain` into the DTO |
| `GET /api/products/SKU-1001` with `Accept: application/xml` | **406** if no XML converter is on the classpath |
| `PUT /api/products/SKU-1001` | **405** — the path matches, the method does not |

That table is the whole topic in miniature. Every row is the front desk, not your code.

---

## Example 2 — production scenario: the `orderflow` REST API

### The constraints, stated concretely

| Constraint | Number |
|---|---|
| Peak throughput | **1,800 rps**, mix: 70% catalogue read, 20% order read, 10% order placement |
| Latency SLO | **p99 < 40 ms** on catalogue read, **p99 < 250 ms** on order placement |
| Catalogue size | **120,000 products** |
| Order history | **1.1M orders, 5M order lines** |
| Clients | a web SPA, an iOS app, and two partner integrations that retry aggressively on timeout |
| Tenancy | multi-tenant; every request carries `X-Tenant-Id` and it must never be optional |

Three consequences fall straight out of those numbers:

1. **`GET /api/orders` must never return an unbounded list.** 1.1M orders with lines is not
   a response, it is an outage. Pagination is mandatory and the maximum page size is
   enforced server-side, not requested by the client.
2. **Order placement must be idempotent.** Partners retry on timeout. A retried
   `POST /api/orders` must not create a second order or debit the wallet twice.
3. **Tenant resolution must be impossible to forget.** Forty controller methods each
   starting with `String tenant = request.getHeader("X-Tenant-Id")` will eventually have one
   that forgets, and that is a cross-tenant data leak. This is exactly what an argument
   resolver is for.

### The API surface

```
GET    /api/products                      list, paged, filterable
GET    /api/products/{sku}                one product
GET    /api/inventory/{sku}               current available stock
POST   /api/orders                        place an order          (idempotent)
GET    /api/orders                        list mine, paged
GET    /api/orders/{orderId}              one order with lines
POST   /api/orders/{orderId}/cancel       cancel
GET    /api/payments/{paymentId}          payment status
POST   /api/payments/{paymentId}/capture  capture an authorisation
GET    /api/wallet                        balance
POST   /api/wallet/topups                 add funds              (idempotent)
```

### The DTO boundary — the rule that is not negotiable

```java
package com.orderflow.orders.web;

import java.time.Instant;
import java.util.List;

/** Response DTO. Flat, explicit, and completely independent of the entity graph. */
public record OrderResponse(
        String orderId,
        String status,
        long totalMinor,
        String currency,
        Instant placedAt,
        List<LineResponse> lines,
        PaymentSummary payment
) {
    public record LineResponse(String sku, String name, int quantity, long unitPriceMinor) {}
    public record PaymentSummary(String paymentId, String status) {}
}
```

```java
/** Request DTO. Validation is Topic 45; the shape is the point here. */
public record PlaceOrderRequest(
        List<LineRequest> lines,
        String walletId
) {
    public record LineRequest(String sku, int quantity) {}
}
```

**Three reasons the DTO is separate from the entity, in order of how expensive ignoring
them is:**

1. **Serialization would walk the lazy graph.** `Order` has `@OneToMany List<OrderLine>`,
   each `OrderLine` has `@ManyToOne Product`, `Product` has a `Category`. Serializing the
   entity means Jackson touching every proxy, which either explodes (Trap 1) or issues
   several hundred queries per response (Topic 50's N+1) — at 1,800 rps that is not
   survivable.
2. **The entity is your database schema.** Renaming a column becomes a breaking API change.
   Adding a column silently adds a field to your public contract, possibly one you did not
   intend to publish.
3. **Entities carry things you must not send.** An internal cost price, a fraud score, a
   soft-delete flag. "We'll add `@JsonIgnore`" is a deny-list, and deny-lists fail on the
   next field somebody adds.

> **The rule:** entities never cross the controller boundary in either direction. Not as a
> parameter, not as a return value, not nested inside something else. If you find yourself
> writing `@JsonIgnore` on an entity field, you have already lost the argument.

### The custom argument resolver — the payoff for knowing the seams

The naive version, repeated in forty methods:

```java
@GetMapping("/{sku}")
public ProductResponse get(@PathVariable String sku, HttpServletRequest request) {
    String tenantId = request.getHeader("X-Tenant-Id");
    if (tenantId == null) throw new MissingTenantException();
    return products.findBySku(TenantId.of(tenantId), sku);
}
```

The version that cannot be forgotten:

```java
package com.orderflow.tenancy;

import java.lang.annotation.*;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface CurrentTenant {}
```

```java
package com.orderflow.tenancy;

import org.springframework.core.MethodParameter;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;

public class CurrentTenantArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentTenant.class)
                && TenantId.class.equals(parameter.getParameterType());
    }

    @Override
    public Object resolveArgument(MethodParameter parameter,
                                  ModelAndViewContainer mav,
                                  NativeWebRequest webRequest,
                                  WebDataBinderFactory binderFactory) {
        String raw = webRequest.getHeader("X-Tenant-Id");
        if (raw == null || raw.isBlank()) {
            throw new MissingTenantException();      // -> 400 via @ControllerAdvice, Topic 46
        }
        return TenantId.of(raw);                     // validated, typed, never a bare String
    }
}
```

```java
package com.orderflow.web;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentTenantArgumentResolver());
    }
}
```

Now every controller method reads:

```java
@GetMapping("/{sku}")
public ProductResponse get(@CurrentTenant TenantId tenant, @PathVariable String sku) {
    return products.findBySku(tenant, sku);
}
```

**Why this is better than a filter putting the tenant in a `ThreadLocal`:**
- The dependency is **visible in the method signature**, so a reader knows the endpoint is
  tenant-scoped without reading a filter.
- It is **typed** — `TenantId`, not `String`, so it cannot be passed where a SKU is
  expected (Topic 02's nominal typing, doing real work).
- It is **testable** — a `@WebMvcTest` (Topic 60) exercises the resolver, and a plain unit
  test calls the controller method with a `TenantId` and no request at all.
- It does **not** leak across threads. A `ThreadLocal` on a pooled Tomcat thread that is
  not cleared gives the next request the previous tenant's identity. That is the same leak
  shape as Topic 120's MDC problem, and here it is a cross-tenant data breach.

> **`WebMvcConfigurer` vs `@EnableWebMvc`:** implementing `WebMvcConfigurer` *adds to*
> Boot's auto-configured MVC setup. Putting `@EnableWebMvc` on a configuration class
> **disables** `WebMvcAutoConfiguration` entirely (it is guarded by
> `@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)` — Topic 42) and you lose
> the message converters, static resource handling and error handling that Boot configured
> for you. Almost nobody wants that. If your JSON suddenly stops working after someone adds
> a config class, look for `@EnableWebMvc` first.

### Pagination that cannot be abused

```java
@GetMapping
public PageResponse<ProductResponse> list(
        @CurrentTenant TenantId tenant,
        @RequestParam(defaultValue = "0")  @Min(0)  int page,
        @RequestParam(defaultValue = "50") @Min(1) @Max(200) int size,
        @RequestParam(required = false) String category) {

    return products.list(tenant, category, page, size);
}
```

`@Max(200)` on the parameter is a **method-level** constraint, which needs `@Validated` on
the controller class to fire — that is Topic 45, and it is the one part of validation that
does not work by default. If you do not want that dependency here, clamp it explicitly:

```java
int effectiveSize = Math.min(Math.max(size, 1), 200);
```

Either is defensible. What is not defensible is trusting the client. A partner integration
that discovers `?size=1000000` will use it, and at 1,800 rps one such request is enough to
saturate the heap.

### Idempotent order placement

```java
@PostMapping
public ResponseEntity<OrderResponse> place(
        @CurrentTenant TenantId tenant,
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody @Valid PlaceOrderRequest request,
        UriComponentsBuilder uriBuilder) {

    OrderResult result = orders.place(tenant, idempotencyKey, request);

    URI location = uriBuilder.path("/api/orders/{id}")
                             .buildAndExpand(result.orderId())
                             .toUri();

    return result.wasCreated()
            ? ResponseEntity.created(location).body(result.response())   // 201
            : ResponseEntity.ok().location(location).body(result.response()); // 200, replayed
}
```

Note that `@RequestHeader("Idempotency-Key")` with no `required = false` makes the header
**mandatory** — a missing one is a 400 with a message naming the header, produced by the
front desk before your method runs. That is the correct place for this rule: a client that
cannot supply an idempotency key cannot place an order, and no code of yours has to
remember to check.

The 201-vs-200 distinction matters to the partner integrations: 201 means "I created it",
200 means "you already sent this, here it is again". Both carry `Location`. Storage and
transactional semantics of the idempotency key are Topic 54; the HTTP contract is here.

### Content negotiation, narrowed deliberately

```java
@RestController
@RequestMapping(path = "/api/orders",
                produces = MediaType.APPLICATION_JSON_VALUE)
public class OrderController { }
```

Declaring `produces` at class level does two things:

1. A client sending `Accept: application/xml` gets a **406 immediately**, from the front
   desk, instead of a surprise if an XML converter ever lands on the classpath transitively.
2. It documents the contract in the code, which OpenAPI generation reads.

For `orderflow` this is the right call: the API is JSON, deliberately and permanently.
Being explicit costs one line and removes a class of "why did the response format change
after a dependency bump" incident.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — returning a JPA entity from a controller

**Wrong:**

```java
@GetMapping("/{orderId}")
public Order get(@PathVariable String orderId) {
    return orderRepository.findById(orderId).orElseThrow();   // the ENTITY
}
```

where `Order` has `@OneToMany(fetch = LAZY) List<OrderLine> lines`.

**Exact symptom — one of four, and which one you get depends on configuration:**

| What you see | What happened |
|---|---|
| HTTP **500**, and the log shows `HttpMessageNotWritableException` caused by `LazyInitializationException: could not initialize proxy - no Session` | the transaction ended when the service method returned; serialization began afterwards; Jackson touched a lazy proxy with no session. |
| HTTP **200** with a correct-looking body, and the SQL log shows **several hundred queries for one request** | `spring.jpa.open-in-view=true` (the Boot default). The session is held open through serialization, so every lazy touch works — by issuing a query. Topic 50's N+1, invisible until you count. |
| HTTP **500** with `StackOverflowError` or an infinite JSON body | a bidirectional association (`Order` → `OrderLine` → `Order`) serialized in a cycle. |
| HTTP **200** with fields you did not intend to publish | the entity has a `costPrice` or `fraudScore` column and Jackson serialized it, because Jackson serializes what is there. |

The second row is the dangerous one. **Nothing errors.** The endpoint works, the tests
pass, and it falls over at 1,800 rps because each request became 300 queries. You discover
it at the Topic 65 gate, or in production.

**Root cause:** serialization happens in a return-value handler *after* your method returns
(step 6 of the dispatch diagram), which is outside your `@Transactional` boundary. An
entity is a live object graph with lazy proxies, not a data structure.

**Fix:** a DTO, mapped inside the transaction.

```java
@GetMapping("/{orderId}")
public OrderResponse get(@CurrentTenant TenantId tenant, @PathVariable String orderId) {
    return orderQueryService.findOrderResponse(tenant, orderId);   // returns a DTO
}
```

and set, in `application.yml`, permanently:

```yaml
spring:
  jpa:
    open-in-view: false
```

Turning `open-in-view` off converts the silent row-two behaviour into the loud row-one
behaviour. That is an improvement: you want lazy-loading-outside-a-transaction to be an
exception at development time, not a query storm in production. Topic 49 covers the full
argument.

---

### Trap 2 — parameter names missing because the build did not pass `-parameters`

**Wrong:**

```java
@GetMapping("/{sku}")
public ProductResponse get(@PathVariable String sku) { }     // name inferred
```

built by a Maven or Gradle setup that does not enable `-parameters` — typically a
hand-rolled `maven-compiler-plugin` configuration that overrode Boot's parent, or a module
that does not inherit `spring-boot-starter-parent`.

**Exact symptom:** the application **fails to start**, or fails on first request, with:

```
java.lang.IllegalArgumentException: Name for argument of type [java.lang.String]
not specified, and parameter name information not found in class file either.
```

The confusing part is that it works perfectly in your IDE (which compiles with debug
information) and fails in CI, or works in one Maven module and not another.

**Root cause:** parameter names are not in a class file unless javac is told to keep them
with `-parameters`. Without them, Spring has an annotation but no name to look up.
`spring-boot-starter-parent` and the Spring Boot Gradle plugin enable the flag for you —
which is why this only appears in builds that stepped outside them.

**Fix — either of two, and do both in library code:**

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <parameters>true</parameters>
  </configuration>
</plugin>
```

and/or be explicit at every binding site:

```java
@PathVariable("sku") String sku
@RequestParam("page") int page
```

**Verify it:**

```bash
javap -v -p target/classes/com/orderflow/catalog/web/ProductController.class \
  | grep -A5 MethodParameters
```

A `MethodParameters` attribute present means the names survived. Absent means they did not.
This is the same reflex as Topic 01's `javap -c` — look at the class file rather than
arguing about it.

---

### Trap 3 — the trailing slash 404, and its cousin the ambiguous mapping

**Wrong (part A):** a client calls `GET /api/products/` and you handle `/api/products`.

**Exact symptom:** **404**, with nothing in the log except the access line. The URL is
obviously right. It worked on the old service. A colleague on an older Boot version cannot
reproduce it.

**Root cause:** Spring Framework 5 matched a trailing slash by default
(`setUseTrailingSlashMatch(true)`). **Spring 6 removed that**, deliberately, because "two
URLs for one resource" is bad for caching, security rules and analytics. Boot 3 and 4
inherit the new behaviour. It is one of the most-hit breaking changes in the whole 2 → 3
migration.

**Fix — pick one and be consistent:**

1. **Preferred:** fix the clients, and redirect at the edge. A 301 from `/x/` to `/x` in
   your ingress or load balancer is one rule, applies to everything, and keeps one
   canonical URL per resource.
2. If you cannot change clients, add an explicit `PathPatternParser` configuration or list
   both paths on the mapping. Both are worse; treat them as migration scaffolding with a
   removal date.

**Wrong (part B):** two methods that can match the same request.

```java
@GetMapping("/{sku}")      public ProductResponse bySku(@PathVariable String sku) { }
@GetMapping("/{id}")       public ProductResponse byId(@PathVariable String id)  { }
```

**Exact symptom:** the application **fails to start**:

```
java.lang.IllegalStateException: Ambiguous mapping. Cannot map 'productController'
method ... to {GET /api/products/{id}}: There is already 'productController' bean method
... mapped.
```

**Root cause:** `RequestMappingHandlerMapping` builds its lookup table at startup and
refuses to register two handlers for an identical mapping. This is a genuinely good design:
a routing ambiguity is a startup failure, not a coin flip at request time.

**Fix:** disambiguate with a real distinction — a different path segment
(`/by-sku/{sku}`), a regex constraint (`/{id:[0-9]+}` vs `/{sku:[A-Z]+-[0-9]+}`), or a
`params`/`headers` condition. Do not disambiguate with `produces` unless the media type is
genuinely the difference.

---

### Trap 4 — `@Controller` where you meant `@RestController`

**Wrong:**

```java
@Controller                       // not @RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping("/{sku}")
    public ProductResponse get(@PathVariable String sku) { ... }

    @GetMapping("/health-token")
    public String token() { return "ok"; }
}
```

**Exact symptom — two different failures from the same mistake:**

| Endpoint | What you see |
|---|---|
| `GET /api/products/SKU-1001` | **500**, `ServletException: Circular view path [product]`, or a whitelabel error page. Spring treated the returned object's name as a **view** and tried to render it. |
| `GET /api/products/health-token` | **404**, or a whitelabel error page saying the view `ok` could not be resolved. The literal string `ok` was treated as a template name. |

The `String`-returning case is the more confusing: the response is a 404 for a URL that
clearly matched a handler. The handler ran. It is the *return value handling* (step 6) that
failed.

**Root cause:** without `@ResponseBody`, the default return-value handler for a `String` is
`ViewNameMethodReturnValueHandler` — it interprets the return value as a logical view name.
`@RestController` is `@Controller + @ResponseBody`, and `@ResponseBody` switches the handler
to `RequestResponseBodyMethodProcessor`, which writes the body.

**Fix:** `@RestController` on the class. Or `@ResponseBody` on the method if you genuinely
have a mixed controller (you almost certainly do not — split it).

**How to recognise it instantly:** the words "view", "template", "Thymeleaf" or "whitelabel"
appearing in an error from a JSON API mean `@ResponseBody` is missing somewhere.

---

### Trap 5 — blocking work on the servlet thread, at 1,800 rps

**Wrong:**

```java
@PostMapping("/{paymentId}/capture")
public PaymentResponse capture(@PathVariable String paymentId) {
    // synchronous HTTP call to the payment gateway, p99 ~ 800 ms
    return gatewayClient.capture(paymentId);
}
```

**Exact symptom, in this order over about ninety seconds:**

1. p99 on `/api/payments/*/capture` rises, which is expected.
2. p99 on **`GET /api/products`** rises — an endpoint that touches nothing but a cache.
3. Tomcat's `busy threads` metric pins at `server.tomcat.threads.max` (default 200).
4. New connections queue in the accept backlog; clients see connection timeouts, not 503s.
5. The Kubernetes **readiness** probe times out. The pod is removed from the load balancer.
   Its traffic shifts to the remaining pods, which now saturate faster.
6. Cascading removal of every pod. A downstream slowdown became a full outage.

The signature is step 2: **unrelated endpoints degrade**. That is always thread starvation,
never a slow query on the endpoint you are looking at.

**Root cause:** the classic Servlet model is thread-per-request. A blocked thread is a
consumed thread. At 1,800 rps with 10% capture traffic and an 800 ms downstream, you need
roughly `180 × 0.8 = 144` threads just for capture — most of a 200-thread pool — and the
other 1,620 rps are competing for what is left.

This is Node's event-loop intuition in reverse: in Node a blocking call blocks
*everything*; in Java it blocks *one thread*, which feels safe right up to the point where
you run out of threads. The failure is later and much harder to attribute.

**Fixes, in the order you should consider them:**

1. **Do not block on a slow downstream inside a request.** Accept the capture, return
   **202 Accepted** with a status URL, and complete it asynchronously. This is a contract
   change and the right answer if the client can tolerate it.
2. **Bound and time-box the call.** A hard client timeout well under your SLO, plus a
   circuit breaker (Topic 111). A downstream that is slow should fail fast, not consume
   your capacity.
3. **Virtual threads** (`spring.threads.virtual.enabled=true`, Topic 101). Blocking a
   virtual thread does not consume a platform thread, which removes *this* failure mode —
   but it moves the bottleneck to whatever the blocked work holds, most often a database
   connection (Topic 109). It is a real fix and it is not free; measure it.
4. Raising `server.tomcat.threads.max` is **not** a fix. It raises the ceiling on
   concurrent blocked work and therefore on memory and on downstream load. It buys minutes.

**And the one that must never happen:** a slow HTTP call inside a `@Transactional` method.
Then each blocked thread also pins a database connection, and pool exhaustion takes out
every endpoint including ones that never call the gateway. That is Topic 55's drill, and it
is worth reading before you write the capture endpoint for real.

---

## Hands-on proof

Every command is one **you** run. I have no JVM and no running `orderflow`, so nothing
below is captured output — it is the command, what to look for, and what each possible
result means. Use `curl -i` throughout so you see status and headers, not just the body.

### Setup

```yaml
# application-local.yml
logging:
  level:
    org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping: TRACE
    org.springframework.web.servlet.DispatcherServlet: DEBUG

management:
  endpoints:
    web:
      exposure:
        include: mappings,beans,conditions,health,env,configprops
```

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
```

### Proof 1 — list every route the application actually has

```bash
curl -s localhost:8080/actuator/mappings \
  | jq '.contexts.application.mappings.dispatcherServlets.dispatcherServlet[]
        | select(.details.handlerMethod.className | test("com.orderflow"))
        | {predicate: .predicate, method: .details.handlerMethod.name}'
```

| What you see | What it means |
|---|---|
| one entry per handler method, with `predicate` showing the full mapping | this is the routing table `RequestMappingHandlerMapping` built at startup. It is the ground truth about what URLs exist. |
| an endpoint you expected is **missing** | the controller is not a bean — check component scanning (Topic 36) and that the class is under the `@SpringBootApplication` package. |
| `predicate` shows `produces [application/json]` where you did not write it | inherited from a class-level `@RequestMapping`. |
| 404 from `/actuator/mappings` | the endpoint is not exposed. Add `mappings` to `include`. |

**Why this beats grepping the source:** it shows the *effective* mapping after class-level
prefixes, inherited attributes and any programmatic registration. When a URL "should work"
and does not, compare it against this list before doing anything else.

### Proof 2 — walk the dispatch with TRACE logging

With the logging config above, issue one request and read the log:

```bash
curl -i localhost:8080/api/products/SKU-1001
```

**What to look for**, in order, in the application log:

1. A `DispatcherServlet` DEBUG line naming the incoming `GET /api/products/SKU-1001`.
2. A `RequestMappingHandlerMapping` TRACE line showing which handler method was **mapped**
   to the request.
3. A `DispatcherServlet` DEBUG line reporting the completion status.

| What you see | What it means |
|---|---|
| a "Mapped to" line naming your method | routing succeeded. Any failure after this is in argument resolution, your method, or serialization. |
| "No mapping for GET /api/products/" | routing failed — 404 from the front desk. Compare against Proof 1's list. Suspect a trailing slash (Trap 3). |
| mapped, then a 500 | your method or serialization. Get the stack trace: `HttpMessageNotWritableException` means serialization (Trap 1); anything else is your code. |
| mapped, then 415 | the request `Content-Type` could not be read. See Proof 4. |
| mapped, then 406 | no converter could produce what `Accept` asked for. See Proof 4. |

Turn this logging **off** before any measurement run. TRACE logging on the mapping class
costs real time per request and will corrupt a Topic 65 baseline.

### Proof 3 — the status-code matrix

Run all of these and record the status line from `curl -i`. Predict each one first.

```bash
# 1. happy path
curl -i localhost:8080/api/products/SKU-1001

# 2. unknown resource -> depends on your @ControllerAdvice (Topic 46)
curl -i localhost:8080/api/products/SKU-DOES-NOT-EXIST

# 3. trailing slash
curl -i localhost:8080/api/products/

# 4. wrong HTTP method on a valid path
curl -i -X PUT localhost:8080/api/products/SKU-1001

# 5. bad type on a query parameter
curl -i "localhost:8080/api/products?size=abc"

# 6. missing required parameter
curl -i "localhost:8080/api/orders"          # if a required param is declared

# 7. missing required header
curl -i -X POST localhost:8080/api/orders \
  -H 'Content-Type: application/json' -d '{"lines":[]}'

# 8. wrong request Content-Type
curl -i -X POST localhost:8080/api/orders \
  -H 'Content-Type: text/plain' -H 'X-Tenant-Id: t1' -d 'hello'

# 9. unsatisfiable Accept
curl -i localhost:8080/api/products/SKU-1001 -H 'Accept: application/xml'

# 10. malformed JSON body
curl -i -X POST localhost:8080/api/orders \
  -H 'Content-Type: application/json' -H 'X-Tenant-Id: t1' -d '{not json'
```

| # | Expected | If you get something else |
|---|---|---|
| 1 | 200 | see Proof 2 |
| 2 | 404 or 400 depending on your advice | a 500 means an unmapped exception — Topic 46 |
| 3 | **404** | a 200 means you are on Spring 5 / Boot 2, or someone re-enabled trailing-slash matching |
| 4 | **405** with an `Allow` header listing permitted methods | a 404 means the path itself did not match |
| 5 | **400** (`MethodArgumentTypeMismatchException`) | a 500 means the exception is unmapped |
| 6 | **400** (`MissingServletRequestParameterException`) | a 200 with a default means the parameter was not required |
| 7 | **400** naming the missing header | a 500 means an NPE inside your method — the header was optional when it should not have been |
| 8 | **415** | remember: 415 is about what you **sent** |
| 9 | **406** if no XML converter is present; 200 with XML if `jackson-dataformat-xml` is transitively on the classpath | a surprise 200 with XML is exactly why you declare `produces` |
| 10 | **400** (`HttpMessageNotReadableException`) | a 500 means the parse failure is not mapped — Topic 46 |

**Rows 8 and 9 are the pair to internalise.** 415 = "I can't read yours". 406 = "I can't
write yours". Say it out loud once and you will never confuse them in an interview.

### Proof 4 — prove content negotiation is a real decision

Add an XML converter to the classpath (Jackson's XML dataformat module — resolve the
artifact from your Boot BOM, do not type a version), rebuild, and repeat:

```bash
curl -i localhost:8080/api/products/SKU-1001 -H 'Accept: application/json'
curl -i localhost:8080/api/products/SKU-1001 -H 'Accept: application/xml'
curl -i localhost:8080/api/products/SKU-1001 -H 'Accept: */*'
curl -i localhost:8080/api/products/SKU-1001 -H 'Accept: application/xml;q=0.9, application/json;q=1.0'
```

| What you see | What it means |
|---|---|
| JSON, then XML, then JSON, then JSON | negotiation is working. `*/*` picks the first converter that can write the type; the `q` values are honoured. |
| XML for the second request even though you never wrote XML code | **the point of the exercise.** A transitive dependency changed your API's output format. Nothing in your code changed. |
| 406 for the XML request | the converter is not on the classpath, or `produces` narrows the endpoint to JSON. |

Now add `produces = MediaType.APPLICATION_JSON_VALUE` at class level and rerun. The XML
request should become a **406**. That single line is how you make your content type a
deliberate contract instead of a consequence of your dependency graph.

Confirm which converters are registered:

```bash
curl -s localhost:8080/actuator/beans \
  | jq '.contexts.application.beans | keys | map(select(test("[Cc]onverter")))'
```

### Proof 5 — prove your argument resolver runs

```bash
# with the header
curl -i localhost:8080/api/products/SKU-1001 -H 'X-Tenant-Id: acme'

# without it
curl -i localhost:8080/api/products/SKU-1001
```

| What you see | What it means |
|---|---|
| 200 with the header, 400 without | the resolver ran and rejected the missing tenant. Correct. |
| 200 in **both** cases | `supportsParameter` returned false — check the annotation is `@Target(PARAMETER)` with `RUNTIME` retention and that the parameter type matches exactly. |
| 500 without the header | your exception is not mapped to a status. Topic 46. |
| the resolver never runs at all | `addArgumentResolvers` was not called — check the class implements `WebMvcConfigurer` and is a `@Configuration` bean. Verify with `/actuator/beans`. |

The third and fourth rows are the common mistakes and they are distinguishable in one
request, which is why running this is worth five minutes.

### Proof 6 — confirm parameter names survived compilation

```bash
javap -v -p target/classes/com/orderflow/catalog/web/ProductController.class \
  | grep -A6 "MethodParameters"
```

| What you see | What it means |
|---|---|
| a `MethodParameters` section listing `sku`, `page`, `size` | `-parameters` is enabled. `@PathVariable` without an explicit name is safe. |
| no `MethodParameters` section at all | the flag is off. Trap 2 is waiting for you. Add `<parameters>true</parameters>` or name every binding explicitly. |

Do this once per module, especially in any module that does not inherit
`spring-boot-starter-parent`.

---

## Practice exercises

### 1 — Easy: the status-code matrix, by hand

Implement the minimal `ProductController` from Example 1. Then run all ten requests from
Hands-on Proof 3.

**Before running each one, write down your predicted status code.** Then run it.

**Deliverable:** a ten-row table of predicted vs actual, and for every row where you were
wrong, one sentence naming the dispatch step (routing / argument resolution / your method /
return-value handling) responsible for the difference. Getting three wrong is normal and
the wrong ones are the ones you will remember.

---

### 2 — Medium: the custom argument resolver (combines Topics 02, 26, 27, 38, 39, 42)

Implement `@CurrentTenant` end to end, then justify each design choice.

Requirements:

1. **Topic 02:** `TenantId` must be a nominal type, not a `String`. Write two sentences on
   what a `String` tenant id makes possible that `TenantId` makes impossible — with a
   concrete `orderflow` example, not an abstraction.
2. **Topic 27:** implement `TenantId` as a record with a compact constructor that validates
   the format. State what "shallowly immutable" means here and why it is sufficient.
3. **Topic 39:** the resolver has a collaborator — a `TenantRegistry` that checks the tenant
   exists. Wire it by constructor injection, register the resolver as a bean, and explain
   why `new CurrentTenantArgumentResolver()` inside `addArgumentResolvers` would be a
   mistake once it has a dependency.
4. **Topic 38:** implement the same feature a **second** way — a servlet `Filter` that puts
   the tenant into a request-scoped bean, injected with a scoped proxy. Get it working.
5. **Topic 26:** design an `@CurrentTenant(required = false)` variant. Should the parameter
   type be `Optional<TenantId>`, a nullable `TenantId`, or should the feature not exist?
   Argue it.
6. **Topic 42:** package the resolver as an auto-configuration in your
   `orderflow-pricing-starter` sibling module, with `@ConditionalOnMissingBean` so a
   consuming service can supply its own. Prove the back-off with the condition evaluation
   report.

**Then compare 3 and 4 directly.** Write 300 words on which you would ship, covering:
visibility in the method signature, testability, thread-leak risk on pooled threads, and
what happens in an `@Async` method (Topic 91). Name a situation where the filter approach
is genuinely the better one.

---

### 3 — Hard: build the `orderflow` API and find its thread ceiling

**Part A — build the API.** Implement all eleven endpoints from Example 2 against the
in-memory stubs from Topic 35. Non-negotiable rules:

- No entity crosses the controller boundary in either direction.
- Every list endpoint is paged with a server-enforced maximum.
- `POST /api/orders` and `POST /api/wallet/topups` require an `Idempotency-Key` header and
  return 201 on create, 200 on replay, both with `Location`.
- Every controller declares `produces = application/json` at class level.
- Zero uses of `HttpServletRequest` in any controller method signature.

**Part B — prove the contract.** Write a `curl` script that exercises every endpoint plus
every failure mode from Proof 3, asserting on status codes. Commit it. This is the artefact
Topic 62's contract testing will replace with something better — build it now so you
understand what it is replacing.

**Part C — find the thread ceiling.** Add an artificial delay to the payment-capture
endpoint that matches a realistic gateway (`Thread.sleep(800)` — deliberately crude; Topic
101 will make this honest).

Using k6 or Gatling with an **open arrival model**:

1. Establish a baseline: `GET /api/products` alone at 1,200 rps. Record p50/p95/p99.
2. Add capture traffic at 20 rps. Re-measure `GET /api/products`. Did it move?
3. Raise capture traffic until `GET /api/products` p99 **doubles**. Record the capture rate
   at which that happened, and the Tomcat `busy threads` metric at that moment
   (`/actuator/metrics/tomcat.threads.busy`).
4. Predict, arithmetically, the capture rate that should saturate 200 threads at 800 ms.
   Compare with what you measured. Explain any gap — and be honest if your prediction was
   badly wrong.

**Part D — fix it three ways and measure each.**

1. `server.tomcat.threads.max: 400`.
2. `spring.threads.virtual.enabled: true` (Topic 101 — note you are running JDK 25).
3. Change the contract: return **202 Accepted** with a status URL and complete the capture
   on a bounded executor.

For each: the capture rate at which catalogue p99 doubles, and the new failure mode. **All
three still fail somewhere** — your deliverable is naming *where*, for each. Option 1's
ceiling is memory and downstream load; option 2 moves the bottleneck to the connection pool
(Topic 109); option 3 moves it to the executor's queue and to your ability to report status.

**Part E — the write-up.** One page: which fix you would ship for `orderflow`, what it
costs the partner integrations, and what you would need to see in Topic 65's baseline to
change your mind. Numbers, not adjectives.

---

## Interview questions

### Q1 — "Walk me through what happens between an HTTP request arriving and your controller method being called."

**Mid-level answer:** "Spring receives the request, matches it to a controller method based
on the URL and HTTP method, converts the JSON body into your DTO, and calls your method.
Then it converts the return value back to JSON."

**Senior answer:** "There are more seams than that, and knowing them is what makes this
debuggable.

Tomcat accepts the connection, parses HTTP and hands the request to a worker thread. It goes
through the servlet filter chain first — that's where Spring Security lives, so authorisation
happens **before** `DispatcherServlet`, and at that point there is no controller method yet,
only a URL and a method.

Then `DispatcherServlet.doDispatch` — the front controller. It asks each `HandlerMapping` in
turn; `RequestMappingHandlerMapping` matches against the `@RequestMapping` metadata it
indexed at startup and returns a `HandlerExecutionChain` — the handler method plus any
interceptors. No match is a 404 right there.

Then it picks a `HandlerAdapter` — `RequestMappingHandlerAdapter` for annotated methods —
which resolves each parameter by walking a list of `HandlerMethodArgumentResolver`s and
taking the first that supports it. `@RequestBody` goes through
`RequestResponseBodyMethodProcessor`, which picks an `HttpMessageConverter` by the request's
`Content-Type`. **That is where deserialization actually happens** — before my method is
entered — and it's also where `@Valid` runs.

My method runs. The return value goes to a `HandlerMethodReturnValueHandler`, which does
content negotiation against the `Accept` header and the method's `produces`, picks a
converter, and writes the body. **Serialization happens after my method returns**, which is
why a `LazyInitializationException` during serialization is thrown outside my transaction.

Any exception goes to the `HandlerExceptionResolver` chain, where
`ExceptionHandlerExceptionResolver` finds my `@ControllerAdvice`.

The practical value: every one of those is a bean I can list and replace. If I need to
resolve a tenant from a header on forty endpoints, that's one argument resolver rather than
forty copies of `request.getHeader`."

**What separates them:** naming the components rather than describing effects, placing
security *before* the dispatcher, and knowing that deserialization and serialization happen
on opposite sides of your method — which is the fact that explains half of Spring MVC's
confusing failures.

**Interviewer's follow-up:** "Where would you plug in support for a new media type?" An
`HttpMessageConverter`, registered via `WebMvcConfigurer.configureMessageConverters` or
`extendMessageConverters`. It works in both directions everywhere at once, which is the
whole reason it is a single extension point. Volunteer the trap: adding `@EnableWebMvc`
disables Boot's auto-configuration and you lose the converters you already had.

---

### Q2 — "What's the difference between `@Controller` and `@RestController`? What actually breaks?"

**Mid-level answer:** "`@RestController` is `@Controller` plus `@ResponseBody`. You use
`@RestController` for REST APIs and `@Controller` when you're rendering views."

**Senior answer:** "That's the definition; the interesting part is the failure it produces.

Without `@ResponseBody`, the return value goes to a return-value handler that treats it as
a **logical view name**. So a method returning the `String` `"ok"` makes Spring look for a
template called `ok`. You get a 404 or a whitelabel error page for a URL that clearly
matched a handler — the handler ran, it's the return-value handling that failed. With an
object return type you often get `Circular view path`, which reads like a Spring bug.

The diagnostic: if a JSON API produces an error mentioning views, templates, Thymeleaf or
the whitelabel page, `@ResponseBody` is missing somewhere.

`@ResponseBody` also works at method level, so a mixed controller is possible — but I'd
split it. A class that both renders views and serves JSON has two different error-handling
contracts, two different content-negotiation stories, and two different security postures."

**What separates them:** describing the exact observable failure and giving a recognition
rule, rather than restating the annotation composition.

**Interviewer's follow-up:** "Where does `@ResponseBody` actually take effect?" In the
return-value handler selection — `RequestResponseBodyMethodProcessor` instead of
`ViewNameMethodReturnValueHandler`. The method already ran either way, which is why the
failure appears *after* your code.

---

### Q3 — "A client gets a 406. Another gets a 415. Explain both."

**Mid-level answer:** "406 means the content type isn't acceptable and 415 means the media
type isn't supported. Usually it's a missing or wrong `Accept` or `Content-Type` header."

**Senior answer:** "They're mirror images and the direction is the whole distinction.

**415 Unsupported Media Type** is about the **request**. The client sent a body with a
`Content-Type` that no `HttpMessageConverter` can read into my parameter type, or that my
method's `consumes` excludes. Common causes: `Content-Type: text/plain` on a JSON body, a
missing `Content-Type` entirely on a POST, or a `consumes` narrower than the client
expects.

**406 Not Acceptable** is about the **response**. Content negotiation intersects what the
client's `Accept` header asks for, what my method's `produces` declares, and what a
registered converter can write. Empty intersection is a 406.

The one worth knowing about is the *inverse* of a 406: a request with
`Accept: application/xml` succeeding and returning XML on an API you thought was JSON-only,
because `jackson-dataformat-xml` arrived transitively and registered a converter. Nothing in
my code changed and my API's output format did. That's why I declare
`produces = application/json` at class level on a JSON API — it makes the content type a
contract rather than a consequence of my dependency graph.

I'd also note that path-extension negotiation — `/products.json` — was removed in
Spring 6 for security reasons, so anything relying on it broke in the 2 to 3 migration."

**What separates them:** the direction as the organising idea, the transitive-converter
scenario (which shows they have actually been bitten), and volunteering the Spring 6
removal.

**Interviewer's follow-up:** "How do you see which converters are registered?"
`/actuator/beans` filtered on converter, or the condition evaluation report from
`--debug` — the JSON converter arrives from an auto-configuration guarded by
`@ConditionalOnClass` on the Jackson types.

---

### Q4 — "Should a controller return a JPA entity? Why not?"

**Mid-level answer:** "No, you should use DTOs. Entities can cause lazy loading issues and
you might expose fields you don't want to."

**Senior answer:** "No, and the reason is a timing fact rather than a style preference.

Serialization happens in a return-value handler **after** my method returns, which is after
the transaction has committed. So Jackson walks a live entity graph with no session. A lazy
`@OneToMany` becomes a `LazyInitializationException` wrapped in
`HttpMessageNotWritableException` — a 500.

The worse case is when it *doesn't* fail. `spring.jpa.open-in-view` defaults to true, which
holds the session open through serialization, so every lazy touch succeeds — by issuing a
query. One `GET /api/orders/{id}` becomes several hundred queries and nothing errors. Tests
pass. It falls over under load, and it also pins a connection for the whole request
including response rendering, which is a pool-exhaustion story at 1,800 rps. I set
`open-in-view: false` on every service, precisely so that mistake is loud in development
instead of silent in production.

Then the two design reasons. The entity is my schema: a column rename becomes a breaking API
change and a new column silently joins my public contract. And entities carry fields I must
not publish — cost price, fraud score. `@JsonIgnore` is a deny-list, and deny-lists fail on
the next field somebody adds.

So DTOs, mapped inside the transaction. For read-heavy endpoints I'd go further and use an
interface projection so the query only selects the columns the DTO needs — that's Topic 47
territory, and it avoids loading the entity at all."

**What separates them:** the *silent* open-in-view failure named as the dangerous one, the
connection-pinning consequence, allow-list versus deny-list framing, and reaching for
projections rather than stopping at "use a DTO".

**Interviewer's follow-up:** "What about accepting an entity as a `@RequestBody`?" Worse.
The client controls which fields get set, including `id` and `version`, so a crafted request
can overwrite rows or defeat optimistic locking. That is mass assignment, and it is a
security bug rather than a design smell.

---

### Q5 — "You need the tenant id from a header in forty controller methods. How do you do it?"

**Mid-level answer:** "I'd add `@RequestHeader("X-Tenant-Id") String tenantId` to each
method signature, or write a helper that pulls it out of `HttpServletRequest`."

**Senior answer:** "`@RequestHeader` on every method works, but it's forty places to forget
one — and a forgotten tenant check in a multi-tenant system is a cross-tenant data leak,
not a bug. I want the compiler and the framework to make it unforgettable.

I'd write a `HandlerMethodArgumentResolver` for a `@CurrentTenant TenantId` parameter.
`supportsParameter` matches the annotation and the type; `resolveArgument` reads the header,
validates it against a tenant registry, and returns a `TenantId` — a nominal type, not a
`String`, so it can't be confused with a SKU at a call site. A missing header throws, and
`@ControllerAdvice` maps that to a 400. Registered once via
`WebMvcConfigurer.addArgumentResolvers`.

That gives me four things: the dependency is visible in the method signature so a reader
knows the endpoint is tenant-scoped; it's typed; it's unit-testable without a request; and
it's one place to change when the header name changes.

The alternative is a filter putting the tenant in a `ThreadLocal`. It works and it's
sometimes right — a filter can enforce it for *every* request including ones I forget to
annotate. But it's invisible at the call site, and a `ThreadLocal` on a pooled Tomcat
thread that isn't cleared in a `finally` gives the next request the previous tenant's
identity. Same leak shape as MDC across `@Async`, except here it's a data breach.

If I needed both — universal enforcement and a clean signature — I'd use a filter to
*reject* requests with no tenant header, and an argument resolver to *supply* the typed
value. They're doing different jobs."

**What separates them:** treating the repetition as a security risk rather than a style
issue, the nominal-type argument, the `ThreadLocal` leak with its concrete consequence, and
the synthesis at the end that shows the two mechanisms are not competing.

**Interviewer's follow-up:** "How would you test the resolver?" A `@WebMvcTest` slice
(Topic 60) — it loads the web layer including your `WebMvcConfigurer`, so the resolver
participates, and `MockMvc` lets you assert 400 on a missing header. Note that the slice
loads a *smaller* set of auto-configurations than the full app, so a resolver registered by
a non-web configuration class may be absent — which is Topic 42's `@ImportAutoConfiguration`
showing up in a test.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Deserialization happens *before* your method and serialization happens *after* it. Name
   three distinct bugs that this single ordering fact explains, and say which dispatch step
   each one lives in.

2. `params` and `headers` on `@RequestMapping` participate in **matching**, so a request
   that fails them gets a 404 rather than a 400. Argue that this is the right design, then
   argue it is the wrong one. Which would you have chosen?

3. Spring 6 removed trailing-slash matching, breaking working applications on upgrade.
   Reconstruct the argument the Spring team must have made. What class of bug does "two
   URLs for one resource" actually cause — name one in caching and one in security.

4. `@EnableWebMvc` disables Boot's MVC auto-configuration entirely. Given that the
   annotation's name suggests "turn MVC on", why does it turn Boot's MVC *off*? Answer in
   terms of Topic 42's `@ConditionalOnMissingBean`.

5. In Node, a blocking call halts everything immediately and you notice in development. In
   Java it consumes one thread and you notice at 1,800 rps in production. Which failure
   mode would you rather have on a team of ten? Justify it as a *staffing and process*
   answer, not a technical one.

6. An argument resolver makes a dependency visible in the method signature; a filter plus a
   `ThreadLocal` makes it invisible but universal. Construct a requirement where the
   invisible one is unambiguously correct, and one where it is unambiguously wrong.

7. Your API returns XML because a transitive dependency added a converter. No code of yours
   changed. Whose defect is this — yours, the library's, or Spring's? Defend your answer,
   then say what you would change in your build to make the question not arise.

---

## Quick reference card

### The dispatch order

```
Tomcat -> Filter chain (Security) -> DispatcherServlet
  -> HandlerMapping        (route; no match = 404)
  -> HandlerInterceptor.preHandle
  -> HandlerAdapter
  -> ArgumentResolvers     (@RequestBody deserializes HERE; @Valid runs HERE)
  -> YOUR METHOD
  -> ReturnValueHandlers   (content negotiation + serialization HERE)
  -> HandlerInterceptor.postHandle / afterCompletion
  -> (on exception) HandlerExceptionResolver -> @ControllerAdvice
```

### Mapping

```java
@RestController                                   // @Controller + @ResponseBody
@RequestMapping("/api/orders")
@GetMapping @PostMapping @PutMapping @PatchMapping @DeleteMapping
@GetMapping(path="/{id}", produces=..., consumes=..., params="!draft", headers="X-V=2")
```

`params` / `headers` affect **matching** (404/405), not validation (400).

### Argument binding

```java
@PathVariable("id") String id           // required always; missing = no match = 404
@RequestParam(defaultValue="0") int p   // missing + no default = 400
@RequestParam(required=false) String c  // missing = null
@RequestParam Optional<String> q        // missing = Optional.empty()
@RequestBody @Valid CreateRequest r     // read by Content-Type; @Valid = 400 on violation
@RequestHeader("X-Tenant-Id") String t  // required by default = 400 if absent
@CookieValue("session") String s
@RequestPart("file") MultipartFile f
UriComponentsBuilder uriBuilder         // for Location headers
```

### Return values

```java
ProductResponse                                  // 200 + body
@ResponseStatus(HttpStatus.CREATED)              // constant status
ResponseEntity.created(location).body(x)         // 201 + Location
ResponseEntity.ok().eTag("\"7\"").body(x)        // 200 + ETag
ResponseEntity.noContent().build()               // 204
ResponseEntity.status(202).location(u).build()   // 202 Accepted
```

### Status codes the front desk produces for you

| Code | Cause |
|---|---|
| 400 | missing required param/header, type mismatch, unreadable body, `@Valid` violation |
| 404 | no `HandlerMapping` matched — including a trailing slash on Spring 6+ |
| 405 | path matched, method did not. Response carries `Allow`. |
| **406** | **response**: `Accept` ∩ `produces` ∩ converters = empty |
| **415** | **request**: `Content-Type` cannot be read into your parameter |
| 500 | your method threw, or serialization threw (check for `HttpMessageNotWritableException`) |

### Extension points

| I want to… | Use |
|---|---|
| resolve a custom parameter | `HandlerMethodArgumentResolver` |
| handle a custom return type | `HandlerMethodReturnValueHandler` |
| support a new media type | `HttpMessageConverter` |
| run before/after every handler | `HandlerInterceptor` |
| run before `DispatcherServlet` | a servlet `Filter` |
| map exceptions to responses | `@ControllerAdvice` (Topic 46) |
| register any of the above | `WebMvcConfigurer` — **never** `@EnableWebMvc` |

### Diagnostics

```bash
curl -i ...                                          # always -i; status and headers
curl -s :8080/actuator/mappings | jq .                # the real routing table
curl -s :8080/actuator/beans | jq '...Converter...'   # registered converters
javap -v -p Target.class | grep -A6 MethodParameters  # did -parameters survive?
```

```yaml
logging.level.org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping: TRACE
logging.level.org.springframework.web.servlet.DispatcherServlet: DEBUG
```

### Gotchas checklist

- [ ] Never accept or return a JPA entity. Ever. In either direction.
- [ ] `spring.jpa.open-in-view: false` in every profile.
- [ ] `/products/` is a **404** on Spring 6+.
- [ ] `@RestController`, not `@Controller`, or a returned `String` becomes a view name.
- [ ] `-parameters` must be on, or name every `@PathVariable`/`@RequestParam` explicitly.
- [ ] `@RequestParam(required=false) int` throws on a missing value. Use `Integer`.
- [ ] Declare `produces` at class level so a transitive converter cannot change your format.
- [ ] Never `@EnableWebMvc` in a Boot app.
- [ ] Server-enforced maximum page size. The client does not get to choose.
- [ ] `jakarta.servlet.*`, not `javax.servlet.*`.
- [ ] No blocking call to a slow downstream on the servlet thread without a timeout.

---

## When would I use this at work?

**1. A 404 on a URL that obviously exists.**
Instead of adding log statements and restarting, you hit `/actuator/mappings`, compare the
predicate against the request, and see the trailing slash or the `params` condition in
fifteen seconds. This is the most common day-to-day payoff and it works on someone else's
service as well as your own.

**2. Removing a repeated block from forty controllers.**
Tenant resolution, correlation IDs, idempotency keys, an `If-None-Match` check, an
`@AuthenticatedUser` parameter. Each is one argument resolver or one interceptor. Spotting
that a repeated block is a framework seam is a large part of what "senior" means in a Spring
codebase, and it is the difference between a review comment and a refactor.

**3. Reviewing a PR that returns an entity.**
You catch it in review with a concrete reason — "serialization runs after the transaction,
and with `open-in-view` off this is a 500; with it on it's 300 queries" — instead of "we
prefer DTOs". The first version changes the author's model. The second gets argued with.

---

## Connected topics

**Prerequisites:**

- **02 — Nominal typing**: why `TenantId` beats `String` in a resolver signature.
- **08–09 — Exceptions**: the exceptions the dispatch pipeline throws
  (`MethodArgumentNotValidException`, `HttpMessageNotReadableException`,
  `HttpMediaTypeNotAcceptableException`) and how your domain hierarchy maps onto them.
- **19 — Serialization**: Jackson, schema evolution, and why the API contract is a
  compatibility obligation.
- **26–27 — Optional and records**: `Optional<String>` request parameters, and records as
  request and response DTOs.
- **31–32 — Maven**: a transitive converter changing your content negotiation is a
  dependency-resolution story. `-parameters` is a compiler-plugin story.
- **35–37 — Container, scanning, lifecycle**: controllers are singletons found by component
  scanning. Singleton means **no mutable instance state in a controller** — every request
  thread shares it.
- **38 — Bean scopes**: request scope and scoped proxies, the alternative to an argument
  resolver.
- **39 — Injection styles**: constructor injection into controllers, `final` fields.
- **42 — Auto-configuration**: `WebMvcAutoConfiguration` supplies `DispatcherServlet`, the
  handler mappings and the message converters. `@EnableWebMvc` switches it off via
  `@ConditionalOnMissingBean`. Read the condition evaluation report when JSON stops working.
- **43 — Configuration**: `server.*`, `spring.mvc.*`, `spring.jackson.*` and the profile
  that turns request logging off before a measurement run.

**This unlocks:**

- **45 — Bean Validation**: `@Valid` on `@RequestBody` runs inside the argument resolver.
  Method-level constraints on a controller need `@Validated`. The immediate sequel.
- **46 — Error handling**: `@ControllerAdvice` and RFC 9457 `ProblemDetail` — turning every
  status code in this document into a stable, machine-readable contract.
- **47 — Spring Data JPA**: interface projections so a read endpoint never loads an entity.
- **49–50 — Lazy associations and N+1**: the mechanism behind Trap 1, in full.
- **55 / 109 — Transactions and the connection pool**: why a blocking call in a controller,
  and especially inside a transaction, is a whole-service outage rather than a slow
  endpoint.
- **56 — Spring Security**: the filter chain that runs *before* `DispatcherServlet`, and why
  that placement changes how authorisation is expressed.
- **60 — Test slices**: `@WebMvcTest` loads exactly this layer and nothing else — the
  fastest way to test a controller, a resolver or a converter.
- **62 — Contract testing**: replaces the `curl` script from Exercise 3 with an artefact the
  provider's build verifies.
- **65 — GATE**: the endpoints in Example 2 are what the load generator hammers, at the
  numbers stated there.
- **101 — Virtual threads**: the honest version of Trap 5's fix, on JDK 25.
- **103–108 — Reactive**: WebFlux replaces `DispatcherServlet` with `DispatcherHandler` and
  the same conceptual seams, without the thread-per-request model.
- **111 — Circuit breakers**: the correct handling for the slow downstream in Trap 5.
- **120 — Logging and MDC**: correlation IDs, the filter that sets them, and the
  `ThreadLocal` leak shape that also threatens tenant resolution.

---

*Java baseline 21, runtime JDK 25. Spring Boot 4.1 / Framework 7.0 / Jakarta EE 11.
The dispatch pipeline described here has been structurally stable since Spring 3 and is
shared, with different component names, by WebFlux. Two Boot 4 features — built-in API
versioning and Jackson 3 — are referenced by name only: I have deliberately not invented
their exact property keys, attribute names or package names, and you should read your
resolved version's reference documentation before using either. No Spring artifact version
numbers appear in this document; the Boot BOM supplies them.*
