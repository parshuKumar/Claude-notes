# 57 — Spring Security II: Authorization, JWT, OAuth2 Resource Server, CSRF, Session Fixation

## Phase: 5 — Spring Boot & Persistence
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: role-based authorization on `orderflow` — three principal kinds (`customer`, `admin`, `internal service`), enforced at the filter chain for coarse URL rules and at the method boundary for ownership rules. The API becomes a stateless OAuth2 resource server validating JWTs; the admin console stays a cookie session, and therefore keeps CSRF protection.

---

## ELI5 anchor

Topic 56 was the **bouncer at the door**: it checks that your ID is real and writes
your name on a wristband.

Topic 57 is everything after the door.

- **Authorization** is the **room list**. The wristband says `customer`. The
  wristband gets you into the shop floor. It does not get you into the stockroom.
  Someone checks the list at every door, not just the front one.
- **A JWT** is a **wristband with your details printed on it and a tamper-proof
  seal**. The venue does not have to phone head office to ask who you are — the
  band says so, and the seal proves nobody wrote on it. That is the whole trick,
  and it is also the whole problem: **once printed, the band cannot be recalled.**
  If you get thrown out, your band still opens doors until it expires. That is why
  bands have a short printed expiry and you swap them at the desk.
- **CSRF** is the trick where a **stranger's poster tells your hand to press a
  button** while you are still wearing your wristband. It only works if the door
  checks the *wristband you are wearing* rather than something you have to type in
  each time. A wristband the venue reads automatically (a **cookie**) is
  vulnerable. A code you type in each time (an `Authorization` header) is not,
  because the stranger's poster cannot make you type it.
- **Session fixation** is a stranger handing you a **wristband they already own**
  before you check in, so that after you check in they are wearing a band that is
  now yours. The fix is that the venue **rips up your band and issues a new one at
  the moment you check in**.

Four ideas. The rest of this document is the mechanics, in Java, with the failure
modes.

---

## The bridge from what you know

You have shipped production NestJS auth: Guards, Passport strategies, JWT, role
decorators. That knowledge transfers about 70%. This section is only the 30% that
does not, because that 30% is where the bugs are.

### Bridge 1 — `Nest Guard` ≈ `filter chain + @PreAuthorize` is PARTIAL

In Nest, `@UseGuards(JwtAuthGuard, RolesGuard)` on a controller method is *one
mechanism*. Both guards run in the same place: inside Nest's request pipeline,
**after** the router has already resolved which handler will run, and after the
pipes have parsed the params.

In Spring there are **two different mechanisms in two different places**, and
conflating them is the single most common source of "why is this returning 401 /
403 / 200 when it should not".

| | Where it runs | What it can see | What it cannot see |
|---|---|---|---|
| **Filter chain** (`authorizeHttpRequests`) | Servlet filter, **before** `DispatcherServlet` | The HTTP method, the URI path, the headers | Which controller method will handle it, the parsed `@PathVariable`, the deserialized `@RequestBody`, the loaded entity |
| **Method security** (`@PreAuthorize`) | An AOP proxy around your bean (Topic 40) | The method arguments, the principal, and — with `@PostAuthorize` — the return value | Nothing about HTTP; it is not an HTTP concept at all |

The consequence you must internalise:

> **Filters run before the dispatcher. Authorization at the filter chain rejects a
> request before Spring MVC has consulted a single `@RequestMapping`.**

A request to `POST /api/admin/products` that fails the URL rule is refused by
`AuthorizationFilter` and **the controller method does not exist as far as that
request is concerned**. There is no handler resolution, no argument resolver, no
`@Valid` run, no `@ControllerAdvice` of yours invoked (the error is written by
`ExceptionTranslationFilter`, not by your `@RestControllerAdvice` — Topic 46's
`ProblemDetail` contract does not automatically cover it unless you wire an `AuthenticationEntryPoint` and an `AccessDeniedHandler`, as Example 2 does).

In Nest, a failed guard still happened *inside* the framework, so your exception
filter caught it and your response shape was uniform. In Spring you have to
deliberately make the filter-chain rejections match your API's error contract.

Second consequence, and it is the one that decides your design:

> The filter chain **cannot** express "a customer may read *their own* order". It
> has a path and a method and nothing else. `/api/orders/8213` is just a string.

Ownership rules are **method security**. URL shape rules are **the chain**. You
need both. In Nest you expressed both with the same decorator and never had to
think about the split.

### Bridge 2 — `SecurityContextHolder` is `ThreadLocal`, and `AsyncLocalStorage` it is not

In Node, `AsyncLocalStorage` follows the async context. You `await`, you spawn a
promise chain, you go through a `setTimeout` — the store follows. This is a
runtime-level feature: the async hooks machinery propagates it for you.

Java has no such machinery. `SecurityContextHolder` defaults to
`ThreadLocalSecurityContextHolderStrategy`: a `ThreadLocal<SecurityContext>`.

> **A `ThreadLocal` is a map keyed by the current thread. Cross a thread boundary
> and the value is simply not there.**

Concretely, in `orderflow`, the context is **silently empty** in:

- an `@Async` method (Topic 91 — it runs on a `TaskExecutor` thread);
- anything you submit to a raw `ExecutorService` you constructed yourself;
- a `parallelStream()` (Topic 25 — the common `ForkJoinPool`);
- a Reactor operator that hops schedulers (Topics 104, 108 — Reactor uses a
  `Context` carried in the subscription, not a `ThreadLocal`);
- a Kafka listener thread (Topic 113) — there was never an HTTP request there at
  all;
- an outbox relay thread (Topic 115) — same.

"Silently empty" is the dangerous part. `SecurityContextHolder.getContext()` never
returns `null`. It returns a **fresh empty `SecurityContext`**. So
`getAuthentication()` returns `null`, and a `@PreAuthorize("hasRole('ADMIN')")` on
the async path denies — or, in the shapes shown in Trap 4, an `isAdmin()` helper
written as `auth != null && auth.getAuthority()...` collapses to `false` and your
code takes the *non-admin* branch without any exception at all.

Forward references, because this problem recurs and gets harder each time:
**Topic 91** (`@Async` and executor propagation), **Topic 101** (virtual threads —
a `ThreadLocal` on a virtual thread works, but a virtual thread you created is
still a *different* thread), **Topic 119** (the identical problem for the OTel
trace context), **Topic 120** (the identical problem for the MDC correlation ID).
Three separate `ThreadLocal`s, one bug shape. Learn it once here.

### Bridge 3 — Passport strategy ≈ `AuthenticationProvider`, but the JWT case skips it

In Nest you write a `JwtStrategy extends PassportStrategy(Strategy)` with a
`validate()` method, register it, and `@UseGuards(AuthGuard('jwt'))`.

In Spring, for the **resource server** case, you usually write **none of that**.
`oauth2ResourceServer(oauth2 -> oauth2.jwt(...))` installs
`BearerTokenAuthenticationFilter`, which delegates to a `JwtDecoder` bean.
Signature verification, `exp`/`nbf` checks, issuer and audience validation are
library code you configure with properties, not code you write.

The piece you *do* write is the **claims → authorities mapping** (a
`JwtAuthenticationConverter`), because only you know that your identity provider
puts roles in `realm_access.roles` and your code wants `ROLE_ADMIN`. That
conversion is the direct analogue of Passport's `validate()` return value, and it
is the only place your code touches the token.

### Bridge 4 — what does not transfer at all

- **`ROLE_` prefix.** `hasRole("ADMIN")` checks for the authority string
  `ROLE_ADMIN`. `hasAuthority("ADMIN")` checks for the literal `ADMIN`. Getting
  this wrong produces a 403 with a perfectly correct-looking token. There is no
  Nest equivalent of this convention and it catches everyone once.
- **The `PasswordEncoder` is mandatory.** Spring Security has no "just compare the
  strings" mode. A `NoOpPasswordEncoder` exists, is `@Deprecated`, and will fail a
  security review.
- **Order matters in the matcher list**, first-match-wins, and it is not sorted for
  you by specificity the way some routers do. Trap 1 is entirely about this.

---

## What is this?

Four distinct subjects that share one configuration file. Keeping them distinct in
your head is most of the work.

### 1. Authorization

**Authentication** answers *who are you*. **Authorization** answers *may you do
this*. Topic 56 delivered a populated `SecurityContext` with an `Authentication`
holding a principal and a `Collection<? extends GrantedAuthority>`. Authorization
is the decision made from those authorities.

Spring Security 6 and 7 express every decision through one interface:

```java
// The shape of the abstraction. Check the current reference docs for the exact
// generic signature in Security 7 before implementing your own.
public interface AuthorizationManager<T> {
    AuthorizationDecision check(Supplier<Authentication> authentication, T object);
}
```

Two things about that signature are load-bearing:

- The `Authentication` arrives as a **`Supplier`**, not a value. That is
  deliberate: for a `permitAll()` rule the manager never calls `get()`, so the
  session is never read and no authentication work is done for a public endpoint.
- `T` is the thing being guarded. For the filter chain it is a
  `RequestAuthorizationContext` (the request plus any URI variables the matcher
  captured). For method security it is a `MethodInvocation`.

Two enforcement points use it:

- **`AuthorizationFilter`** — the last filter in the chain, running
  `authorizeHttpRequests` rules against the request.
- **`AuthorizationManagerBeforeMethodInterceptor`** (and its `After` sibling) — the
  AOP interceptor behind `@PreAuthorize` / `@PostAuthorize`.

### 2. JWT and the OAuth2 resource server

A **JWT** (JSON Web Token) is three base64url segments joined by dots:
`header.payload.signature`.

*Illustration of the structure, not captured output. `xxx` stands for base64url
characters you would see in a real token.*

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6Inh4eCJ9 . xxxxxxxxxxxxxxxx . xxxxxxxxxxxxxxxx
|--------------- header ---------------------------|   |-- payload --|    |- signature -|

header  (decoded, illustrative):  {"alg":"RS256","typ":"JWT","kid":"<key-id>"}
payload (decoded, illustrative):  {
                                    "iss":"https://idp.internal/realms/orderflow",
                                    "sub":"<user-id>",
                                    "aud":"orderflow-api",
                                    "exp":<unix-seconds>,
                                    "iat":<unix-seconds>,
                                    "jti":"<token-id>",
                                    "scope":"orders:read orders:write",
                                    "realm_access":{"roles":["customer"]}
                                  }
