# 45 — Bean Validation vs class-validator

## Phase: 5 — Spring Boot & Persistence
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: request DTO validation across the whole `orderflow` API — products, inventory, orders, payments and wallet. Every request body, path variable, query parameter and header gets a declared contract, and a violation becomes an RFC 9457 `ProblemDetail` (Topic 46) rather than a stack trace.

---

## ELI5 anchor

Imagine a government office that processes forms.

At the **front desk**, a clerk checks your form before it goes anywhere. Is the name field
filled in? Is the date in the past? Is the postcode shaped like a postcode? If anything is
wrong, you get the form back immediately with a list of every problem — not just the first
one. Nothing behind the front desk ever sees a bad form.

That front desk is `@Valid` on a controller's `@RequestBody`. It is built into the machine
that reads the form. It always runs. You do not have to ask.

Now, deeper inside the building, there is a **back-office clerk** who also handles forms —
ones that arrive by internal post from other departments, not from the public. That clerk
*can* check the same rules. But they only do it if someone has put a sticker on their office
door saying "this clerk checks forms".

That sticker is `@Validated`. Without it, the clerk processes whatever arrives, however
broken. And here is the part that catches people: **you can write all the rules on the form
and the clerk will happily ignore every one of them**, because nobody put the sticker on the
door. Nothing warns you. The rules are just ink on paper that nobody read.

Two more details that matter.

**The rules can be conditional.** A new-application form and a change-of-address form use
the same paper, but different fields are required. Rather than printing two forms, you print
one with a note beside each field saying which *kind* of submission it applies to. Those are
**validation groups**.

**Some rules are about the form as a whole**, not one field. "The end date must be after the
start date" cannot be written next to either date on its own. That rule goes at the top of
the form, and it needs a person who can see both fields at once. That is a **class-level
constraint**.

---

## The bridge from what you know

### `class-validator` ≈ Jakarta Bean Validation: **HONEST ANALOGUE**

This is one of the cleanest mappings in the whole curriculum. `class-validator` was
explicitly modelled on the Java Bean Validation specification. The decorator names are
frequently identical.

```ts
// NestJS + class-validator
import { IsNotEmpty, IsInt, Min, Max, MaxLength, Matches, ValidateNested, IsOptional } from 'class-validator';
import { Type } from 'class-transformer';

export class LineDto {
  @IsNotEmpty() @Matches(/^[A-Z0-9-]{3,32}$/) sku!: string;
  @IsInt() @Min(1) @Max(999) quantity!: number;
}

export class PlaceOrderDto {
  @ValidateNested({ each: true }) @Type(() => LineDto) lines!: LineDto[];
  @IsOptional() @MaxLength(64) note?: string;
}
```

```java
// Java + Jakarta Bean Validation
public record LineRequest(
        @NotBlank @Pattern(regexp = "^[A-Z0-9-]{3,32}$") String sku,
        @NotNull @Min(1) @Max(999) Integer quantity) {}

public record PlaceOrderRequest(
        @NotEmpty List<@Valid LineRequest> lines,
        @Size(max = 64) String note) {}
```

### The translation table

Keep this open for your first week. It is a straight lookup.

| `class-validator` | Jakarta Bean Validation | Notes |
|---|---|---|
| `@IsDefined()` | `@NotNull` | |
| `@IsNotEmpty()` on a string | `@NotBlank` | `@NotBlank` also rejects whitespace-only. `@NotEmpty` does not. |
| `@IsNotEmpty()` on an array | `@NotEmpty` | works on `Collection`, `Map`, array, `CharSequence` |
| `@IsOptional()` | *(omit `@NotNull`)* | Java's default is "null passes every constraint except `@NotNull`" |
| `@IsString()` | *(the type system)* | **Not needed.** The field is declared `String`. |
| `@IsInt()` | *(the type system)* | **Not needed.** The field is declared `Integer`. |
| `@IsNumber()` | *(the type system)* | **Not needed.** |
| `@Min(n)` / `@Max(n)` | `@Min(n)` / `@Max(n)` | identical |
| `@Min` on a decimal | `@DecimalMin("...")` | string-valued, to avoid float imprecision |
| `@Length(min,max)` / `@MaxLength` | `@Size(min=, max=)` | one annotation for both bounds; also works on collections |
| `@Matches(/re/)` | `@Pattern(regexp = "...")` | Java regex syntax, no delimiters, no flags in the string |
| `@IsEmail()` | `@Email` | both are permissive; neither proves deliverability |
| `@IsPositive()` | `@Positive` | also `@PositiveOrZero`, `@Negative`, `@NegativeOrZero` |
| `@IsDate()` + custom | `@Past`, `@PastOrPresent`, `@Future`, `@FutureOrPresent` | operate on `java.time` types |
| `@IsEnum(E)` | *(the type system)* | **Not needed.** Declare the field as the enum; Jackson rejects unknown values. |
| `@IsUUID()` | *(declare it `UUID`)* or `@Pattern` | Jackson parses `UUID` directly |
| `@IsBoolean()` | *(the type system)* | **Not needed.** |
| `@ValidateNested()` | `@Valid` on the field | |
| `@ValidateNested({ each: true })` | `@Valid` on the **container element**: `List<@Valid X>` | note *where* the annotation goes |
| `@ArrayMinSize` / `@ArrayMaxSize` | `@Size(min=, max=)` | same annotation |
| `groups: ['create']` | `groups = OnCreate.class` | Java groups are **interfaces**, not strings |
| `@Validate(CustomConstraint)` | custom annotation + `ConstraintValidator` | more ceremony, more reusable |
| `ValidationPipe` (global) | `@Valid` per parameter | Spring's is per-parameter, not a global pipe |

Notice how many rows say "not needed". A meaningful fraction of a `class-validator` DTO is
spent asserting things Java's type system asserts for free at compile time. `@IsString()`,
`@IsInt()`, `@IsBoolean()`, `@IsEnum()` all disappear. That is Topic 02's nominal typing
paying a dividend you can see.

### Difference 1 — Nest has `whitelist`. Java has nothing equivalent, and you must handle it elsewhere.

```ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,              // strip properties with no decorator
  forbidNonWhitelisted: true,   // or reject the request outright
}));
```

This is a **security control**, not a tidiness feature. Without it, a client that posts
`{"sku":"X","quantity":1,"internalDiscountApplied":true}` gets that extra field onto the
object, and if anything downstream reads it, you have a mass-assignment vulnerability.

**Jakarta Bean Validation has no `whitelist` concept at all.** It validates the fields it
knows about and says nothing about extras. In Spring, the equivalent control lives in
**Jackson**, one layer earlier:

```yaml
spring:
  jackson:
    deserialization:
      fail-on-unknown-properties: true      # reject unknown fields with a 400
```

`FAIL_ON_UNKNOWN_PROPERTIES` defaults to **false** in Spring Boot, which means unknown
fields are silently discarded — Nest's `whitelist: true` behaviour, not
`forbidNonWhitelisted`. Turning it on gives you the strict behaviour.

There is a second, structural defence Java gives you that Nest does not: **use records for
DTOs**. A record has a canonical constructor with a fixed parameter list. There is no
setter, no `Object.assign`, and no way for a stray JSON property to land anywhere. Mass
assignment is impossible by construction, not by configuration.

> **Take this to your first Java code review:** DTOs are records. Extra JSON fields are
> either rejected loudly or cannot land at all. That combination is what `whitelist: true`
> was for.

### Difference 2 — the one that actually costs people a day: `@Valid` vs `@Validated`

In Nest, the `ValidationPipe` is a pipe. You register it globally or per route, and after
that every decorated DTO passing through a route handler is validated. One mechanism, one
place.

In Spring there are **two separate mechanisms**, and they behave differently:

| | Bean-graph validation | Method validation |
|---|---|---|
| Triggered by | `@Valid` (or `@Validated`) on a `@RequestBody` / `@ModelAttribute` **parameter** | `@Validated` on the **class**, plus constraints on method parameters |
| Implemented by | the argument resolver in `DispatcherServlet` (Topic 44) | a **proxy** created by `MethodValidationPostProcessor` |
| Validates | the object's fields, cascading through `@Valid` | the method's parameters and optionally its return value |
| Exception on failure | `MethodArgumentNotValidException` → 400 | `ConstraintViolationException` → 500 unless you map it |
| Works on a self-call? | n/a | **No** — Topic 40's self-invocation trap, in full |
| Works with no extra annotation? | yes, `@Valid` on the parameter is enough | **no** |

That second column is the whole reason this topic is not "just add annotations". Method
validation is **proxy-based**, so it inherits every constraint from Topic 40:

- A self-call inside the same class bypasses it entirely. Silent.
- A `final` method is not overridden by CGLIB, so it is not validated. Silent.
- A `final` class cannot be proxied — that one fails loudly at startup.
- A `private` method is not advised.
- `@Validated` on the class is mandatory for a non-controller bean; without it,
  `MethodValidationPostProcessor` never selects the bean for proxying and **every constraint
  annotation on every method parameter is dead metadata**.

**Verdict: HONEST ANALOGUE for the annotations, PARTIAL for the mechanism, and the mechanism
is where the day goes.**

### Difference 3 — it is a specification, not a library

`class-validator` is one npm package. Jakarta Bean Validation is a **specification**
(Jakarta Validation 3.1, part of Jakarta EE 11) with **Hibernate Validator** as the
reference implementation. Three consequences you actually feel:

1. **The same annotations work in several places.** `@NotNull` on a JPA entity field makes
   Hibernate validate before insert (see the production example). `@Size` on a
   `@ConfigurationProperties` class (Topic 43) validates your configuration at startup. The
   annotations are not a web-layer feature.
2. **You can swap implementations.** In practice nobody does — Hibernate Validator is what
   Boot ships — but it is why the API is in `jakarta.validation` and the implementation is
   in `org.hibernate.validator`. Constraints from the second package are **not portable**;
   `@org.hibernate.validator.constraints.Length` and `@URL` are Hibernate extensions.
3. **You must add the dependency yourself.** Since Boot 2.3, `spring-boot-starter-web` does
   **not** pull in validation. This is Trap 2 and it is the single most common "my
   annotations do nothing" cause.

### Difference 4 — nothing transforms your input

`class-validator` is usually paired with `class-transformer`, and `ValidationPipe({
transform: true })` converts the plain JSON object into a real class instance with
`@Type()` handling nested types. You have probably been bitten by forgetting `@Type()`.

