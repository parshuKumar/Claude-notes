# 46 — `@ControllerAdvice`, Error Handling, and RFC 9457 `ProblemDetail`

## Phase: 5 — Spring Boot & Persistence
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: a uniform `ProblemDetail` error contract for `orderflow` — one advice class that maps the Topic 09 domain exception hierarchy onto HTTP status codes, with a body shape that never changes between endpoints and never leaks an entity or a stack trace

---

## ELI5 anchor

A shop gives you a receipt when things go right.

When things go wrong, most shops hand you *whatever*. One till prints "ERROR 4".
Another prints a paragraph of apology. A third hands you the internal stock-system
printout with the manager's phone number on it.

Now imagine you are a machine reading those. You cannot. Every till speaks a
different language, so you have to write three parsers, and a fourth one next month
when someone installs a new till.

An **error contract** is the rule that says: when something goes wrong, every till
in the chain prints the same shaped slip. Same fields, same order, same meaning.
One parser works everywhere. And the slip never contains the wiring diagram of the
building.

`@ControllerAdvice` is the one place in a Spring app where you print that slip.
`ProblemDetail` is the agreed shape of the slip.

---

## The bridge from what you know

### `@ControllerAdvice` ≈ Nest exception filters — **HONEST ANALOGUE**

You already have this. In NestJS:

```ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse();
    res.status(exception.getStatus()).json({ message: exception.message });
  }
}
```

In Spring:

```java
@RestControllerAdvice
public class OrderflowExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    public ProblemDetail handleProductNotFound(ProductNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }
}
```

Same job. Same position in the request pipeline: after your handler throws, before
the response is written. `@Catch(SomeException)` maps to
`@ExceptionHandler(SomeException.class)`. A global filter maps to a
`@ControllerAdvice` with no narrowing attributes.

**Transfer your intuition directly.** There is nothing to unlearn here. Move on.

### What is actually new

Three things, and only one of them is Spring-specific.

| Thing | What is new |
|---|---|
| **RFC 9457 `ProblemDetail`** | A standardised, registered media type (`application/problem+json`) with defined field names. Nest has no built-in equivalent — you invent your body shape per project. Java has a class in the framework and an IETF document behind it. |
| **The exception-to-status mapping lives in one file** | Nest lets you throw `new NotFoundException()` from a service, which couples your domain layer to HTTP. Spring's idiom is a domain exception that knows nothing about HTTP, plus one advice that does the mapping. |
| **Exceptions thrown *before* your dispatcher exist** | The Spring Security filter chain (Topic 56) runs before `DispatcherServlet`. An exception there never reaches your `@ControllerAdvice`. In Nest, guards run *inside* the framework pipeline, so your filter sees them. This is the one place your Nest intuition is actively wrong, and it is Trap 5 below. |

### The translation table row

| You know | Java | Verdict |
|---|---|---|
| Nest exception filters | `@ControllerAdvice` + `@ExceptionHandler` | **HONEST ANALOGUE** |
| `HttpException` thrown from a service | Domain exception + advice mapping | **PARTIAL** — the Java idiom deliberately keeps HTTP out of the domain layer |
| Ad-hoc `{ message, statusCode }` body | `ProblemDetail` per RFC 9457 | **NO ANALOGUE** — a specified wire format, not a convention |
| `class-validator` errors auto-formatted by Nest | `MethodArgumentNotValidException` you must format yourself | **PARTIAL** — Spring gives you a default, and the default is not good enough |

---

## What is this?

### RFC 9457 in one paragraph

RFC 9457, *Problem Details for HTTP APIs*, defines a JSON object for describing an
error, served as `Content-Type: application/problem+json`. It obsoletes RFC 7807,
which was the same idea with less guidance; the media type did not change, so
anything that understood 7807 understands 9457.

The document defines five members:

| Member | Type | Meaning |
|---|---|---|
| `type` | URI string | **The stable identifier of the error kind.** This is the field machines branch on. Defaults to `"about:blank"`. |
| `title` | string | A short human-readable summary of the *kind*. Should not change per occurrence. |
| `status` | integer | The HTTP status code, duplicated into the body. |
| `detail` | string | Human-readable explanation of *this specific occurrence*. |
| `instance` | URI string | Identifies this specific occurrence — often the request path or a correlation ID URI. |

Anything else you add is an **extension member**, and extensions are explicitly
allowed. That is where `errors`, `correlationId` and `traceId` go.

**The single most important sentence in the whole RFC, for your purposes:**
consumers must branch on `type`, not on `title` or `detail`. `title` and `detail`
are for humans and may be reworded or translated at any time. `type` is the API
contract and must never change once published.

### Spring's implementation

Spring Framework 6.0 added `org.springframework.http.ProblemDetail` — a plain
mutable carrier for those five fields plus a `Map` of extensions. Spring Framework
7.0 (your target) has it unchanged in the parts that matter.

Three related types you will meet:

- **`ProblemDetail`** — the body. Not an exception. Just data.
- **`ErrorResponse`** — an interface meaning "I can produce a status, headers and a
  `ProblemDetail`". Spring's own exceptions implement it.
- **`ErrorResponseException`** — a ready-made `RuntimeException` that implements
  `ErrorResponse`, so you can throw a fully-formed problem from anywhere.

### Where in the request does this happen?

You learned the `DispatcherServlet` loop in Topic 44. Error handling hangs off the
end of it:

```
Request
  -> Servlet filters            (Spring Security lives HERE - Topic 56)
  -> DispatcherServlet
       -> HandlerMapping        -> which controller method?
       -> HandlerAdapter        -> argument resolvers, invoke your method
            *** your method throws ***
       -> HandlerExceptionResolver chain:
            1. ExceptionHandlerExceptionResolver   <- finds your @ExceptionHandler
            2. ResponseStatusExceptionResolver     <- handles @ResponseStatus / ResponseStatusException
            3. DefaultHandlerExceptionResolver     <- Spring MVC's own exceptions
       -> if none resolved it: forward to /error -> BasicErrorController (the whitelabel page)
  -> Response written
```

Two facts to keep from that diagram:

1. **Your advice is resolver number one.** It wins over everything else.
2. **The filter layer is outside the whole box.** An exception in a filter never
   enters this chain.

---

## Why does it matter?

**1. Error responses are an API surface with the same compatibility obligations as
the success path.**

You would never rename a field in `GET /orders/{id}` without a version bump. Teams
rename error fields weekly, because errors feel like debugging output rather than
API. They are not. A client's retry logic, its circuit breaker, and its "show the
user a friendly message" branch all read your error body. Change its shape and you
break them, silently, with a 200-level deploy.

**2. The default is a leak.**

If nothing handles an exception, Boot forwards to `/error`. Depending on your
properties, that response can contain the exception class name, the message, and
the stack trace. The exception message frequently contains data —
`"could not execute statement; constraint [uq_wallet_customer_id]"` tells an
attacker your schema. A Hibernate `LazyInitializationException` message names your
entity classes and field names.