```

The payload is **signed, not encrypted**. Anyone holding the token can read every
claim. Never put anything in a JWT that you would not print on a postcard.

A **resource server** is a service that *consumes* tokens it did not issue. It does
not have a login page. It does not have a user database for authentication
purposes. It receives `Authorization: Bearer <token>`, verifies the signature
against the issuer's public keys, checks `exp`, `iss` and `aud`, maps claims to
authorities, and proceeds. `orderflow`'s `/api/**` is a resource server.

The public keys come from the issuer's **JWKS** (JSON Web Key Set) endpoint,
discovered from the issuer URL. `NimbusJwtDecoder` fetches and caches them, and
selects the right one using the token header's `kid`. This is why key rotation at
the identity provider does not require a deploy of `orderflow`.

### 3. CSRF

**Cross-Site Request Forgery.** `evil.example` serves a page containing:

```html
<form action="https://orderflow.internal/api/wallet/transfer" method="POST">
  <input type="hidden" name="to" value="attacker-account">
  <input type="hidden" name="amount" value="500">
</form>
<script>document.forms[0].submit()</script>
```

Your logged-in user visits that page. The browser submits the form to
`orderflow.internal`. **The browser attaches the `orderflow.internal` session
cookie automatically**, because that is what cookies are for. Your server sees a
fully authenticated request to transfer money.

The precise definition you must be able to state:

> **CSRF is an attack on *ambient* credentials — credentials the browser attaches
> to a cross-origin request without the page asking.** Cookies, HTTP Basic, and
> TLS client certificates are ambient. An `Authorization: Bearer` header is not.

The attacker's page cannot set an `Authorization` header on a cross-origin request
that the browser will send without a successful CORS preflight, and it cannot read
your token out of `localStorage` on your origin. That is why a purely
header-authenticated API does not need CSRF tokens — **and why "disable CSRF for
APIs" is dangerous advice repeated without its precondition.**

### 4. Session fixation

If your app has a session (the admin console does), the attack is:

1. Attacker obtains a session id from your server — trivially, by visiting it.
2. Attacker gets the victim's browser to adopt that session id (a link with the id
   in the URL if you allow URL rewriting, a subdomain cookie injection, an XSS).
3. Victim logs in. The server **upgrades the existing session** to authenticated.
4. Attacker now holds an authenticated session id.

The fix is a one-liner and Spring Security does it by default: **on successful
authentication, change the session identifier**. `changeSessionId()` keeps the
session attributes and issues a new id; `newSession()` starts a clean one.

---

## Why does it matter?

Four reasons, in the order they will bite you.

**1. Authorization failures are silent successes.** A missing `@PreAuthorize` does
not throw. It returns `200 OK` with somebody else's order in the body. Every other
class of bug in this curriculum announces itself — an exception, a timeout, a
latency spike. This one ships, passes tests, and is discovered by a customer or a
penetration tester. It is the only Phase 5 topic where the *absence* of an error is
the failure.

**2. The `permitAll()`-ordering bug is a two-way failure.** Ordered wrong one way
you get a 401 on your health endpoint and Kubernetes restarts your pods (Topic
121). Ordered wrong the other way you get `anyRequest().permitAll()` shadowing your
admin rules and the stockroom door is open. The mechanism is identical; only the
direction of the damage changes.

**3. "Stateless" is a claim about *your server*, not about *the token*.** The
moment you choose JWTs you have chosen "an issued credential is valid until it
expires, and I cannot take it back". Every incident-response runbook that says
"revoke the compromised user's access" now needs a specific answer. Deciding this
at 3am during an incident is not a plan.

**4. This is the last topic in Phase 5.** After this the `orderflow` gate says the
service has "a full REST API, Postgres persistence, transactional order placement,
JWT auth, optimistic locking" — and cannot prove any of it. Phase 6 tests it,
Phase 7 loads it. Everything from Topic 65 onward runs against an authenticated
API, which means your k6 script needs a token-acquisition step and your load
profile now includes the cost of signature verification per request. Getting the
auth design wrong here means redoing the baseline.

---

## Syntax breakdown

### A standing honesty note on Spring Security 7 API shapes

This document targets **Spring Security 7.0** (riding along with Spring Boot 4.1).
Security 7 removed a long tail of APIs deprecated across the 6.x line, and renamed
some builder entry points. Where I am confident, the code below is exact. Where the
7.0 method name or nesting could plausibly differ from what I write, I say so **on
the line itself**, and you should confirm against the current
`docs.spring.io/spring-security/reference/` before typing it. I would rather flag
four uncertainties than have you debug an invented method name.

The specific things I am flagging up front, collected in one place so you can check
them in one sitting:

| Thing | Why I am flagging it | How to settle it |
|---|---|---|
| The **request-matcher builder** for non-trivial patterns | `AntPathRequestMatcher` was deprecated during 6.x in favour of a `PathPatternRequestMatcher`-style builder. I am not certain of the exact 7.0 factory-method name or whether `requestMatchers(String...)` still covers every case you need. | Reference docs, "Authorize HTTP Requests" → "Matching Requests". Plain `requestMatchers("/path/**")` and `requestMatchers(HttpMethod.POST, "/path")` are the forms I use below and are the ones least likely to have moved. |
| The exact nesting of the **session-fixation** sub-DSL | I write `sessionManagement(s -> s.sessionFixation(f -> f.changeSessionId()))`. The lambda-of-a-lambda nesting is the part I am least sure survived unchanged into 7.0. | Reference docs, "Session Management". The **default is already `changeSessionId()`**, so in practice you rarely write it — you write it to state intent. |
| Whether `AbstractHttpConfigurer::disable` is still the idiom for `.csrf(...)` | It has been stable for years, but 7.0 tightened several DSL entry points. | Reference docs, "CSRF". |
| The precise `AuthorizationManager` generic signature and the composition helpers (`AuthorizationManagers.allOf` / `anyOf`) | These moved and gained overloads across 6.x. | Reference docs, "Authorization Architecture". |
| The default algorithm id emitted by `PasswordEncoderFactories.createDelegatingPasswordEncoder()` | It has changed once already (from `bcrypt`); it may be `bcrypt` or `argon2` on your version. | Print `encoder.encode("x")` in a test and read the `{prefix}`. This is a one-line experiment, not a docs question. |

Everything below that is **not** in that table is API I am confident about.

### The `SecurityFilterChain` bean, and the lambda DSL

Since Security 5.7 the configuration is a `@Bean` of type `SecurityFilterChain`
built from an `HttpSecurity` object. The `WebSecurityConfigurerAdapter` you will
find in every older tutorial is **gone** — not deprecated, removed.

```java
package com.orderflow.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
class ApiSecurityConfig {

    @Bean
    SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
        return http
            // 1. WHICH REQUESTS DOES THIS CHAIN HANDLE AT ALL?
            .securityMatcher("/api/**")

            // 2. AUTHORIZATION RULES. FIRST MATCH WINS. ORDER IS THE SEMANTICS.
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/orders").hasRole("CUSTOMER")
                .anyRequest().authenticated()
            )

            // 3. HOW IS AUTHENTICATION ESTABLISHED?
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))

            // 4. NO SESSION. NO COOKIE. THEREFORE NO CSRF SURFACE.
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())

            .build();
    }
}
```

Read it as four questions, and answer them in that order every time you write one.

**`securityMatcher(...)` vs `requestMatchers(...)` — the distinction that causes
Trap 2.**

- `securityMatcher` decides **whether this whole `SecurityFilterChain` applies to
  the request at all**. A request that matches no chain's `securityMatcher` goes
  through **no security filters** and lands on your controller unauthenticated.
- `requestMatchers` inside `authorizeHttpRequests` decides **which rule applies**,
  once a chain has been selected.

One is chain selection. The other is rule selection. They read almost identically
and mean completely different things.

**`Customizer.withDefaults()`** is the lambda-DSL idiom for "enable this feature
with no further configuration". It exists because the DSL takes a
`Customizer<T>` everywhere and `withDefaults()` is the no-op customizer. You will
see it constantly.

**There is no `.and()`.** In the pre-lambda DSL you chained
`.csrf().disable().and().authorizeRequests()...`. The lambda DSL replaced it: each
top-level call returns `HttpSecurity`, so you just keep chaining. If you find
`.and()` in a snippet, that snippet predates Security 5.7 and probably contains
other removed API too.

### `authorizeHttpRequests` — the rule vocabulary

```java
.authorizeHttpRequests(auth -> auth

    // --- matchers: WHICH requests ---
    .requestMatchers("/api/products/**")                 // path pattern, any method
    .requestMatchers(HttpMethod.GET, "/api/orders/**")   // method + path
    .requestMatchers("/api/a/**", "/api/b/**")           // several patterns, one rule
    .anyRequest()                                        // catch-all. ALWAYS LAST.

    // --- rules: WHAT is required ---
    .permitAll()                       // no authentication performed at all
    .denyAll()                         // always 403. Useful to shut a path off hard.
    .authenticated()                   // any authenticated principal
    .hasRole("ADMIN")                  // authority "ROLE_ADMIN"
    .hasAnyRole("ADMIN", "SUPPORT")    // "ROLE_ADMIN" or "ROLE_SUPPORT"
    .hasAuthority("SCOPE_orders:write")// literal authority string, NO prefix added
    .hasAnyAuthority("SCOPE_a", "SCOPE_b")
    .access(customAuthorizationManager) // anything else
)
```

Three facts about this block that you must hold simultaneously:

1. **Rules are evaluated top to bottom and the first matching rule wins.** Not the
   most specific. The first. This is Trap 1 and it is the highest-frequency bug in
   the whole topic.
2. **`hasRole("X")` prepends `ROLE_`.** `hasAuthority("X")` does not. A JWT whose
   claims map to the authority `ADMIN` will fail `hasRole("ADMIN")` and pass
   `hasAuthority("ADMIN")`. Pick one convention for the codebase and enforce it in
   your `JwtAuthenticationConverter`.
3. **`permitAll()` is not "skip the check", it is "the check passes trivially".**
   The rest of the chain still runs. A `permitAll()` endpoint on a chain with
   `oauth2ResourceServer` will still **parse and validate a token if one is
   present** — and a *malformed* token on a `permitAll()` path still produces a
   401, because `BearerTokenAuthenticationFilter` ran before `AuthorizationFilter`
   and rejected it. "Public" does not mean "unparsed".

### `@PreAuthorize` and `@PostAuthorize`

Enable them once:

```java
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableMethodSecurity   // prePostEnabled is TRUE by default here (unlike the old annotation)
class MethodSecurityConfig { }
```

> **If you forget `@EnableMethodSecurity`, every `@PreAuthorize` in your codebase
> is an inert comment.** No warning, no startup failure, no log line. Your tests
> pass because your tests call the service directly and the rule was never going to
> run. This is Trap 5.

```java
@Service
public class OrderService {

    // Runs BEFORE the method body. Denial -> AccessDeniedException, body never runs.
    @PreAuthorize("hasRole('ADMIN')")
    public void cancelAnyOrder(OrderId id) { ... }

    // #orderId binds to the METHOD PARAMETER named orderId.
    // Requires -parameters at compile time, or @P("orderId") on the parameter.
    @PreAuthorize("hasRole('ADMIN') or @orderOwnership.isOwner(#orderId, authentication)")
    public OrderView get(OrderId orderId) { ... }

    // Runs AFTER the body, with the result bound to returnObject.
    // The method HAS ALREADY EXECUTED. Side effects have happened.
    @PostAuthorize("returnObject.customerId == authentication.name")
    public OrderView getUnchecked(OrderId orderId) { ... }
}
```

**The expression language.** These are SpEL strings evaluated at runtime against a
`MethodSecurityExpressionRoot`. That gives you `authentication`, `principal`,
`hasRole()`, `hasAuthority()`, `hasPermission()`, `#parameterName`, and
`@beanName.method(...)` to call any bean in the context.

Two consequences of "SpEL string":

- **It is not type-checked.** `hasRole('ADMN')` compiles, deploys, and denies
  everyone. Only a test catches it.
- **It is evaluated per invocation.** A complex expression that hits the database
  runs on every call. `@orderOwnership.isOwner(...)` below is a repository lookup;
  at Topic 65's load mix that is an extra query on 20% of traffic. Measure it.

**`@PreAuthorize` vs `@PostAuthorize` — a decision, not a preference.**

`@PostAuthorize` runs *after* the method. If the method was `@Transactional` and
wrote something, **the write has happened**; the `AccessDeniedException` thrown by
the interceptor then propagates out through the transaction proxy and — because
`AccessDeniedException` is a `RuntimeException` — rolls it back, *provided the
security interceptor sits outside the transaction interceptor*. That ordering is
an `@Order` question (Topic 41) and you should not want to depend on it.

> Use `@PostAuthorize` **only on read methods**. For anything that writes, the
> check must be `@PreAuthorize`.

**And the proxy caveat, in full, because it is Topic 40 again.** `@PreAuthorize` is
implemented by an AOP proxy. Therefore:

- A call from inside the same bean (`this.get(id)`) **bypasses the check entirely**.
- A `private` or `final` method cannot be advised.
- A method called from `@PostConstruct` is called before the proxy exists from that
  bean's own point of view (Topic 37).

Self-invocation on a `@Transactional` method loses atomicity, which you will
eventually notice. Self-invocation on a `@PreAuthorize` method loses the
authorization check, which you will not.

### `oauth2ResourceServer(jwt)` and the `JwtDecoder`

The minimum viable configuration is two properties and zero Java:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://idp.internal/realms/orderflow
```

From `issuer-uri`, Boot auto-configures a `NimbusJwtDecoder` that:

1. fetches `${issuer-uri}/.well-known/openid-configuration` at startup;
2. reads `jwks_uri` from it and fetches the key set;
3. caches the keys and refreshes them when it sees an unknown `kid`;
4. validates signature, `exp`, `nbf`, and `iss`.

`jwk-set-uri` is the alternative when you do not want the discovery round-trip at
startup — worth knowing because **`issuer-uri` makes your identity provider a
startup dependency**, and an IdP that is slow to respond becomes a slow
`orderflow` boot, which becomes a failing Kubernetes startup probe (Topic 121).

**Audience validation is not on by default and you must add it.** Without it, a
token minted by your IdP for a *different* service is accepted by `orderflow`,
because it is correctly signed by an issuer you trust. That is a real
privilege-escalation path across a microservice estate.

```java
@Bean
JwtDecoder jwtDecoder(OAuth2ResourceServerProperties props) {
    var issuer = props.getJwt().getIssuerUri();
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuer);
    OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer(issuer);
    OAuth2TokenValidator<Jwt> withAudience = new JwtClaimValidator<List<String>>(
            "aud", aud -> aud != null && aud.contains("orderflow-api"));
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(withIssuer, withAudience));
    return decoder;
}
```

I am confident about `JwtDecoders.fromIssuerLocation`, `JwtValidators`,
`DelegatingOAuth2TokenValidator` and `JwtClaimValidator` as concepts and as names
carried forward from 6.x. If `JwtClaimValidator`'s constructor arity differs on
your version, the fallback that certainly works is a two-line
`OAuth2TokenValidator<Jwt>` lambda-style implementation returning
`OAuth2TokenValidatorResult.success()` or `.failure(...)`.

**Claims to authorities.** Default behaviour: the `scope` (or `scp`) claim is split
on whitespace and each value becomes an authority prefixed `SCOPE_`. So
`"scope":"orders:read orders:write"` yields `SCOPE_orders:read` and
`SCOPE_orders:write`. Nothing produces `ROLE_` authorities by default — which is
exactly why `hasRole("CUSTOMER")` returns 403 against a token that plainly contains
`"roles":["customer"]`.

```java
@Bean
JwtAuthenticationConverter jwtAuthenticationConverter() {
    var converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        // Scopes keep their SCOPE_ prefix, for fine-grained API permissions.
        var scopes = new JwtGrantedAuthoritiesConverter().convert(jwt);

        // Realm roles become ROLE_*, for coarse role checks.
        Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
        List<String> roles = realmAccess == null
                ? List.of()
                : (List<String>) realmAccess.getOrDefault("roles", List.of());

        var authorities = new ArrayList<GrantedAuthority>(scopes);
        roles.stream()
             .map(r -> new SimpleGrantedAuthority("ROLE_" + r.toUpperCase(Locale.ROOT)))
             .forEach(authorities::add);
        return authorities;
    });
    // Make Authentication.getName() the stable user id, not whatever "sub" defaults to.
    converter.setPrincipalClaimName("sub");
    return converter;
}
```

That converter is the **entire** trust boundary between your IdP's claim schema and
your codebase's authority vocabulary. Write it once, test it directly, and never
scatter `jwt.getClaim(...)` calls through your services.

### `PasswordEncoder`

You still need this even as a resource server, because the admin console
authenticates locally and because internal service credentials have to be stored
somewhere.

```java
@Bean
PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

A `DelegatingPasswordEncoder` stores the algorithm **inside the hash**:

*Illustration of the stored format, not captured output.*

```
{bcrypt}$2a$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
 ^------^ ^--------------------- the bcrypt hash ---------------------^
 the id
```

Why this matters more than it looks: it makes **algorithm migration possible
without a password reset**. On a successful login you can re-encode with the new
default and store it; users with old hashes are upgraded as they log in. A bare
`new BCryptPasswordEncoder()` stores no prefix and gives you no migration path.

- `encode(raw)` produces a new hash **with a fresh random salt** — so encoding the
  same password twice gives two different strings. This surprises people writing
  their first test.
- `matches(raw, encoded)` is the only comparison. Never `equals`.
- The bcrypt work factor (default 10) is a **CPU cost per login by design**. At
  factor 12 a login is roughly four times the CPU of factor 10. That belongs in
  your capacity model, and it is a reason login endpoints get their own bulkhead
  (Topic 111).

### CSRF configuration, for the chain that needs it

```java
.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .ignoringRequestMatchers("/internal/webhooks/**")   // machine callers, header auth
)
```

`withHttpOnlyFalse()` makes the CSRF cookie readable by JavaScript, which a SPA
needs in order to echo it back in the `X-XSRF-TOKEN` header. The session cookie
stays `HttpOnly`; only the CSRF cookie is readable, and that is fine — the token is
not a credential, it is a proof that the request came from your own page.

Note: Security 6 introduced a `CsrfTokenRequestAttributeHandler` /
`XorCsrfTokenRequestAttributeHandler` distinction to mitigate BREACH, and the
defaults around deferred token loading changed. If your SPA sends the token and
still gets 403, that handler configuration is the first place to look — and it is
the second thing on my "check the current reference docs" list.

### `[BOOT 3.x DELTA]` — Security 6 vs Security 7 configuration idioms

| Concern | Security 6.x (Boot 3.x) | Security 7.0 (Boot 4.x) | Migration note |
|---|---|---|---|
| `WebSecurityConfigurerAdapter` | Removed already in 5.7 | Removed | If you see it, the code is ancient |
| `authorizeRequests()` | Deprecated | **Removed** | Mechanical rename to `authorizeHttpRequests` |
| `antMatchers()` / `mvcMatchers()` | Removed in 6.0 | Removed | Use `requestMatchers` |
| `.and()` chaining | Deprecated | **Removed** | Convert to the lambda DSL |
| `@EnableGlobalMethodSecurity` | Deprecated | **Removed** | `@EnableMethodSecurity`; note `prePostEnabled` now defaults `true` |
| Ant-style matcher class | `AntPathRequestMatcher` | Deprecated / replaced by a path-pattern matcher | **Flagged above — check the docs** |
| Saving the `SecurityContext` | 6.0 split `SecurityContextPersistenceFilter` into `SecurityContextHolderFilter`; **explicit save required** | Same | If you authenticate manually in a filter, you must call `securityContextRepository.saveContext(...)` yourself |
| Authorization internals | `AccessDecisionManager` deprecated, `AuthorizationManager` introduced | `AuthorizationManager` only | Custom voters need rewriting |

The practical migration rule: **a Security 5.x snippet from a blog will not compile
on 7.0, and a Security 6.x snippet mostly will.** Treat `.and()` in a snippet as a
date stamp.

---

## Example 1 — minimal

The smallest thing that demonstrates all four subjects. One chain, two roles, a
locally-signed JWT so you do not need an identity provider to run it, and a
`@PreAuthorize` for the rule the chain cannot express.

**Dependencies** (versions come from the Boot BOM — do not pin them yourself,
Topic 32):

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

**The chain.** For this minimal example the tokens are signed with a symmetric
secret so there is no IdP to run. Symmetric signing is fine for a lab and wrong for
production — see Example 2.

```java
package com.orderflow.lab;

import javax.crypto.spec.SecretKeySpec;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
class LabSecurityConfig {

    // 32 bytes minimum for HS256. In a lab only.
    private static final byte[] SECRET =
        "orderflow-lab-secret-key-32-bytes!!".getBytes(StandardCharsets.UTF_8);

    @Bean
    SecurityFilterChain chain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/products/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())   // justified: no cookie, no ambient credential
            .build();
    }

    @Bean
    JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder
            .withSecretKey(new SecretKeySpec(SECRET, "HmacSHA256"))
            .build();
    }

    @Bean
    JwtEncoder jwtEncoder() {
        return new NimbusJwtEncoder(
            new ImmutableSecret<>(new SecretKeySpec(SECRET, "HmacSHA256")));
    }

    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter() {
        var authorities = new JwtGrantedAuthoritiesConverter();
        authorities.setAuthorityPrefix("ROLE_");     // not SCOPE_
        authorities.setAuthoritiesClaimName("roles");// not scope
        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authorities);
        return converter;
    }
}
```

Note what those three lines in the converter do: they say "the claim called `roles`
holds my authorities, and prefix each with `ROLE_`". Without them,
`hasRole("ADMIN")` cannot possibly pass, no matter what the token says.

**A token minting endpoint, so you can run the proof yourself.** This is a lab
affordance and must never exist in a deployed service.

```java
@RestController
class LabTokenController {

    private final JwtEncoder encoder;
    LabTokenController(JwtEncoder encoder) { this.encoder = encoder; }

    // GET /lab/token?sub=alice&roles=CUSTOMER&ttl=300
    @GetMapping("/lab/token")
    String token(@RequestParam String sub,
                 @RequestParam List<String> roles,
                 @RequestParam(defaultValue = "300") long ttl) {
        var now = Instant.now();
        var claims = JwtClaimsSet.builder()
            .issuer("orderflow-lab")
            .subject(sub)
            .issuedAt(now)
            .expiresAt(now.plusSeconds(ttl))     // NEVER omit this. See Trap 3.
            .claim("roles", roles)
            .build();
        return encoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
    }
}
```

**The controller under test:**

```java
@RestController
class LabOrderController {

    @GetMapping("/products/{sku}")           // permitAll
    String product(@PathVariable String sku) { return "product " + sku; }

    @GetMapping("/orders/{id}")              // anyRequest().authenticated()
    String order(@PathVariable String id, Authentication auth) {
        return "order " + id + " read by " + auth.getName();
    }

    @DeleteMapping("/admin/orders/{id}")     // hasRole("ADMIN") at the chain
    String adminDelete(@PathVariable String id) { return "deleted " + id; }
}
```

**Run it.** Substitute the token you mint; do not expect the value below, and do
not paste a token from anywhere else.

```bash
# 1. No token on a public path.
curl -i http://localhost:8080/products/SKU-1001

# 2. No token on a protected path.
curl -i http://localhost:8080/orders/8213

# 3. Mint a customer token, then use it.
CUST=$(curl -s 'http://localhost:8080/lab/token?sub=alice&roles=CUSTOMER')
curl -i -H "Authorization: Bearer $CUST" http://localhost:8080/orders/8213
curl -i -H "Authorization: Bearer $CUST" -X DELETE http://localhost:8080/admin/orders/8213

# 4. Mint an admin token and repeat the delete.
ADMIN=$(curl -s 'http://localhost:8080/lab/token?sub=root&roles=ADMIN')
curl -i -H "Authorization: Bearer $ADMIN" -X DELETE http://localhost:8080/admin/orders/8213

# 5. Corrupt the token by one character.
curl -i -H "Authorization: Bearer ${CUST}X" http://localhost:8080/orders/8213

# 6. Mint a token that expires in 2 seconds, wait, then use it.
SHORT=$(curl -s 'http://localhost:8080/lab/token?sub=alice&roles=CUSTOMER&ttl=2')
sleep 4
curl -i -H "Authorization: Bearer $SHORT" http://localhost:8080/orders/8213
```

**WHAT TO LOOK FOR** — the status line and, on failures, the
`WWW-Authenticate` response header. That header is where the resource server tells
you *why*, and almost nobody reads it.

| What you see | What it means |
|---|---|
| `1` returns `200` | `permitAll()` matched before `anyRequest()`. Rule order is correct. |
| `2` returns `401` with `WWW-Authenticate: Bearer` | No credentials presented. `ExceptionTranslationFilter` invoked the entry point. |
| `3` first call `200`, second call `403` | Authenticated but not authorised. **401 vs 403 is the whole distinction**: 401 = I do not know who you are; 403 = I know, and no. |
| `4` returns `200` | `ROLE_ADMIN` present. Your converter's prefix and claim name are right. |
| `5` returns `401` with `error="invalid_token"` and a description | Signature verification failed **before** any authorization rule ran. |
| `6` returns `401` with `invalid_token` and an expiry-related description | `exp` is enforced by the decoder's default validators. If this returns `200`, you have no expiry validation and Trap 3 is live in your config. |
| `3`'s DELETE returns `401` rather than `403` | Your token was not accepted at all. Check the converter, not the rule. |

That last row is the diagnostic habit worth forming: **401 means fix the token or
the decoder; 403 means fix the authorities or the rule.** Never debug them
together.

---

## Example 2 — production scenario (on the project spine)

### The requirement

`orderflow` at the Topic 65 baseline: containerised, 100k products, 1M orders, 5M
order lines, driven by k6 in **open-loop arrival-rate** mode with the recorded
70/20/10 mix (catalogue read / order read / order placement). Postgres, Redis,
Kafka. HikariCP `maximum-pool-size: 16` from Topic 109's sizing work.

Three principal kinds, with genuinely different needs:

| Principal | How it authenticates | What it may do | Session? |
|---|---|---|---|
| **customer** | JWT from the customer IdP realm, obtained by the mobile app / SPA | Read the catalogue; read and place **their own** orders; read their own wallet | No — stateless |
| **admin** | Interactive login to the back-office console at `/console/**` | Read any order, cancel any order, adjust inventory | **Yes — cookie session, therefore CSRF applies** |
| **internal service** | JWT from the machine realm, client-credentials grant | Call `/api/internal/**` — reconciliation, the payment-callback ingest | No — stateless |

The design decision this table forces: **you need two `SecurityFilterChain` beans**,
because one is stateless with CSRF disabled and one is session-based with CSRF on.
A single chain cannot be both, and trying to make it be both is how the "disable
CSRF because the API needs it" bug gets shipped.

### Chain 1 — the stateless API

```java
package com.orderflow.security;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class OrderflowSecurityConfig {

    /**
     * Order 1: evaluated first. securityMatcher restricts it to /api/**,
     * so anything else falls through to the next chain.
     */
    @Bean
    @Order(1)
    SecurityFilterChain apiChain(HttpSecurity http,
                                 JwtAuthenticationConverter converter) throws Exception {
        return http
            .securityMatcher("/api/**")

            .authorizeHttpRequests(auth -> auth
                // --- public catalogue: 70% of the Topic 65 load mix ---
                .requestMatchers(HttpMethod.GET, "/api/products", "/api/products/**")
                    .permitAll()

                // --- machine-to-machine ---
                .requestMatchers("/api/internal/**")
                    .hasAuthority("SCOPE_orderflow.internal")

                // --- admin operations exposed over the API too ---
                .requestMatchers(HttpMethod.POST,   "/api/inventory/*/adjust").hasRole("ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/orders/**").hasRole("ADMIN")

                // --- customer operations ---
                .requestMatchers(HttpMethod.POST, "/api/orders").hasRole("CUSTOMER")
                .requestMatchers(HttpMethod.GET,  "/api/orders/**").hasAnyRole("CUSTOMER", "ADMIN")
                .requestMatchers("/api/wallet/**").hasRole("CUSTOMER")

                // --- everything else under /api requires SOMETHING ---
                .anyRequest().authenticated()
            )

            .oauth2ResourceServer(o -> o
                .jwt(jwt -> jwt.jwtAuthenticationConverter(converter))
                // Make failures speak the same ProblemDetail dialect as the rest of
                // the API. Without these two, filter-level rejections bypass your
                // @RestControllerAdvice entirely (Topic 46).
                .authenticationEntryPoint(new ProblemDetailAuthEntryPoint())
                .accessDeniedHandler(new ProblemDetailAccessDeniedHandler())
            )

            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())   // JUSTIFIED: no cookie is consulted on this chain
            .cors(Customizer.withDefaults())
            .headers(h -> h.frameOptions(f -> f.deny()))
            .build();
    }