In Spring, **Jackson has already done this** by the time validation runs. `@RequestBody`
deserialization (Topic 44) constructs a real `PlaceOrderRequest` with a real
`List<LineRequest>` inside it, using the declared generic types. There is no `@Type()`
equivalent because there is nothing to tell Jackson — the type is in the signature and
generics are recovered from the field's declared type, not from erasure at the call site.

You *do* still have to write `@Valid` on the element (`List<@Valid LineRequest>`) to make
validation **cascade** into the nested objects. That is Trap 3, and it is the exact analogue
of forgetting `@ValidateNested({ each: true })`.

---

## What is this?

Jakarta Bean Validation is a declarative constraint system. You annotate fields, method
parameters, method return values and type arguments; a `Validator` walks the object graph
and produces a `Set<ConstraintViolation<T>>`.

### Getting it on the classpath

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

No version — the Boot BOM manages it (Topic 32). That starter brings Hibernate Validator
and the Jakarta Validation API. Boot's `ValidationAutoConfiguration` then registers a
`LocalValidatorFactoryBean` (which implements both `jakarta.validation.Validator` and
Spring's own `org.springframework.validation.Validator`) and a
`MethodValidationPostProcessor`.

> `[BOOT 3.x DELTA]` The starter and its auto-configuration behave the same on Boot 3.x and
> 4.1. The change that will actually break you when reading older material is the package
> move: **`javax.validation.*` became `jakarta.validation.*` in Boot 3.0**. Every import in
> every Stack Overflow answer written before 2023 is wrong for your stack. If you paste one
> and the annotation silently does nothing, check the import first — a `javax.validation`
> annotation on a Boot 3/4 application is not recognised by Hibernate Validator, compiles
> fine if the old API is transitively present, and is ignored at runtime. That failure mode
> is Topic 127's headline.

### The two validation surfaces, in detail

**Surface 1 — bean-graph validation.**

```java
Set<ConstraintViolation<PlaceOrderRequest>> violations = validator.validate(request);
```

The validator reflects over the object's fields and property getters, evaluates each
constraint, and recurses into any field marked `@Valid`. It collects **all** violations, not
just the first. Each `ConstraintViolation` carries a property path
(`lines[2].quantity`), an interpolated message, the invalid value, and the constraint
descriptor.

In Spring MVC this is triggered by `@Valid` on a `@RequestBody` or `@ModelAttribute`
parameter, inside the argument resolver — **before your controller method body runs**, which
is why a controller can assume its input is already valid.

**Surface 2 — method validation.**

```java
@Validated                                   // <-- required on a non-controller bean
@Service
public class InventoryService {
    public void reserve(@NotBlank String sku, @Min(1) int units) { ... }
}
```

Here the constraints are on the **parameters themselves**, not on a DTO's fields. Spring's
`MethodValidationPostProcessor` (a `BeanPostProcessor` — Topic 37) sees `@Validated` on the
class and wraps the bean in an AOP proxy whose interceptor calls
`ExecutableValidator.validateParameters(...)` before delegating.

Failure throws `jakarta.validation.ConstraintViolationException`. Note what it does *not*
do: it does not produce a 400. Unless you map it in a `@ControllerAdvice` (Topic 46), a
constraint violation from a service becomes a **500 Internal Server Error**, which is
arguably correct — an invalid call from your own code is a bug, not a client error.

### The three exception types, and why it matters

| Exception | Thrown when | Default status |
|---|---|---|
| `MethodArgumentNotValidException` | `@Valid` on `@RequestBody` / `@ModelAttribute` fails | 400 |
| `HandlerMethodValidationException` | constraints directly on **controller method parameters** (`@RequestParam @Max(200) int size`) fail | 400 |
| `ConstraintViolationException` | method validation on a non-controller `@Validated` bean fails | 500 unless mapped |

`HandlerMethodValidationException` is a Spring Framework 6.1 addition. Before it, controller
method-parameter constraints threw `ConstraintViolationException` and needed
`@Validated` on the controller class.

> **Honest flag, one line:** I am confident `HandlerMethodValidationException` exists and is
> the modern shape, and reasonably confident that from Framework 6.1 onward controller
> method parameter constraints are validated **without** `@Validated` on the class. I am
> **not** going to assert the exact behaviour on Framework 7.0 from memory. **Settling
> command:** write a `@ControllerAdvice` with
> `@ExceptionHandler(Exception.class)` that logs `ex.getClass().getName()`, remove
> `@Validated` from a controller that has `@RequestParam @Max(200) int size`, send
> `?size=999`, and read which class you actually got. Proof 4 in the Hands-on section is
> that experiment. Do not take this paragraph's word for it and do not take a blog's.

The practical consequence, whichever way it lands on your version: **`@Validated` on a
non-controller bean is always required.** That part is not in question, and it is Trap 1.

### The built-in constraints

| Annotation | Applies to | Passes on `null`? |
|---|---|---|
| `@NotNull` | anything | **no** |
| `@Null` | anything | yes (only `null` passes) |
| `@NotEmpty` | `CharSequence`, `Collection`, `Map`, array | **no** |
| `@NotBlank` | `CharSequence` | **no** — also rejects `"   "` |
| `@Size(min,max)` | `CharSequence`, `Collection`, `Map`, array | yes |
| `@Min` / `@Max` | integral numbers | yes |
| `@DecimalMin` / `@DecimalMax` | numbers, as `String` bounds | yes |
| `@Positive` / `@PositiveOrZero` | numbers | yes |
| `@Negative` / `@NegativeOrZero` | numbers | yes |
| `@Digits(integer,fraction)` | numbers | yes |
| `@Pattern(regexp, flags)` | `CharSequence` | yes |
| `@Email` | `CharSequence` | yes |
| `@Past` / `@PastOrPresent` | `java.time` types, `Date` | yes |
| `@Future` / `@FutureOrPresent` | `java.time` types, `Date` | yes |
| `@AssertTrue` / `@AssertFalse` | `boolean`, `Boolean` | yes |
| `@Valid` | cascades into the object | n/a |

**The single most important row is "passes on `null`".** Every constraint except `@NotNull`,
`@NotEmpty` and `@NotBlank` is satisfied by `null`. `@Size(min = 3)` on a `null` string
**passes**. `@Min(1)` on a `null` `Integer` **passes**. This is deliberate and specified: a
constraint expresses a property of a value, and absence is the concern of `@NotNull`.

So `@Min(1) Integer quantity` alone does not make quantity required. You need
`@NotNull @Min(1) Integer quantity`. Coming from `class-validator`, where `@IsInt()` fails
on `undefined`, this is a genuine behavioural difference and it produces silent nulls.

---

## Why does it matter?

**1. Validation at the boundary is a security control.**

Every unvalidated field is an input your persistence layer, your pricing logic and your
downstream calls have to defend against individually. `@Size(max = 64)` on a string that
ends up in a log line, a database column and an outbound header is one annotation replacing
three defensive checks — and the one you would have forgotten. A `?size=1000000` page
request with no `@Max` is a heap exhaustion at 1,200 rps (Topic 44 already flagged this).

**2. The failure mode of the `@Validated` gap is silence.**

An annotation that nobody reads looks exactly like an annotation that works. The code passes
review because the constraint is right there in the signature. There is no warning, no log
line, no startup check. You discover it when a null `sku` produces a
`DataIntegrityViolationException` from Postgres three layers away, or worse, when the column
is nullable and you get a row you cannot reconcile.

**3. Groups are what stop your DTO count from doubling.**

Without groups, `CreateProductRequest` and `UpdateProductRequest` are two nearly identical
classes that drift apart over eighteen months. With groups, one class carries both
contracts and the difference is visible in one place.

**4. It is the input half of the API contract, and Topic 46 is the output half.**

A `MethodArgumentNotValidException` carries a structured list of field paths and messages.
Turning that into a stable, machine-readable `ProblemDetail` with a `violations` array is
what makes an API usable by a partner integration. Returning `{"message": "Validation
failed"}` is not an error contract; it is an apology.

---

## Syntax breakdown

### The annotations on a DTO

```java
package com.orderflow.catalog.web;

import jakarta.validation.constraints.*;

public record CreateProductRequest(

        @NotBlank(message = "sku is required")
        @Pattern(regexp = "^[A-Z0-9-]{3,32}$", message = "sku must be 3-32 chars of A-Z, 0-9 or -")
        String sku,

        @NotBlank
        @Size(max = 200)
        String name,

        @NotNull
        @Positive
        Long priceMinor,                       // WRAPPER, not long. See Trap 4.

        @NotNull
        @Pattern(regexp = "^[A-Z]{3}$")
        String currency
) {}
```

| Piece | What it does |
|---|---|
| `jakarta.validation.constraints.*` | The specification's package. **Not** `javax.`, **not** `org.hibernate.validator.constraints`. |
| `message = "..."` | Overrides the default message. Prefer a key (`{orderflow.sku.invalid}`) plus a `ValidationMessages.properties` file if you need i18n. |
| `@Pattern(regexp = ...)` | Java regex. No `/` delimiters. `^` and `$` are not implied — `@Pattern` matches the **whole** string only if you anchor it. |
| `Long priceMinor` | A wrapper. A primitive `long` cannot be `null`, so `@NotNull` on it is always satisfied and a missing JSON field silently becomes `0`. |
| Order of annotations | Irrelevant. All constraints on a field are evaluated; there is no short-circuit. |

### `@Valid` — cascading

```java
public record PlaceOrderRequest(

        @NotEmpty(message = "an order must have at least one line")
        @Size(max = 50, message = "an order may not exceed 50 lines")
        List<@Valid LineRequest> lines,        // <-- @Valid on the ELEMENT

        @Valid
        DeliveryAddress address,               // <-- @Valid on the FIELD

        @Size(max = 64)
        String note
) {}
```

| Placement | Meaning |
|---|---|
| `@Valid` on a field | cascade into that object's constraints |
| `@Valid` inside the generic (`List<@Valid X>`) | cascade into **each element** |
| `@Valid` on the field holding the list (`@Valid List<X>`) | **also** cascades into elements — both forms work, but the element form is clearer and is the one the specification calls a container element constraint |
| **no `@Valid` anywhere** | the nested object's constraints are **never evaluated**. Silent. Trap 3. |

Container element constraints work on more than `@Valid`:

```java
List<@NotBlank String> tags;                  // each element must be non-blank
Map<String, @Positive Integer> quantities;    // each value must be positive
Optional<@Size(max = 20) String> couponCode;  // the contained value, if present
```

### `@Valid` vs `@Validated` — the distinction to memorise

```java
import jakarta.validation.Valid;                       // the SPECIFICATION annotation
import org.springframework.validation.annotation.Validated;   // a SPRING annotation
```

| | `@Valid` | `@Validated` |
|---|---|---|
| Package | `jakarta.validation` | `org.springframework.validation.annotation` |
| Supports groups? | **no** | **yes** — `@Validated(OnCreate.class)` |
| Cascades into nested objects? | **yes** | no (it is not a cascade marker) |
| Enables method validation on a bean? | no | **yes** — this is its main job |
| Where you put it | on a parameter, a field, or a type argument | on a **class** (to enable method validation) or on a controller parameter (to select groups) |

The rule that covers 95% of cases:

> **`@Valid` on parameters and nested fields. `@Validated` on the class when you want method
> validation, and on a parameter when you need groups.**

### Groups

```java
package com.orderflow.catalog.web;

public interface OnCreate {}      // marker interfaces. No members, ever.
public interface OnUpdate {}
```

```java
public record ProductRequest(

        @Null(groups = OnCreate.class,  message = "id must not be supplied on create")
        @NotNull(groups = OnUpdate.class, message = "id is required on update")
        Long id,

        @NotNull(groups = OnUpdate.class, message = "version is required on update")
        Long version,                                  // optimistic locking, Topic 52

        @NotBlank(groups = {OnCreate.class, OnUpdate.class})
        @Pattern(regexp = "^[A-Z0-9-]{3,32}$", groups = {OnCreate.class, OnUpdate.class})
        String sku,

        @NotBlank(groups = {OnCreate.class, OnUpdate.class})
        @Size(max = 200, groups = {OnCreate.class, OnUpdate.class})
        String name,

        @NotNull(groups = OnCreate.class)
        @Positive(groups = {OnCreate.class, OnUpdate.class})
        Long priceMinor
) {}
```

```java
@PostMapping("/api/products")
ProductResponse create(@Validated(OnCreate.class) @RequestBody ProductRequest req) { ... }

@PutMapping("/api/products/{id}")
ProductResponse update(@PathVariable Long id,
                       @Validated(OnUpdate.class) @RequestBody ProductRequest req) { ... }
```

Four rules about groups that people get wrong:

1. **A group is an interface.** Empty, no methods. Its only purpose is to be a type token.
2. **A constraint with no `groups` attribute belongs to `Default.class` only.** So the
   moment you start using groups on a class, a constraint you forgot to tag is **not
   evaluated** by `@Validated(OnCreate.class)`. This is the most common group bug. Either tag
   everything, or include `Default.class` in the group list you pass.
3. **`@Valid` cannot select a group.** Only `@Validated(SomeGroup.class)` can. So a DTO
   using groups must be validated with `@Validated`, not `@Valid`.
4. **Group inheritance works**: `interface OnUpdate extends Default {}` makes
   `@Validated(OnUpdate.class)` also evaluate untagged constraints. This is usually what you
   actually want, and it removes rule 2's footgun.

```java
public interface OnCreate extends jakarta.validation.groups.Default {}
public interface OnUpdate extends jakarta.validation.groups.Default {}
```

**`@GroupSequence`** orders groups so a later group is skipped if an earlier one failed:

```java
@GroupSequence({ Basics.class, Expensive.class })
public interface FullValidation {}
```

Use it when a later constraint is expensive — a database lookup, a remote call — and is
pointless if the cheap format checks already failed.

### A custom cross-field constraint

Two pieces: the annotation, and the validator.

```java
package com.orderflow.orders.web;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = NoDuplicateSkusValidator.class)
@Target({ ElementType.TYPE, ElementType.ANNOTATION_TYPE })   // TYPE = class-level
@Retention(RetentionPolicy.RUNTIME)
public @interface NoDuplicateSkus {
    String message() default "an order may not list the same sku twice";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

| Required member | Why |
|---|---|
| `message()` | the specification requires it; it is what gets interpolated |
| `groups()` | the specification requires it; omitting it is a compile-time failure from the validator engine at startup |
| `payload()` | the specification requires it; used for carrying metadata like severity. You will almost never use it, and you must still declare it. |

```java
package com.orderflow.orders.web;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.util.HashSet;
import java.util.Set;

public class NoDuplicateSkusValidator
        implements ConstraintValidator<NoDuplicateSkus, PlaceOrderRequest> {

    @Override
    public boolean isValid(PlaceOrderRequest req, ConstraintValidatorContext ctx) {
        if (req == null || req.lines() == null) {
            return true;                     // null is @NotNull's problem, not ours
        }
        Set<String> seen = new HashSet<>();
        for (LineRequest line : req.lines()) {
            if (line != null && line.sku() != null && !seen.add(line.sku())) {
                ctx.disableDefaultConstraintViolation();
                ctx.buildConstraintViolationWithTemplate(
                        "duplicate sku: " + line.sku())
                   .addPropertyNode("lines")
                   .addConstraintViolation();
                return false;
            }
        }
        return true;
    }
}
```

```java
@NoDuplicateSkus                              // class-level: sees the whole object
public record PlaceOrderRequest(...) {}
```

Three points:

- **`isValid` returns `true` for `null`.** Every custom validator should. Mixing "is it
  present" into "is it well-formed" makes the constraint unusable on an optional field.
- **`addPropertyNode("lines")`** attaches the violation to a field path so the client sees
  *which* field is wrong rather than a bare object-level message.
- **A `ConstraintValidator` can be a Spring bean.** Hibernate Validator's Spring integration
  resolves validators from the `ApplicationContext`, so you can constructor-inject a
  repository. **Do this sparingly** — a validator that hits the database runs on the request
  thread inside the argument resolver, before any transaction exists, and turns a format
  check into a query. For `orderflow`, "does this SKU exist" belongs in the service with a
  proper domain exception, not in a constraint.

### Method validation on a service

```java
package com.orderflow.inventory;

import jakarta.validation.Valid;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;

@Validated                                    // <-- WITHOUT THIS, NOTHING BELOW RUNS
@Service
public class InventoryService {

    public void reserve(@NotBlank String sku, @Min(1) int units) { ... }

    public void applyAdjustment(@Valid StockAdjustment adjustment) { ... }
}
```

`@Validated` at class level makes `MethodValidationPostProcessor` proxy the bean. Everything
in Topic 40 then applies: no self-calls, no `final` methods, no `final` class, no `private`
methods.

---

## Example 1 — minimal

One DTO, one controller, one handler. The point is to see all three failure paths.

```java
package com.orderflow.lab;

import jakarta.validation.constraints.*;

public record CreateProductRequest(
        @NotBlank @Pattern(regexp = "^[A-Z0-9-]{3,32}$") String sku,
        @NotBlank @Size(max = 200) String name,
        @NotNull @Positive Long priceMinor
) {}
```

```java
package com.orderflow.lab;

import jakarta.validation.Valid;
import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/lab/products")
public class LabProductController {

    private final LabProductService service;

    public LabProductController(LabProductService service) { this.service = service; }

    /** Path A — bean-graph validation. Fails with MethodArgumentNotValidException -> 400. */
    @PostMapping
    public ResponseEntity<String> create(@Valid @RequestBody CreateProductRequest req) {
        return ResponseEntity.ok("created " + req.sku());
    }

    /** Path B — controller method-parameter validation. See the honest flag above. */
    @GetMapping
    public String list(@RequestParam(defaultValue = "50") @Min(1) @Max(200) int size) {
        return "listing " + size;
    }

    /** Path C — service method validation. Only works if LabProductService is @Validated. */
    @PostMapping("/direct")
    public String direct(@RequestBody CreateProductRequest req) {
        service.save(req.sku(), req.priceMinor());
        return "saved";
    }
}
```

```java
package com.orderflow.lab;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;

@Validated                                   // comment this out to reproduce Trap 1
@Service
public class LabProductService {
    public void save(@NotBlank String sku, @Positive Long priceMinor) {
        System.out.println("SAVE sku=" + sku + " price=" + priceMinor);
    }
}
```

Three distinct paths, three distinct exception types, and one of them — path C — quietly
does nothing if you remove one annotation. That is the whole lesson in twenty lines.

---

## Example 2 — production scenario (on the project spine)

### The constraints, stated concretely

- **1,200 rps** across 6 replicas. 10% is `POST /api/orders`.
- **40 partner tenants** send orders. Partner integrations are written once and never
  touched again, so error responses must be **machine-readable and stable**, and a
  validation failure must name every bad field in one response — a partner cannot iterate
  request-by-request against a production API.
- Order bodies: **1 to 50 lines**, quantity **1 to 999** per line, SKU matching
  `^[A-Z0-9-]{3,32}$`, no duplicate SKUs, currency an ISO-4217 code.
- The `Idempotency-Key` header is **mandatory** and at most 64 characters (Topics 44, 116).
- Product create and update share one DTO. Create must reject a client-supplied id; update
  must require both an id and an optimistic-locking version (Topic 52).
- Every validation failure returns RFC 9457 `ProblemDetail` (Topic 46) with a `violations`
  array. Nothing else.

### The DTOs

```java
package com.orderflow.orders.web;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import java.util.List;

@NoDuplicateSkus
public record PlaceOrderRequest(

        @NotEmpty(message = "an order must contain at least one line")
        @Size(max = 50, message = "an order may contain at most 50 lines")
        List<@Valid LineRequest> lines,

        @NotBlank
        @Pattern(regexp = "^[A-Z]{3}$", message = "currency must be a 3-letter ISO-4217 code")
        String currency,

        @NotNull(message = "walletId is required")
        java.util.UUID walletId,

        @Size(max = 500, message = "note may be at most 500 characters")
        String note
) {

    public record LineRequest(

            @NotBlank(message = "sku is required")
            @Pattern(regexp = "^[A-Z0-9-]{3,32}$")
            String sku,

            @NotNull(message = "quantity is required")
            @Min(value = 1,   message = "quantity must be at least 1")
            @Max(value = 999, message = "quantity must be at most 999")
            Integer quantity                    // WRAPPER. A primitive int would default to 0.
    ) {}
}
```

Five decisions worth defending:

**1. `Integer quantity`, not `int quantity`.** With `int`, a body omitting `quantity`
deserializes to `0`, `@NotNull` is trivially satisfied because a primitive is never null,
and `@Min(1)` then catches it — but with a *confusing* message ("must be at least 1" for a
field the client never sent). With `Integer`, a missing field is `null`, `@NotNull` fires,
and the message says "quantity is required". Same protection, honest diagnosis. This is
Topic 01's boxing distinction with a business consequence.

**2. `UUID walletId`, not `String walletId`.** Jackson parses it. A malformed value is
rejected by the message converter before validation even runs, and you never write a
`@Pattern` for UUID format. Let the type system do the work — Topic 02.

**3. `@Size(max = 50)` on `lines` is a capacity control, not a business rule.** A partner
posting 100,000 lines at 120 rps is a heap-exhaustion incident. The bound is chosen from
what the business actually needs (largest observed real order plus headroom), and it is
enforced at the boundary where rejecting is cheap.

**4. `@Size(max = 500)` on `note`.** It goes into a Postgres column, a log line and
possibly an email. Unbounded client strings crossing three systems is how you get a log
pipeline outage.

**5. Messages are written for the partner, not for you.** "must not be blank" is Hibernate
Validator's default and tells an integrator nothing about which field or why. Naming the
field and the rule in the message removes a support ticket per partner per year.

### The controller

```java
package com.orderflow.orders.web;

import jakarta.validation.Valid;
import jakarta.validation.constraints.Size;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.util.UriComponentsBuilder;

@RestController
@RequestMapping(path = "/api/orders", produces = "application/json")
@Validated                                       // enables the header constraint below
public class OrderController {

    private final OrderPlacementService placement;

    public OrderController(OrderPlacementService placement) { this.placement = placement; }

    @PostMapping
    public ResponseEntity<OrderResponse> place(
            @RequestHeader("Idempotency-Key") @Size(min = 8, max = 64) String idempotencyKey,
            @Valid @RequestBody PlaceOrderRequest request,
            UriComponentsBuilder uriBuilder) {

        var result = placement.place(idempotencyKey, request.toCommand());

        var location = uriBuilder.path("/api/orders/{id}")
                                 .buildAndExpand(result.orderId()).toUri();

        return result.wasCreated()
                ? ResponseEntity.created(location).body(result.response())
                : ResponseEntity.ok().location(location).body(result.response());
    }
}
```

`@Validated` on the controller class covers the `@Size` on the header parameter. On
Framework 6.1+ this may be unnecessary for controllers — see the honest flag above — but it
is harmless, explicit, and portable to Boot 3.x. **Leaving it on is the defensible choice.**

### The product DTO with groups — one class, two contracts

```java
package com.orderflow.catalog.web;

import jakarta.validation.constraints.*;
import jakarta.validation.groups.Default;

public interface OnCreate extends Default {}
public interface OnUpdate extends Default {}

public record ProductRequest(

        @Null(groups = OnCreate.class,
              message = "id is assigned by the server and must not be supplied on create")
        @NotNull(groups = OnUpdate.class,
              message = "id is required on update")
        Long id,

        @NotNull(groups = OnUpdate.class,
              message = "version is required on update; fetch the product first")
        Long version,

        @NotBlank
        @Pattern(regexp = "^[A-Z0-9-]{3,32}$")
        String sku,

        @NotBlank @Size(max = 200)
        String name,

        @NotNull(groups = OnCreate.class)
        @Positive
        Long priceMinor,

        @NotNull(groups = OnCreate.class)
        @Pattern(regexp = "^[A-Z]{3}$")
        String currency
) {}
```

```java
@PostMapping("/api/products")
@ResponseStatus(HttpStatus.CREATED)
ProductResponse create(@Validated(OnCreate.class) @RequestBody ProductRequest req) { ... }

@PutMapping("/api/products/{id}")
ProductResponse update(@PathVariable long id,
                       @Validated(OnUpdate.class) @RequestBody ProductRequest req) { ... }
```

`OnCreate` and `OnUpdate` **extend `Default`**, so untagged constraints (`sku`, `name`) are
evaluated by both. Without that inheritance, `@Validated(OnCreate.class)` would skip every
untagged constraint and a product could be created with a blank name. That is the group
footgun from rule 2, disarmed structurally rather than by remembering to tag everything.

**`@Null(groups = OnCreate.class)` on `id` is a real security control**, not tidiness.
Without it, a partner posting `{"id": 41, "sku": "X", ...}` to the create endpoint could
cause your mapper to write to product 41 — which belongs to a different tenant. That is
mass assignment via an id field, and it is one of the most common API vulnerabilities.
Rejecting a supplied id on create closes it declaratively.

### Turning violations into a stable `ProblemDetail`

```java
package com.orderflow.web;

import jakarta.validation.ConstraintViolationException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.net.URI;
import java.util.Comparator;
import java.util.List;

@RestControllerAdvice
public class ValidationProblemAdvice {

    private static final URI TYPE =
            URI.create("https://orderflow.example/errors/validation-failed");

    /** @Valid on a @RequestBody. The common case. */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail onBodyInvalid(MethodArgumentNotValidException ex) {
        List<Violation> violations = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> new Violation(fe.getField(), fe.getDefaultMessage()))
                .sorted(Comparator.comparing(Violation::field))     // STABLE ORDER
                .toList();
        return problem(violations);
    }

    /** Method validation on a @Validated bean, including controller parameters/headers. */
    @ExceptionHandler(ConstraintViolationException.class)
    ProblemDetail onConstraintViolation(ConstraintViolationException ex) {
        List<Violation> violations = ex.getConstraintViolations().stream()
                .map(cv -> new Violation(cv.getPropertyPath().toString(), cv.getMessage()))
                .sorted(Comparator.comparing(Violation::field))
                .toList();
        return problem(violations);
    }

    private ProblemDetail problem(List<Violation> violations) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setType(TYPE);
        pd.setTitle("Validation failed");
        pd.setDetail(violations.size() + " field(s) failed validation");
        pd.setProperty("violations", violations);
        return pd;
    }

    record Violation(String field, String message) {}
}
```

Four decisions:

- **Sorted violations.** Hibernate Validator's violation order is not specified. An
  unsorted list makes response bodies non-deterministic, which breaks contract tests
  (Topic 62) and makes partner-side diffing useless. Sorting costs nothing and removes a
  class of flaky test.
- **The `type` URI is stable and documented.** That is what makes the error machine-readable
  — a client switches on `type`, not on a prose `title`.
- **Never put the invalid value in the response.** `cv.getInvalidValue()` is tempting and
  will eventually echo a password, a token, or a card number into a log aggregator.
- **`ConstraintViolationException` mapped to 400 here.** Note the judgement: for a
  *controller parameter* that is correct, because the client caused it. For a violation from
  a deep service method, 400 is arguably a lie — that was your bug, not theirs. If you need
  the distinction, handle `HandlerMethodValidationException` separately for the controller
  layer and let service-layer `ConstraintViolationException` be a 500. Whether your version
  throws the newer type is Proof 4.

> Also note where `@ControllerAdvice` **cannot** help: a validation failure raised in a
> Spring Security filter, or a rejection by the message converter before dispatch. Topic 56
> covers the security entry point; Topic 46 covers `ErrorResponse` and the converter path.

### Where else the same annotations run — and the trap in it

```java
@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    private Long id;

    @NotBlank                                  // Bean Validation
    @Column(nullable = false, length = 32)     // JPA schema metadata
    private String sku;

    @NotNull @Positive
    @Column(name = "price_minor", nullable = false)
    private long priceMinor;
}
```

Hibernate integrates Bean Validation and, by default, validates entities on
pre-insert/pre-update. That sounds like free defence in depth. It has a sharp edge:

> **An entity constraint violation surfaces at flush, not at the setter.** Inside a
> `@Transactional` method, that is usually at commit — potentially hundreds of lines after
> the code that set the bad value, with a `ConstraintViolationException` wrapped in a
> `RollbackException` and a stack trace that names Hibernate, not your code. The transaction
> is already marked rollback-only, so nothing you do in a catch block can save it.

The rule this produces:

> **Validate at the boundary so bad data never reaches an entity. Keep entity constraints as
> a last-resort backstop, and expect their diagnostics to be poor.** `@Column(nullable =
> false)` is doing more useful work than `@NotNull` here: it puts a `NOT NULL` in the
> generated DDL, which every writer must obey — including the batch job and the Kafka
> consumer that never go through your controller.

The layered defence for `orderflow`, in order of when it fires and how good the diagnosis is:

| Layer | Fires | Diagnosis quality | Covers |
|---|---|---|---|
| Jackson type binding | before validation | good (400, names the field) | wrong types, malformed UUIDs |
| `@Valid` on the DTO | before the controller body | **best** — all violations, named paths | every HTTP write path |
| Domain invariants in the entity constructor | at object creation | good — a real exception at the cause | every path, including consumers |
| Bean Validation on the entity | at flush | **poor** — far from the cause | a backstop |
| Postgres `NOT NULL` / `CHECK` | at flush | poor, but **absolute** | every writer, including `psql` |

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `@Valid` on a service method does nothing without `@Validated`

**Wrong:**

```java
@Service                                        // <-- no @Validated
public class InventoryService {

