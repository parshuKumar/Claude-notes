# 56 — Spring Security I — Filter Chain, Authentication, `SecurityContext`

## Phase: 5 — Spring Boot & Persistence
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: the authenticated `orderflow` API — every endpoint under `/api/**` requires a real identity, and the identity is available to the service layer

---

## ELI5 anchor

Imagine an office building.

Your **application** is an office on the fourth floor. Requests are visitors.

A **Nest guard** is a receptionist *inside your office*. The visitor has already come
through the front door, into the lift, down the corridor, and knocked on your door. The
receptionist looks at their badge and decides whether to let them into the meeting room.
By the time the receptionist speaks to them, the building already knows which office they
were looking for.

**Spring Security is the security desk in the lobby.** It sits at the front door. A
visitor with no badge is turned away in the lobby. Nobody upstairs — not the lift
operator, not the receptionist, not your office — ever learns that anyone came. The
building never even looks up which office they wanted.

The desk is not one guard. It is a **line of guards**, each with one job, in a fixed
order:

1. One writes down that a visitor arrived (logging).
2. One checks whether this visitor already has a valid pass from earlier (session).
3. One checks their badge and issues a pass (authentication).
4. One decides whether *this* visitor is allowed into *this* part of the building
   (authorization).

Each guard can wave the visitor on to the next, or stop them dead. If a visitor makes it
past all of them, they go upstairs.

And one more thing, which is the detail that bites people: when a guard issues a pass,
they **write the visitor's name on a whiteboard that belongs to that one guard's desk**.
If your office hands the work to a colleague at a different desk, that colleague looks at
*their* whiteboard, sees nothing, and concludes nobody is there. That whiteboard is
`SecurityContextHolder`, and the "different desk" is a different thread.

---

## The bridge from what you know

### Nest guards ≈ Spring Security filter chain: PARTIAL, and the difference matters

This is the most important paragraph in the doc, so read it slowly.

A Nest guard runs **inside the framework's request pipeline, after routing**. By the time
`canActivate` is called, Nest has already matched the URL to a controller and a handler
method — that is precisely how `@UseGuards` on a specific route works, and how
`context.getHandler()` gives you the method's metadata.

```ts
// NestJS — the guard runs AFTER routing, INSIDE the pipeline
@Injectable()
export class JwtAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest();
    const token = req.headers.authorization?.slice(7);
    req.user = this.jwt.verify(token);      // attach to the request object
    return true;
  }
}

@UseGuards(JwtAuthGuard)
@Controller('orders')
export class OrdersController { ... }
```

Spring Security is a **servlet filter chain**. It runs **before `DispatcherServlet`** —
before Spring MVC has consulted a single `HandlerMapping`, before any `@RequestMapping`
has been matched, before your controller class has been touched.

```
HTTP request
   |
   v
Servlet container (Tomcat)
   |
   v
FilterChainProxy  ("springSecurityFilterChain")   <-- Spring Security lives HERE
   |   -> a list of ~15 filters, in a fixed order
   |   -> can reject with 401/403 and return, right here
   v
DispatcherServlet                                  <-- Spring MVC starts HERE
   |
   v
HandlerMapping -> HandlerAdapter -> your @RestController
```

**Three consequences that follow, and that you should be able to state:**

1. **Authorization can reject a request before any controller mapping is consulted.** A
   request to `/api/admin/refunds` with no token gets a 401 without Spring MVC ever
   determining whether that URL maps to anything. It is genuinely cheaper, and it is
   genuinely a different security posture — an unmapped URL and a forbidden URL are
   rejected by the same code.
2. **The rules are URL-pattern-based by default, not annotation-based.** In Nest you
   decorate the controller. In Spring you write patterns in a central config. Both are
   available in Spring (`@PreAuthorize` is Topic 57), but the filter chain's
   `authorizeHttpRequests` is the primary mechanism and it does not know your controller
   exists.
3. **A `@ControllerAdvice` exception handler (Topic 46) does not see security failures.**
   Your `ProblemDetail` error contract stops at `DispatcherServlet`; a 401 from the filter
   chain never reaches it. You have to configure the error response *in the security
   config*, and forgetting this is why so many APIs have one error format for business
   errors and a completely different one for auth errors.

**Verdict: PARTIAL analogue.** The mental model of "a pluggable pre-handler that can
reject a request" transfers. The position in the stack does not, and it is the position
that produces the differences you will actually trip over.

### Passport strategies ≈ `AuthenticationProvider` / `AuthenticationFilter`: PARTIAL

| Passport (Nest) | Spring Security | Verdict |
|---|---|---|
| A `Strategy` class with a `validate()` method | `AuthenticationProvider.authenticate(Authentication)` | **PARTIAL** — same role: turn a credential into a principal |
| `passport-local` reading a username/password body | `UsernamePasswordAuthenticationFilter` + `DaoAuthenticationProvider` | **PARTIAL** |
| `passport-jwt` extracting a bearer token | `BearerTokenAuthenticationFilter` + a `JwtDecoder` (Topic 57) | **PARTIAL** |
| `req.user` | `SecurityContextHolder.getContext().getAuthentication().getPrincipal()` | **PARTIAL** — see below |
| Strategy registered per-route via `@UseGuards` | Filter registered once, applied by URL pattern | **DIFFERENT** |

The `req.user` row is the one that costs people time. In Nest, the principal is a
property on the **request object**, which is passed explicitly down the call chain, so it
survives anything you do with it. In Spring, the principal lives in a **`ThreadLocal`**.
It is available anywhere on the current thread with no plumbing — which is wonderfully
convenient and is exactly why it silently disappears the moment you change threads.

### What does not transfer at all

| Your world | Spring Security | Verdict |
|---|---|---|
| `req.user` travels with the request object | `SecurityContext` travels with the **thread** | **NO ANALOGUE** — new failure mode |
| Guards are opt-in per route | The chain applies to everything; you opt *out* with `permitAll()` | **INVERTED** |
| Rule order is the decorator you wrote | Rule order is **the order of your matchers**, first match wins | **NO ANALOGUE** — Trap 1 |
| An unhandled guard error hits your exception filter | A filter-chain rejection never reaches `@ControllerAdvice` | **NO ANALOGUE** |
| `AsyncLocalStorage` follows async context automatically | `ThreadLocal` does not follow anything | **NO ANALOGUE** — Trap 4, and Topics 91, 101, 119, 120 |

---

## What is this?

Spring Security is a chain of servlet filters that runs before your application code, plus
a small set of objects for representing "who is this".

### The four nouns

Learn these four. Every conversation about Spring Security uses them.

| Noun | What it is | Nest equivalent |
|---|---|---|
| `Authentication` | An object holding the principal, the credentials, the authorities, and an `authenticated` boolean | `req.user`, roughly |
| `SecurityContext` | A holder with one field: the current `Authentication` | — |
| `SecurityContextHolder` | A static accessor backed by a `ThreadLocal<SecurityContext>` | `AsyncLocalStorage`, except it does not propagate |
| `SecurityFilterChain` | A list of filters plus a `RequestMatcher` saying which requests they apply to | your global guard registration |

An `Authentication` object does double duty, which confuses everyone once:

- **Before** authentication, it is a *request*: "here is a username and a password,
  please verify them." `isAuthenticated()` is false.
- **After** authentication, it is a *result*: "this is Alice, and she has these
  authorities." `isAuthenticated()` is true, and the credentials are usually erased.

The thing that converts one into the other is an `AuthenticationManager`, which delegates
to one or more `AuthenticationProvider`s.

### How a request actually flows

```
Tomcat
  -> DelegatingFilterProxy  (a plain servlet filter registered with the container;
     its only job is to find the Spring bean named springSecurityFilterChain)
  -> FilterChainProxy       (a Spring bean; holds a LIST of SecurityFilterChains)
       -> picks the FIRST SecurityFilterChain whose RequestMatcher matches this request
       -> runs that chain's filters in order:
            DisableEncodeUrlFilter
            WebAsyncManagerIntegrationFilter
            SecurityContextHolderFilter       <- loads the SecurityContext for this thread
            HeaderWriterFilter
            CorsFilter
            CsrfFilter                        <- Topic 57
            LogoutFilter
            UsernamePasswordAuthenticationFilter   (form login, if enabled)
            BearerTokenAuthenticationFilter        (resource server, if enabled — Topic 57)
            BasicAuthenticationFilter              (http basic, if enabled)
            RequestCacheAwareFilter
            SecurityContextHolderAwareRequestFilter
            AnonymousAuthenticationFilter     <- gives unauthenticated requests a principal
            ExceptionTranslationFilter        <- turns exceptions into 401/403
            AuthorizationFilter               <- LAST. enforces authorizeHttpRequests
  -> DispatcherServlet
  -> your controller
```