```

Two things to notice.

**`.securityMatcher("/api/**")` plus `@Order(1)`.** The chain applies only to
`/api/**`. A request to `/console/login` does not match, so `FilterChainProxy` moves
to the next chain. If *no* chain matches, the request bypasses security entirely —
which is why the last chain must have no `securityMatcher` (see chain 2's sibling
below) or you must add a deny-all catch-all chain.

**The entry point and denied handler.** By default, a filter-chain rejection writes
a Spring Security error body, not your `ProblemDetail`. Your API contract from
Topic 46 says every error is RFC 9457. These two lines are what make the 401 and
403 conform:

```java
    static final class ProblemDetailAuthEntryPoint implements AuthenticationEntryPoint {
        @Override
        public void commence(HttpServletRequest req, HttpServletResponse res,
                             AuthenticationException ex) throws IOException {
            var pd = ProblemDetail.forStatus(HttpStatus.UNAUTHORIZED);
            pd.setTitle("Unauthorized");
            pd.setDetail("Authentication is required.");   // deliberately vague
            pd.setType(URI.create("https://orderflow.internal/problems/unauthorized"));
            write(res, HttpStatus.UNAUTHORIZED, pd);
        }
    }

    static final class ProblemDetailAccessDeniedHandler implements AccessDeniedHandler {
        @Override
        public void handle(HttpServletRequest req, HttpServletResponse res,
                           AccessDeniedException ex) throws IOException {
            var pd = ProblemDetail.forStatus(HttpStatus.FORBIDDEN);
            pd.setTitle("Forbidden");
            pd.setDetail("You do not have permission to perform this action.");
            pd.setType(URI.create("https://orderflow.internal/problems/forbidden"));
            write(res, HttpStatus.FORBIDDEN, pd);
        }
    }
```

"Deliberately vague" is a decision, not laziness. `"detail": "user alice lacks
ROLE_ADMIN on order 8213"` is a gift to an attacker enumerating your authority
model. Log the specific reason at `WARN` with the correlation ID (Topic 120); return
the generic one.

### Chain 2 — the admin console, with a session and therefore with CSRF

```java
    @Bean
    @Order(2)
    SecurityFilterChain consoleChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/console/**")

            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/console/login", "/console/css/**", "/console/js/**")
                    .permitAll()
                .anyRequest().hasRole("ADMIN")
            )

            .formLogin(form -> form
                .loginPage("/console/login")
                .defaultSuccessUrl("/console/orders", true)
                .failureUrl("/console/login?error")
            )
            .logout(logout -> logout
                .logoutUrl("/console/logout")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
            )

            // CSRF STAYS ON. This chain authenticates with an ambient cookie.
            .csrf(Customizer.withDefaults())

            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                // Session fixation: rotate the id at authentication. This is the
                // DEFAULT; written here to make the intent explicit in review.
                // I am not fully certain this nested-lambda shape is unchanged in
                // Security 7.0 -- check the reference docs' Session Management page.
                .sessionFixation(fixation -> fixation.changeSessionId())
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)   // new login evicts the old session
            )
            .build();
    }

    /**
     * Order 3, NO securityMatcher: the catch-all. Anything that reached here
     * matched neither /api/** nor /console/**. Deny it rather than let it through.
     */
    @Bean
    @Order(3)
    SecurityFilterChain fallbackChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/liveness",
                                 "/actuator/health/readiness").permitAll()
                .anyRequest().denyAll()
            )
            .csrf(csrf -> csrf.disable())
            .build();
    }
}
```

The `fallbackChain` is the part teams forget. Without it, a new endpoint mounted at
`/metrics-internal` — outside both matchers — is served with **no security filters
at all**. It is not "denied by default"; there is simply nothing there to deny it.
Topic 56's Trap 2 in its production form.

Also note the two probe paths are `permitAll` and everything else on Actuator is
`denyAll`. Liveness and readiness must answer without a token (Topic 121); `/env`,
`/heapdump` and `/threaddump` must not answer at all from outside the cluster.

### The ownership rule the chain cannot express

`GET /api/orders/{id}` is allowed for `CUSTOMER` and `ADMIN` at the chain. But a
customer must only read **their own** order. The chain sees `/api/orders/8213`; it
has no idea whose order that is.

```java
@Service
public class OrderQueryService {