**3. It is the cheapest possible operational win.**

One advice class means: every 4xx and 5xx in the service carries a correlation ID,
a stable `type`, and a `status`. Your on-call runbook can say "search logs for the
`correlationId` in the problem body" and it works for every endpoint. Without it,
that sentence has to be written per endpoint and is wrong for half of them.

---

## Syntax breakdown

### `@ControllerAdvice` and `@RestControllerAdvice`

```java
@RestControllerAdvice
public class OrderflowExceptionHandler { }
```

| Bit | What it means |
|---|---|
| `@ControllerAdvice` | Marks a bean whose `@ExceptionHandler` / `@InitBinder` / `@ModelAttribute` methods apply across **many** controllers, not just one. It is itself a `@Component`, so component scanning finds it (Topic 36). |
| `@RestControllerAdvice` | `@ControllerAdvice` + `@ResponseBody`. Return values are serialized to the response body instead of treated as view names. **This is the one you want** for a JSON API. |

Narrowing which controllers an advice applies to:

```java
@RestControllerAdvice(basePackages = "com.orderflow.payments")     // by package
@RestControllerAdvice(assignableTypes = { OrderController.class }) // by controller class
@RestControllerAdvice(annotations = InternalApi.class)             // by annotation on the controller
```

Multiple advices are allowed and are consulted in `@Order` order. Lower value wins.
An advice with no narrowing attributes applies to everything.

### `@ExceptionHandler`

```java
@ExceptionHandler({ ProductNotFoundException.class, OrderNotFoundException.class })
public ProblemDetail handleNotFound(RuntimeException ex, HttpServletRequest request) {
    ...
}
```

| Bit | What it means |
|---|---|
| The `value` array | Which exception types this method handles. If omitted, Spring infers them from the method's parameter types. |
| Matching rule | **Most specific type wins.** If you have a handler for `Exception` and one for `ProductNotFoundException`, the specific one is chosen. This is resolved by walking the exception's class hierarchy, not by declaration order. |
| Parameters | You may declare the exception, plus `HttpServletRequest`, `WebRequest`, `HttpHeaders`, `Locale`, `HandlerMethod` and more. Spring resolves them the same way it resolves controller-method arguments. |
| Return type | `ProblemDetail`, `ResponseEntity<ProblemDetail>`, `ErrorResponse`, a DTO, or `void`. Returning `ResponseEntity` is how you set custom headers such as `Retry-After`. |
| Cause unwrapping | If the thrown exception has no handler, Spring also checks its `getCause()` chain. Useful, and occasionally surprising. |

### `ProblemDetail`

```java
ProblemDetail problem = ProblemDetail.forStatusAndDetail(
        HttpStatus.CONFLICT,
        "Wallet balance 1250 is below the required 4999");

problem.setType(URI.create("https://errors.orderflow.com/insufficient-funds"));
problem.setTitle("Insufficient funds");
problem.setInstance(URI.create("/api/orders"));
problem.setProperty("correlationId", correlationId);   // extension member
problem.setProperty("walletId", walletId);             // extension member
```

| Factory | Use |
|---|---|
| `ProblemDetail.forStatus(HttpStatusCode)` | Status only. `title` is derived from the status reason phrase. |
| `ProblemDetail.forStatusAndDetail(HttpStatusCode, String)` | The one you will use 90% of the time. |

`setProperty(String, Object)` adds an extension member, serialized as a top-level
JSON key alongside the five standard ones. Setting a property to `null` removes it.

The serialized shape (this is an **illustration of the format**, not captured
output):

```json
{
  "type": "https://errors.orderflow.com/insufficient-funds",
  "title": "Insufficient funds",
  "status": 409,
  "detail": "Wallet balance 1250 is below the required 4999",
  "instance": "/api/orders",
  "correlationId": "9f2c1b7e-4a3d-4f21-8c6b-2e1a5d0c7f44",
  "walletId": 88213
}
```

### `ErrorResponseException`

When you want to throw a complete problem from a place that is not worth giving its
own domain exception:

```java
throw new ErrorResponseException(HttpStatus.SERVICE_UNAVAILABLE);
```

or with a body you build:

```java
ProblemDetail body = ProblemDetail.forStatusAndDetail(
        HttpStatus.SERVICE_UNAVAILABLE, "Payment gateway unreachable");
body.setType(URI.create("https://errors.orderflow.com/gateway-unavailable"));
throw new ErrorResponseException(HttpStatus.SERVICE_UNAVAILABLE, body, null);
```

It implements `ErrorResponse`, so Spring renders it without any advice at all.
Useful — but for a domain error that appears in more than one place, a named domain
exception plus one advice method is better, because the *mapping* stays in one file.

### `ResponseEntityExceptionHandler`

An abstract base class that already has `@ExceptionHandler` methods for Spring
MVC's own exceptions — `MethodArgumentNotValidException`,
`HttpMessageNotReadableException`, `HttpRequestMethodNotSupportedException`,
`NoResourceFoundException` and about a dozen others — each producing a
`ProblemDetail`.

```java
@RestControllerAdvice
public class OrderflowExceptionHandler extends ResponseEntityExceptionHandler {

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {
        // customise the validation error body here
    }
}
```

Extending it is the difference between "my domain errors are RFC 9457 and my
framework errors are something else" and "everything is RFC 9457".

### The property that turns on the defaults

```properties
spring.mvc.problemdetails.enabled=true
```

This makes Boot register a `ProblemDetail`-producing handler for Spring MVC's
built-in exceptions **without** you extending `ResponseEntityExceptionHandler`.
Off by default, because turning it on changes the body shape of existing errors —
which is exactly the compatibility point made above.

> **Pick one, not both.** If you extend `ResponseEntityExceptionHandler` you are
> already covering those exceptions. Enabling the property as well gives you two
> registered handlers for the same types; yours wins, but the configuration now
> lies about intent.

### The properties that control the leak

```properties
server.error.include-message=never          # default: never    <- keep it
server.error.include-stacktrace=never       # default: never    <- keep it
server.error.include-binding-errors=never   # default: never
server.error.include-exception=false        # default: false
```

These govern the `/error` fallback (`BasicErrorController`), not your advice. They
are safe by default. People break them by setting `always` while debugging and
committing it. See Trap 1.

### `[BOOT 3.x DELTA]`

- `ProblemDetail`, `ErrorResponse` and `ErrorResponseException` all arrived in
  **Spring Framework 6.0 / Boot 3.0**. Everything in this document works on 3.x.
- `spring.mvc.problemdetails.enabled` also exists on 3.x. Same key.
- On **Boot 2.x** none of this exists. The community answer was Zalando's
  `problem-spring-web` library, or a hand-rolled `ApiError` DTO. If you interview
  at a shop on 2.7 and they show you a hand-rolled error DTO, the right answer is
  "that is RFC 7807 reinvented; on Boot 3+ it is a framework type".