You do not need to memorise all fifteen. You need four of them cold, and you need to know
you can print the real list at startup (Hands-on proof).

### The four filters that matter

| Filter | Job | Why you care |
|---|---|---|
| `SecurityContextHolderFilter` | At the start of the request, load a `SecurityContext` into the `ThreadLocal`; **in a `finally` block at the end, clear it** | The `finally` is why the context does not leak to the next request on a pooled thread. It is also why the context is gone by the time an async task runs. |
| `AnonymousAuthenticationFilter` | If nothing has authenticated, install an `AnonymousAuthenticationToken` | This is why `getAuthentication()` is rarely `null` — an unauthenticated request has an *anonymous* principal, not no principal. Checking `!= null` is therefore not an authentication check. |
| `ExceptionTranslationFilter` | Catch `AuthenticationException` → 401 (or redirect to login); catch `AccessDeniedException` → 403 | Translates security failures into HTTP. Sits *above* `AuthorizationFilter` in the chain so it can catch what that throws. |
| `AuthorizationFilter` | Evaluate your `authorizeHttpRequests` rules; throw `AccessDeniedException` if denied | **The last filter.** This is where "authorization happens before your controller" is literally true. |

The 401-vs-403 distinction falls straight out of `ExceptionTranslationFilter`:

- **401 Unauthorized** — you are anonymous. "I do not know who you are." The response
  carries a `WWW-Authenticate` header telling you how to authenticate.
- **403 Forbidden** — you are authenticated, and you still may not do this. "I know who
  you are, and no."

Getting a 403 when you expected a 401 almost always means an anonymous request reached a
rule that denied it without triggering the entry point. Getting a 401 when you sent a
valid token means the token was not accepted at all.

---

## Why does it matter?

**1. "Where does authorization happen?" is a real architectural question and you will be
asked it.** Answering "in a servlet filter, before `DispatcherServlet`, so before any
controller mapping is consulted" — and being able to *show* the chain — is a strong
signal.

**2. Security misconfiguration is silent in the direction that hurts.** A rule that is too
strict fails loudly on the first request and someone fixes it in five minutes. A rule that
is too permissive — `permitAll()` in the wrong position, a matcher that does not match
what you think — succeeds silently and stays in production until someone reports it or a
pen test finds it. Every trap below is of the second kind.

**3. The `ThreadLocal` detail determines the correctness of everything you build later.**
Every async feature in Phases 9 and 10 of this curriculum has to answer "and what happens
to the `SecurityContext`?" If you do not internalise it here, you will write an
authorization check that silently sees an anonymous principal.

---

## Syntax breakdown

### The `SecurityFilterChain` bean

Since Spring Security 5.7 the configuration is a **bean**, not a subclass. This is the
shape you will write for the rest of your career on this stack.

```java
package com.orderflow.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/products/**").permitAll()
                .requestMatchers("/api/orders/**").authenticated()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())          // stateless + header auth. Topic 57.
            .httpBasic(basic -> {})                // lab only. Replaced by JWT in Topic 57.
            .build();
    }
}
```

Line by line, the parts that are new to you:

| Line | What it does |
|---|---|
| `@Bean SecurityFilterChain` | Registers **one** chain. You can register several; they are tried in `@Order`, first match wins. |
| `HttpSecurity http` (parameter) | A builder, injected by Spring. Do **not** create it yourself. |
| `.securityMatcher("/api/**")` | *Which requests this whole chain applies to.* If it does not match, this chain is skipped entirely and the next one is tried. |
| `.authorizeHttpRequests(auth -> ...)` | *Which rules apply within this chain.* Rules are evaluated top to bottom, **first match wins**. |
| `.requestMatchers(...)` | A pattern. `/api/products/**` matches any depth below. |
| `.permitAll()` / `.authenticated()` | The decision for that pattern. |
| `.anyRequest()` | A catch-all. **It must be last** — see Trap 1. |
| `.sessionManagement(... STATELESS)` | Never create an `HttpSession`, never look for one. Required for a genuinely stateless API. |
| `.csrf(csrf -> csrf.disable())` | Correct *only* for header-based auth with no cookies. Topic 57 covers exactly when this is wrong. |
| `.build()` | Produces the `SecurityFilterChain`. |
| `throws Exception` | The builder declares it. Just propagate it. |

**The lambda DSL.** Every configuration method takes a `Customizer<T>` lambda. This is
the only supported style in Security 6 and 7 — the older chained style with `.and()` was
deprecated and then removed. `.httpBasic(basic -> {})` with an empty body means "enable
it with defaults"; `Customizer.withDefaults()` is the idiomatic way to write the same
thing.

> `[BOOT 3.x DELTA]` The `SecurityFilterChain`-bean style is the same on Boot 3.x
> (Security 6) as on Boot 4.1 (Security 7), so the config above is portable. The things
> that changed are **earlier**: `WebSecurityConfigurerAdapter` was deprecated in Security
> 5.7 and **removed in Security 6**, and the pre-lambda `.and()` chaining style was
> removed alongside it. If you meet a 2.x-era codebase extending
> `WebSecurityConfigurerAdapter` and overriding `configure(HttpSecurity)`, that is the
> pattern being replaced, and the migration is mechanical: return a bean instead of
> overriding a method. Security 7 continues the lambda-DSL-only direction and has
> tightened some defaults further; where this document names a specific Security 7 method
> I say so, and where I am unsure I say that too rather than inventing a name.

### `UserDetailsService` and `PasswordEncoder`

These two beans are how Spring Security learns about your users.

```java
@Bean
UserDetailsService userDetailsService(CustomerRepository customers) {
    return username -> customers.findByEmail(username)
        .map(c -> User.withUsername(c.email())
                      .password(c.passwordHash())        // ALREADY hashed. See below.
                      .authorities("ROLE_CUSTOMER")
                      .accountLocked(c.isLocked())
                      .build())
        .orElseThrow(() -> new UsernameNotFoundException(username));
}

@Bean
PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

| Piece | What it means |
|---|---|
| `UserDetailsService` | A functional interface with one method: `loadUserByUsername(String)`. It **loads**; it does not verify. |
| Returning `UserDetails` | An interface with `getUsername()`, `getPassword()`, `getAuthorities()` and four boolean account-status flags |
| `.password(c.passwordHash())` | The **stored hash**, not a plaintext password. `DaoAuthenticationProvider` hashes the submitted password and compares. |
| `UsernameNotFoundException` | Thrown when there is no such user. Spring converts it to a generic `BadCredentialsException` before it reaches the client, so you cannot enumerate accounts. |
| `PasswordEncoder` | Hashes and verifies. Required — Spring Security refuses to compare plaintext. |
| `createDelegatingPasswordEncoder()` | Stores hashes prefixed with the algorithm, e.g. `{bcrypt}$2a$10$...`, so you can migrate algorithms without invalidating existing passwords. **Use this, not `new BCryptPasswordEncoder()` directly.** |

The `{bcrypt}` prefix is the detail worth keeping: it makes the hash self-describing, so a
future move to Argon2 means new passwords get `{argon2}` and old ones keep verifying. A
bare `BCryptPasswordEncoder` gives you no migration path.

### Reading the current user

Three ways, in order of preference:

```java
// 1. BEST in a controller — a method parameter. Testable, explicit, no static access.
@GetMapping("/api/orders")
List<OrderSummary> myOrders(@AuthenticationPrincipal UserDetails principal) {
    return orders.findByCustomerEmail(principal.getUsername());
}

// 2. Also fine — Spring MVC resolves the Authentication as an argument.
@GetMapping("/api/me")
Map<String, Object> me(Authentication authentication) {
    return Map.of("name", authentication.getName(),
                  "authorities", authentication.getAuthorities());
}

// 3. Anywhere on the request thread — static, convenient, and the source of Trap 4.
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

Prefer (1). Its advantages are the same as constructor injection's (Topic 39): the
dependency is visible in the signature and the method is testable without any container.

### `SecurityContextHolder` and its strategies

```java
SecurityContextHolder.getContext()                  // never null; may hold an anonymous auth
SecurityContextHolder.getContext().getAuthentication()
SecurityContextHolder.setContext(ctx)               // for propagating across threads
SecurityContextHolder.clearContext()                // MUST be called on pooled threads
SecurityContextHolder.setStrategyName(...)          // MODE_THREADLOCAL (default),
                                                    // MODE_INHERITABLETHREADLOCAL,
                                                    // MODE_GLOBAL
```

The strategy is the mechanism, so it is worth being exact:

| Strategy | Behaviour | When it is right |
|---|---|---|
| `MODE_THREADLOCAL` (default) | One context per thread; child threads get nothing | Almost always |
| `MODE_INHERITABLE_THREADLOCAL` | A thread created *from* this one inherits the context | Only for threads you `new` per request. **Useless for a thread pool**, because pool threads were created long before the request and inherit nothing. |
| `MODE_GLOBAL` | One context for the whole JVM | Standalone single-user applications only. Never in a server. |

People reach for `MODE_INHERITABLE_THREADLOCAL` to fix `@Async` and are surprised it does
not work. The reason is in the table: inheritance happens at **thread creation**, and pool
threads are created before any request exists. The correct fix is
`DelegatingSecurityContextExecutor` (Trap 4).

---

## Example 1 — minimal

The smallest useful configuration: one protected endpoint, one open endpoint, one
in-memory user. Run it and probe it with `curl`.

```java
package com.orderflow.security;

@Configuration
@EnableWebSecurity
public class MinimalSecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/products/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())
            .build();
    }

    @Bean
    UserDetailsService users(PasswordEncoder encoder) {
        return new InMemoryUserDetailsManager(
            User.withUsername("alice@orderflow.test")
                .password(encoder.encode("correct-horse"))
                .authorities("ROLE_CUSTOMER")
                .build());
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```

```java
@RestController
class ProbeController {

    @GetMapping("/api/products/1001")
    Map<String, Object> product() {
        return Map.of("sku", "SKU-1001", "name", "Espresso beans, 1kg");
    }

    @GetMapping("/api/orders")
    Map<String, Object> orders(Authentication authentication) {
        return Map.of(
            "user", authentication.getName(),
            "authorities", authentication.getAuthorities().toString(),
            "thread", Thread.currentThread().getName(),
            "authenticated", authentication.isAuthenticated());
    }
}
```

Probe it:

```bash
curl -i localhost:8080/api/products/1001
curl -i localhost:8080/api/orders
curl -i -u alice@orderflow.test:correct-horse localhost:8080/api/orders
curl -i -u alice@orderflow.test:wrong localhost:8080/api/orders
```

| Request | Expected status | Why |
|---|---|---|
| `/api/products/1001`, no credentials | **200** | `permitAll()` matched first |
| `/api/orders`, no credentials | **401** with a `WWW-Authenticate: Basic` header | `anyRequest().authenticated()` denied an anonymous principal; `ExceptionTranslationFilter` invoked the entry point |
| `/api/orders`, correct credentials | **200**, body naming alice | `BasicAuthenticationFilter` authenticated, `AuthorizationFilter` allowed |
| `/api/orders`, wrong password | **401** | `BadCredentialsException` from `DaoAuthenticationProvider` |

`@EnableWebSecurity` is optional in a Boot application — auto-configuration supplies it —
but writing it makes the intent explicit and costs nothing.

The `thread` field in the response is deliberate. Note the value; you will use it in the
`@Async` trap to prove the context did not follow.

---

## Example 2 — production scenario on the `orderflow` spine

### The requirement

`orderflow` has three distinct kinds of caller, and one chain cannot serve all three
sensibly:

| Caller | How they authenticate | What they may do | Volume |
|---|---|---|---|
| **Customer** (mobile app / web) | JWT bearer token from the identity provider | Place orders, view *their own* orders, view their wallet | ~1,150 rps |
| **Admin** (internal back office) | JWT with an admin claim, plus stricter rules | Refunds, inventory adjustments, view any order | ~5 rps |
| **Internal service** (the settlement batch job) | mTLS client certificate, or a service token | Bulk settlement endpoints only | ~2 rps, bursty |
| **Anonymous** | nothing | Product catalogue, health, docs | ~840 rps of the total |

Plus operational endpoints that must not be reachable from the internet at all.

### Multiple filter chains, ordered

This is the pattern to learn: **one `SecurityFilterChain` bean per audience**, each with a
`securityMatcher`, ordered explicitly. Chains are evaluated in `@Order`, and **the first
chain whose matcher matches handles the request**. Later chains never see it.

```java
package com.orderflow.security;

@Configuration
@EnableWebSecurity
public class OrderflowSecurityConfig {

    // ---------- 1. Actuator: most specific, so it must be FIRST ----------
    @Bean
    @Order(1)
    SecurityFilterChain actuatorChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/liveness",
                                 "/actuator/health/readiness").permitAll()
                .anyRequest().hasAuthority("SCOPE_orderflow.ops")
            )
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .build();
    }

    // ---------- 2. Internal service-to-service ----------
    @Bean
    @Order(2)
    SecurityFilterChain internalChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/internal/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .x509(x509 -> x509.subjectPrincipalRegex("CN=(.*?)(?:,|$)"))
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .build();
    }

    // ---------- 3. The public API ----------
    @Bean
    @Order(3)
    SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth
                // ORDER IS SIGNIFICANT. Specific rules first, catch-all last.
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/inventory/*/availability").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/orders").hasRole("CUSTOMER")
                .requestMatchers("/api/orders/**").authenticated()
                .requestMatchers("/api/wallet/**").authenticated()
                .anyRequest().denyAll()          // <-- deny, not authenticate. See below.
            )
            .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))  // Topic 57
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())        // header auth only, no cookies. Topic 57.
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(problemDetailEntryPoint())
                .accessDeniedHandler(problemDetailAccessDeniedHandler()))
            .build();
    }

    // ---------- 4. Everything else: deny by default ----------
    @Bean
    @Order(4)
    SecurityFilterChain fallbackChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth.anyRequest().denyAll())
            .csrf(csrf -> csrf.disable())
            .build();
    }
}
```

Six decisions worth defending:

**1. `@Order(1)` on the most specific matcher.** `/actuator/**` must be evaluated before
any broader chain. If a chain matching `/**` came first, it would swallow actuator
requests and the actuator chain would be dead configuration that never runs — and, worse,
would look correct in code review. The rule is: **most specific first, catch-all last**,
at both the chain level and the rule level.

**2. `anyRequest().denyAll()` in the API chain, not `.authenticated()`.** With `denyAll()`,
a new endpoint someone adds under `/api/` without adding a rule returns 403 for everyone,
including the developer who added it, on the first request. That is a fast, loud, free
failure. With `.authenticated()`, the new endpoint is quietly available to *every* logged-in
customer — including endpoints that should have been admin-only. Deny-by-default converts a
silent authorization hole into an immediate bug report.

**3. A separate `fallbackChain`.** Without it, a request matching no chain passes through
Spring Security entirely and reaches `DispatcherServlet` unprotected. Making the fallback
explicit and denying means an unmatched URL cannot become an accidental hole. This is the
single highest-value line in the whole configuration.

**4. HTTP method in the matcher.** `GET /api/products/**` is public; `POST /api/products`
is not. Writing `.requestMatchers("/api/products/**").permitAll()` without the method makes
your product catalogue writable by anonymous users. Method-aware matchers are not optional
on a REST API.

**5. `STATELESS` everywhere.** No `HttpSession` is created and none is consulted. Three
consequences: no session-fixation risk (Topic 57), no sticky-session requirement in the
load balancer, and the `SecurityContext` genuinely lives only for the duration of one
request on one thread. At 1,200 rps across 6 replicas, sessions would also be a
significant memory and replication cost you have no reason to pay.

**6. A `ProblemDetail` error contract at the security boundary.** Topic 46 established
RFC 9457 `ProblemDetail` as `orderflow`'s error format, and `@ControllerAdvice` enforces it
— but only for exceptions that reach `DispatcherServlet`. A 401 from the filter chain
never does. Without the `exceptionHandling` block, your API has one error format for
business errors and a completely different one for auth errors, which every client then
has to special-case:

```java
private AuthenticationEntryPoint problemDetailEntryPoint() {
    return (request, response, authException) -> {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/problem+json");
        response.getWriter().write("""
            {"type":"https://orderflow.example/errors/unauthenticated",
             "title":"Authentication required",
             "status":401,
             "detail":"A valid bearer token is required for this endpoint."}
            """);
    };
}
```

Do not put the exception's message in `detail`. `authException.getMessage()` can leak
whether a user exists or why a token failed. A generic message plus a correlation ID
(Topic 120) is the right shape: specific in your logs, opaque in the response.

### Making the identity available to the service layer

The controller should not thread the principal down through every service call. Two
patterns, both used in real systems:

```java
// A. Pass it explicitly. Boring, testable, always correct.
@PostMapping("/api/orders")
OrderResponse place(@AuthenticationPrincipal Jwt jwt, @Valid @RequestBody PlaceOrderRequest req) {
    return placementService.place(req.toCommand(UUID.fromString(jwt.getSubject())));
}
```