    private final InventoryRepository repo;

    public void reserve(@NotBlank String sku, @Min(1) int units) {
        repo.decrement(sku, units);
    }
}
```

**Exact symptom.** Nothing at startup. Nothing in the logs. `reserve(null, -5)` runs the
body. Then, depending on the schema, one of:

- `org.springframework.dao.DataIntegrityViolationException` wrapping
  `org.postgresql.util.PSQLException: ERROR: null value in column "sku" of relation
  "inventory" violates not-null constraint` — *illustration of the message shape, not
  captured output*. A 500, from a stack that names Hibernate and Postgres and never mentions
  `InventoryService`.
- Or, if the column is nullable: **no error at all**, and a row with a null SKU that
  reconciliation finds three weeks later.
- Or, with `units = -5`: available stock silently **increases**, and inventory drifts away
  from physical reality with no alert.

The direct confirmation, once you suspect it:

```java
System.out.println(inventoryService.getClass().getName());
```

If it prints `com.orderflow.inventory.InventoryService` with no `$$SpringCGLIB$$` marker,
the bean was never proxied, so no interceptor exists to run the validation. That one line
is the diagnosis.

**Root cause.** Method validation is implemented by `MethodValidationPostProcessor`, a
`BeanPostProcessor` (Topic 37) that selects beans for proxying **by looking for
`@Validated`**. No `@Validated`, no proxy. No proxy, no interceptor. No interceptor, and the
constraint annotations are what Topic 40 called them: inert bytes in the class file's
`RuntimeVisibleAnnotations` attribute that nothing read.

**Fix:**

```java
@Validated                                      // the whole fix
@Service
public class InventoryService { ... }
```

**And the three follow-on traps that come free with it**, because `@Validated` means a
proxy:

```java
@Validated
@Service
public class InventoryService {