- The RFC number changed (7807 → 9457) but the media type `application/problem+json`
  did not. Do not "upgrade" the media type; there is nothing to upgrade.

---

## Example 1 — minimal

A single endpoint, a single domain exception, a single advice.

**The domain exception** — note it knows nothing about HTTP. This is Topic 09's
design applied.

```java
package com.orderflow.catalog;

public class ProductNotFoundException extends RuntimeException {

    private final String sku;

    public ProductNotFoundException(String sku) {
        super("No product with SKU " + sku);
        this.sku = sku;
    }

    public String getSku() {
        return sku;
    }
}
```

**The controller** — no try/catch, no HTTP concerns beyond the mapping.

```java
package com.orderflow.catalog;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService products;

    ProductController(ProductService products) {
        this.products = products;
    }

    @GetMapping("/{sku}")
    public ProductResponse getBySku(@PathVariable String sku) {
        return products.findBySku(sku)                       // Optional<Product>, Topic 26
                       .map(ProductResponse::from)
                       .orElseThrow(() -> new ProductNotFoundException(sku));
    }
}
```

**The advice.**

```java
package com.orderflow.api;

@RestControllerAdvice
public class OrderflowExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    ProblemDetail handleProductNotFound(ProductNotFoundException ex) {
        ProblemDetail problem =
                ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setType(URI.create("https://errors.orderflow.com/product-not-found"));
        problem.setTitle("Product not found");
        problem.setProperty("sku", ex.getSku());
        return problem;
    }
}
```

Three properties of this that are worth naming:

1. `ProductService` returns `Optional<Product>` and the *controller* decides that
   absence is a 404. The service does not know what HTTP is.
2. The `sku` travels as a typed field on the exception and lands as a typed
   extension member. The client does not have to regex it out of `detail`.
3. `type` is a URL under a domain you control. It does not have to resolve to
   anything — but if it does, it should be a page documenting that error. That page
   is the cheapest API documentation you will ever write.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` in production, at the Topic 65 baseline:

- 100,000 products, 1,000,000 orders, 5,000,000 order lines in Postgres.
- Sustained traffic: ~400 rps catalogue reads, ~120 rps order reads, ~60 rps order
  placements. Peak is 3× that for 20 minutes around a promotion.
- Six client teams consume the API: the web storefront, the iOS app, the Android
  app, a partner integration, an internal admin console, and a reconciliation batch
  job.
- **SLO:** p99 latency under 250 ms for reads, under 600 ms for order placement;
  error rate under 0.5% excluding client 4xx.
- **Hard requirement from the security review:** no response body may contain a
  stack trace, a SQL fragment, a database constraint name, a class name, or any
  field of a JPA entity that is not in a published DTO.

At 60 order placements per second, an insufficient-funds rejection rate of even 2%
is 1.2 rejections per second — about 100,000 per day. Every one of those is a
response the iOS app has to categorise. If it cannot, it shows "Something went
wrong" and the user calls support. That is the business case for a stable `type`.

### The domain exception hierarchy (Topic 09)

```java
package com.orderflow.shared;

/** Base for every business-rule failure in orderflow. Carries no HTTP knowledge. */
public abstract class OrderflowException extends RuntimeException {

    protected OrderflowException(String message) {
        super(message);
    }