```java
// B. A thin accessor over the ThreadLocal, so exactly one class knows about
//    SecurityContextHolder and the rest of the codebase is testable.
@Component
public class CurrentUser {

    public Optional<CustomerId> id() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()
                || auth instanceof AnonymousAuthenticationToken) {
            return Optional.empty();       // note: NOT just a null check
        }
        return Optional.of(new CustomerId(UUID.fromString(auth.getName())));
    }

    public CustomerId requireId() {
        return id().orElseThrow(() -> new AccessDeniedException("no authenticated user"));
    }
}
```

Prefer (A) at the boundary and (B) for cross-cutting needs like the audit aspect from
Topic 41. The `AnonymousAuthenticationToken` check in (B) is the important part: because
`AnonymousAuthenticationFilter` installs a principal for unauthenticated requests, a null
check alone will happily return "authenticated" for an anonymous caller. **A null check is
not an authentication check.**

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `permitAll()` ordered after `anyRequest().authenticated()`

**Wrong:**
```java
.authorizeHttpRequests(auth -> auth
    .anyRequest().authenticated()
    .requestMatchers("/api/products/**").permitAll()      // never reached
)
```

**Exact symptom (and there are two, depending on your version):**

*Symptom A — a startup failure.* The application refuses to start with an
`IllegalStateException` whose message says a matcher cannot be configured after
`anyRequest`. Recent Spring Security versions detect this specific mistake and fail fast.
This is the good outcome.

*Symptom B — the silent one.* With a subtler ordering mistake that the check does not
catch — say `/api/**` before `/api/products/**` — there is no error at all. The product
catalogue simply returns 401 for anonymous users. Your mobile app's home screen is empty
for logged-out users. QA reports "the app doesn't load products" and nobody connects it to
a security config change.

```bash
curl -i localhost:8080/api/products/1001     # expect 200, get 401
```

**Root cause:** `authorizeHttpRequests` builds an **ordered list** of matcher-to-decision
pairs, evaluated top to bottom, and **the first match wins**. `anyRequest()` matches
everything, so any rule after it is unreachable. Broader patterns shadow narrower ones the
same way, without any error.

If you know routing tables or firewall rules, this is exactly that model, and the same
discipline applies.