    public void reserveAll(List<Reservation> rs) {
        rs.forEach(r -> reserve(r.sku(), r.units()));   // SELF-CALL -> not validated
    }

    public final void reserve(@NotBlank String sku, @Min(1) int units) { }
    //         ^^^^^ final -> CGLIB cannot override -> not validated, silently
}
```

Self-invocation and `final` are Topic 40 in full. A `final` **class** at least fails loudly
at startup with `AopConfigException`.

**The regression barrier:**

```java
@SpringBootTest
class MethodValidationIsActive {

    @Autowired InventoryService inventory;

    @Test
    void service_is_proxied_for_method_validation() {
        assertThat(AopUtils.isAopProxy(inventory)).isTrue();
    }

    @Test
    void blank_sku_is_rejected_before_the_repository_is_touched() {
        assertThatThrownBy(() -> inventory.reserve("  ", 1))
            .isInstanceOf(ConstraintViolationException.class);
    }
}
```

The second test is the one that matters: it fails the moment someone deletes `@Validated`,
makes the method `final`, or refactors the call into a self-invocation.

---

### Trap 2 — `spring-boot-starter-validation` is not on the classpath

**Wrong:** a `pom.xml` with `spring-boot-starter-web` and no `spring-boot-starter-validation`.

**Exact symptom.** The application compiles and starts. `POST /api/products` with
`{"sku":"","name":"","priceMinor":-1}` returns **201 Created**. Every `@NotBlank`,
`@Positive` and `@Size` is ignored. A blank-SKU product is now in the catalogue, and
`GET /api/products` returns it.

There is no warning. `@Valid` on the parameter is not an error when no validator exists —
the argument resolver simply finds no `Validator` bean and skips validation.

A second observable: the annotations still **compile**, because
`jakarta.validation-api` often arrives transitively via Hibernate/JPA even without the
starter. So the code looks completely correct.

**Root cause.** Since Spring Boot 2.3, validation was removed from
`spring-boot-starter-web` to reduce the default dependency footprint. It must be added
explicitly. Boot's `ValidationAutoConfiguration` is `@ConditionalOnClass(ExecutableValidator.class)`
and backs off silently when Hibernate Validator is absent — silence is the designed
behaviour of a conditional back-off (Topic 42), and here it is exactly what hurts.

**Fix:**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

**Confirm it, do not assume it:**

```bash
./mvnw dependency:tree | grep -i -E "hibernate-validator|jakarta.validation"
```

You want a line for `org.hibernate.validator:hibernate-validator`. `jakarta.validation-api`
on its own is the API with no engine — annotations that compile and do nothing.

**And write the test that makes it impossible to regress:**

```java
@WebMvcTest(ProductController.class)
class ValidationIsWired {
    @Autowired MockMvc mvc;