    private final OrderRepository orders;

    @PreAuthorize("hasRole('ADMIN') or @orderOwnership.isOwner(#orderId, authentication)")
    @Transactional(readOnly = true)
    public OrderView get(OrderId orderId) {
        return orders.findViewById(orderId)
                     .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}

@Component("orderOwnership")
public class OrderOwnership {

    private final OrderRepository orders;
    OrderOwnership(OrderRepository orders) { this.orders = orders; }

    public boolean isOwner(OrderId orderId, Authentication authentication) {
        if (authentication == null || !authentication.isAuthenticated()) return false;
        // ONE indexed lookup. Not a full entity load -- this runs on every request
        // to this endpoint, which is 20% of the Topic 65 mix.
        return orders.findCustomerIdById(orderId)
                     .map(customerId -> customerId.equals(authentication.getName()))
                     .orElse(false);
    }
}
```

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // A projection, not the entity. Topic 47: never load an aggregate to read one column.
    @Query("select o.customerId from Order o where o.id = :id")
    Optional<String> findCustomerIdById(@Param("id") OrderId id);
}
```

**Three production constraints this design has to respect, stated as numbers:**

1. **It adds a query.** At 20% of the mix, this is one extra indexed lookup per
   order-read request. `orders(id)` is the primary key, so it is an index-only
   fetch of one column. Confirm with `EXPLAIN (ANALYZE, BUFFERS)` and confirm the
   p95 delta against the Topic 65 baseline before and after. Do not assume it is
   free and do not assume it is expensive.
2. **It must not hold a connection longer than needed.** `isOwner` runs *before*
   the `@Transactional` method body. It opens and closes its own short transaction
   through the repository. That is a second connection acquisition per request, and
   `maximum-pool-size` is 16. Topic 109's arithmetic applies: this is fine because
   the two acquisitions are **sequential, not nested** — the ownership check has
   returned before the service transaction begins. If you were to move the check
   *inside* the transaction you would be holding two connections and Topic 109's
   deadlock bound changes.
3. **Caching it is tempting and mostly wrong.** An order's owner never changes, so
   `@Cacheable` (Topic 110) looks safe. It is, on that one key — but you have just
   created an authorization decision cache, and an authorization decision cache is
   a thing you must be able to invalidate during an incident. If you cache it, put
   it on a short TTL and name the cache `authz-*` so it is obvious in the Redis key
   space what it is.

### The internal-service caller

```java
@RestController
@RequestMapping("/api/internal")
class InternalReconciliationController {

    // Belt and braces: the chain requires SCOPE_orderflow.internal, and so does this.
    // The chain rule protects the URL space; the method rule survives someone
    // remounting the controller at a different path.
    @PreAuthorize("hasAuthority('SCOPE_orderflow.internal')")
    @PostMapping("/reconcile-payments")
    ReconcileReport reconcile(@RequestBody @Valid ReconcileRequest request) { ... }
}
```

The internal token comes from a **client-credentials** grant: no user is involved,
the service authenticates with a client id and secret and receives a token with
`"scope":"orderflow.internal"` and no `roles` claim. Its `sub` is the client id.
Two consequences:

- `authentication.getName()` is a service identity, not a user id. Anything that
  assumes `getName()` is a customer id — including `OrderOwnership.isOwner` — will
  quietly return `false` for internal callers. That is why the expression is
  `hasRole('ADMIN') or isOwner(...)` and internal callers use a different endpoint
  entirely rather than sharing the customer one.
- Its token TTL should be **shorter** than a user's, not longer. A service can
  refresh silently; a human cannot. The common mistake is a 24-hour service token
  "because it is only internal", which is the longest-lived, highest-privilege
  credential in the estate.

### Load-testing implication (Topic 65, and why this changes your baseline)

Your k6 script now needs a token. Two rules:

- **Mint tokens in `setup()`, once, not per iteration.** A token request per
  iteration means you are load-testing your identity provider and your recorded
  `orderflow` numbers include its latency. That is coordinated omission of a
  different kind: you have added an unmodelled dependency to every measurement.
- **Do not let the tokens expire mid-run.** A 5-minute TTL with a 20-minute run
  produces a cliff at minute five where every request 401s. It looks like a
  catastrophic regression and it is a test-harness bug. Mint with a TTL longer than
  the run, or refresh in `setup` per stage.

And record the **cost of verification**. RS256 signature verification is an
asymmetric operation per request. Run the baseline with `permitAll()` on the
catalogue path versus `authenticated()` on the same path and compare p95. That
difference is the price of your auth design, and it is a number you should be able
to state.

---

## Wrong approach → exact symptom → root cause → fix

Five traps. Each one is written so you can reproduce the symptom deliberately.

### Trap 1 — `permitAll()` ordered after `anyRequest().authenticated()`

**Wrong approach**

```java
.authorizeHttpRequests(auth -> auth
    .anyRequest().authenticated()                                  // <-- catch-all, FIRST
    .requestMatchers("/actuator/health/**").permitAll()             // never reached
    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()// never reached
)
```

**Exact symptom**

```bash
curl -i http://localhost:8080/actuator/health/readiness
curl -i http://localhost:8080/api/products/SKU-1001
```

Both return `401` with `WWW-Authenticate: Bearer`. Under Kubernetes, the readiness
probe fails, the pod is removed from the Service endpoints, and — if you also
pointed liveness at a secured path — the pod restarts in a loop (Topic 121's drill,
arriving from an unexpected direction). Under the Topic 65 load, 70% of the mix
(the catalogue read) fails with 401 and your error rate is 70%.

Some versions of the DSL detect the unreachable rule and fail at startup with a
message about a matcher that cannot be reached. **Do not rely on that.** It catches
the literal `anyRequest()`-before-others case; it does not catch
`requestMatchers("/api/**").authenticated()` placed before
`requestMatchers("/api/products/**").permitAll()`, which is the same bug wearing a
disguise and starts up perfectly happily.

**Root cause**

`authorizeHttpRequests` builds an **ordered list** of (matcher, rule) pairs.
`AuthorizationFilter` walks that list and stops at the **first matcher that
matches**. It does not look for the most specific match. `anyRequest()` matches
everything, so nothing after it is ever consulted.

This is the same first-match-wins semantics as an nginx `location` block or an
Express `app.use` ordering, and the opposite of a router that sorts by specificity.
Your instinct from Nest — where the decorator sits on the handler and specificity is
structural — is exactly wrong here.

**The dangerous mirror image.** Reverse the mistake and the damage inverts:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/**").permitAll()          // "let the API through for now"
    .requestMatchers("/api/admin/**").hasRole("ADMIN")// never reached
)
```

Now `/api/admin/**` is **public** and nothing fails. No 401, no 403, no log line,
no test failure unless you wrote a test that asserts a 403. This ships.

**Fix**

Order from most specific to least specific, and put `anyRequest()` last, always.

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/actuator/health/liveness",
                     "/actuator/health/readiness").permitAll()
    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/internal/**").hasAuthority("SCOPE_orderflow.internal")
    .anyRequest().authenticated()                     // LAST. Always last.
)
```

**And prove it**, because ordering is not something to hold in your head:

```java
@WebMvcTest
@Import(OrderflowSecurityConfig.class)
class AuthorizationRuleTest {

    @Autowired MockMvc mvc;

    @Test void catalogue_is_public() throws Exception {
        mvc.perform(get("/api/products/SKU-1001")).andExpect(status().isOk());
    }

    @Test void admin_path_is_not_public() throws Exception {
        mvc.perform(delete("/api/orders/8213")).andExpect(status().isUnauthorized());
    }

    @Test @WithMockUser(roles = "CUSTOMER")
    void customer_cannot_delete_orders() throws Exception {
        mvc.perform(delete("/api/orders/8213")).andExpect(status().isForbidden());
    }

    @Test @WithMockUser(roles = "ADMIN")
    void admin_can_delete_orders() throws Exception {
        mvc.perform(delete("/api/orders/8213")).andExpect(status().isOk());
    }
}
```

That test class is the artefact. A rule you have not asserted is a rule that will be
reordered by the next person who adds a matcher.

---

### Trap 2 — disabling CSRF reflexively on a cookie-session application

**Wrong approach**

```java
// One chain for everything, including the /console/** pages that use a session.
@Bean
SecurityFilterChain everything(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable())     // "we're an API, CSRF is for forms"
        .build();
}
```

The reasoning is always the same and always half-true: "we are a REST API, we do
not use forms, CSRF does not apply". The half that is true is about the *API*. The
half that is missing is that `formLogin()` creates a **session cookie**, and now
every `/console/**` endpoint is authenticated by an ambient credential with no CSRF
defence.

**Exact symptom**

There is no symptom in your application. That is the point. You reproduce it
deliberately:

1. Log into the console in a browser so `JSESSIONID` is set for
   `orderflow.internal`.
2. Save this to a file and open it from a **different** origin — `file://` or any
   other host. Do this against a local instance only.

```html
<!-- csrf-poc.html : open from a DIFFERENT origin than the app -->
<body onload="document.forms[0].submit()">
  <form action="http://localhost:8080/console/orders/8213/cancel" method="POST">
    <input type="hidden" name="reason" value="csrf-proof">
  </form>
</body>
```

| What you see | What it means |
|---|---|
| The order is cancelled; app logs show the admin's username as the actor | **CSRF is live.** The browser attached `JSESSIONID` to a cross-origin form POST and your server trusted it. |
| `403` with a body mentioning a missing or invalid CSRF token | CSRF protection is on and working. |
| `401` | The session was not attached — check that you are actually logged in in that browser and that the cookie's `SameSite` is not already blocking it. |

Note the `SameSite` interaction, because it changes what you will observe: modern
browsers default cookies to `SameSite=Lax`, which **already blocks** a cross-site
POST from attaching the cookie. That is a real mitigation and it is also why this
proof can appear to "pass" on a mis-set-up test. `SameSite=Lax` is defence in depth,
not a replacement: it does not protect against a same-site subdomain, it does not
protect non-browser clients, and it is a browser behaviour you do not control. If
you set `SameSite=None` for any reason — a common consequence of an embedded
iframe or a third-party checkout widget — the protection evaporates and only the
CSRF token remains.

**Root cause**

The rule, stated exactly:

> **CSRF protection is required exactly when the server authenticates a
> state-changing request using a credential the browser attaches automatically.**

Cookies: yes. HTTP Basic: yes (browsers re-send it). TLS client certs: yes.
`Authorization: Bearer` read from `localStorage` and set by your own JS: **no** —
the attacker's page cannot make the victim's browser set that header on a
cross-origin request, and it cannot read your origin's `localStorage`.

The advice "disable CSRF for APIs" is correct **only for the header-authenticated
chain**, and people copy it onto the chain that has `formLogin()`.

**Fix**

Split the chains — exactly as Example 2 does. One stateless chain with CSRF off
and one session chain with CSRF on. Then write down the justification next to the
`disable()` call so the next reviewer sees the precondition:

```java
// CSRF disabled: this chain is STATELESS and authenticates only from the
// Authorization header. No ambient credential is consulted, so there is no
// CSRF surface. If a cookie is ever introduced to this chain, this line
// becomes a vulnerability.
.csrf(csrf -> csrf.disable())
```

Additional hardening on the session chain, all cheap:

```yaml
server:
  servlet:
    session:
      cookie:
        http-only: true
        secure: true
        same-site: strict
      tracking-modes: cookie   # never URL: URL rewriting IS session fixation
```

`tracking-modes: cookie` deserves a sentence. If the container is allowed to put
the session id in the URL (`;jsessionid=...`), an attacker can simply email a link
containing a session id they control, which is the classic fixation vector, and
session ids leak through `Referer` headers and access logs.

---

### Trap 3 — a JWT with no expiry, and the discovery that stateless means unrevokable

**Wrong approach**

Two versions, and the second is the one that ships.

```java
// Version A -- the lab shortcut that escapes.
var claims = JwtClaimsSet.builder()
    .subject(userId)
    .claim("roles", roles)
    .build();                       // no .expiresAt(...) -- "it was annoying in dev"

// Version B -- an expiry that is a rounding error away from forever.
    .expiresAt(Instant.now().plus(30, ChronoUnit.DAYS))
```

**Exact symptom**

For version A:

```bash
TOKEN=$(curl -s 'http://localhost:8080/lab/token?sub=alice&roles=CUSTOMER')
# Decode the payload locally -- this reads YOUR token, it does not invent one.
echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
```

| What you see | What it means |
|---|---|
| The decoded JSON has **no `exp` field** | The token never expires. Every copy of it ever logged, cached in a proxy, or stored in a crash report is a permanent credential. |
| `exp` present, and the request 401s after that instant | Expiry is being enforced. This is the correct state. |
| `exp` present but the request still succeeds well past it | Your `JwtDecoder` has a non-default validator that dropped `JwtTimestampValidator`, or your clock skew allowance is enormous. Read your `setJwtValidator` call. |

For version B the symptom is the incident, and it arrives later: a customer reports
account takeover, you disable the account in Postgres, and **the attacker keeps
placing orders**. Every request they make is authenticated. Your database says the
user is disabled. Your API never asks the database.

**Root cause**

This is not a bug in Spring Security. It is the defining property of the design.

> **A JWT is a bearer credential that your server validates by arithmetic, not by
> lookup. Validation touches no shared state. Therefore no change to shared state
> can invalidate it.**

Disabling the user, deleting them, changing their password, revoking their roles —
none of it is consulted, because consulting it is precisely the round trip you
adopted JWTs to avoid. You did not lose revocation by accident; you traded it away.

**Fix — four options, none free. Choose deliberately.**

| Option | Mechanism | What it costs | Revocation latency |
|---|---|---|---|
| **1. Short access token + refresh rotation** | Access token TTL 5–15 min; a long-lived, **stored** refresh token exchanged for new access tokens. Revoke by deleting the refresh token. | The refresh token is state — so you are not stateless, you have just moved the state to a lower-traffic path. | Up to one access-token TTL |
| **2. Denylist by `jti`** | Every token carries a unique `jti`. On revocation, write `jti` to Redis with a TTL equal to the token's remaining life. Check on every request. | **A Redis lookup on every authenticated request.** You have reintroduced the round trip. At the Topic 65 mix that is a new hard dependency in the hot path — if Redis is down, you either fail closed (outage) or fail open (revocation silently stops working). | Immediate |
| **3. Token version / epoch claim** | Store `token_epoch` per user. Put it in the token. On revocation, bump the user's epoch. Compare on each request. | Same round trip as option 2 but cacheable per user, and it revokes **all** of a user's tokens at once rather than one. | Immediate, but needs a per-user lookup |
| **4. Opaque tokens + introspection** | Do not use JWTs. Issue a random string; the resource server calls the IdP's introspection endpoint. | A network call per request unless cached — and if you cache it you are back to option 2's latency window. Spring supports this via `oauth2ResourceServer(o -> o.opaqueToken(...))`. | Immediate |

**Refresh rotation, stated precisely, because "we use refresh tokens" is not a
design.** Rotation means: when a refresh token is used, it is **invalidated** and a
new one issued. If an old refresh token is presented again, that is either a replay
or a race — and the correct response is to **revoke the entire token family** for
that user, because you cannot tell the attacker from the victim. Without that
detection step, a stolen refresh token is a permanent credential and rotation
bought you nothing.

**What to actually do for `orderflow`:** option 1 as the default (15-minute access
tokens, rotating refresh tokens held by the IdP), plus option 3 as the break-glass
lever, wired but with the per-user epoch cached in Redis with a 30-second TTL so the
hot path cost is bounded. Then write down, in the runbook, the sentence "revoking a
user takes effect within 15 minutes, or immediately if you bump the epoch". That
sentence is the deliverable. Topic 133's postmortems will thank you.

Also: **log the `jti`, never the token.** A token in a log file is a live
credential, and logs go to a system with a different access-control model than your
database.

---

### Trap 4 — the `SecurityContext` is lost across `@Async` and the authorization check sees anonymous

**Wrong approach**

```java
@Service
public class OrderService {

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        var order = orders.save(Order.from(cmd));
        auditService.recordAsync(order.id());        // fire and forget
        return order.id();
    }
}

@Service
public class AuditService {

    @Async
    public void recordAsync(OrderId id) {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        // Intent: record WHO placed the order.
        audit.save(new AuditEntry(id, auth.getName(), Instant.now()));
    }
}
```

**Exact symptom**

Two possible symptoms, and which one you get depends on a detail that has nothing
to do with your intent.

```bash
# Place an order as a real customer, then read the audit table.
curl -s -X POST http://localhost:8080/api/orders \
     -H "Authorization: Bearer $CUST" \
     -H 'Content-Type: application/json' \
     -d '{"productSku":"SKU-1001","quantity":1}'

psql -c "select order_id, actor, created_at from audit_entry order by created_at desc limit 5;"
```

| What you see | What it means |
|---|---|
| A `NullPointerException` in the async thread's log, order still created, no audit row | `getAuthentication()` returned `null`. The exception is on the executor thread, so the HTTP response was already `201` — **the caller never learns the audit failed**. |
| An audit row with `actor = 'anonymousUser'` | An anonymous authentication was present instead of the real one. Worse than the NPE: it is silent and it is wrong data. |
| Correct `actor` | Either you propagated the context, or you are on a version/configuration where the executor is already decorated. Verify which, do not assume. |

Now the dangerous variant, where the missing context becomes an **authorization**
bug rather than a logging bug:

```java
@Async
public void recordAsync(OrderId id) {
    if (isAdmin()) {
        audit.saveDetailed(id);      // includes PII
    } else {
        audit.saveMinimal(id);
    }
}

private boolean isAdmin() {
    var auth = SecurityContextHolder.getContext().getAuthentication();
    return auth != null && auth.getAuthorities().stream()
             .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
}
```

`auth` is `null`, `isAdmin()` returns `false`, no exception is thrown, and the code
takes the non-admin branch. **A defensive null check turned a crash into a silently
wrong authorization decision.** This is the failure mode to fear.

**Root cause**

`SecurityContextHolder` defaults to `ThreadLocalSecurityContextHolderStrategy`.
`@Async` executes the method on a thread from a `TaskExecutor`. Different thread,
different `ThreadLocal` map, no value.

Two aggravating details:

- `getContext()` **never returns `null`.** It lazily creates an empty
  `SecurityContext` if there is none. So the "is the context set" check that would
  catch this does not exist naturally; you have to check
  `getAuthentication() == null`, which is exactly the check that produces the
  silent-wrong-branch variant above.
- `SecurityContextHolderFilter` **clears** the context in a `finally` block at the
  end of the request. If your async task starts late enough it would find a cleared
  context even on a shared thread — so "it worked in my test" is not evidence.

**Fix — in preference order.**

*Fix 1 (best): do not need the context on the other thread.* Pass the actor as a
parameter. The identity is an input to the operation, not ambient state.

```java
@Transactional
public OrderId place(PlaceOrderCommand cmd, String actorId) {
    var order = orders.save(Order.from(cmd));
    auditService.recordAsync(order.id(), actorId);   // explicit
    return order.id();
}

@Async
public void recordAsync(OrderId id, String actorId) {
    audit.save(new AuditEntry(id, actorId, Instant.now()));
}
```

This is the fix. Everything below is for when you cannot restructure — a library
that reads the holder, or a deep call stack you do not own.

*Fix 2: decorate the executor.* Spring ships
`DelegatingSecurityContextAsyncTaskExecutor`, which captures the context at submit
time and installs it on the worker thread for the duration of the task.

```java
@Configuration
@EnableAsync
class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(500);            // BOUNDED. Topic 90.
        executor.setThreadNamePrefix("of-async-");
        executor.initialize();
        return new DelegatingSecurityContextAsyncTaskExecutor(executor);
    }
}
```

*Fix 3 (do not do this): `MODE_INHERITABLETHREADLOCAL`.*

```java
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
```

An `InheritableThreadLocal` is copied when a thread is **created**, not when a task
is **submitted**. With a pool, threads are created once and reused for thousands of
unrelated requests. So thread `of-async-3` inherits whoever happened to trigger its
creation and keeps that identity forever. You have converted "no identity" into
"someone else's identity", which is strictly worse. It works only with a
thread-per-task model and no pooling, which is not a thing you have.

*Verify the fix, do not assume it:*

```java
@Async
public void recordAsync(OrderId id) {
    log.info("thread={} auth={}",
        Thread.currentThread().getName(),
        SecurityContextHolder.getContext().getAuthentication());
    ...
}
```

**WHAT TO LOOK FOR:** the thread name must be the async prefix (proving you really
crossed a boundary) *and* the authentication must be non-null with the right
principal. Both, in the same line. If the thread name is `http-nio-8080-exec-*`,
`@Async` did not apply at all — which is Topic 40 again, and is Trap 5.

The same fix shape is needed for the MDC correlation id (Topic 120) and the OTel
trace context (Topic 119). Decorate the executor **once**, with all three, or you
will do this three times.

---

### Trap 5 — `@PreAuthorize` that never runs

**Wrong approach** — two independent causes, identical symptom.

```java
// Cause A: @EnableMethodSecurity is missing from the whole application.
@Service
public class OrderService {
    @PreAuthorize("hasRole('ADMIN')")
    public void cancelAnyOrder(OrderId id) { ... }
}
```

```java
// Cause B: self-invocation. The annotation is on a method called from within
// the same bean, so the call never goes through the proxy. Topic 40.
@Service
public class OrderService {

    public void bulkCancel(List<OrderId> ids) {
        ids.forEach(this::cancelAnyOrder);   // <-- 'this', not the proxy
    }

    @PreAuthorize("hasRole('ADMIN')")
    public void cancelAnyOrder(OrderId id) { ... }
}
```

**Exact symptom**

A `CUSTOMER` token calls the endpoint that reaches `cancelAnyOrder`, and gets
`200 OK`. The order is cancelled. Nothing is logged, nothing is thrown, and the
happy-path test passes because it uses an admin.

Distinguishing A from B takes one command each.

```bash
# Cause A test: does the annotation do ANYTHING?
curl -i -H "Authorization: Bearer $CUST" -X POST http://localhost:8080/api/orders/8213/cancel
```

| What you see | What it means |
|---|---|
| `403` with `access_denied` | Method security is enabled and the annotation fired. Cause A is excluded. |
| `200` and the order is cancelled | Either method security is off (A) or the call path bypassed the proxy (B). Go to the next check. |

```java
// Cause B test: print the runtime class of the injected bean.
@Component
class ProxyProbe {
    ProxyProbe(OrderService orderService) {
        System.out.println("OrderService is: " + orderService.getClass().getName());
    }
}
```

| What you see | What it means |
|---|---|
| `com.orderflow.orders.OrderService$$SpringCGLIB$$0` | A proxy exists. If the check still does not fire, the call is self-invocation (B) — the proxy is bypassed from inside. |
| `com.orderflow.orders.OrderService` (plain, no `$$`) | **No proxy was created at all.** Method security is not enabled (A), or nothing else advises this bean either. |

And the definitive instrument for both:

```yaml
logging:
  level:
    org.springframework.security: DEBUG
```

**WHAT TO LOOK FOR:** with `@EnableMethodSecurity` active, an authorized method
invocation produces a log line naming the `MethodInvocation` and the authorization
decision. **No such line for `cancelAnyOrder` means the interceptor never ran.**
Absence of the line is the evidence.

**Root cause**

- **A:** `@PreAuthorize` is not an active annotation on its own. It is inert
  metadata until `@EnableMethodSecurity` registers the
  `AuthorizationManagerBeforeMethodInterceptor` and the advisor that matches it.
  Spring does not warn about annotations it has no processor for — this is the same
  class of silent no-op as `@Transactional` without a transaction manager.
- **B:** the interceptor lives on the proxy. `this.cancelAnyOrder(...)` is a plain
  virtual call on the target object. Topic 40, exactly and completely.

**Fix**

For A, add the annotation, and add a test that proves the *denial* path — not just
the allow path:

```java
@Configuration
@EnableMethodSecurity
class MethodSecurityConfig { }
```

```java
@SpringBootTest
class MethodSecurityIsEnabledTest {

    @Autowired OrderService orderService;

    @Test @WithMockUser(roles = "CUSTOMER")
    void customer_cannot_cancel_any_order() {
        assertThatThrownBy(() -> orderService.cancelAnyOrder(new OrderId(8213L)))
            .isInstanceOf(AccessDeniedException.class);
    }
}
```

That test fails loudly the day someone removes `@EnableMethodSecurity` during a
config refactor. A test that only checks the admin can cancel would pass.

For B, apply Topic 40's fixes in the same preference order: move the guarded method
to a separate collaborator bean (best — it makes the boundary explicit), or inject
an `ObjectProvider<OrderService>` and call through it, or self-inject. Do **not**
put `@PreAuthorize` on `bulkCancel` and consider it handled — that changes which
rule is checked, and if `bulkCancel` and `cancelAnyOrder` ever need different rules
you have hidden the problem rather than fixed it.

**The general principle worth taking away from this trap:** defence in depth here
means the chain rule *and* the method rule, because they fail differently. The
chain rule survives a proxy problem. The method rule survives someone remounting
the controller at a new path. Neither is redundant.

---

## Hands-on proof

Four proofs. None of them requires you to trust anything in this document.

### Proof 1 — see which chain and which rule handled a request

```yaml
logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.web.FilterChainProxy: DEBUG
```

Then issue one request and read the log for that request only.

```bash
curl -i -H "Authorization: Bearer $CUST" http://localhost:8080/api/orders/8213
```

**WHAT TO LOOK FOR** — three things, in order:

1. A `FilterChainProxy` line naming **which `SecurityFilterChain` was selected** and
   the request URI. This settles `securityMatcher` questions instantly.
2. The **sequence of filters invoked**, one line each. `BearerTokenAuthenticationFilter`
   must appear before `AuthorizationFilter`.
3. An authorization line naming the decision.

| What you see | What it means |
|---|---|
| The selected chain is not the one you expected | Your `securityMatcher` or `@Order` is wrong. Fix chain selection before looking at rules. |
| `BearerTokenAuthenticationFilter` absent | `oauth2ResourceServer` is not configured on the chain that was selected. |
| `AuthorizationFilter` never reached | An earlier filter rejected the request. Read the filter immediately above it in the log. |
| `Anonymous` authentication reaching `AuthorizationFilter` on a request that had a token | The token was present but produced no authentication — decoder or converter problem, not a rule problem. |

**Turn this off before load testing.** DEBUG logging on the security package is a
per-request allocation and I/O cost that will move your Topic 65 numbers.

### Proof 2 — the `curl` matrix

This is the artefact to keep. Run it after every change to the security config.

```bash
BASE=http://localhost:8080
CUST=$(...)   # a customer token
ADMIN=$(...)  # an admin token
SVC=$(...)    # an internal-service token

probe() { printf '%-8s %-38s -> %s\n' "$1" "$3" \
  "$(curl -s -o /dev/null -w '%{http_code}' -X "$1" ${2:+-H "Authorization: Bearer $2"} "$BASE$3")"; }

probe GET    ""      /api/products/SKU-1001
probe GET    ""      /api/orders/8213
probe GET    "$CUST" /api/orders/8213
probe DELETE "$CUST" /api/orders/8213
probe DELETE "$ADMIN" /api/orders/8213
probe POST   "$SVC"  /api/internal/reconcile-payments
probe POST   "$CUST" /api/internal/reconcile-payments
probe GET    ""      /actuator/health/readiness
probe GET    ""      /actuator/env
```

**WHAT TO LOOK FOR** — write down the expected column *before* you run it, then
diff. Any cell you did not predict is a rule you do not understand.

| Row | Expected | If you see something else |
|---|---|---|
| public catalogue, no token | `200` | Trap 1: `permitAll()` is unreachable |
| protected order, no token | `401` | `200` means the path matches no chain, or matches a `permitAll()` you did not intend |
| customer reads own order | `200` | `403` usually means the `ROLE_` prefix is missing from your converter |
| customer deletes order | `403` | `200` is Trap 1's mirror image and is a live vulnerability |
| service token on internal path | `200` | `403` means the scope claim is not mapping to `SCOPE_orderflow.internal` |
| customer token on internal path | `403` | `200` means the internal rule is shadowed |
| readiness, no token | `200` | `401` restarts your pods (Topic 121) |
| `/actuator/env`, no token | `401` or `403` | `200` is an information disclosure — that endpoint prints your configuration |

### Proof 3 — watch the `ThreadLocal` fail to propagate

Add a temporary endpoint and read the two log lines it produces.

```java
@RestController
class ContextProbeController {

    private final ProbeAsync probeAsync;

    @GetMapping("/api/probe/context")
    String probe() {
        log.info("SYNC  thread={} auth={}", Thread.currentThread().getName(),
                 SecurityContextHolder.getContext().getAuthentication());
        probeAsync.run();
        return "check the logs";
    }
}

@Component
class ProbeAsync {
    @Async
    void run() {
        log.info("ASYNC thread={} auth={}", Thread.currentThread().getName(),
                 SecurityContextHolder.getContext().getAuthentication());
    }
}
```

*Illustration of the log shape, not captured output:*

```
... SYNC  thread=http-nio-8080-exec-<n> auth=JwtAuthenticationToken [Principal=..., Authorities=[ROLE_CUSTOMER]]
... ASYNC thread=of-async-<n>           auth=null
```

| What you see | What it means |
|---|---|
| `ASYNC` thread name differs and `auth=null` | The `ThreadLocal` did not cross. This is the default and expected state. |
| `ASYNC` thread name is the same `http-nio-*` thread | `@Async` did not apply — self-invocation or missing `@EnableAsync`. Fix that before concluding anything. |
| `ASYNC` `auth` non-null after you install `DelegatingSecurityContextAsyncTaskExecutor` | The fix works. Keep this probe as a test. |

### Proof 4 — decode your own token and read its lifetime

```bash
echo "$CUST" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq '{iss,sub,aud,exp,iat,jti,scope}'
```

**WHAT TO LOOK FOR:** `exp` present; `exp - iat` equal to the TTL you intended;
`aud` naming `orderflow-api`; `jti` present so revocation and log correlation are
possible. `exp - iat` measured in days is Trap 3.

---

## Practice exercises

### Easy — predict, then verify

Given this rule list, predict the status code for each of the six requests below
**before running anything**. Then build it and run Proof 2's matrix.

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/**").authenticated()
    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .anyRequest().permitAll()
)
```

1. `GET /api/products/SKU-1001` with no token
2. `GET /api/orders/1` with a customer token
3. `DELETE /api/admin/orders/1` with a customer token
4. `GET /internal/debug` with no token
5. `GET /api/products/SKU-1001` with a *malformed* token
6. `GET /actuator/env` with no token

Then write the corrected rule list and re-predict. Success criterion: your
predictions match the observed codes on the first run, and you can say in one
sentence why rules 2 and 3 were unreachable.

### Medium — the ownership rule (combines Topics 40, 44, 46, 47, 54, 56)

Implement `GET /api/orders/{id}` so that a customer may read only their own order
and an admin may read any, with all of the following true:

1. The chain rule allows `CUSTOMER` and `ADMIN`; the ownership rule is
   `@PreAuthorize` on the service method.
2. A customer reading another customer's order gets `403` with a `ProblemDetail`
   body matching your Topic 46 contract — **not** a Spring Security default body.
3. A customer reading a **non-existent** order gets `404`, and a customer reading
   *someone else's* order gets `403` — then argue in three sentences whether that
   distinction leaks the existence of orders, and change it if you decide it does.
4. Add a `bulkGet` method on the same service that calls `get(id)` in a loop.
   Demonstrate that the `@PreAuthorize` does **not** fire (Topic 40), then fix it,
   then prove the fix with a test that a customer cannot bulk-read others' orders.
5. The ownership check must issue exactly **one** query. Prove it with a Hibernate
   `Statistics` query counter in the test (Topic 50's technique).

### Hard — production simulation under load

Run `orderflow` at the Topic 65 baseline with the full three-chain configuration
and produce a written result for each item.

1. **Baseline the auth cost.** Run the recorded 70/20/10 mix twice: once with the
   catalogue path `permitAll()`, once with it `authenticated()` and every VU
   sending a token. Report the p50/p95/p99 delta and the CPU delta. State the cost
   of signature verification per request in your own numbers.
2. **Break rule ordering under load.** Move `anyRequest().authenticated()` to the
   top. Run the mix. Report the error rate, which scenarios failed, and how long it
   took you to identify the cause from logs alone with
   `org.springframework.security=DEBUG` **off** (it will be off in production).
   Then turn it on and time it again. The difference is the value of that log
   level, quantified.
3. **The expiry cliff.** Mint tokens with a 3-minute TTL and run a 10-minute test.
   Plot error rate over time. Describe exactly what an on-call engineer would see
   in the RED dashboard (Topic 118) and what they would wrongly conclude.
4. **Revocation drill.** With option 3 from Trap 3 wired (a per-user epoch in
   Redis, 30-second cache), measure: the added p99 on the authenticated path, the
   Redis QPS added at baseline load, and the observed time from "bump the epoch" to
   "the user's requests 401". Then kill Redis mid-run and report which way your
   implementation failed — open or closed — and whether that is what you intended.
5. **Write the runbook entry.** Six lines maximum: how to revoke one user, how to
   revoke everyone, what the propagation delay is, and what breaks if Redis is
   down. This is the Phase 12 artefact shape (Topic 131) in miniature.

---

## Interview questions

### Q1 — "Why doesn't a stateless JWT API need CSRF protection?"

**MID-LEVEL:** "Because CSRF is about forms and sessions, and we're a REST API with
tokens, so it doesn't apply."

**SENIOR:** "CSRF exploits *ambient* credentials — anything the browser attaches to
a cross-origin request without the page doing it. Cookies, HTTP Basic, TLS client
certs. If my API authenticates only from an `Authorization` header that my own
JavaScript sets, an attacker's page cannot make the victim's browser send it: it
can't read my origin's storage, and a custom header on a cross-origin request needs
a CORS preflight my server won't approve. So there's no CSRF surface and disabling
the filter is correct.

The precondition is the whole answer, though. The moment any chain in the app has a
session cookie — a form login, an admin console, an OAuth2 login flow — CSRF is
back on that chain. So I split the chains: stateless with CSRF off, session-based
with CSRF on. And I'd note `SameSite=Lax` is real defence in depth but not a
substitute, because I don't control the browser and `SameSite=None` gets set for
embedding reasons that have nothing to do with security."

**What separates them:** the mid-level answer states a conclusion; the senior answer
states the *precondition under which the conclusion holds*, and then observes that
the precondition is per-chain rather than per-application. That is the difference
between repeating advice and being able to apply it to a codebase you have not seen.

**Follow-up:** *"Your app is one chain with both an API and a form login. What
exactly is broken?"* — Every state-changing endpoint reachable with the session
cookie is CSRF-able, including the API ones, because the browser attaches the
cookie regardless of which authentication mechanism you *intended* for that path.

### Q2 — "How do you revoke a JWT?"

**MID-LEVEL:** "You add it to a blacklist in Redis and check on every request."

**SENIOR:** "You don't, in the general case — that's the trade you made. A JWT is
validated by arithmetic, so nothing in shared state can invalidate it. There are
four options and all of them reintroduce state somewhere.

Short access tokens with rotating refresh tokens is the default: revocation
latency is one access-token TTL, and the state lives on the low-traffic refresh
path rather than the hot path. A `jti` denylist gives immediate revocation and
costs a Redis lookup on every authenticated request — which is the round trip JWTs
were adopted to avoid, and creates a new hard dependency: you have to decide now
whether a Redis outage fails open or closed. A per-user epoch claim is usually the
better version of that, because it revokes all of a user's tokens at once and is
cacheable per user rather than per token. Or you use opaque tokens with
introspection and stop pretending.

For `orderflow` I'd run 15-minute access tokens with rotating refresh, plus an
epoch as break-glass, and I'd write the revocation latency in the runbook so nobody
discovers it during an incident."

**What separates them:** the mid-level answer names one mechanism. The senior answer
names the property that makes revocation hard, enumerates the options with their
costs, and — critically — mentions **refresh rotation with family revocation**,
which is the part people skip and which is what makes a stolen refresh token
detectable.

**Follow-up:** *"Your denylist is in Redis and Redis goes down. What happens?"* —
The honest answer is "whatever I coded, and I need to have decided". Fail closed
means a Redis outage is a full authentication outage. Fail open means revocation
silently stops working during exactly the incident where you need it. Most teams
fail open and do not know it.

### Q3 — "We put the role check in the controller with an `if`. What's wrong with that?"

**MID-LEVEL:** "It's not idiomatic; you should use `@PreAuthorize`."

**SENIOR:** "Three concrete problems. First, it's easy to forget: security by
convention on every new endpoint is security that decays. A chain rule is
enumerable — I can read the whole policy in one file and write a test matrix
against it.

Second, it runs late. The filter chain rejects before `DispatcherServlet` resolves a
handler, before argument resolution, before `@Valid`, before any transaction opens.
A controller `if` runs after all of that, so a request that should have been
rejected has already consumed deserialization and possibly a connection.

Third, and this is the real one: an `if` in a controller can't express ownership any
better than the chain can — you still need the entity — but it *looks* like it can,
so people write `if (order.customerId().equals(principal))` after loading the
order, and then someone adds a second code path that loads the order without that
check. The rule belongs on the service method as `@PreAuthorize`, where every caller
of that method inherits it.

I'd use both layers: coarse URL rules at the chain, ownership rules at the method
boundary. They fail differently, so neither is redundant."

**What separates them:** the mid-level answer is about style. The senior answer is
about *where in the request lifecycle the decision happens* and *how the rule
survives the next refactor*.

**Follow-up:** *"You moved it to `@PreAuthorize`. What could still make it not
run?"* — Missing `@EnableMethodSecurity`; self-invocation through `this`; a `final`
or `private` method; the annotation on the interface while the proxy is CGLIB-based
on the class. All Topic 40.

### Q4 — "Where does the current user live, and what breaks?"

**MID-LEVEL:** "In `SecurityContextHolder`, you call
`SecurityContextHolder.getContext().getAuthentication()`."

**SENIOR:** "In a `ThreadLocal`, by default. Which means it is present exactly on
the thread that ran the filter chain, and nowhere else.

So it is absent in an `@Async` method, in anything I submit to an executor I built,
in a parallel stream on the common ForkJoinPool, in a Kafka listener, and in an
outbox relay — the last two because there was never an HTTP request on that thread
at all. And `getContext()` never returns null; it manufactures an empty context. So
the failure isn't an exception you'd notice, it's `getAuthentication()` returning
null and a defensively-written `isAdmin()` helper collapsing to `false`, silently
taking the wrong branch.

The fix I prefer is to pass the actor as a parameter — make identity an input, not
ambient state. Where I can't, I decorate the executor with
`DelegatingSecurityContextAsyncTaskExecutor`. What I won't do is
`MODE_INHERITABLETHREADLOCAL`, because inheritance happens at thread *creation* and
a pooled thread is created once, so it pins whichever request happened to create it
and hands that identity to every subsequent task. That turns 'no identity' into
'someone else's identity'.

It's the same bug shape as the MDC correlation id and the OTel trace context, so I
decorate the executor with all three at once."

**What separates them:** the mid-level answer names the API. The senior answer names
the *storage mechanism*, derives the failure set from it, and knows the
`InheritableThreadLocal` trap — which is the fix most people reach for and which is
worse than the bug.

**Follow-up:** *"Virtual threads — does this get better?"* — `ThreadLocal` works on
a virtual thread, so a request served on one is fine. But a virtual thread you
*create* is still a new thread with an empty context. Virtual threads change the
cost of threads, not the semantics of `ThreadLocal`. Topic 101.

### Q5 — "Our health endpoint started returning 401 after a security change. Walk me through it."

**MID-LEVEL:** "Someone secured it by accident; I'd add `permitAll()` for
`/actuator/**`."

**SENIOR:** "First I'd establish whether the rule is unreachable or the chain is
wrong, because those are different fixes. `FilterChainProxy` at DEBUG tells me which
`SecurityFilterChain` was selected for that URI; if it's not the one I expected, the
problem is `securityMatcher` or `@Order`, not the rules.

If the chain is right, it's almost certainly rule ordering.
`authorizeHttpRequests` is first-match-wins, not most-specific-wins, so an
`anyRequest().authenticated()` above the `permitAll()` makes it dead code. That's
the same mechanism as an nginx location block, and the opposite of what people
expect coming from a decorator-based framework.

Then I'd fix the ordering rather than blanket-permitting `/actuator/**`, because
`/actuator/env`, `/actuator/heapdump` and `/actuator/threaddump` must not be
public — `env` prints configuration and `heapdump` is every secret in memory. Only
`health/liveness` and `health/readiness` go in the `permitAll()` list.

And I'd add the four-line test matrix, because ordering bugs come back every time
someone adds a matcher."

**What separates them:** the mid-level answer fixes the symptom and creates an
information-disclosure vulnerability doing it. The senior answer separates chain
selection from rule selection as the first diagnostic step, and knows which Actuator
endpoints are dangerous.

**Follow-up:** *"Same config, but now `/api/admin/**` returns 200 for everyone.
Same root cause?"* — Yes: identical mechanism, `permitAll()` above the admin rule
instead of below it. Same bug, inverted damage, and this one produces no error and
no alert.

---

## Mental model checkpoint

Answer without scrolling up.

1. A request arrives at `/api/orders/8213`. Name every decision point between the
   socket and your controller method body that can reject it, in order.
2. `hasRole("ADMIN")` returns 403 for a token whose payload contains
   `"roles":["ADMIN"]`. Give two distinct causes and the command that tells them
   apart.
3. Why is `permitAll()` not the same as "no security filters run for this path"?
   What is still done, and what can still return 401?
4. State the exact condition under which a state-changing endpoint needs CSRF
   protection. Then say why `SameSite=Lax` does not remove the need.
5. A user's account is disabled in Postgres at 10:00. They place an order at 10:02
   and it succeeds. Explain why, and give three mitigations with their revocation
   latency.
6. `@PostAuthorize` denies a request. Has the method body executed? What does that
   mean if the method was `@Transactional` and wrote a row?
7. You add `DelegatingSecurityContextAsyncTaskExecutor` and the async method still
   logs `auth=null`. Give two causes.

---

## Quick reference card

**The four questions, in configuration order**

```java
.securityMatcher(...)          // 1. does this chain apply?
.authorizeHttpRequests(...)    // 2. what is required? FIRST MATCH WINS
.oauth2ResourceServer(...)     // 3. how is identity established?
.sessionManagement(...).csrf(...)  // 4. is there a session, and therefore CSRF?
```

**Status codes**

| Code | Meaning | Fix direction |
|---|---|---|
| 401 | I do not know who you are | Token, decoder, converter |
| 403 | I know who you are, and no | Authorities, rule, `ROLE_` prefix |
| 200 when you expected 403 | A rule is shadowed or method security is inert | Rule order, `@EnableMethodSecurity`, self-invocation |

**Authority prefixes**

| Written | Checked for |
|---|---|
| `hasRole("ADMIN")` | `ROLE_ADMIN` |
| `hasAuthority("ADMIN")` | `ADMIN` |
| default JWT scope mapping | `SCOPE_<scope-value>` |

**Diagnostics**

```yaml
logging.level.org.springframework.security: DEBUG
logging.level.org.springframework.security.web.FilterChainProxy: DEBUG
```

```bash
curl -i -H "Authorization: Bearer $T" $URL          # read the status AND WWW-Authenticate
echo "$T" | cut -d. -f2 | tr '_-' '/+' | base64 -d | jq .   # decode YOUR token
```

**Gotchas checklist**

- [ ] `anyRequest()` is the **last** rule in every chain.
- [ ] A catch-all chain exists so no path escapes all `securityMatcher`s.
- [ ] `@EnableMethodSecurity` is present, and a **denial** test proves it.
- [ ] No `@PreAuthorize` method is called via `this.`
- [ ] Every JWT has `exp`, and `exp - iat` is minutes, not days.
- [ ] Audience (`aud`) is validated, not just issuer and signature.
- [ ] CSRF is on for every chain that has a session cookie.
- [ ] `/actuator/env`, `/heapdump`, `/threaddump` are not `permitAll()`.
- [ ] Executors are decorated for `SecurityContext` + MDC + trace context together.
- [ ] Filter-chain 401/403 bodies match your `ProblemDetail` contract.
- [ ] Security DEBUG logging is **off** in the load-test profile.

---

## When would I use this at work?

**1. Adding a new endpoint to an existing service.** The two-minute habit: decide
whether the rule is expressible as method-and-path (chain) or needs the entity
(method security), add it in the right place, add the row to the test matrix. Most
authorization vulnerabilities are new endpoints that inherited `anyRequest()` and
nobody noticed the rule was wrong for them.

**2. A security review or a penetration-test finding.** "Broken access control" and
"CSRF on the admin console" are the two most common findings in Java web services,
and both are in this document. Being able to say "here is the chain that has a
session, here is why CSRF is on for it, and here is the matrix that proves the rules"
turns a two-week remediation into a one-hour conversation.

**3. An incident where you must cut off a compromised account right now.** This is
the one that decides whether your JWT design was a decision or a default. The
question "how fast can we revoke, and what breaks if the revocation store is down"
has a number as its answer, and you either know it or you are finding out at 3am.

---

## Connected topics

**Backwards**

- **40 — proxying, JDK/CGLIB, self-invocation.** `@PreAuthorize` is proxy-based.
  Every caveat there applies here, and here the failure is silent.
- **46 — `ProblemDetail`.** Filter-chain rejections bypass `@RestControllerAdvice`;
  you must wire an `AuthenticationEntryPoint` and `AccessDeniedHandler` to keep the
  error contract uniform.
- **47 — Spring Data JPA.** The ownership check should be a projection query, not an
  entity load.
- **54–55 — transactions.** `@PostAuthorize` runs after the body; a transaction
  holds a connection for its whole life, and the ownership check must not nest
  inside it.
- **56 — the filter chain.** The direct prequel: filters, `SecurityContextHolder`,
  authentication. This topic is everything after authentication succeeds.
- **65 — the baseline.** Authentication is now on the hot path. Re-baseline with
  tokens, mint them in `setup()`, and record the verification cost.

**Forwards**

- **91 — `@Async` and executors.** The propagation fix in Trap 4, generalised.
- **101 — virtual threads.** `ThreadLocal` semantics on virtual threads; why they
  change cost, not correctness.
- **109 — HikariCP.** The ownership query is an extra connection acquisition; keep
  it sequential, never nested.
- **113 / 115 / 116 — Kafka, outbox, idempotency.** No `SecurityContext` exists on a
  consumer or relay thread. The actor must travel **in the message**, which makes it
  a schema decision.
- **118 — metrics.** Emit authorization-denial counts by rule, never by user id
  (cardinality).
- **119 / 120 — tracing and MDC.** The same `ThreadLocal` propagation problem, twice
  more. Decorate executors once.
- **121 — Actuator and probes.** Probe paths must be `permitAll()`; sensitive
  Actuator endpoints must not be.
- **123 — graceful shutdown.** Sessions and in-flight authenticated requests during
  a rolling deploy.
- **133 — postmortems.** "We could not revoke the token" is a postmortem action item
  you can pre-empt today.