**Fix:**
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()   // most specific
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/orders/**").authenticated()
    .anyRequest().denyAll()                                            // catch-all LAST
)
```

**Prevent it structurally:** write a test that asserts the actual HTTP status for every
significant path. Ordering bugs are invisible to code review and completely visible to a
`MockMvc` test.

```java
@WebMvcTest
@Import(OrderflowSecurityConfig.class)
class SecurityRuleTest {

    @Autowired MockMvc mvc;

    @Test void catalogueIsPublic() throws Exception {
        mvc.perform(get("/api/products/1001")).andExpect(status().isOk());
    }

    @Test void ordersRequireAuth() throws Exception {
        mvc.perform(get("/api/orders")).andExpect(status().isUnauthorized());
    }

    @Test void unmappedApiPathsAreDenied() throws Exception {
        mvc.perform(get("/api/definitely-not-a-real-endpoint"))
           .andExpect(status().isForbidden());          // proves denyAll() is the catch-all
    }
}
```

That third test is the one that catches the next person's mistake as well as your own.

---

### Trap 2 — a request that matches no filter chain at all

**Wrong:**
```java
@Bean
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")                       // ONLY /api/**
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .build();
}
// ... and no other SecurityFilterChain bean exists.
```

Then someone adds a controller at `/internal/debug/dump-config`.

**Exact symptom:** `curl localhost:8080/internal/debug/dump-config` with no credentials
returns **200 and the configuration**. No 401, no 403, no log line, no test failure — the
endpoint's own test passes because it never tested authentication. The gap is found by a
pen test, or by an incident.

**Root cause:** `FilterChainProxy` walks its list of chains and uses the first whose
`securityMatcher` matches. **If none matches, no security filters run at all** and the
request goes straight to `DispatcherServlet`. Defining a `securityMatcher` narrows what
that chain protects; it does not extend protection to anything else.

This is genuinely counter-intuitive coming from Nest, where a global guard registered with
`APP_GUARD` applies to every route by default and you opt out with `@Public()`. Spring's
chains are opt-in by pattern.

**Fix:**
```java
// Always define a lowest-priority catch-all chain that denies.
@Bean
@Order(Ordered.LOWEST_PRECEDENCE)
SecurityFilterChain fallbackChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth.anyRequest().denyAll())
        .csrf(csrf -> csrf.disable())
        .build();
}
```

**Verify it, do not assume it.** Enable `logging.level.org.springframework.security.web.FilterChainProxy=DEBUG`
and confirm that a request to an unexpected path is handled by the fallback chain (see
Hands-on proof). And write the "unmapped path returns 403" test above — it is the cheapest
possible guard against this entire class of bug.

---

### Trap 3 — using `@Secured`-style checks in the controller instead of the chain

**Wrong:**
```java
@GetMapping("/api/admin/refunds")
List<Refund> refunds(Authentication auth) {
    if (!auth.getName().endsWith("@orderflow.internal")) {   // hand-rolled check
        throw new AccessDeniedException("admins only");
    }
    return refundService.all();
}
```

**Exact symptom:** it works. That is the problem. Then a colleague adds
`GET /api/admin/refunds/{id}` and forgets the check, because the check is not a rule
anywhere — it is four lines of code in one method. The new endpoint is available to every
authenticated customer. The route also does not appear in any security configuration, so
nobody reviewing `SecurityConfig` sees a gap, and any test that enumerates admin routes
finds nothing to enumerate.

There is a subtler cost too: the check runs **after** request-body deserialization, after
validation, after argument resolution, and possibly after a `@Transactional` boundary
opened — so an unauthorized caller has already consumed real resources and possibly taken
a connection from the pool (Topic 55).

**Root cause:** authorization has become an implementation detail of one method instead of
a declared policy. Policy that is not centralised cannot be audited, and policy that
cannot be audited drifts.

**Fix — declare it, in one of the two supported places:**

```java
// 1. In the chain, by URL pattern. Runs before DispatcherServlet. Preferred for
//    coarse-grained, resource-shaped rules.
.requestMatchers("/api/admin/**").hasRole("ADMIN")
```

```java
// 2. As a method annotation, for rules that depend on the arguments. Topic 57.
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/api/admin/refunds")
List<Refund> refunds() { ... }
```

Use the chain for "who may reach this resource" and `@PreAuthorize` for "may this user act
on *this specific object*" — the ownership check, which a URL pattern cannot express. Both
are declarative and both are greppable. Neither is an `if` statement in a handler.

---

### Trap 4 — `SecurityContext` lost across an `@Async` boundary

**Wrong:**
```java
@RestController
class OrderController {

    @PostMapping("/api/orders")
    OrderResponse place(@Valid @RequestBody PlaceOrderRequest req) {
        OrderId id = placementService.place(req.toCommand());
        auditService.recordAsync(id);                     // @Async
        return new OrderResponse(id);
    }
}

@Service
class AuditService {

    @Async
    public void recordAsync(OrderId id) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        auditRepository.save(new Audit(id, auth.getName()));   // who?
    }
}
```

**Exact symptom, in three flavours depending on exactly what you do:**

- Every audit row's `actor` column reads `anonymousUser`. Your audit trail is complete,
  well-formatted and useless. **No exception is thrown**, which is why this survives code
  review and testing.
- Or a `NullPointerException` on `auth.getName()` in an async thread, appearing in the
  logs with **no request context and no correlation ID** (Topic 120 — the MDC did not
  propagate either), so it is very hard to attribute to a request.
- Or, in the worst variant, a `@PreAuthorize`-annotated method called from the async task
  evaluates against an anonymous principal and **denies** — turning a security check into
  an intermittent failure that only appears under async load.

**Root cause:** `SecurityContextHolder` is `ThreadLocal`-backed. The `@Async` method runs
on a `TaskExecutor` thread. That thread has its own empty `ThreadLocal`, so
`getContext()` returns a fresh empty `SecurityContext` and
`AnonymousAuthenticationFilter`'s work — which happened on the *request* thread — is not
visible.

Worse, `SecurityContextHolderFilter` clears the context in a `finally` block when the
request completes. So even a naive "capture the context and use it later" fails if the
request finishes first.

**Fix:**

```java
// 1. BEST: pass the identity explicitly. It is data. Treat it as data.
auditService.recordAsync(id, currentUser.requireId());
```

```java
// 2. Wrap the executor so it copies the context across the boundary.
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(500);          // BOUNDED. Topic 90.
        executor.setThreadNamePrefix("orderflow-async-");
        executor.initialize();
        return new DelegatingSecurityContextExecutor(executor);   // <-- the fix
    }
}
```

```java
// 3. For a single call site, wrap the task rather than the executor.
executor.execute(new DelegatingSecurityContextRunnable(() -> audit(id)));
```

```java
// 4. Do NOT reach for this and think you are done:
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
// Inheritance happens at THREAD CREATION. Pool threads were created at startup,
// before any request existed. They inherit nothing. This "fix" appears to work in a
// test that creates a fresh thread and fails in production with a pool.
```

**The general principle, which you will meet four more times in this curriculum:**
`ThreadLocal`-backed context does not cross thread boundaries, and every mechanism that
changes threads needs an explicit propagation strategy. That is true for the
`SecurityContext` here, for the transaction binding (Topics 54 and 55), for the MDC
(Topic 120) and for the trace context (Topic 119). Forward: Topic 91
(`CompletableFuture` continuations), Topic 101 (virtual threads — each has its own
`ThreadLocal`s, so the problem is unchanged in kind but far larger in scale).

---

### Trap 5 — treating `getAuthentication() != null` as "is authenticated"

**Wrong:**
```java
public boolean isLoggedIn() {
    return SecurityContextHolder.getContext().getAuthentication() != null;
}
```

**Exact symptom:** an anonymous request is treated as logged in. Downstream,
`auth.getName()` returns the literal string `anonymousUser`, and you get order rows whose
customer is `anonymousUser`, or an authorization branch that grants access to a caller who
sent no credentials whatsoever. Because the string is non-null and non-empty, every null
check and every `StringUtils.hasText` passes.

**Root cause:** `AnonymousAuthenticationFilter` installs an `AnonymousAuthenticationToken`
for any request that reached it without being authenticated. That token has a principal
(`anonymousUser`), an authority (`ROLE_ANONYMOUS`), and — this is the sharp edge —
`isAuthenticated()` returns **`true`**, because from the framework's point of view the
anonymous identity was successfully established.

So *neither* a null check *nor* `isAuthenticated()` alone is sufficient.

**Fix:**
```java
public boolean isLoggedIn() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    return auth != null
        && auth.isAuthenticated()
        && !(auth instanceof AnonymousAuthenticationToken);    // the necessary third check
}
```

Better still: do not write this at all. Let the filter chain decide, with
`.authenticated()`, which uses `AuthenticatedAuthorizationManager` and gets the anonymous
case right by construction. Hand-rolled identity checks are how this bug gets written in
the first place — the same lesson as Trap 3.

---

## Hands-on proof

Every step below is something **you** run. I have no JVM and no running Spring context, so
nothing here is captured output — what you get is the exact configuration, what to look
for, and how to read each possible result.

### Setup

`src/main/resources/application-lab.yml`:

```yaml
logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.web.FilterChainProxy: DEBUG
    org.springframework.security.web.access: TRACE
    org.springframework.security.authentication: TRACE
server:
  port: 8080
```

Run with `--spring.profiles.active=lab`.

> **Never leave these on in production.** `org.springframework.security=DEBUG` logs
> request details and authentication outcomes, and at DEBUG the framework itself prints a
> warning telling you the same thing. It is a lab instrument.

### Proof 1 — print the actual filter chain at startup

This is the centrepiece of this topic. Stop guessing which filters are active and read the
list the framework built.

`logging.level.org.springframework.security.web.FilterChainProxy=DEBUG` makes Spring
Security log, at startup, each configured `SecurityFilterChain` with its request matcher
and its ordered list of filters.

**What to look for**, in order:

1. **How many chains were registered.** One line per `SecurityFilterChain` bean.
2. **The `RequestMatcher` for each chain**, and crucially **their order**.
3. **The ordered filter list inside each chain.**

The shape of what you are reading is like the following — **this is an illustration of the
format, not captured output**:

```
Will secure Ant [pattern='/actuator/**'] with filters: DisableEncodeUrlFilter,
  WebAsyncManagerIntegrationFilter, SecurityContextHolderFilter, HeaderWriterFilter,
  ... , AuthorizationFilter
Will secure Ant [pattern='/api/**'] with filters: ...
Will secure any request with filters: ...
```

**How to read it:**

| What you see | What it means |
|---|---|
| Your chains listed in the order you expected | `@Order` is correct. The most specific matcher is first. |
| A broad matcher (`any request`) listed **before** a specific one | **Bug.** The broad chain will swallow every request and the specific chain is dead configuration. Fix the `@Order` values. |
| Only **one** chain, matching `any request`, when you defined several | Your `SecurityFilterChain` beans are not all being registered. Check that the `@Configuration` class is component-scanned and that you did not accidentally define two beans with the same name. |
| `BearerTokenAuthenticationFilter` present | `oauth2ResourceServer(jwt)` took effect (Topic 57). |
| `BearerTokenAuthenticationFilter` **absent** when you configured it | The chain you are reading is not the one you configured, or the `spring-security-oauth2-resource-server` dependency is missing. |
| `CsrfFilter` present on a chain where you called `.csrf(c -> c.disable())` | You are reading a different chain. Match the pattern first. |
| `AuthorizationFilter` is **last** in every chain | Correct and expected. Authorization is the final gate before `DispatcherServlet`. |
| `AnonymousAuthenticationFilter` present | Why unauthenticated requests have a principal (Trap 5). |

At DEBUG level, per request, you will also get lines showing which chain was selected and
which filter is being invoked. That per-request trace is how you answer "why did this
request get a 403" without guessing.

### Proof 2 — prove authorization runs before your controller

Add a controller and a deliberate marker:

```java
@RestController
class MarkerController {
    private static final Logger log = LoggerFactory.getLogger(MarkerController.class);

    @GetMapping("/api/orders")
    Map<String, String> orders() {
        log.info(">>> CONTROLLER REACHED <<<");
        return Map.of("ok", "true");
    }
}
```

```bash
curl -i localhost:8080/api/orders                       # no credentials
curl -i -u alice@orderflow.test:correct-horse localhost:8080/api/orders
```

| What you see | What it means |
|---|---|
| 401, and **`>>> CONTROLLER REACHED <<<` does not appear** in the log | **The proof.** The request was rejected in the filter chain. `DispatcherServlet` never ran, no handler was matched, no argument was resolved. This is the concrete difference from a Nest guard. |
| 200 with credentials, and the marker **does** appear | The chain allowed it through. Normal. |
| The marker appears even on the 401 | Something is wrong: either the endpoint is matched by a `permitAll()` rule and something else produced the 401, or you are looking at a different endpoint. Re-check with the per-request chain logging. |

Now do the same for a URL that does not exist at all:

```bash
curl -i localhost:8080/api/this-endpoint-does-not-exist
```

| What you see | What it means |
|---|---|
| **403** (with `anyRequest().denyAll()`) | Spring Security rejected it before Spring MVC could determine that nothing maps to it. This is the "authorization before mapping" property, demonstrated. |
| **401** (with `anyRequest().authenticated()`) | Same property, different rule. |
| **404** | The request escaped the filter chain entirely and reached `DispatcherServlet`. **This is Trap 2.** You have no catch-all chain, or its matcher does not cover this path. |

That last row is worth internalising: **a 404 on an unauthenticated request to a
nonexistent path means your security is not covering that path.**

### Proof 3 — see the `ThreadLocal` and watch it not propagate

```java
@RestController
class ContextProbeController {

    private final AsyncProbe asyncProbe;

    ContextProbeController(AsyncProbe asyncProbe) { this.asyncProbe = asyncProbe; }

    @GetMapping("/api/probe")
    Map<String, String> probe() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        asyncProbe.fromAnotherThread();
        return Map.of(
            "thread", Thread.currentThread().getName(),
            "principal", auth == null ? "null" : auth.getName(),
            "authenticated", auth == null ? "null" : String.valueOf(auth.isAuthenticated()),
            "class", auth == null ? "null" : auth.getClass().getSimpleName());
    }
}

@Service
class AsyncProbe {
    private static final Logger log = LoggerFactory.getLogger(AsyncProbe.class);

    @Async
    public void fromAnotherThread() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        log.info("ASYNC thread={} principal={} class={}",
                 Thread.currentThread().getName(),
                 auth == null ? "null" : auth.getName(),
                 auth == null ? "null" : auth.getClass().getSimpleName());
    }
}
```

```bash
curl -s localhost:8080/api/probe | jq                                  # anonymous
curl -s -u alice@orderflow.test:correct-horse localhost:8080/api/probe | jq
```

| What you see | What it means |
|---|---|
| Anonymous call: `principal: "anonymousUser"`, `class: "AnonymousAuthenticationToken"`, `authenticated: "true"` | **Trap 5 proven.** An unauthenticated request has a non-null principal and `isAuthenticated()` is true. A null check is not an authentication check. |
| Authenticated call: `principal` is alice's username, `class: "UsernamePasswordAuthenticationToken"` | Normal. |
| HTTP response `thread` is `http-nio-8080-exec-N`; the ASYNC log line's thread is a task-executor thread with a **different** name | The thread boundary, visible. |
| ASYNC log line: `principal=anonymousUser` or `principal=null` while the HTTP response shows alice | **Trap 4 proven.** The context did not cross the boundary. |
| ASYNC log line shows alice | You have already wrapped the executor with `DelegatingSecurityContextExecutor` — or `@Async` is not actually applying (it is proxy-based too; Topic 40 rules apply, so check for self-invocation). |

Now apply the fix — wrap the executor in `DelegatingSecurityContextExecutor` — restart,
and re-run. The ASYNC line should now name alice while the thread name still differs.
**Both facts together are the point:** a different thread, and the context arrived anyway,
because you copied it explicitly.

### Proof 4 — read the authorization decision for one request

With `logging.level.org.springframework.security.web.access=TRACE`, each request produces
lines showing which authorization rule was consulted and what it decided.

| What to look for | How to read it |
|---|---|
| A line naming the matcher that matched your request | If it is not the rule you expected, your ordering is wrong (Trap 1) |
| A line showing the decision (granted / denied) and the authorities considered | If denied, compare the required authority against the ones in the `Authentication` — a `ROLE_` prefix mismatch is the usual cause (Topic 57) |
| No authorization lines at all | The request never reached `AuthorizationFilter`. Either an earlier filter rejected it, or no chain matched (Trap 2). |

### Proof 5 — the full `curl` matrix

Run every combination and record the status. This table is your security specification,
and it should become the `MockMvc` test suite from Trap 1.

```bash
BASE=localhost:8080
GOOD='-u alice@orderflow.test:correct-horse'

curl -s -o /dev/null -w '%{http_code} GET  /api/products/1001 anon\n'   $BASE/api/products/1001
curl -s -o /dev/null -w '%{http_code} GET  /api/orders        anon\n'   $BASE/api/orders
curl -s -o /dev/null -w '%{http_code} GET  /api/orders        auth\n'   $GOOD $BASE/api/orders
curl -s -o /dev/null -w '%{http_code} GET  /api/admin/refunds auth\n'   $GOOD $BASE/api/admin/refunds
curl -s -o /dev/null -w '%{http_code} POST /api/products      anon\n'   -X POST $BASE/api/products
curl -s -o /dev/null -w '%{http_code} GET  /api/nonexistent   anon\n'   $BASE/api/nonexistent
curl -s -o /dev/null -w '%{http_code} GET  /actuator/health   anon\n'   $BASE/actuator/health
curl -s -o /dev/null -w '%{http_code} GET  /internal/anything anon\n'   $BASE/internal/anything
```

| Row | Expected | If you see something else |
|---|---|---|
| `GET /api/products/1001` anon | 200 | 401 → Trap 1, a broader rule shadowed `permitAll()` |
| `GET /api/orders` anon | 401 | 403 → the entry point is not configured, so an anonymous denial is reported as forbidden rather than unauthenticated |
| `GET /api/orders` auth | 200 | 403 → the user lacks the required authority; check the `ROLE_` prefix |
| `GET /api/admin/refunds` as a customer | 403 | 200 → **an authorization hole.** Stop and fix. |
| `POST /api/products` anon | 401 or 403 | 200 → your matcher lacks an HTTP method and the catalogue is writable by anyone |
| `GET /api/nonexistent` anon | 403 (with `denyAll()`) | 404 → **Trap 2**, the request escaped the filter chain |
| `GET /actuator/health` anon | 200 | 401 → the health path is not in your `permitAll()` list; Kubernetes probes will fail and your pods will restart (Topic 121) |
| `GET /internal/anything` anon | 401/403 | 200 → the internal chain is not matching; check `@Order` |

---

## Practice exercises

### 1 — Easy: read the chain, then predict it

1. Enable `logging.level.org.springframework.security.web.FilterChainProxy=DEBUG` and
   start `orderflow`. Write down, in order, every chain and every filter in the chain that
   matches `/api/**`.
2. Now **predict** what changes before you make each of these edits, then make them and
   check:
   - add `.csrf(Customizer.withDefaults())` (re-enable CSRF)
   - add `.formLogin(Customizer.withDefaults())`
   - change `SessionCreationPolicy.STATELESS` to `IF_REQUIRED`
   - remove `.httpBasic(...)` entirely
3. For each edit, name which filter appeared or disappeared, and say in one sentence why.
4. Finally, run the Proof 5 `curl` matrix and reconcile every status code with a specific
   filter or rule. Any status you cannot explain is a gap in your model — find it.

### 2 — Medium: the audit (combines Topics 37, 39, 40, 44, 46, 54)

The configuration below contains **six** defects. For each: name it, state the exact
observable symptom (an HTTP status, a log line, or a database row — not "bad practice"),
and write the fix.

```java
@Configuration
public class SecurityConfig {

    @Autowired private CustomerRepository customers;

    @Bean
    SecurityFilterChain chain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
                .requestMatchers("/api/products/**").permitAll()
                .requestMatchers("/actuator/**").permitAll()
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }

    @Bean
    UserDetailsService userDetailsService() {
        return username -> {
            Customer c = customers.findByEmail(username).orElse(null);
            return User.withUsername(c.email())
                       .password(c.plaintextPassword())
                       .roles("CUSTOMER")
                       .build();
        };
    }

    @Bean
    PasswordEncoder encoder() {
        return NoOpPasswordEncoder.getInstance();
    }
}

@RestController
class AdminController {

    @GetMapping("/api/admin/refunds")
    List<Refund> refunds() {
        return refundService.all();       // no check anywhere
    }

    @PostMapping("/api/admin/refunds")
    @Async
    public void issueRefund(RefundRequest req) {
        String actor = SecurityContextHolder.getContext().getAuthentication().getName();
        refundService.issue(req, actor);
    }
}
```

Hints by topic: 39 (what injection style is that, and what does it cost?), 40 (is `@Async`
on a controller method going to work?), 44 (what does `@Async` return type imply for the
HTTP response?), 46 (what error format do those 401s use?), 54 (is the refund
transactional, and on which thread?), and this topic (rule order, encoder, anonymous
principal, missing catch-all).

### 3 — Hard: production simulation — secure the `orderflow` API and prove it

**Part A — build the chains.** Implement the four-chain configuration from Example 2
against `orderflow`: actuator, internal, API, fallback. Use HTTP Basic for now; Topic 57
replaces it with JWT. Requirements: `GET /api/products/**` public, `POST /api/orders`
requires `ROLE_CUSTOMER`, `/api/admin/**` requires `ROLE_ADMIN`, `/actuator/health/*`
public and everything else under `/actuator` restricted, and every unmatched path denied.

**Part B — prove the ordering.** Capture the startup `FilterChainProxy` DEBUG output and
annotate it: for each chain, say which requests it will handle and which it will not.
Deliberately swap two `@Order` values, capture the output again, and show which chain
became unreachable. Restore.

**Part C — the specification as a test suite.** Write a `MockMvc` test with **at least
twelve** cases covering: anonymous vs customer vs admin, each significant path, and both
`GET` and `POST` where the methods differ. Include the "unmapped path is denied" case.
This suite is the deliverable — it is what stops the next person reintroducing Traps 1
and 2.

**Part D — the async boundary.** Add an `@Async` audit write that records who placed each
order. Run it **without** `DelegatingSecurityContextExecutor` first, place ten orders as
alice, and show the `actor` column. Then add the wrapper and repeat. Produce a
before/after table of the actual database rows. Explain in three sentences why no
exception was thrown in the broken version, and why that makes it more dangerous than a
crash.

**Part E — the error contract.** Make 401 and 403 responses use the same RFC 9457
`ProblemDetail` shape as your `@ControllerAdvice` from Topic 46. Prove with `curl -i` that
a business error and an auth error are indistinguishable in *shape* to a client. Then
explain why `authException.getMessage()` must not appear in the `detail` field.

**Part F — argue the other side.** Your team lead proposes dropping the URL-pattern rules
entirely and using only `@PreAuthorize` on controller methods. Write the strongest case
for that position, then the strongest case against. Name one `orderflow` rule that is
easier to express each way.

---

## Interview questions

### Q1 — "How does Spring Security differ from a Nest guard or an Express middleware?"

**Mid-level answer:** "Spring Security uses filters. It's basically middleware that checks
authentication before your controller."

**Senior answer:** "Structurally it's a servlet filter chain, which means it runs before
`DispatcherServlet` — before Spring MVC has consulted any `HandlerMapping`. A Nest guard
runs inside the framework pipeline, after routing, which is why `context.getHandler()` can
give you the handler's metadata. The practical consequences are three. First, authorization
can reject a request before any controller mapping is consulted, so an unmapped URL and a
forbidden URL are rejected by the same code — a request to a nonexistent path under a
protected prefix returns 403, not 404. Second, the rules are URL-pattern-based and live in
a central configuration rather than as decorators on the controller, which makes them
auditable but means the config and the controllers can drift. Third, a rejection never
reaches `@ControllerAdvice`, so your `ProblemDetail` error contract has to be configured
separately in the security config or you end up with two error formats. And the piece that
actually bites in production is that the principal lives in a `ThreadLocal` rather than on
the request object, so it silently doesn't cross `@Async`, an executor, or a Reactor
operator."

**What separates them:** the position in the stack and its three concrete consequences,
plus the `ThreadLocal` detail — which is the one that shows they have debugged it rather
than read about it.

**Follow-up:** "You said a request to a nonexistent path returns 403. What if it returns
404 instead?" *(Then no `SecurityFilterChain` matched that path and the request escaped
Spring Security entirely — Trap 2. The fix is a lowest-precedence catch-all chain with
`denyAll()`.)*

---

### Q2 — "Name the filters in the chain and what each does."

**Mid-level answer:** "There's an authentication filter and an authorization filter, and
some others for CSRF and CORS."

**Senior answer:** "There are around fifteen and I wouldn't recite them — I'd print them
with `logging.level.org.springframework.security.web.FilterChainProxy=DEBUG`, which logs
every chain with its matcher and its ordered filters at startup. Four matter for
debugging. `SecurityContextHolderFilter` loads the context into the `ThreadLocal` at the
start of the request and clears it in a `finally` at the end — that `finally` is why the
context doesn't leak to the next request on a pooled thread, and also why it's gone by the
time an async task runs. `AnonymousAuthenticationFilter` installs an
`AnonymousAuthenticationToken` if nothing authenticated, which is why `getAuthentication()`
is rarely null and why a null check is not an authentication check.
`ExceptionTranslationFilter` turns an `AuthenticationException` into a 401 via the entry
point and an `AccessDeniedException` into a 403. And `AuthorizationFilter` is last — it
evaluates `authorizeHttpRequests` and is literally the last thing before
`DispatcherServlet`. Knowing the order matters because a custom filter added in the wrong
position — say, before the context is loaded — sees no authentication."

**What separates them:** refusing to recite, naming the instrument, and explaining *why*
each of the four matters rather than what it is called.

**Follow-up:** "You need to add a custom filter that reads a tenant header and needs the
authenticated user. Where does it go and how do you register it?"
*(After the authentication filter and before `AuthorizationFilter`, registered with
`http.addFilterAfter(...)` or `addFilterBefore(...)` naming a filter class as the
position anchor.)*

---

### Q3 — "Why did my `permitAll()` endpoint start returning 401?"

**Mid-level answer:** "Maybe the matcher pattern is wrong, or CSRF is blocking it."

**Senior answer:** "First hypothesis is rule ordering. `authorizeHttpRequests` builds an
ordered list evaluated top to bottom with first-match-wins, so a broader pattern earlier —
`/api/**` before `/api/products/**`, or an `anyRequest()` anywhere but last — shadows the
`permitAll()` and it never runs. Recent versions fail fast if you literally put a matcher
after `anyRequest()`, but the subtler shadowing cases produce no error at all. Second
hypothesis is chain selection: if there are several `SecurityFilterChain` beans, the first
one whose `securityMatcher` matches handles the request, so I might be editing a chain
that never runs. I'd confirm both by turning on `FilterChainProxy` DEBUG, which prints
every chain and its matcher at startup and, per request, which chain was selected. Third,
if it's a POST rather than a GET, check whether the matcher specified an HTTP method and
whether CSRF is enabled. And the structural fix is a `MockMvc` test asserting the actual
status for every significant path, because ordering bugs are invisible to code review."

**What separates them:** an ordered set of hypotheses, knowing the first-match-wins
semantics at both the chain and the rule level, naming the exact diagnostic, and finishing
with prevention.

**Follow-up:** "Ordering bugs are invisible in review. How do you prevent them at the team
level?" *(A test suite that asserts status per path, including an unmapped path; and
`denyAll()` as the catch-all so a missing rule fails loudly.)*

---

### Q4 — "Where does the current user live, and what breaks?"

**Mid-level answer:** "In `SecurityContextHolder`. You call
`SecurityContextHolder.getContext().getAuthentication()`."

**Senior answer:** "In a `SecurityContext` held in a `ThreadLocal` by
`SecurityContextHolder`, populated by `SecurityContextHolderFilter` at the start of the
request and cleared in a `finally` at the end. Two things break. First, it doesn't cross
thread boundaries: an `@Async` method, a task on a raw `ExecutorService`, a
`CompletableFuture` continuation on the common pool, or a Reactor operator all see an
empty context, so an audit records `anonymousUser` and a `@PreAuthorize` check silently
evaluates against an anonymous principal — with no exception, which is what makes it
dangerous. The fix is `DelegatingSecurityContextExecutor` or
`DelegatingSecurityContextRunnable`, or, better, passing the identity explicitly as data.
`MODE_INHERITABLETHREADLOCAL` is the trap answer: inheritance happens at thread creation
and pool threads were created before the request, so it works in a test that news up a
thread and fails in production. Second, `getAuthentication()` is rarely null even for an
unauthenticated request, because `AnonymousAuthenticationFilter` installs an
`AnonymousAuthenticationToken` whose `isAuthenticated()` returns true — so the correct
check also excludes `AnonymousAuthenticationToken`, and better still is to let the filter
chain's `.authenticated()` decide. It's the same `ThreadLocal` propagation problem as the
transaction binding, the MDC and the trace context — one lesson, four places."

**What separates them:** the `finally`-clear detail, the specific fix, pre-empting the
`INHERITABLETHREADLOCAL` trap, the anonymous-token subtlety, and generalising to the other
three `ThreadLocal` contexts.

**Follow-up:** "Does switching to virtual threads fix this?" *(No. Each virtual thread has
its own `ThreadLocal`s, so the semantics are identical — you just have far more threads.
`ScopedValue` is the direction the platform is heading, and Spring Security is a
`ThreadLocal`-based API today. Topics 101 and 102.)*

---

### Q5 — "A request reached your controller with no authentication. How?"

**Mid-level answer:** "The endpoint must be in a `permitAll()` rule."

**Senior answer:** "Three possibilities, in the order I'd check them. One: it matches a
`permitAll()` rule, possibly one that's broader than intended — a missing HTTP method on
the matcher is a common version of this, where `GET /api/products/**` was meant to be
public and `POST /api/products` accidentally is too. Two: **no `SecurityFilterChain`
matched the request at all**, in which case `FilterChainProxy` runs no filters and the
request goes straight to `DispatcherServlet` unprotected. That happens whenever every
chain has a narrowing `securityMatcher` and someone adds a controller outside all of them.
The tell is that an unauthenticated request to a nonexistent path returns 404 instead of
403. The fix is a lowest-precedence catch-all chain with `anyRequest().denyAll()`. Three:
the request never went through the filter chain at all — an internal forward, a scheduled
job, or a message-listener path that isn't HTTP. I'd confirm which by turning on
`FilterChainProxy` DEBUG and looking at which chain was selected per request, and I'd add
a test asserting that an unmapped path under each prefix is denied."

**What separates them:** knowing that no-chain-matched means no-security, using the 404
signal as a diagnostic, and remembering non-HTTP entry points.

**Follow-up:** "Given (2), why doesn't Spring just deny by default when nothing matches?"
*(Because `FilterChainProxy` is a servlet filter among possibly many, and denying
everything it wasn't configured for would break every non-Spring-Security servlet in the
application. Boot's default single chain does secure everything — the hole only appears
once you add a narrowing `securityMatcher`, which is exactly why the catch-all chain is
your responsibility.)*

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Spring Security could have been implemented as a `HandlerInterceptor` inside Spring MVC,
   which would have made it routing-aware like a Nest guard. It is a servlet filter
   instead. Name what that buys and what it costs, and give one security rule that is
   *harder* to express because of the choice.

2. `AnonymousAuthenticationFilter` exists so that unauthenticated requests have a
   principal rather than none. Argue that this is good design, then make the strongest
   case that it is a footgun. Which side wins, and under what team conditions?

3. `SecurityContextHolderFilter` clears the context in a `finally` block. Describe exactly
   what would go wrong if it did not, in a server using a thread pool — and connect that to
   Topic 120's MDC leak.

4. You have four `SecurityFilterChain` beans and first-match-wins. Devise an ordering
   mistake that produces **no** error, **no** failing test, and a real security hole. What
   single practice would have caught it?

5. In Nest, `req.user` travels with the request object; in Spring it travels with the
   thread. Each choice makes a different set of bugs easy. Name one bug that is easy in
   Nest and impossible in Spring, and one that is easy in Spring and impossible in Nest.

6. `@PreAuthorize` (Topic 57) is proxy-based and therefore inherits every caveat of Topic
   40. Give a concrete `orderflow` example where a `@PreAuthorize` check silently does not
   run, and say what the resulting security impact would be.

7. Your API returns 403 for a path that does not exist, which leaks less information than
   404 but confuses developers. Argue for each behaviour as a policy, and say which you
   would pick for `orderflow` and why.

---

## Quick reference card

### The filter chain, abridged

```
DelegatingFilterProxy -> FilterChainProxy
   -> first SecurityFilterChain whose securityMatcher matches:
        SecurityContextHolderFilter        load/clear the ThreadLocal SecurityContext
        HeaderWriterFilter                 security response headers
        CorsFilter                         CORS preflight
        CsrfFilter                         CSRF token check (Topic 57)
        LogoutFilter
        <authentication filters>           UsernamePassword / BearerToken / Basic / X509
        AnonymousAuthenticationFilter      install anonymous principal if nothing else did
        ExceptionTranslationFilter         AuthenticationException -> 401
                                           AccessDeniedException  -> 403
        AuthorizationFilter                evaluate authorizeHttpRequests  <-- LAST
   -> DispatcherServlet -> your controller
```

### The configuration shape

```java
@Bean
@Order(1)                                        // most specific chain first
SecurityFilterChain chain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")              // WHICH requests this chain handles
        .authorizeHttpRequests(auth -> auth      // rules, first match wins
            .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().denyAll())             // catch-all LAST, and deny
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .csrf(csrf -> csrf.disable())            // only correct for header auth. Topic 57.
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .exceptionHandling(ex -> ex.authenticationEntryPoint(entryPoint))
        .build();
}
```

### Reading the current user

```java
@AuthenticationPrincipal UserDetails principal          // controller param — PREFERRED
Authentication authentication                           // controller param
SecurityContextHolder.getContext().getAuthentication()  // anywhere on this thread
```

Correct "is authenticated" check:
```java
auth != null && auth.isAuthenticated() && !(auth instanceof AnonymousAuthenticationToken)
```

### Diagnostics

```yaml
logging.level.org.springframework.security: DEBUG
logging.level.org.springframework.security.web.FilterChainProxy: DEBUG   # prints the chain
logging.level.org.springframework.security.web.access: TRACE
```

```bash
curl -i localhost:8080/api/orders                       # expect 401
curl -i -u user:pass localhost:8080/api/orders          # expect 200
curl -i localhost:8080/api/no-such-path                 # 403 = secured, 404 = ESCAPED
```

### Status code meanings

| Status | Meaning | Produced by |
|---|---|---|
| 401 | "I don't know who you are" | `ExceptionTranslationFilter` → `AuthenticationEntryPoint` |
| 403 | "I know who you are, and no" | `ExceptionTranslationFilter` → `AccessDeniedHandler` |
| 404 on an unauthenticated request to a protected prefix | **No chain matched.** Security escaped. | nothing — that is the problem |

### Gotchas checklist

- [ ] Rules are first-match-wins; `anyRequest()` must be last
- [ ] Chains are first-match-wins; most specific `@Order` first
- [ ] Always define a lowest-precedence catch-all chain with `denyAll()`
- [ ] Put the HTTP method in the matcher when GET and POST differ
- [ ] `getAuthentication() != null` is **not** an authentication check
- [ ] `isAuthenticated()` is `true` for anonymous — exclude `AnonymousAuthenticationToken`
- [ ] `SecurityContext` does **not** cross `@Async`, executors, or Reactor operators
- [ ] `MODE_INHERITABLETHREADLOCAL` does not fix thread pools
- [ ] Filter-chain rejections never reach `@ControllerAdvice` — configure the entry point
- [ ] Never put an exception message in an auth error response body
- [ ] Use `PasswordEncoderFactories.createDelegatingPasswordEncoder()`, never `NoOp`
- [ ] `org.springframework.security=DEBUG` is a lab instrument, never production

> `[BOOT 3.x DELTA]` The `SecurityFilterChain`-bean-plus-lambda-DSL style in this doc is
> identical on Boot 3.x (Security 6) and Boot 4.1 (Security 7), so everything here ports
> directly. What changed *before* 3.x: `WebSecurityConfigurerAdapter` was deprecated in
> Security 5.7 and removed in Security 6, and the `.and()` chaining style went with it —
> if you meet `extends WebSecurityConfigurerAdapter`, that is a Boot 2.x codebase.
> Security 7 continues the lambda-DSL-only direction and has tightened further defaults.
> **Where I am not certain of an exact Security 7 method name I have said so rather than
> guessing; check the current Spring Security reference for the version on your
> classpath.** In particular, verify against the reference docs before relying on any
> specific `x509(...)` customizer method or `RequestMatcher` factory name — those helper
> APIs have moved more than the core DSL has.

---

## When would I use this at work?

**1. Joining a codebase and needing to know what is actually protected.**
You do not read `SecurityConfig` and hope. You turn on `FilterChainProxy` DEBUG, read the
chains the framework actually built, and run the `curl` matrix. In twenty minutes you have
a truthful map of the security posture, including the gaps the config *looks* like it
covers. This is one of the highest-leverage things you can do in your first week.

**2. Reviewing a PR that adds an endpoint.**
Three questions, every time: does a rule cover this path and this HTTP method; if it is an
admin path, does the catch-all deny rather than authenticate; and does anything in the
handler run on another thread and then read the principal. Those three catch the large
majority of security regressions in a Spring codebase, and they take thirty seconds.

**3. Debugging "it works for me but the audit log says anonymous".**
You recognise the shape immediately: a `ThreadLocal` context and a thread boundary. You
check whether the write happens in an `@Async` method, a `CompletableFuture`, a scheduled
job or a message listener, and you fix it with explicit identity passing or a delegating
executor. Without this topic, that investigation is an afternoon; with it, it is a
question.

---

## Connected topics

**Prerequisites:**
- **37 — Bean lifecycle**: `SecurityFilterChain` is a bean like any other, and
  `@PreAuthorize` (Topic 57) is applied by a `BeanPostProcessor`-created proxy.
- **39 — Injection styles**: `HttpSecurity` is injected into your `@Bean` method — do not
  construct it.
- **40 — Proxying**: `@PreAuthorize` is proxy-based, so every self-invocation caveat
  applies to method security exactly as it does to `@Transactional`.
- **44 — REST controllers**: the filter chain sits *before* `DispatcherServlet`; knowing
  that request path is what makes the ordering claim meaningful.
- **46 — Error handling**: `@ControllerAdvice` does not see filter-chain rejections, which
  is why the security config needs its own `ProblemDetail` entry point.

**This unlocks:**
- **57 — Authorization, JWT, OAuth2, CSRF, session fixation**: the direct sequel.
  `oauth2ResourceServer(jwt)`, `@PreAuthorize`, and why CSRF depends on how the credential
  travels.
- **60 — Spring test slices**: `@WebMvcTest` plus `spring-security-test`
  (`@WithMockUser`) is how the Trap 1 test suite gets written.
- **91 — `CompletableFuture`**: the same context-propagation problem on a different
  mechanism.
- **101 — Virtual threads**: each has its own `ThreadLocal`s; the problem is identical in
  kind and much larger in scale.
- **119 — Tracing**: trace context is another `ThreadLocal` that drops at exactly the same
  boundaries.
- **120 — MDC**: the third `ThreadLocal`, and the one where the leak-on-pooled-threads
  failure is most visible.
- **121 — Actuator and k8s probes**: `/actuator/health/liveness` and `readiness` must be
  `permitAll()` or your pods fail their probes and restart.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0 / Spring Security
7.0, Jakarta EE 11. The `SecurityFilterChain` bean and lambda DSL shown here are the
supported style on both Security 6 and 7. Filter names and ordering are stable across
those versions, but rather than trusting any document — including this one — print the
chain with `FilterChainProxy` DEBUG and read what your application actually built.*