    @Test
    void blank_sku_is_a_400() throws Exception {
        mvc.perform(post("/api/products").contentType(APPLICATION_JSON)
              .content("""
                  {"sku":"","name":"x","priceMinor":100,"currency":"GBP"}
                  """))
           .andExpect(status().isBadRequest());
    }
}
```

One test. It catches a missing dependency, a wrong import, a missing `@Valid`, and a
misconfigured advice. Every API should have one per DTO.

---

### Trap 3 — nested objects and collection elements are never validated

**Wrong:**

```java
public record PlaceOrderRequest(
        @NotEmpty List<LineRequest> lines,     // <-- no @Valid on the element
        @NotBlank String currency
) {}
```

**Exact symptom.** `POST /api/orders` with

```json
{"lines":[{"sku":"","quantity":-4}],"currency":"GBP","walletId":"..."}
```

returns **201 Created**. `@NotEmpty` on `lines` passed — the list has one element. Then
`@NotBlank` on `sku` and `@Min(1)` on `quantity` were **never evaluated**, because nothing
told the validator to descend into the element.

Downstream: an order line with a blank SKU and quantity −4. The inventory decrement runs
with −4, which *increases* available stock. Now your inventory says you have four more units
of a product than physically exist, you oversell, and the reconciliation shows a phantom
inventory gain with no purchase order behind it. That takes a day to trace and the root
cause is one missing annotation.

**Root cause.** Bean Validation does **not** cascade by default. Cascading is opt-in per
field or per container element, marked with `@Valid`. This is the exact analogue of
forgetting `@ValidateNested({ each: true })` in `class-validator` — and it fails the same
way: silently, with a green build.

**Fix:**

```java
public record PlaceOrderRequest(
        @NotEmpty @Size(max = 50) List<@Valid LineRequest> lines,
        @NotBlank String currency
) {}
```

Also cascade into single nested objects and map values:

```java
@Valid DeliveryAddress address;                          // single nested object
Map<String, @Valid LineRequest> linesBySku;              // map values
Optional<@Valid CouponRequest> coupon;                   // Optional contents
```

**How to find every instance of this in an existing codebase:** for each DTO with a nested
type or a collection of a custom type, write a test posting an outer-valid, inner-invalid
body and assert 400. Any that return 2xx are missing a `@Valid`. This is mechanical, and it
is the kind of sweep worth doing once across a whole API.

---

### Trap 4 — `@NotNull` on a primitive is always satisfied, and a missing field becomes zero

**Wrong:**

```java
public record LineRequest(
        @NotBlank String sku,
        @NotNull int quantity                  // primitive
) {}
```

**Exact symptom.** A body of `{"sku":"WIDGET-1"}` — with `quantity` **absent entirely** —
passes validation and produces `quantity = 0`. The order is created with a zero-quantity
line. The order total is £0.00. The wallet debit is zero. Inventory is decremented by zero.
Every downstream system succeeds. The customer receives an order confirmation for nothing
and is charged nothing, and finance sees a £0.00 order with a real line on it.

With `@Min(1)` present as well, it *is* caught — but the client gets "quantity must be at
least 1" for a field they never sent, which is a confusing message that produces a support
ticket instead of a fix.

Money is the sharper version:

```java
        @NotNull long priceMinor                // primitive
```

A body omitting `priceMinor` creates a product priced at zero. That is a live pricing
incident, and the `@NotNull` sitting right there in the source is why nobody suspects
validation.

**Root cause.** A Java primitive cannot hold `null` (Topic 01). `@NotNull` on an `int` is
therefore a tautology — the validator receives an autoboxed `Integer` holding `0`, which is
not null, so the constraint passes. Meanwhile Jackson, finding no JSON property, leaves the
primitive at its default: `0` for integral types, `0.0` for floating point, `false` for
`boolean`.

**Fix:** **every optional-shaped field in a request DTO is a wrapper type.**

```java
public record LineRequest(
        @NotBlank String sku,
        @NotNull @Min(1) @Max(999) Integer quantity
) {}
```

Now a missing field is `null`, `@NotNull` fires, and the message says "quantity is required".
The cost is one allocation per field per request (Topic 01) — utterly irrelevant at DTO
scale, where you are already allocating a `String` per field.

**The general rule for the whole `orderflow` API:**

> **Request DTOs use wrapper types. Domain objects and entities use primitives.** The DTO's
> job is to distinguish "absent" from "zero". The domain's job is to hold a value that is
> already known to be present, where a primitive is both cheaper and more honest — it makes
> the absent state unrepresentable.

Grep for the smell across a codebase:

```bash
grep -rnE "@NotNull[[:space:]]+(int|long|double|float|short|byte|boolean|char) " src/main/java
```

Every hit is either this bug or a redundant annotation. Both are worth fixing.

---

### Trap 5 — one DTO for create and update, with no groups

**Wrong:**

```java
public record ProductRequest(
        Long id,                                // no constraint — can't require it on update
        @NotBlank String sku,
        @NotBlank String name,
        @NotNull @Positive Long priceMinor
) {}
```

used by both `POST /api/products` and `PUT /api/products/{id}`.

**Exact symptom — two of them, and both are real incidents.**

*Symptom A — mass assignment on create.* A partner posts to `POST /api/products`:

```json
{"id": 41, "sku":"NEW-SKU", "name":"New", "priceMinor": 100}
```

`id` has no constraint, so it binds. If the mapper does `repository.save(toEntity(req))`
and the entity's id is populated, Hibernate treats it as **detached** and issues an
`UPDATE` (or a merge) instead of an `INSERT`. Product 41 — belonging to a different tenant —
now has a new SKU, name and price. Response: `201 Created`. Nothing anywhere logs a problem.

*Symptom B — lost updates on update.* The version field is optional, so a partner omits it.
The service falls back to "load and overwrite", which drops optimistic locking entirely
(Topic 52). Two concurrent price updates: the second silently overwrites the first. The
observable is a price that reverts, reported as "the price change didn't save" and
irreproducible on demand.

*And symptom C, the slow one:* someone eventually creates `UpdateProductRequest` by
copy-paste. Eighteen months later the two classes have drifted — a constraint was tightened
on one and not the other — and a field is validated on create but not on update. This is the
default outcome and it is the reason groups exist.

**Root cause.** One DTO expressing two different contracts, with no way to say which
constraints apply to which. "Required on update, forbidden on create" is not expressible
with an untagged annotation.

**Fix — groups extending `Default`:**

```java
public interface OnCreate extends jakarta.validation.groups.Default {}
public interface OnUpdate extends jakarta.validation.groups.Default {}