    protected OrderflowException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

```java
package com.orderflow.orders;

public class InsufficientStockException extends OrderflowException {

    private final String sku;
    private final int requested;
    private final int available;

    public InsufficientStockException(String sku, int requested, int available) {
        super("Requested " + requested + " of " + sku + " but only " + available + " available");
        this.sku = sku;
        this.requested = requested;
        this.available = available;
    }

    public String getSku()      { return sku; }
    public int getRequested()   { return requested; }
    public int getAvailable()   { return available; }
}
```

The others follow the same shape: `InsufficientFundsException` (wallet), 
`OrderNotFoundException`, `PaymentDeclinedException`, `DuplicateOrderException`,
`OrderNotCancellableException`.

**Why typed fields and not just a message:** the iOS team wants to show
"Only 2 left" in the UI. If `available` is only inside the `detail` string, they
will parse the string. Then you reword the string and their app breaks. Extension
members exist to stop that.

### The error catalogue

Write this table *before* the code. It is the actual deliverable; the advice class
is just its implementation.

| Domain exception | Status | `type` (stable, never changes) | Extension members |
|---|---|---|---|
| `ProductNotFoundException` | 404 | `.../product-not-found` | `sku` |
| `OrderNotFoundException` | 404 | `.../order-not-found` | `orderId` |
| `InsufficientStockException` | 409 | `.../insufficient-stock` | `sku`, `requested`, `available` |
| `InsufficientFundsException` | 409 | `.../insufficient-funds` | `requiredMinor`, `availableMinor`, `currency` |
| `OrderNotCancellableException` | 409 | `.../order-not-cancellable` | `orderId`, `currentStatus` |
| `DuplicateOrderException` | 409 | `.../duplicate-order` | `idempotencyKey`, `existingOrderId` |
| `PaymentDeclinedException` | 402 | `.../payment-declined` | `declineCode` |
| `MethodArgumentNotValidException` | 400 | `.../validation-failed` | `errors[]` |
| `OptimisticLockingFailureException` | 409 | `.../concurrent-modification` | — (client should retry) |
| anything else | 500 | `.../internal` | — (nothing; see below) |

Two decisions in that table worth defending in a review:

- **409 for insufficient stock, not 400.** The request was well-formed. The
  *current state of the resource* makes it impossible. That is the definition of
  409 Conflict. A 400 tells the client "you sent something malformed", which sends
  them debugging their own serializer.
- **402 for a declined payment.** Rarely used, but it is exactly what it means, and
  it lets a client distinguish "your card was refused" from "you have no money in
  your wallet" (409) without reading `type`. Both are still distinguished by `type`;
  the status is a coarser hint for generic middleware.

### The advice

```java
package com.orderflow.api;

import com.orderflow.orders.*;
import com.orderflow.payments.PaymentDeclinedException;
import com.orderflow.catalog.ProductNotFoundException;
import com.orderflow.wallet.InsufficientFundsException;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.net.URI;
import java.util.List;

@RestControllerAdvice
public class OrderflowExceptionHandler extends ResponseEntityExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(OrderflowExceptionHandler.class);
    private static final String ERROR_BASE = "https://errors.orderflow.com/";

    private final CorrelationIdProvider correlationIds;   // request-scoped, Topic 38

    OrderflowExceptionHandler(CorrelationIdProvider correlationIds) {
        this.correlationIds = correlationIds;
    }

    // ---------- 404 ----------

    @ExceptionHandler({ ProductNotFoundException.class, OrderNotFoundException.class })
    ProblemDetail handleNotFound(OrderflowException ex, HttpServletRequest request) {
        String slug = (ex instanceof ProductNotFoundException) ? "product-not-found" : "order-not-found";
        ProblemDetail problem = problem(HttpStatus.NOT_FOUND, "Resource not found", slug,
                                        ex.getMessage(), request);
        if (ex instanceof ProductNotFoundException pnf) {
            problem.setProperty("sku", pnf.getSku());
        } else if (ex instanceof OrderNotFoundException onf) {
            problem.setProperty("orderId", onf.getOrderId());
        }
        return problem;
    }

    // ---------- 409 ----------

    @ExceptionHandler(InsufficientStockException.class)
    ProblemDetail handleInsufficientStock(InsufficientStockException ex, HttpServletRequest request) {
        ProblemDetail problem = problem(HttpStatus.CONFLICT, "Insufficient stock",
                                        "insufficient-stock", ex.getMessage(), request);
        problem.setProperty("sku", ex.getSku());
        problem.setProperty("requested", ex.getRequested());
        problem.setProperty("available", ex.getAvailable());
        return problem;
    }

    @ExceptionHandler(InsufficientFundsException.class)
    ProblemDetail handleInsufficientFunds(InsufficientFundsException ex, HttpServletRequest request) {
        ProblemDetail problem = problem(HttpStatus.CONFLICT, "Insufficient funds",
                                        "insufficient-funds", ex.getMessage(), request);
        problem.setProperty("requiredMinor", ex.getRequiredMinor());
        problem.setProperty("availableMinor", ex.getAvailableMinor());
        problem.setProperty("currency", ex.getCurrency());
        return problem;
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    ResponseEntity<ProblemDetail> handleConcurrentModification(
            OptimisticLockingFailureException ex, HttpServletRequest request) {

        // Deliberately does NOT use ex.getMessage(): Spring's message names the entity
        // class and its identifier. That is an internal detail. See Trap 3.
        ProblemDetail problem = problem(HttpStatus.CONFLICT, "Concurrent modification",
                                        "concurrent-modification",
                                        "The resource was modified concurrently. Retry the request.",
                                        request);
        return ResponseEntity.status(HttpStatus.CONFLICT)
                             .header(HttpHeaders.RETRY_AFTER, "1")
                             .body(problem);
    }

    // ---------- 402 ----------

    @ExceptionHandler(PaymentDeclinedException.class)
    ProblemDetail handlePaymentDeclined(PaymentDeclinedException ex, HttpServletRequest request) {
        ProblemDetail problem = problem(HttpStatus.PAYMENT_REQUIRED, "Payment declined",
                                        "payment-declined", ex.getMessage(), request);
        problem.setProperty("declineCode", ex.getDeclineCode());
        return problem;
    }

    // ---------- 400: validation, overriding the framework's version ----------

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        List<FieldViolation> violations = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> new FieldViolation(fe.getField(), fe.getDefaultMessage()))
                .toList();

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.BAD_REQUEST, "One or more fields are invalid");
        problem.setType(URI.create(ERROR_BASE + "validation-failed"));
        problem.setTitle("Validation failed");
        problem.setProperty("errors", violations);
        problem.setProperty("correlationId", correlationIds.current());

        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                             .contentType(MediaType.APPLICATION_PROBLEM_JSON)
                             .body(problem);
    }

    /** Records as the projection shape for a violation. Topic 27. */
    record FieldViolation(String field, String message) { }

    // ---------- 500: the catch-all ----------

    @ExceptionHandler(Exception.class)
    ProblemDetail handleUnexpected(Exception ex, HttpServletRequest request) {
        String correlationId = correlationIds.current();

        // The stack trace goes to the LOG, with the correlation id.
        // It does NOT go to the response body. This split is the whole point.
        log.error("Unhandled exception correlationId={} path={}",
                  correlationId, request.getRequestURI(), ex);

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR,
                "An unexpected error occurred. Quote the correlation id when contacting support.");
        problem.setType(URI.create(ERROR_BASE + "internal"));
        problem.setTitle("Internal server error");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("correlationId", correlationId);
        return problem;
    }

    // ---------- shared builder ----------

    private ProblemDetail problem(HttpStatus status, String title, String typeSlug,
                                  String detail, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(status, detail);
        problem.setType(URI.create(ERROR_BASE + typeSlug));
        problem.setTitle(title);
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("correlationId", correlationIds.current());
        return problem;
    }
}
```

### What this design buys you, stated concretely

- **The iOS team writes one switch.** On `type`, with a `default` branch that shows
  the generic message. Adding a new error type never crashes an old client.
- **The 500 body is information-free by construction.** It carries a correlation ID
  and nothing else. There is no code path that can put a stack trace in it, because
  the only 500 handler builds the body from literals.
- **Support has a one-step lookup.** "Give me the correlation ID from the error"
  → `grep` the logs → the full stack trace is there.
- **The error catalogue is testable.** One parameterised test (Topic 58) asserts
  that every `OrderflowException` subclass has a handler and a distinct `type`.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — leaking a stack trace or an internal message

**Wrong:**

```properties
# committed during a debugging session in March, never reverted
server.error.include-stacktrace=always
server.error.include-message=always
```

or, in an advice:

```java
@ExceptionHandler(Exception.class)
ProblemDetail handleAll(Exception ex) {
    return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, ex.getMessage());   // <- the leak
}
```

**Exact symptom:** a 500 response body containing a `trace` field several kilobytes
long, or a `detail` field reading something like

```
could not execute statement [ERROR: duplicate key value violates unique constraint
"uq_wallet_customer_id"]; SQL [insert into wallet ...]
```

You usually find this in a penetration-test report, or when a client team pastes a
raw error into a public Slack channel and it contains your table names.

**Root cause:** exception messages are written for developers. They routinely
contain SQL, constraint names, class names, file paths, and — with Hibernate — the
parameter values that were bound, which can include personal data.

**Fix:** keep `server.error.include-*` at their safe defaults, and in the catch-all
handler build `detail` from a **literal string**, never from `ex.getMessage()`.
Domain exceptions you wrote may use their own message, because you control it. Any
exception you did not write may not. Add a test that fires a forced 500 and asserts
the body has exactly the keys `type`, `title`, `status`, `detail`, `instance`,
`correlationId`.

---

### Trap 2 — an error contract that changes shape between endpoints

**Wrong:** three shapes in one service, because three people wrote three endpoints.

```java
// OrderController
return ResponseEntity.badRequest().body(Map.of("error", "invalid quantity"));

// PaymentController
return ResponseEntity.badRequest().body(new ApiError(400, "invalid amount", now()));

// nothing at all in WalletController, so Boot's /error fallback answers:
// { "timestamp": ..., "status": 400, "error": "Bad Request", "path": "/api/wallet" }
```

**Exact symptom:** client code that looks like this, in every consumer:

```ts
const msg = body.error ?? body.message ?? body.detail ?? body.title ?? "Unknown error";
```

and a bug report saying "the Android app shows 'Unknown error' for wallet failures
but the right message for order failures". Nobody logs an incident for it; it just
degrades forever.

**Root cause:** no single place owns the error body, so it is invented per handler.

**Fix:** one `@RestControllerAdvice`, extending `ResponseEntityExceptionHandler` so
framework exceptions are covered too. Then delete every `ResponseEntity.badRequest()`
in every controller — controllers throw, the advice renders. Enforce it with an
ArchUnit rule or a code-review checklist item: *no controller returns a 4xx/5xx
`ResponseEntity` directly.*

---

### Trap 3 — putting an entity in the error body

**Wrong:**

```java
@ExceptionHandler(InsufficientStockException.class)
ProblemDetail handle(InsufficientStockException ex) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, "No stock");
    problem.setProperty("product", ex.getProduct());   // <- a JPA entity
    return problem;
}
```

**Exact symptom:** one of three, depending on your luck:

1. The response contains `costPriceMinor`, `supplierId` and `internalNotes` — fields
   that exist on the `Product` entity and were never meant to leave the building.
2. A `LazyInitializationException` **inside the exception handler**, producing a 500
   from your 409 handler, because Jackson touched a lazy association after the
   transaction closed. That is Topic 49, and it is genuinely confusing to debug
   because the stack trace points at serialization, not at your code.
3. An enormous response, because `Product` → `Inventory` → `Product` serialized the
   whole reachable graph, or a `StackOverflowError` from an infinite recursion on a
   bidirectional association.

**Root cause:** an entity is a *persistence* type. It has a lifecycle, lazy proxies,
and fields chosen for storage rather than for publication. Putting one anywhere near
Jackson couples your wire format to your schema.

**Fix:** extension members are **scalars and records only**. `sku` (a `String`),
`available` (an `int`). If you genuinely need a structure, define a `record`
(Topic 27) for it, as `FieldViolation` above does. Never a `@Entity`.

---

### Trap 4 — the catch-all that eats your 4xx

**Wrong:**

```java
@ExceptionHandler(Exception.class)
ProblemDetail handleAll(Exception ex) { /* returns 500 */ }
```

...written first, with the specific handlers added later in a *different* advice
class that has no `@Order`.

**Exact symptom:** `GET /api/products/NONEXISTENT` returns **500**, not 404. Your
error-rate dashboard shows a 4% 5xx rate that is entirely client mistakes. Your SLO
burn alert fires at 3am for a non-incident. Worse: the alert becomes noise, and you
stop trusting it.

**Root cause:** two things can cause this and you must distinguish them.

- *Within one advice class*, Spring picks the **most specific** matching handler, so
  a `ProductNotFoundException` handler beats the `Exception` handler. This is fine.
- *Across two advice classes*, the class with the lower `@Order` value is consulted
  first, and if it produces a result, the search stops. An unordered advice gets
  `Ordered.LOWEST_PRECEDENCE`, so the winner between two unordered advices is
  effectively undefined.

**Fix:** put every handler in one advice class if you can. If you genuinely need
several (for example a separate advice for `/internal/**`), give each an explicit
`@Order`, with the catch-all advice at `@Order(Ordered.LOWEST_PRECEDENCE)`. Then
write the test: assert a 404 for a missing product. That test is what stops the
regression.

---

### Trap 5 — assuming your advice sees security failures

**Wrong:** believing that a `@ControllerAdvice` covers "all errors from my API",
because that is what a Nest global exception filter does.

**Exact symptom:** every business error returns a tidy `application/problem+json`
body, but a request with an expired JWT returns a **completely different** shape —
often an empty body with a `WWW-Authenticate` header, or Boot's default
`/error` JSON — with `Content-Type: application/json`, not `application/problem+json`.
The client's error parser, which was written against your problem shape, fails on
exactly the case it most needs to handle: "your token expired, re-authenticate".

**Root cause:** Spring Security is a **servlet filter chain** (Topic 56). It runs
*before* `DispatcherServlet`. An `AuthenticationException` or `AccessDeniedException`
thrown in a filter never enters the `HandlerExceptionResolver` chain, so no
`@ExceptionHandler` can ever see it. Your Nest intuition — guards run inside the
framework pipeline, so filters catch their errors — does not hold here.

**Fix:** configure Security's own hooks, and have them produce the *same*
`ProblemDetail` shape:

```java
http.exceptionHandling(ex -> ex
        .authenticationEntryPoint(problemAuthenticationEntryPoint)   // 401
        .accessDeniedHandler(problemAccessDeniedHandler));           // 403
```

Both write a `ProblemDetail` to the response with the same `type` scheme. Share the
builder between them and the advice so there is one implementation. This is
finished properly in Topic 57; know today that the gap exists.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM and no running service, so
I will not print output and call it real. What follows is exactly what to run, what
to look at, and how to read every result you might get.

### Setup

```bash
cd ~/orderflow
./mvnw -q spring-boot:run
# in another terminal:
java --version     # expect 21 or 25
./mvnw -q dependency:list | grep -i "spring-boot:jar"   # confirm your Boot version
```

Knowing your exact Boot version matters for the `[BOOT 3.x DELTA]` notes. Do not
take my word for which line you are on — read it from the build.

### Proof 1 — the Content-Type is the contract

```bash
curl -i http://localhost:8080/api/products/NO-SUCH-SKU
```

**What to look for:** the status line and the `Content-Type` header.

| What you see | What it means |
|---|---|
| `HTTP/1.1 404` and `Content-Type: application/problem+json` | Your advice ran and returned a `ProblemDetail`. Spring sets the media type automatically for a `ProblemDetail` return value. Correct. |
| `HTTP/1.1 404` and `Content-Type: application/json` | You returned a hand-rolled DTO or a `Map`, not a `ProblemDetail`, or you overrode the content type. Clients doing content negotiation on `problem+json` will not recognise it. |
| `HTTP/1.1 500` | Either no handler matched and the catch-all took it (check the logs), or your handler itself threw. Handler exceptions are the sneaky case — see Trap 3. |
| `HTTP/1.1 200` with an empty body | Your service returned `Optional.empty()` serialized, rather than throwing. The `orElseThrow` is missing. |
| A full HTML page | The whitelabel error page. Nothing handled the exception and content negotiation chose HTML because `curl` sent `Accept: */*` and no advice claimed it. |

### Proof 2 — the body has the five standard members

```bash
curl -s http://localhost:8080/api/products/NO-SUCH-SKU | jq 'keys'
```

**What to look for:** the key list.

| What you see | What it means |
|---|---|
| Contains `type`, `title`, `status`, `detail`, `instance` | RFC 9457 compliant. |
| `type` is `"about:blank"` | You did not call `setType`. Legal per the RFC, but useless — every error looks identical to a machine. Fix it. |
| Contains `trace` or `exception` | A stack trace or class name is leaking. This is Trap 1. Check `server.error.include-stacktrace`. |
| Contains `timestamp`, `error`, `path` but not `type` | This is Boot's `/error` fallback body, not your advice. Nothing handled the exception. |

### Proof 3 — force a 500 and prove nothing leaks

Add a temporary endpoint on a non-production profile only:

```java
@Profile("local")
@RestController
class BlowUpController {
    @GetMapping("/internal/blow-up")
    String blowUp() {
        throw new IllegalStateException("secret: connection string user=orderflow_rw");
    }
}
```

```bash
curl -s http://localhost:8080/internal/blow-up | jq .
```

**What to look for:** whether the string `secret:` appears anywhere in the body.

| What you see | What it means |
|---|---|
| No `secret:`, only `type`/`title`/`status`/`detail`/`instance`/`correlationId` | Your catch-all builds `detail` from a literal. This is the desired result. |
| `secret:` appears in `detail` | Your catch-all uses `ex.getMessage()`. Trap 1. Fix before anything else. |
| A `trace` array | `server.error.include-stacktrace` is not `never`, or you serialized the exception. |

Then confirm the information went to the right place:

```bash
grep "correlationId=" logs/orderflow.log | tail -1
```

The stack trace should be in the log, tied to the same correlation ID that came back
in the body. **That is the test.** The information is not destroyed; it is moved to
the channel that is authenticated and audited.

### Proof 4 — see which resolver handled it

```properties
logging.level.org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver=TRACE
logging.level.org.springframework.web.servlet.DispatcherServlet=DEBUG
```

Restart, re-run Proof 1.

**What to look for:** a log line naming the handler method that was selected.

| What you see | What it means |
|---|---|
| A line naming `OrderflowExceptionHandler#handleProductNotFound` | Your specific handler was chosen. |
| A line naming `#handleUnexpected` | The catch-all won. Either you have no specific handler, or it is in a lower-priority advice — Trap 4. |
| No `ExceptionHandlerExceptionResolver` line at all, then a forward to `/error` | No advice matched. The exception may be from a filter (Trap 5) or your advice is not being component-scanned. |

### Proof 5 — prove the advice is even registered

```bash
curl -s http://localhost:8080/actuator/beans | jq '.contexts[].beans | keys[] | select(test("ExceptionHandler"; "i"))'
```

(Requires `management.endpoints.web.exposure.include=beans` on a local profile.)

**What to look for:** your advice class name in the list.

| What you see | What it means |
|---|---|
| `orderflowExceptionHandler` present | It is a bean. If it still is not firing, the problem is matching or ordering, not registration. |
| Absent | It is outside the component-scan base package (Topic 36), or the class is missing `@RestControllerAdvice`, or you registered it as a plain `@Bean` in a `@Configuration` that is not imported. |

### Proof 6 — the security gap is real

```bash
# with a deliberately malformed token
curl -i -H "Authorization: Bearer not-a-real-token" http://localhost:8080/api/orders
```

**What to look for:** compare this response's `Content-Type` and body keys to Proof 1's.

| What you see | What it means |
|---|---|
| `401` with `Content-Type: application/problem+json` and the same keys | You have already wired Security's entry point to produce a `ProblemDetail`. |
| `401` with an empty body, or a different JSON shape | Trap 5, confirmed on your own service. The filter chain answered, not your advice. |
| `200` | Security is not protecting this route. Different problem, and a bigger one. |

---

## Practice exercises

### 1 — Easy: build the contract

Add to `orderflow`:

- `ProductNotFoundException` and `OrderNotFoundException` as shown.
- One `@RestControllerAdvice` producing `ProblemDetail` for both, with a `type` URI,
  a `title`, and one typed extension member each.
- A catch-all `Exception` handler whose `detail` is a literal.

Then prove all four of these with `curl -i`, pasting each response:

1. A 404 for a missing product, `Content-Type: application/problem+json`.
2. A 404 for a missing order, with a *different* `type` from the product case.
3. A forced 500 containing no class name, no message from the exception, and no
   stack trace.
4. The same 500's correlation ID appearing in the application log alongside the
   stack trace.

**The question to answer in writing:** why should `type` be a URI rather than a
short string like `"PRODUCT_NOT_FOUND"`? Give the reason the RFC gives, and then
give the practical reason that matters more.

### 2 — Medium: combining earlier topics (01–45)

Extend the advice to cover validation, using what you built in Topics 26, 27 and 45.

Requirements:

- Extend `ResponseEntityExceptionHandler` and override
  `handleMethodArgumentNotValid`.
- The body carries an `errors` extension member: a list of `record FieldViolation(String field, String message)` (Topic 27 — and note the record is a
  *DTO*, which is legal; a record cannot be a JPA entity, which you will meet in
  Topic 48).
- Validation uses the Topic 45 constraints on `PlaceOrderRequest`, including a
  validation group so create and update differ.
- The service layer returns `Optional<Product>` (Topic 26) and the controller does
  the `orElseThrow`. The service must not import anything from
  `org.springframework.http`.
- Add a JUnit 5 parameterised test that enumerates every subclass of
  `OrderflowException` and asserts each one has a handler producing a distinct
  `type`. Use reflection or an explicit list — either is acceptable, but justify
  your choice.

**Deliberately break it, then fix it:** add a second `@RestControllerAdvice`
containing only an `Exception` handler, with no `@Order` on either advice. Run the
404 test repeatedly, including after a restart. Report what you observe, explain it
using the ordering rule, and fix it with explicit `@Order` values.

**The question to answer:** your service now imports nothing from
`org.springframework.http` below the controller layer. What does that buy you when
this logic is later reused from a Kafka consumer (Topic 113)?

### 3 — Hard: production simulation on the spine

You are the API owner. Six client teams consume `orderflow`. Deliver an error
contract that survives them.

**Part A — the catalogue.** Write `docs/orderflow-error-catalogue.md`: every error
`type`, its status, its `title`, its extension members with types, and one sentence
on what a client should *do* about it (retry, re-authenticate, show to user, give
up). This document is the deliverable; the code implements it.

**Part B — implement it**, including:
- Domain exceptions for all of: product not found, order not found, insufficient
  stock, insufficient funds, order not cancellable, duplicate order (idempotency
  key collision), payment declined.
- `OptimisticLockingFailureException` → 409 with a `Retry-After` header. (You have
  not written the `@Version` field yet — that is Topic 52. Trigger it in a test by
  throwing the exception from a stub service.)
- A `correlationId` extension member on every single problem, sourced from the
  request-scoped bean from Topic 38.

**Part C — the compatibility test.** Write a test that:
1. Serializes one `ProblemDetail` per error type to JSON.
2. Compares each against a committed golden file in
   `src/test/resources/error-contract/`.
3. Fails if any key is *removed or renamed*, but passes if a key is *added*.

Explain in a comment why "added keys pass" is the correct policy, and what would
have to change for that to become false.

**Part D — the security gap.** Wire Spring Security's `authenticationEntryPoint`
and `accessDeniedHandler` to emit the same `ProblemDetail` shape. Prove with `curl`
that a 401, a 403, a 404 and a 409 all have identical key sets apart from the
type-specific extensions. Then write, in three sentences, why this could not be
solved inside `@ControllerAdvice`.

**Part E — argue the other side.** Under what circumstances is RFC 9457 the wrong
choice for a service? Name at least one real case and be specific about the cost.

---

## Interview questions

### Q1 — "How do you handle exceptions globally in a Spring Boot API?"

**Mid-level answer:** "I use `@ControllerAdvice` with `@ExceptionHandler` methods
for each exception type, and return a `ResponseEntity` with the right status code
and an error DTO."

**Senior answer:** "`@RestControllerAdvice` extending
`ResponseEntityExceptionHandler`, returning `ProblemDetail` per RFC 9457, so
framework exceptions and my domain exceptions produce the same body shape. My
domain exceptions carry typed fields and no HTTP knowledge — the advice owns the
exception-to-status mapping, so the service layer stays reusable from a Kafka
consumer or a batch job. The `type` URI is the stable contract clients branch on;
`title` and `detail` are human text I reserve the right to reword. And I know the
advice does not cover the Security filter chain, so the entry point and
access-denied handler are wired to emit the same shape."

**What separates them:** the senior answer treats the error body as a *versioned
API surface*, names which field is the machine contract, and volunteers the
filter-chain gap unprompted. The mid answer describes the mechanism correctly and
stops there.

**Follow-up the interviewer asks:** "What happens if two `@ControllerAdvice` classes
both have a handler for the same exception?" They want `@Order`, and they want you
to distinguish *within-class* specificity matching from *across-class* ordering.

---

### Q2 — "A client says your error responses are inconsistent. How do you investigate?"

**Mid-level answer:** "I'd check each controller and make sure they all use the same
error format."

**Senior answer:** "First I'd enumerate the actual shapes rather than assume there
are two. There are usually four sources: my advice, controllers returning
`ResponseEntity.badRequest()` directly, Boot's `/error` fallback for anything
unhandled, and the Security filter chain — which never reaches the advice at all.
I'd hit each error path with `curl -i` and diff the key sets, including the
`Content-Type`. Then I'd fix it in one place, ban direct 4xx `ResponseEntity`
returns from controllers with an ArchUnit rule, and add a golden-file test per
error type so it can't drift again. The test is the actual fix; the code change is
just today's instance of the problem."

**What separates them:** naming all four sources, and finishing with the regression
guard rather than the code change.

**Follow-up:** "How would you roll out a change to the error format without breaking
the six clients?" Good answers: additive-only for a deprecation window, both shapes
behind content negotiation or an API version (Boot 4 has `spring.mvc.apiversion.*`),
metrics on which clients still parse the old field, and a dated removal.

---

### Q3 — "Why not just return `ex.getMessage()` in the response?"

**Mid-level answer:** "Because it could contain sensitive information."

**Senior answer:** "Two separate reasons. Security: framework and driver messages
routinely contain SQL, constraint names, class names and bound parameter values —
a Postgres unique-violation message names your index, and a Hibernate message can
name your entity fields and, with bind-parameter logging on, actual customer data.
Contract: the message is written for a developer and changes with any library
upgrade, so any client that parses it is coupled to my dependency versions. My rule
is that `detail` is a literal for anything I didn't write, and for my own domain
exceptions it's a message I own and version. The real information isn't destroyed —
it goes to the log, keyed by the correlation ID that *is* in the response, so
support can find it in one step."

**What separates them:** two distinct reasons rather than one, a concrete example of
a leaky message, and the observation that the information is relocated rather than
lost.

**Follow-up:** "So how does support debug a customer's 500?" They are checking
whether you have actually thought through the operational consequence, or just
recited a security rule.

---

### Q4 — "What is RFC 9457 and why would you use it over your own error DTO?"

**Mid-level answer:** "It's a standard format for error responses with fields like
type, title, status and detail."

**Senior answer:** "It's the IETF spec for problem details, served as
`application/problem+json`; it obsoletes RFC 7807 with the same media type. The
value isn't the field names — it's that they're *agreed*, so client libraries, API
gateways and OpenAPI tooling already understand them without per-service
configuration. It also forces a distinction most hand-rolled DTOs miss: `type` is
the stable machine identifier, `title` is the human summary of the kind, `detail` is
about this occurrence. Most hand-rolled DTOs collapse all three into one `message`
field, which is why clients end up string-matching. And extension members are
explicitly permitted, so I lose nothing by adopting it."

**What separates them:** naming the type/title/detail distinction as the actual
design content, and knowing the RFC number changed while the media type did not.

**Follow-up:** "When would you *not* use it?" Honest answers: an internal RPC where
both sides ship together and you'd rather have a typed schema; a GraphQL API, which
has its own error envelope; a legacy public API where clients already parse a
different shape and the migration cost exceeds the benefit.

---

### Q5 — "Where should the exception-to-HTTP-status mapping live?"

**Mid-level answer:** "In the controller, or in the service where the error is
detected — you can throw `ResponseStatusException` with the right status."

**Senior answer:** "In exactly one place: the advice. A domain exception should say
*what business rule failed*, not *what status code that is*. The moment
`WalletService` imports `HttpStatus`, that service can no longer be called from a
Kafka consumer, a scheduled job or a CLI without dragging a web concern along —
and the mapping decision becomes scattered across dozens of files, so nobody can
answer 'what does this API return for insufficient stock?' without grepping.
`ResponseStatusException` is fine for a genuinely one-off case in a controller, but
using it as the default pushes HTTP down the stack. The mapping table is a document
first and code second."

**What separates them:** treating the mapping as a single owned artefact, and giving
the concrete cost — reuse from a non-HTTP entry point — rather than saying
"separation of concerns".

**Follow-up:** "Does that mean `@ResponseStatus` on an exception class is wrong?"
Nuanced answer: it works, and it is concise, but it puts HTTP knowledge on the
exception itself and gives you no place for a `type` URI or extension members. It is
acceptable in a small service; it does not scale to an error catalogue.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The RFC says clients should branch on `type`, not `title` or `detail`. What
   concretely breaks in a mobile app if a team ignores that advice and branches on
   `detail`? Why is a mobile client worse affected than a web client?

2. `@ControllerAdvice` sits inside `DispatcherServlet`; the Security filter chain
   sits outside it. Given that, is it *possible* to have exactly one implementation
   of your error body, or must there always be at least two call sites? What does
   your answer imply about how you structure the code?

3. Your catch-all handler builds `detail` from a literal string. A colleague argues
   this makes production debugging harder. Steelman their position, then rebut it.
   Under what conditions would they actually be right?

4. Adding a key to the error body is backward-compatible; removing one is not. Is
   that always true? Construct a case where *adding* a key breaks a client.

5. You have `@ExceptionHandler(Exception.class)` in one advice and
   `@ExceptionHandler(ProductNotFoundException.class)` in a second, with no `@Order`
   anywhere. Predict the behaviour, then say what makes your prediction unsafe to
   rely on.

6. `ProblemDetail` is a mutable class with a `Map` of extensions, in a codebase
   where Topic 17 told you to prefer immutability. Why did the framework choose
   mutable here, and does that choice cost you anything given how it is used?

7. Your error catalogue has 11 types. A new team wants to add "wallet frozen by
   compliance" as a 12th. Walk through what has to happen — in the catalogue, the
   code, the tests, and the six client teams — before that error can be returned in
   production.

---

## Quick reference card

### The five RFC 9457 members

| Member | Stable? | Who reads it |
|---|---|---|
| `type` (URI) | **Yes — never change it** | machines |
| `title` | no | humans |
| `status` | yes | machines and middleware |
| `detail` | no | humans |
| `instance` (URI) | per-occurrence | support |

Everything else is an extension member. Scalars and records only.

### Annotations

```java
@RestControllerAdvice                       // global, JSON body           <- use this
@RestControllerAdvice(basePackages = "...") // narrow by package
@RestControllerAdvice(assignableTypes = X.class)
@ExceptionHandler(SomeException.class)      // most specific type wins within a class
@Order(Ordered.LOWEST_PRECEDENCE)           // ordering ACROSS advice classes
```

### Building a problem

```java
ProblemDetail.forStatus(HttpStatus.CONFLICT)
ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, "human text")
problem.setType(URI.create("https://errors.orderflow.com/insufficient-stock"))
problem.setTitle("Insufficient stock")
problem.setInstance(URI.create(request.getRequestURI()))
problem.setProperty("available", 2)         // extension member
throw new ErrorResponseException(HttpStatus.SERVICE_UNAVAILABLE)
```

### Properties

```properties
spring.mvc.problemdetails.enabled=true   # framework exceptions -> ProblemDetail
server.error.include-message=never       # keep the defaults
server.error.include-stacktrace=never
server.error.include-binding-errors=never
server.error.include-exception=false
```

### Status choices you will keep re-deciding

| Situation | Status |
|---|---|
| Malformed request body / failed validation | 400 |
| No or bad credentials | 401 (filter chain, not the advice) |
| Authenticated but not allowed | 403 (filter chain, not the advice) |
| Resource does not exist | 404 |
| Well-formed but conflicts with current state (no stock, no funds, wrong status) | 409 |
| Payment refused by the gateway | 402 |
| Optimistic lock failure the client may retry | 409 + `Retry-After` |
| Anything you did not anticipate | 500 with a literal `detail` |

### Checklist

- [ ] Exactly one advice owns the body shape.
- [ ] It extends `ResponseEntityExceptionHandler` so framework errors match.
- [ ] Every problem has a `type` URI. None is `about:blank`.
- [ ] Every problem has a `correlationId` extension member.
- [ ] The 500 handler's `detail` is a literal, never `ex.getMessage()`.
- [ ] The 500 handler logs the stack trace with the correlation ID.
- [ ] No `@Entity` is ever passed to `setProperty`.
- [ ] Security's entry point and access-denied handler emit the same shape.
- [ ] A golden-file test per error type guards the contract.
- [ ] No controller returns a 4xx/5xx `ResponseEntity` directly.

---

## When would I use this at work?

**1. Week one on a new team, reading an unfamiliar service.**
You `curl` three failing endpoints and diff the bodies. If they differ, you have
found the highest-leverage, lowest-risk improvement available to a new joiner: it
touches no business logic, it is trivially testable, and every client team will
thank you. It is also an excellent way to learn the domain, because building the
error catalogue forces you to enumerate every business rule.

**2. During a security review or a penetration test.**
"Does any error response contain internal information?" is a standard finding. If
your 500 body is built from literals and your catch-all is the only 500 path, the
answer is a one-line "no, here is the test that proves it". If it is not, you spend
a sprint auditing every handler under time pressure.

**3. When a client team files "the app shows a generic error for X".**
With a stable `type` per error kind, this is a five-minute conversation: here is the
type, here is what it means, here is the extension member with the number you want
to display. Without it, it becomes a negotiation about string parsing that will
break again at the next reword.

---

## Connected topics

**Prerequisites:**
- **08 — Exceptions fundamentals**: the `Throwable`/`Error`/`RuntimeException`
  hierarchy that decides what your advice can even catch.
- **09 — Exception API design**: the domain exception hierarchy this topic maps onto
  HTTP. If you have not designed it, you are mapping nothing.
- **26 — Optional**: `findBySku` returns `Optional<Product>`; the controller's
  `orElseThrow` is where absence becomes a 404.
- **27 — Records**: the correct type for an extension member with structure.
- **36 — Component scanning**: why your advice must be under the scanned package.
- **38 — Bean scopes**: the request-scoped correlation ID bean.
- **44 — REST controllers**: the `DispatcherServlet` loop that the
  `HandlerExceptionResolver` chain hangs off.
- **45 — Bean Validation**: produces `MethodArgumentNotValidException`, the
  highest-volume error your advice handles.

**This unlocks:**
- **47 — Spring Data JPA**: `Optional<T>` repository returns and `DataAccessException`
  subclasses that need mapping here.
- **49 — Hibernate associations and lazy loading**: why serializing an entity —
  including inside an error handler — is a 500 waiting to happen.
- **52 — Locking**: `OptimisticLockingFailureException` → 409 + `Retry-After` is the
  contract half of optimistic locking.
- **54 — `@Transactional`**: rollback happens *before* your advice runs, which is
  why the advice must never assume a live persistence context.
- **56 / 57 — Spring Security**: the filter-chain gap, closed properly.
- **111 — Retries and backoff**: a client can only retry safely if your error tells
  it whether the failure is retryable. That is a `type` and a `Retry-After`.
- **118 — Metrics**: error `type` is a *good* metric tag (bounded cardinality);
  order ID is a catastrophic one.
- **119 — Tracing**: the trace ID belongs in the problem body next to the
  correlation ID, so a support ticket links straight to a trace.

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0,
`jakarta.*` throughout. `ProblemDetail` has been in the framework since 6.0, so
everything here works unchanged on Boot 3.x; only Boot 2.x lacks it entirely.
RFC 9457 obsoleted RFC 7807 in July 2023 without changing the
`application/problem+json` media type.*