public record ProductRequest(
        @Null(groups = OnCreate.class,
              message = "id is server-assigned and must not be supplied on create")
        @NotNull(groups = OnUpdate.class, message = "id is required on update")
        Long id,

        @NotNull(groups = OnUpdate.class,
              message = "version is required on update; GET the product first")
        Long version,

        @NotBlank @Pattern(regexp = "^[A-Z0-9-]{3,32}$") String sku,
        @NotBlank @Size(max = 200) String name,
        @NotNull(groups = OnCreate.class) @Positive Long priceMinor
) {}
```

```java
create(@Validated(OnCreate.class) @RequestBody ProductRequest req)
update(@Validated(OnUpdate.class) @RequestBody ProductRequest req)
```

**Two things that will bite you the first time:**

1. **`@Valid` cannot select a group.** You must write `@Validated(OnCreate.class)`. Leaving
   `@Valid` there means the `Default` group runs and the create/update distinction silently
   does not apply.
2. **Extend `Default`.** Without it, `@Validated(OnCreate.class)` evaluates *only*
   `OnCreate`-tagged constraints, so `@NotBlank String sku` — untagged — is skipped, and a
   blank SKU is created. Extending `Default` fixes it structurally so nobody has to remember
   to tag every constraint.

**And the defence in depth that costs nothing:** put a `@Null(groups = OnCreate.class)` on
every server-assigned field (`id`, `version`, `createdAt`, `tenantId`). It is a two-word
declarative statement that closes the entire mass-assignment class of vulnerability for
that DTO.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM, no Postgres and no running service,
and I will not print output and call it captured. What follows is the exact configuration,
the exact command, what to look for, and how to read every result you might get.

### Setup

```bash
mkdir -p ~/java-lab/45 && cd ~/java-lab/45
java --version           # expect 21 or 25

curl https://start.spring.io/starter.zip \
  -d dependencies=web,validation \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=validation-lab \
  -d type=maven-project -o validation-lab.zip && unzip validation-lab.zip -d validation-lab
cd validation-lab
```

Add the Example 1 classes. Add this to `src/main/resources/application.properties`:

```properties
# Show the field-level messages in the default error body while you experiment.
server.error.include-message=always
server.error.include-binding-errors=always

# Nest's whitelist:true / forbidNonWhitelisted equivalent
spring.jackson.deserialization.fail-on-unknown-properties=true

logging.level.org.springframework.web=DEBUG
```

> `server.error.include-*` are **development-only**. Leaving them on in production leaks
> internal detail into error bodies. In `orderflow` the real answer is the `ProblemDetail`
> advice from Example 2, which controls exactly what is exposed.

### Proof 0 — confirm the engine is actually present

```bash
./mvnw dependency:tree | grep -i -E "hibernate-validator|jakarta.validation|jakarta.el"
```

| What you see | What it means |
|---|---|
| A line for `org.hibernate.validator:hibernate-validator` | The engine is present. Validation can work. |
| Only `jakarta.validation:jakarta.validation-api` | **Trap 2.** You have the annotations and no engine. Everything compiles; nothing validates. Add `spring-boot-starter-validation`. |
| Neither line | The annotations will not even compile. You will notice this one. |
| `jakarta.el` present | Expected — Hibernate Validator uses Expression Language for message interpolation. Its absence produces a startup error about a missing EL implementation, which is loud and easy to fix. |

### Proof 1 — the three exception types, side by side

Add a temporary advice that reports the exception class rather than handling it nicely:

```java
@RestControllerAdvice
class DiagnosticAdvice {
    @ExceptionHandler(Exception.class)
    ResponseEntity<String> any(Exception ex) {
        return ResponseEntity.status(500)
                .body("EXCEPTION=" + ex.getClass().getName() + " :: " + ex.getMessage());
    }
}
```

```bash
# Path A — body validation
curl -i -s -X POST localhost:8080/lab/products \
  -H 'Content-Type: application/json' \
  -d '{"sku":"","name":"","priceMinor":-1}'

# Path B — controller parameter validation
curl -i -s "localhost:8080/lab/products?size=9999"

# Path C — service method validation
curl -i -s -X POST localhost:8080/lab/products/direct \
  -H 'Content-Type: application/json' \
  -d '{"sku":"  ","name":"x","priceMinor":-5}'
```

| What you see | What it means |
|---|---|
| Path A: `EXCEPTION=org.springframework.web.bind.MethodArgumentNotValidException` | Bean-graph validation fired in the argument resolver. Correct. |
| Path A: 200/201 with no exception | Validation did not run. Check Proof 0, then check that `@Valid` is on the parameter. |
| Path B: `EXCEPTION=org.springframework.web.method.annotation.HandlerMethodValidationException` | Framework 6.1+ controller method validation. Note the class name for your `@ControllerAdvice`. |
| Path B: `EXCEPTION=jakarta.validation.ConstraintViolationException` | The older shape. Your version routes controller parameter constraints through method validation, which means `@Validated` on the controller is required. Keep it. |
| Path B: `listing 9999` returned successfully | Neither mechanism ran. Add `@Validated` to the controller class and retry. **This is the settling result for the honest flag earlier in this document.** |
| Path C: `EXCEPTION=jakarta.validation.ConstraintViolationException` | Method validation on the service fired. `@Validated` is present and the bean is proxied. |
| Path C: `SAVE sku=   price=-5` printed and a 200 returned | **Trap 1, reproduced.** Remove/restore `@Validated` on `LabProductService` to toggle it deliberately, and note that nothing distinguishes the two cases except behaviour. |

**Remove the diagnostic advice afterwards.** It converts every exception into a 500 with an
internal class name in the body — useful for four minutes, dangerous for four months.

### Proof 2 — reveal that method validation is a proxy

```java
@Component
class ValidationProxyProbe implements ApplicationRunner {
    private final LabProductService service;
    ValidationProxyProbe(LabProductService service) { this.service = service; }

    @Override public void run(ApplicationArguments args) {
        System.out.println("VALIDATED-BEAN class = " + service.getClass().getName());
        System.out.println("VALIDATED-BEAN isAopProxy = " + AopUtils.isAopProxy(service));
    }
}
```

```bash
./mvnw spring-boot:run 2>&1 | grep VALIDATED-BEAN
```

| What you see | What it means |
|---|---|
| `class = com.orderflow.lab.LabProductService$$SpringCGLIB$$0`, `isAopProxy = true` | `MethodValidationPostProcessor` wrapped the bean. Constraints on its methods will run. |
| `class = com.orderflow.lab.LabProductService`, `isAopProxy = false` | **No proxy.** `@Validated` is missing, or the class is `final`, or the post-processor is absent. Every method constraint on this bean is dead metadata. |
| The app fails at startup with `AopConfigException: Could not generate CGLIB subclass` | The class is `final` (or is a record). Loud failure — the good kind. |

This is the same probe as Topic 40's Proof 1, applied to a different annotation. Once you
have it, "did the annotation get wired?" is a ten-second question for `@Transactional`,
`@Cacheable`, `@Async`, `@PreAuthorize` and `@Validated` alike.

### Proof 3 — validate programmatically, with no HTTP at all

The fastest way to understand what the engine actually produces:

```java
class RawValidatorProbe {
    public static void main(String[] args) {
        try (var factory = jakarta.validation.Validation.buildDefaultValidatorFactory()) {
            var validator = factory.getValidator();
            var req = new PlaceOrderRequest(
                    List.of(new PlaceOrderRequest.LineRequest("", -4)),
                    "gbp", null, null);
            validator.validate(req).forEach(v ->
                System.out.println("path=" + v.getPropertyPath()
                                 + " message=" + v.getMessage()));
        }
    }
}
```

| What you see | What it means |
|---|---|
| Lines with `path=lines[0].sku`, `path=lines[0].quantity` | Cascading is working. The indexed path is what you surface to the client. |
| Only `path=currency`, nothing for `lines[0].*` | **Trap 3.** No `@Valid` on the list element. Add `List<@Valid LineRequest>`. |
| Nothing printed at all | No violations found — check you passed genuinely invalid values, and check Proof 0. |
| Violations appear in a different order on each run | Expected and unspecified. This is why the advice in Example 2 sorts them. |

### Proof 4 — the group behaviour, proven both ways

```bash
# Create with a client-supplied id -> must be rejected
curl -i -s -X POST localhost:8080/api/products \
  -H 'Content-Type: application/json' \
  -d '{"id":41,"sku":"NEW-SKU","name":"New","priceMinor":100,"currency":"GBP"}'

# Update without a version -> must be rejected
curl -i -s -X PUT localhost:8080/api/products/41 \
  -H 'Content-Type: application/json' \
  -d '{"id":41,"sku":"NEW-SKU","name":"New","priceMinor":100,"currency":"GBP"}'

# Create with a blank name -> must be rejected even though @NotBlank is untagged
curl -i -s -X POST localhost:8080/api/products \
  -H 'Content-Type: application/json' \
  -d '{"sku":"NEW-SKU","name":"","priceMinor":100,"currency":"GBP"}'
```

| What you see | What it means |
|---|---|
| All three return 400 with a `violations` array naming the right field | Groups are configured correctly and `OnCreate`/`OnUpdate` extend `Default`. |
| Request 1 returns 201 | `@Null(groups = OnCreate.class)` is not being applied. Check you wrote `@Validated(OnCreate.class)` and not `@Valid`. |
| Request 3 returns 201 with a blank name | **The group footgun.** Your groups do not extend `Default`, so untagged constraints are skipped. Add `extends Default`. |
| Any of them returns 500 | The exception is not mapped. Check which class you got (Proof 1) and add the matching `@ExceptionHandler`. |

### Proof 5 — the unknown-property control

```bash
curl -i -s -X POST localhost:8080/api/products \
  -H 'Content-Type: application/json' \
  -d '{"sku":"NEW-SKU","name":"New","priceMinor":100,"currency":"GBP","internalCostMinor":1}'
```

| What you see | What it means |
|---|---|
| 400, mentioning an unrecognised field | `spring.jackson.deserialization.fail-on-unknown-properties=true` is active. This is Nest's `forbidNonWhitelisted`. |
| 201, and the extra field was ignored | Boot's default. Equivalent to `whitelist: true` without `forbidNonWhitelisted`. Acceptable *if* your DTOs are records — the field has nowhere to land. |
| 201, and the extra field affected the outcome | You are not using records, or the mapper is copying properties reflectively. That is a mass-assignment vulnerability. Fix the DTO shape, not the config. |

---

## Practice exercises

### 1 — easy: the validation status matrix

For the Example 1 lab, fill in this table **by prediction first**, then verify each row with
`curl -i`.

| Request | Predicted status | Predicted exception class | Actual |
|---|---|---|---|
| `POST /lab/products` with `{"sku":"AB","name":"x","priceMinor":1}` | | | |
| `POST /lab/products` with `{"sku":"AB-1","name":"","priceMinor":1}` | | | |
| `POST /lab/products` with `{"sku":"AB-1","name":"x"}` (priceMinor absent) | | | |
| `POST /lab/products` with `{"sku":"AB-1","name":"x","priceMinor":"free"}` | | | |
| `GET /lab/products?size=9999` | | | |
| `GET /lab/products?size=abc` | | | |
| `POST /lab/products/direct` with a blank sku, `@Validated` present | | | |
| `POST /lab/products/direct` with a blank sku, `@Validated` removed | | | |

**Done when:** you can explain why row 4 and row 6 fail *before* any constraint is
evaluated, and which component produced those two failures.

### 2 — medium: combines Topics 27, 40, 44 and 46

Build the `orderflow` `PlaceOrderRequest` from Example 2, then:

1. Write the `@NoDuplicateSkus` class-level constraint and its validator. Prove it produces a
   violation with the property path `lines`, not an object-level one.
2. Write the `ValidationProblemAdvice` from Example 2. Assert with `MockMvc` that a body with
   three separate problems returns **all three** violations in one 400 response, sorted.
3. Now break method validation deliberately: put `@Validated` on `OrderPlacementService`,
   add `@NotBlank` to a parameter, then call that method from a **sibling method in the same
   class**. Predict the result, then observe it. Explain it purely in terms of Topic 40 —
   say which object the call dispatches on and where the interceptor lives.
4. Make one method on that service `final`. Predict whether the failure is loud or silent.
   Verify. Then make the whole **class** `final` and predict again.
5. Explain why the record DTOs from Topic 27 make `whitelist: true` unnecessary, and what
   would change if they were mutable classes with setters.

**Done when:** you can state, without running anything, which of steps 3 and 4 fail at
startup and which fail silently at runtime.

### 3 — hard: production simulation — validate the whole `orderflow` API and prove the contract

**Scenario.** You are onboarding a partner integration. They will write their client once
against your error contract and will not adapt to changes. Your job is to make the contract
complete and stable.

**Requirements:**

1. Validate every write endpoint: `POST /api/orders`, `POST /api/products`,
   `PUT /api/products/{id}`, `POST /api/payments`, `POST /api/wallet/{id}/topup`.
2. Groups for product create vs update, with `@Null(groups = OnCreate.class)` on every
   server-assigned field.
3. A cross-field constraint on `PlaceOrderRequest`: no duplicate SKUs **and** the declared
   currency must match the wallet's currency. Decide — and write down your reasoning —
   whether the second rule belongs in a constraint or in the service. (There is a defensible
   answer either way. The wrong answer is not having thought about it.)
4. Every failure returns `ProblemDetail` with a stable `type` URI and a sorted `violations`
   array. No stack traces. No invalid values echoed.
5. A `@WebMvcTest` per endpoint asserting: (a) a fully valid body succeeds, (b) a body with
   three problems returns exactly three violations, (c) an inner-invalid nested object
   returns 400, (d) unknown properties behave as you intend.
6. **The load check.** Add validation to the Topic 65 load scenario. Run 10% of order
   placements with invalid bodies. Record p50/p95/p99 and error rate. Then answer: does
   validation appear in the latency profile at all? If you cannot tell, say so — that is the
   honest answer, and Topic 77 explains why a naive timing would not have told you either.
7. **The negative check that matters most.** Delete `spring-boot-starter-validation` from the
   `pom.xml` and re-run the full test suite. **Every validation test must fail.** If any
   passes, it was asserting something other than what you thought.

**Done when:** step 7 produces a wall of red, and you can name — from the failures alone —
which endpoints were genuinely covered.

---

## Interview questions

### Q1 — "What's the difference between `@Valid` and `@Validated`?"

**Mid-level answer:** "`@Valid` is the standard Jakarta annotation and `@Validated` is
Spring's version. `@Validated` also supports validation groups."

**Senior answer:** "They do different jobs, and conflating them is the top validation bug in
Spring codebases. `@Valid` is `jakarta.validation`, and it means 'cascade validation into
this object' — you put it on a `@RequestBody` parameter or on a nested field or container
element. `@Validated` is Spring's, and it does two things `@Valid` cannot: it selects
validation groups, and — the important one — on a **class** it tells
`MethodValidationPostProcessor` to wrap the bean in an AOP proxy so that constraints on
method *parameters* are enforced. That second job is why it matters: a service with
`@NotBlank` on a parameter and no `@Validated` on the class validates nothing at all, with no
warning. And once it is proxy-based, every Topic 40 caveat applies — self-invocation
bypasses it, a `final` method silently is not advised, and a `final` class fails at startup.
So the rule I use is: `@Valid` on parameters and nested fields, `@Validated` on the class
when I want method validation, and `@Validated(Group.class)` on a parameter when I need
groups."

**What separates them:** the mid answer is a documentation lookup. The senior answer names
the *mechanism* (`BeanPostProcessor` selecting beans for proxying), and connects it to the
self-invocation trap. The phrase "with no warning" is the part an interviewer is listening
for — it shows you have debugged it.

**Interviewer's follow-up:** *"You put `@Validated` on the class and it still does nothing.
What now?"* — Print `bean.getClass().getName()`. No `$$SpringCGLIB$$` means no proxy: the
class is `final`, or the bean was created outside the container, or something replaced it.
If it *is* a proxy, then the call is a self-invocation, or the method is `final`.

---

### Q2 — "How does Bean Validation compare to `class-validator`?"

**Mid-level answer:** "They're very similar — `@IsNotEmpty` becomes `@NotBlank`, `@Min` is
`@Min`. Java's is a spec with Hibernate Validator as the implementation."

**Senior answer:** "The annotations map almost one-to-one, and a good fraction of a
`class-validator` DTO — `@IsString`, `@IsInt`, `@IsEnum` — just disappears, because Java's
type system asserts it at compile time. Three real differences. First, Nest's
`ValidationPipe` has `whitelist`/`forbidNonWhitelisted`, which is a mass-assignment defence.
Bean Validation has no equivalent at all — in Spring the control is Jackson's
`FAIL_ON_UNKNOWN_PROPERTIES`, which defaults to false, and the structural answer is to make
DTOs records so extra properties have nowhere to land. Second, Nest has one mechanism; Spring
has two — bean-graph validation in the argument resolver, and proxy-based method validation
that needs `@Validated`. Second one fails silently. Third, `null` semantics: in Java every
constraint except `@NotNull`/`@NotEmpty`/`@NotBlank` passes on null, so `@Min(1)` alone does
not make a field required. That trips up everyone coming from `class-validator`, where
`@IsInt()` rejects `undefined`."

**What separates them:** the mid answer maps names. The senior answer names what does
**not** transfer — `whitelist`, the two mechanisms, and null semantics — because that is
where the bugs are. The `@IsString` observation shows they understand *why* the languages
differ rather than just that they do.

**Interviewer's follow-up:** *"Give me the null-semantics example that would actually bite."*
— `@Size(min = 8) String password` with no `@NotNull`: a null password passes validation.

---

### Q3 — "Show me how you'd handle create versus update with one DTO."

**Mid-level answer:** "Use validation groups — an `OnCreate` interface and an `OnUpdate`
interface, then `@Validated(OnCreate.class)` on the create endpoint."

**Senior answer:** "Groups, with two details that decide whether it works. First, my group
interfaces **extend `jakarta.validation.groups.Default`**. Without that,
`@Validated(OnCreate.class)` evaluates only the constraints tagged with `OnCreate`, so every
untagged constraint — the `@NotBlank` on the name, say — is silently skipped, and you can
create a product with a blank name. That is the single most common group bug and it is
invisible in review. Second, `@Valid` cannot select a group, so the parameter must be
`@Validated(OnCreate.class)`; leaving `@Valid` there means the whole grouping quietly does
nothing. Beyond the mechanics, the reason I care is security: `@Null(groups =
OnCreate.class)` on `id`, `version`, `tenantId` and `createdAt` is a declarative
mass-assignment defence. Without it, a partner posting `{"id": 41, ...}` to the create
endpoint can make your mapper produce a detached entity, and Hibernate issues an UPDATE
against product 41 — which belongs to someone else. That is a real cross-tenant write, and
it returns 201."

**What separates them:** the `extends Default` detail, the `@Valid`-cannot-select-groups
detail, and reframing groups as a security control rather than a DRY convenience. The mass
assignment story is a concrete incident, which is what makes it memorable.

**Interviewer's follow-up:** *"When would you use two DTOs instead?"* — When the shapes
genuinely diverge — different field sets, not just different requiredness. Groups keep one
class honest; they do not justify keeping one class when it has become two contracts wearing
a trench coat.

---

### Q4 — "You have `@NotNull` on a field and a request without that field still succeeds. Why?"

**Mid-level answer:** "Maybe validation isn't enabled, or `@Valid` is missing on the
parameter."

**Senior answer:** "Four candidates, and I'd check them in this order because that is
cheapest to most expensive. One: is the field a **primitive**? `@NotNull int quantity` can
never fail, because a primitive is never null — Jackson leaves it at `0` and the constraint
passes. Two: is `spring-boot-starter-validation` on the classpath? Since Boot 2.3 it is not
in `starter-web`, and without Hibernate Validator the auto-configuration backs off silently
and `@Valid` is a no-op. `mvn dependency:tree | grep hibernate-validator` settles it. Three:
is the import `jakarta.validation` and not `javax.validation`? A `javax` annotation on Boot
3 or 4 compiles and is ignored. Four: is the field **nested** without a cascade? If it is
inside a `List` and I did not write `List<@Valid LineRequest>`, its constraints are never
evaluated even though the outer object validated fine. All four are silent, which is why I
have a `@WebMvcTest` per DTO asserting a 400 on a known-bad body — that one test catches
every one of these."

**What separates them:** a *checklist ordered by cost*, and the observation that all four
failures are silent. The primitive one is the answer most people never reach.

**Interviewer's follow-up:** *"Which of those four would survive a code review?"* — All of
them. That is the point. The annotation is right there in the source in every case.

---

### Q5 — "Where should validation live: the controller, the service, or the entity?"

**Mid-level answer:** "The controller, so bad requests are rejected early. Maybe the entity
too as a backstop."

**Senior answer:** "All three, doing different jobs, and the reason is that they have very
different diagnostic quality. At the controller, `@Valid` on the DTO gives me the best
possible failure: it happens before my method body, it collects every violation at once, and
each one has a field path I can hand to a client. That is where the *format* contract lives.
In the domain, invariants belong in the constructor — an `Order` that cannot be constructed
in an invalid state is stronger than any annotation, and it protects the paths that never
touch HTTP, like a Kafka consumer or a batch job. At the database, `NOT NULL` and `CHECK`
constraints are absolute: they bind every writer including someone in `psql`. What I am
careful about is Hibernate's entity-level Bean Validation on pre-insert. It sounds like free
defence in depth, but the violation surfaces at **flush**, which inside a transaction is
usually at commit — so the exception is hundreds of lines from the cause, wrapped in a
`RollbackException`, with a stack trace naming Hibernate rather than my code, and the
transaction is already rollback-only. I keep it as a backstop and never rely on it for
diagnosis. The design rule is: validate at the boundary so bad data never reaches an entity,
and treat everything deeper as an assertion that the boundary worked."

**What separates them:** knowing *when* each layer fires and how good the resulting error
message is. The flush-time observation is a production-debugging scar and it is the part
that signals seniority.

**Interviewer's follow-up:** *"Your batch importer writes 50,000 rows and one violates a
constraint. What happens?"* — With entity validation, the whole flush fails at commit and
you lose all 50,000 with a message about one row. That is Topic 53's territory: you need
per-chunk transactions and validation before the entity is ever constructed.

---

## Mental model checkpoint

Answer these out loud, without looking above.

1. Two annotations, `@Valid` and `@Validated`. For each, name its package, whether it
   cascades, whether it supports groups, and what it enables on a class.

2. `@Min(1) Integer quantity` with no `@NotNull`. A request omits the field. What happens,
   and why is that the specified behaviour rather than a bug?

3. A `@Validated` service's method constraint does nothing. Give three distinct causes, each
   traceable to Topic 40.

4. `class-validator`'s `whitelist: true` has no Bean Validation equivalent. Name the two
   things you do instead, and say which one is structural rather than configured.

5. Your group interfaces do not extend `Default`. Describe precisely what stops being
   validated, and give the request that would then succeed and should not.

6. Why does a Bean Validation failure on a JPA entity produce a worse error message than the
   same constraint on a DTO? Answer in terms of *when* it fires.

7. Name the three exception types validation can throw in a Spring MVC application, what
   triggers each, and their default status codes.

---

## Quick reference card

### Dependency

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Not in `starter-web` since Boot 2.3. Confirm with
`mvn dependency:tree | grep hibernate-validator`.

### `@Valid` vs `@Validated`

| | `@Valid` | `@Validated` |
|---|---|---|
| package | `jakarta.validation` | `org.springframework.validation.annotation` |
| cascades | yes | no |
| groups | no | **yes** |
| enables method validation on a class | no | **yes** |

### Null semantics

```
@NotNull, @NotEmpty, @NotBlank   -> reject null
EVERY OTHER CONSTRAINT           -> null PASSES
```

### Where `@Valid` goes for cascading

```java
@Valid NestedThing thing;             // single object
List<@Valid Line> lines;              // each element
Map<String, @Valid Line> byId;        // each value
Optional<@Valid Coupon> coupon;       // the contained value
List<@NotBlank String> tags;          // constraint on the element itself
```

### Groups

```java
public interface OnCreate extends jakarta.validation.groups.Default {}   // EXTEND Default
@Null(groups = OnCreate.class) @NotNull(groups = OnUpdate.class) Long id;
create(@Validated(OnCreate.class) @RequestBody Req r)                    // NOT @Valid
```

### Custom constraint skeleton

```java
@Constraint(validatedBy = XValidator.class)
@Target({ ElementType.TYPE, ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface X {
    String message() default "...";
    Class<?>[] groups() default {};                    // required
    Class<? extends Payload>[] payload() default {};   // required
}

class XValidator implements ConstraintValidator<X, T> {
    public boolean isValid(T v, ConstraintValidatorContext ctx) {
        if (v == null) return true;                    // ALWAYS
        ...
    }
}
```

### Exception types

| Exception | From | Status |
|---|---|---|
| `MethodArgumentNotValidException` | `@Valid` on `@RequestBody` | 400 |
| `HandlerMethodValidationException` | controller method parameters (Framework 6.1+) | 400 |
| `ConstraintViolationException` | `@Validated` bean method validation | 500 unless mapped |

### Diagnostics

```bash
./mvnw dependency:tree | grep -i hibernate-validator
```

```java
System.out.println(bean.getClass().getName());     // $$SpringCGLIB$$ = @Validated worked
AopUtils.isAopProxy(bean)
validator.validate(obj).forEach(v -> ... v.getPropertyPath() ...)
```

```properties
spring.jackson.deserialization.fail-on-unknown-properties=true
server.error.include-binding-errors=always     # DEV ONLY
```

### Gotchas checklist

- `spring-boot-starter-validation` missing → everything silently passes.
- `javax.validation` import on Boot 3/4 → silently ignored.
- `@Validated` missing on a service class → method constraints are dead metadata.
- Self-call or `final` method on a `@Validated` bean → not validated, silently.
- `@NotNull` on a primitive → always passes; a missing field becomes `0`.
- Missing `@Valid` on a nested field or list element → nested constraints skipped.
- Groups not extending `Default` → untagged constraints skipped.
- `@Valid` used where a group was needed → grouping does nothing.
- Entity-level validation fires at **flush**, far from the cause.
- Never echo `getInvalidValue()` into a response.

---

## When would I use this at work?

**1. The first thing I write for any new endpoint.** A request DTO as a record with wrapper
types and constraints, plus one `@WebMvcTest` asserting a 400 on a known-bad body. It costs
about ten minutes per endpoint and it is the cheapest defence you will ever add — against
malformed partner requests, against heap-exhausting page sizes, and against mass assignment.
The test is the part people skip and the part that keeps working eighteen months later.

**2. Auditing an inherited API.** Given a service you did not write, three sweeps find most
of the holes: grep for `@NotNull` on primitives; find every DTO with a nested type or a
collection of a custom type and check for `@Valid` on the element; and print
`getClass().getName()` for every `@Validated` bean to confirm it is proxied. Each sweep is
mechanical and each one typically finds something. Then delete
`spring-boot-starter-validation` and run the suite — anything still green was never
testing validation.

**3. Designing an error contract a partner can integrate against once.** Validation is
where most 4xx responses come from, so the shape of a validation failure *is* your error
contract in practice. Sorted violations, a stable `type` URI, field paths that match the
request body's JSON structure, and no internal detail. This is a design decision made once,
enforced by a `@RestControllerAdvice` and a contract test (Topic 62), and it is the
difference between a partner integrating in a day and a partner filing tickets for a month.

---

## Connected topics

**Prerequisites:**

- **01 — Primitives and wrappers**: `@NotNull` on a primitive is always satisfied, and a
  missing JSON field silently becomes `0`. Trap 4 is Topic 01 with money attached.
- **02 — Nominal typing**: `@IsString`, `@IsInt`, `@IsEnum` disappear because the type
  system already asserts them. Declaring `UUID` beats writing a UUID `@Pattern`.
- **27 — Records**: record DTOs make mass assignment structurally impossible — no setters,
  a fixed canonical constructor. This is the Java answer to `whitelist: true`.
- **37 — Bean lifecycle**: `MethodValidationPostProcessor` is a `BeanPostProcessor`; the
  proxy is created at `postProcessAfterInitialization`.
- **40 — Proxying**: method validation is proxy-based, so self-invocation, `final` methods
  and `final` classes all apply exactly as they do to `@Transactional`. Trap 1's follow-ons
  are Topic 40 verbatim.
- **42 — Auto-configuration**: `ValidationAutoConfiguration` is `@ConditionalOnClass`, so a
  missing starter makes it back off **silently**. Trap 2 is a conditional back-off doing
  exactly what it was designed to do.
- **44 — REST controllers**: `@Valid` runs inside the argument resolver, after Jackson
  deserialization and before your method body. That ordering is why a controller may assume
  its input is valid.

**This unlocks / is used by:**

- **46 — Error handling and `ProblemDetail`**: the direct sequel. Validation is the largest
  single source of 4xx responses, so the violation-to-`ProblemDetail` mapping is the
  centrepiece of the error contract.
- **47 / 48 — Spring Data JPA and the persistence context**: entity-level constraints fire
  at flush, which is why boundary validation is not optional.
- **52 — Locking**: `@NotNull(groups = OnUpdate.class)` on the `version` field is what makes
  optimistic locking non-optional for API clients.
- **53 — Batching writes**: a bulk import with entity-level validation fails the entire
  flush on one bad row. Chunked transactions and pre-construction validation are the fix.
- **57 — Authorization**: validation answers "is this request well-formed"; authorization
  answers "is this caller allowed". Both run before your business logic, and confusing the
  two produces a 400 where a 403 belongs.
- **58 / 60 — JUnit 5 and test slices**: `@WebMvcTest` plus `MockMvc` is where the validation
  contract is asserted; a slice test is enough because nothing below the controller is
  involved.
- **62 — Contract testing**: the error body's shape is part of the published contract, which
  is why sorted violations and a stable `type` URI matter.
- **64 — Property-based testing**: constraints are stated invariants, which makes them
  natural properties — generate arbitrary DTOs and assert that a valid one always passes and
  an invalid one always produces a violation naming the right field.
- **116 — Idempotency**: the `Idempotency-Key` header's `@Size` constraint is validation;
  its uniqueness semantics are not, and confusing the two is a common design error.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0, Jakarta Validation
(Jakarta EE 11), Hibernate Validator via `spring-boot-starter-validation`. The
`jakarta.validation` package and the constraint set are specification-defined and stable
across Boot 3.x and 4.x; the `javax` → `jakarta` move happened in Boot 3.0 and is Topic 127.
The exact exception type thrown for controller **method-parameter** constraints changed in
Framework 6.1 and I have flagged my uncertainty about Framework 7.0 in the text — settle it
with Proof 1 on your own build rather than trusting this document, and pick your
`@ExceptionHandler` types from what you actually observe.*
