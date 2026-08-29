# 42 — Auto-configuration Mechanics: `@Conditional` and `AutoConfiguration.imports`

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: a custom `orderflow-pricing-starter` — you will *write* an auto-configuration, not just consume one

---

## Mechanical statement

> Boot reads the file
> `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
> from **every jar on the classpath**, registers each class listed in those files as a
> *candidate* configuration, and then evaluates the `@Conditional*` annotations on each
> candidate against the current classpath and the current bean registry.
> Candidates whose conditions pass contribute their `@Bean` methods. Candidates whose
> conditions fail are discarded and never instantiated.
>
> **It is a registry plus a predicate.** Nothing else.

Every word of that sentence is checkable. The registry is a text file you can `unzip -l`
and `cat`. The predicate is a set of annotations you can read. The result of running the
predicate is printed on demand by `--debug` as the **condition evaluation report**.

There is no step in that chain you have to take on faith. That is the whole point of this
topic, and it is why this topic sits *before* every other Boot topic in the curriculum.

---

## The bridge from what you know

### The honest verdict: **PARTIAL at best — there is no Nest analogue.**

NestJS is explicit by design. If a provider exists, some module declared it. You can
open `app.module.ts` and follow `imports` until you have the entire graph. Nothing
appears that you did not type.

```ts
// NestJS: the graph is a file you can read top to bottom
@Module({
  imports: [ConfigModule.forRoot(), TypeOrmModule.forRoot(dbConfig)],
  providers: [PricingService],
})
export class AppModule {}
```

Spring Boot does the opposite. You add one line to `pom.xml` and roughly a hundred beans
appear that you never declared.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

You now have an embedded Tomcat, a `DispatcherServlet`, a Jackson `ObjectMapper`, a
`ConversionService`, error handling, static resource handling, and more.

### The closest thing you *do* have

The nearest thing in the Node world is a `forRoot()` factory that inspects what is
installed and returns different providers accordingly, or a plugin auto-loader like
Fastify's `@fastify/autoload` scanning a directory. Both are "look at the environment,
decide what to register."

**But there are three differences that matter:**

| | NestJS | Spring Boot auto-configuration |
|---|---|---|
| Where the candidate list lives | in your source, as `imports:` | in a text file inside **someone else's jar** |
| What decides | you did, by typing it | a predicate (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, …) evaluated at startup |
| How you find out what happened | read the file | run with `--debug` and read the **condition evaluation report** |

That third row is the one people skip, and it is the difference between "Boot is magic"
and "Boot is a registry plus a predicate."

### The reframe you are here for

The word **magic** is a confession that you have not read the report. A senior Java
engineer never uses it about auto-configuration, because the mechanism is not only
knowable — it is *printed on request*, and it is **falsifiable**: you can make a
prediction ("this bean will not exist because `HikariDataSource` is not on the
classpath") and the report will tell you whether you were right.

### What transfers from Topics 35 and 36

You already have the load-bearing piece from **Topic 35 — the two-phase startup**:

1. **Phase 1:** bean *definitions* are registered — metadata only, nothing constructed.
2. **Phase 2:** singletons are instantiated.

Conditions are evaluated during phase 1, against the definitions registered so far.
**That phase split is the only reason `@ConditionalOnMissingBean` can work at all.**
If Spring built objects as it discovered them, "is there already a bean of type X?"
would be a question about construction order, not about metadata, and back-off would be
unimplementable.

From **Topic 36** you have component scanning: `@SpringBootApplication` scans its own
package and below. Auto-configuration is the *other* source of bean definitions, and it
is deliberately kept outside your scanned packages — which is why an auto-configuration
class in `com.orderflow` is a design mistake (Trap 5 below).

From **Topics 31–32** you have the classpath: Maven flattens dependencies to exactly one
version of each artifact on one flat classpath. That flat classpath is precisely what
`@ConditionalOnClass` interrogates.

---

## What is this?

**Auto-configuration** is Spring Boot's mechanism for contributing bean definitions from
a library, conditionally, without the application asking for them by name.

It has exactly four moving parts.

### 1. The registry file

Every jar that wants to contribute auto-configuration ships this file:

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Its content is a plain list of fully-qualified class names, one per line. Blank lines
are ignored; `#` starts a comment.

```
com.orderflow.pricing.autoconfigure.PricingAutoConfiguration
com.orderflow.pricing.autoconfigure.PricingMetricsAutoConfiguration
```

That is the entire registry. There is no scanning, no reflection over the jar, no
annotation index. Boot asks the class loader for **every** copy of that exact resource
path and concatenates the lines.

### 2. `@AutoConfiguration` on the listed class

```java
@AutoConfiguration
public class PricingAutoConfiguration { }
```

`@AutoConfiguration` is `@Configuration(proxyBeanMethods = false)` plus ordering
attributes. `proxyBeanMethods = false` matters: it means Spring does **not** create a
CGLIB subclass of the configuration class (Topic 40), so calling one `@Bean` method from
another returns a *new* object rather than the container's singleton. Auto-configurations
use it because it is faster and because a well-written one takes its collaborators as
method parameters instead of calling sibling `@Bean` methods.

### 3. The conditions

Annotations that answer a yes/no question about the current environment.

| Condition | Question it asks |
|---|---|
| `@ConditionalOnClass` | is this class loadable from the classpath? |
| `@ConditionalOnMissingClass` | is this class *absent*? |
| `@ConditionalOnBean` | is a bean of this type/name already defined? |
| `@ConditionalOnMissingBean` | is *no* bean of this type/name defined? |
| `@ConditionalOnProperty` | does this property have this value (or exist at all)? |
| `@ConditionalOnResource` | does this classpath resource exist? |
| `@ConditionalOnWebApplication` | is this a servlet / reactive web app? |
| `@ConditionalOnSingleCandidate` | is there exactly one bean of this type, or one marked `@Primary`? |
| `@ConditionalOnExpression` | does this SpEL expression evaluate true? (last resort) |

You can write your own by implementing `Condition` and using `@Conditional(MyCondition.class)`.

### 4. The report

`ConditionEvaluationReport` is a real bean in the context. It records, for every
candidate and every condition, the outcome and the human-readable reason. `--debug`
prints it. Actuator exposes it at `/actuator/conditions`. It is the ground truth.

### Auto-configuration vs a "starter"

These are different things and the words get used interchangeably, which is a mistake.

- A **starter** is a jar with (almost) no code — just a `pom.xml` full of dependencies.
  `spring-boot-starter-web` contains no classes of consequence. Its job is to put the
  right jars on the classpath in compatible versions.
- An **auto-configuration** is the code that reacts to what is on the classpath.

Boot's own artifacts split them: `spring-boot-starter-web` (dependencies) pulls in
`spring-boot-autoconfigure` (the conditional configuration classes). For your own
library, the convention is a two-module pair:

```
orderflow-pricing-spring-boot-autoconfigure   <- the code and the imports file
orderflow-pricing-spring-boot-starter         <- an empty pom depending on the above
```

**Naming rule:** never start a third-party starter's artifact id with `spring-boot-`.
That prefix is reserved for the Spring team. Yours is
`orderflow-pricing-spring-boot-starter`, not `spring-boot-starter-orderflow-pricing`.

---

## Why does it matter?

**1. Debugging stops being guesswork.**
"Why is there no `DataSource`?" has a printed answer. Without this topic you read blog
posts; with it you read the report and get the actual reason in seconds.

**2. You can ship a library that other teams add with one dependency line.**
`orderflow` will eventually be several services. The pricing rules should not be
copy-pasted into each. A starter is how a Java shop distributes shared behaviour, and
writing one is the difference between consuming the ecosystem and contributing to it.

**3. `@ConditionalOnMissingBean` is the contract that lets users win.**
Every good auto-configuration backs off when you define your own bean. Getting this
wrong in a library you ship means you silently overwrite your users' configuration —
a support nightmare with no error message.

**4. Startup time is a function of how many candidates you evaluate.**
Topic 122 measures `orderflow`'s startup. A large chunk of that is condition evaluation
and bean instantiation for auto-configurations you may not need. You cannot make an
informed decision about `spring.autoconfigure.exclude` without understanding what you
are excluding.

**5. Interviewers ask this to separate mid from senior.**
"How does auto-configuration work?" is asked in most senior Spring loops. The word
"magic" ends that line of questioning badly.

---

## Machine-level reality

This is the section that makes the mechanical statement precise. Read it slowly.

### Step 1 — how the imports file is discovered

`@SpringBootApplication` is a composed annotation. One of its parts is
`@EnableAutoConfiguration`, and that carries `@Import(AutoConfigurationImportSelector.class)`.

`AutoConfigurationImportSelector` is a **`DeferredImportSelector`**, not a plain
`ImportSelector`. That single word is load-bearing.

- A plain `ImportSelector` runs *while* the configuration class that imported it is being
  parsed.
- A `DeferredImportSelector` runs **after all regular `@Configuration` classes have been
  parsed** — including everything your component scan found.

**Consequence:** by the time auto-configuration candidates are considered, every bean
definition *you* declared is already registered. That is why `@ConditionalOnMissingBean`
inside an auto-configuration reliably sees your beans. It is also why the same annotation
on *your own* `@Configuration` class is unreliable — your class is parsed in the same
undeferred pass as everything else you wrote, so "already registered" depends on parsing
order you do not control.

> **Rule, stated once and never violated:**
> `@ConditionalOnMissingBean` belongs in auto-configuration classes only.
> In application code it is a race against your own parsing order.

The selector loads the candidate list via Spring's `ImportCandidates` mechanism, which
does the moral equivalent of:

```java
ClassLoader cl = ...;
Enumeration<URL> urls = cl.getResources(
    "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports");
// read every URL, split on newlines, strip comments and blanks
```

Two things follow from `getResources` (plural):

1. **Every jar's copy is read.** They do not override each other. Twenty jars each
   contributing five lines gives you a hundred candidates.
2. **A duplicate class name across jars is harmless** — the list is de-duplicated.

The result is cached per class loader for the lifetime of the application, so the file
I/O happens once, not once per candidate.

### Step 2 — the fast pre-filter (why 150 candidates do not cost 150 class loads)

Naively, evaluating `@ConditionalOnClass` means attempting to load the named class and
catching `ClassNotFoundException`. Doing that for every condition on every candidate
would be slow, and throwing/catching hundreds of exceptions at startup is measurably bad.

Boot avoids it with a build-time index. An annotation processor
(`spring-boot-autoconfigure-processor`) generates
`META-INF/spring-autoconfigure-metadata.properties` inside the jar, recording each
auto-configuration's `ConditionalOnClass` / `ConditionalOnBean` / `ConditionalOnWebApplication`
values as plain strings.

At startup, `AutoConfigurationImportFilter` implementations (`OnClassCondition`,
`OnBeanCondition`, `OnWebApplicationCondition`) read that metadata and eliminate
candidates in bulk, **before** the full `@Conditional` machinery runs. `OnClassCondition`
even splits the work across two threads when the candidate list is large enough.

> **Genuine uncertainty, flagged:** the exact artifact coordinates of the annotation
> processor changed with Boot 4's modularisation. Do not copy a `groupId:artifactId` from
> me. Find it by searching your resolved dependency tree
> (`./mvnw dependency:tree | grep -i autoconfigure`) or the Boot reference docs for your
> exact version, and let the Boot BOM supply the version.

**Practical consequence for you:** if you write an auto-configuration and *do not* add
that processor, everything still works — you just pay full condition evaluation instead
of the indexed fast path. It is an optimisation, not a requirement. Add it when your
starter is used by more than a couple of services.

### Step 3 — when conditions are evaluated relative to instantiation

Ordering, precisely:

```
1. Refresh begins.
2. ConfigurationClassPostProcessor (a BeanFactoryPostProcessor) parses @Configuration classes.
   -> your @Configuration classes, your component scan: definitions registered.
3. Deferred import selectors run.
   -> auto-configuration candidate list loaded from all imports files
   -> fast pre-filter via spring-autoconfigure-metadata.properties
   -> candidates SORTED (see step 4)
   -> for each surviving candidate, in order:
        evaluate its class-level @Conditional*
        if it passes: parse it, register its @Bean definitions
                      (each @Bean method's own @Conditional* is evaluated here too)
        if it fails:  record the reason in ConditionEvaluationReport, discard
4. All BeanFactoryPostProcessors finish. Definitions are now final.
5. BeanPostProcessors registered.
6. Singleton pre-instantiation: objects are actually constructed.   <- Topic 37 lifecycle
```

**Everything about conditions happens at step 3. Nothing is constructed until step 6.**

This is why a condition can never ask a question about a bean's *state* — only about
whether a *definition* exists. `@ConditionalOnBean(DataSource.class)` means "a bean
definition of that type is registered", not "a working database connection exists".

It also explains a class of confusing behaviour: `@ConditionalOnBean` on a bean whose
type is only known from a `@Bean` method's *return type* can be misjudged when that
return type is a generic or an interface implemented later. Boot's own advice is to
prefer `@ConditionalOnMissingBean` (asking about absence, which is cheap and safe) over
`@ConditionalOnBean` (asking about presence, which is order-dependent).

### Step 4 — ordering, and why it is deterministic

Auto-configurations are sorted before evaluation. The sort has three passes, in this
order:

1. **Alphabetically by fully-qualified class name.** This exists purely so the result is
   stable across machines and class-loader implementations. Never rely on it.
2. **By `@AutoConfigureOrder`.** A plain integer, lower runs first. Same semantics as
   `@Order` (Topic 41) but a separate annotation, because auto-configuration ordering is
   a different concern from advice ordering.
3. **By `@AutoConfigureBefore` / `@AutoConfigureAfter` dependencies.** A topological sort
   over the graph those annotations describe.

`@AutoConfiguration` lets you write the last one inline:

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
public class PricingAutoConfiguration { }
```

There are `beforeName` / `afterName` string variants for referencing a class you do not
want to hard-depend on at compile time.

### Step 5 — why `@ConditionalOnMissingBean` is order-sensitive

Put the previous two steps together.

`@ConditionalOnMissingBean` asks: *"as of right now, is there a definition of this type?"*
"Right now" means "after every candidate sorted before me has already been processed."

So:

```java
// Candidate A, sorted first
@AutoConfiguration
class ACfg {
    @Bean @ConditionalOnMissingBean
    PriceRounder rounder() { return new HalfUpRounder(); }
}

// Candidate B, sorted second
@AutoConfiguration
class BCfg {
    @Bean @ConditionalOnMissingBean
    PriceRounder rounder() { return new BankersRounder(); }
}
```

A wins. Always. B's condition sees A's definition and backs off. Reverse the sort and B
wins. **The annotation alone does not decide the winner; the sort does.** If you want a
specific outcome you must state it with `@AutoConfigureBefore`/`@AutoConfigureAfter`.

And the reason it works reliably *against user beans* is step 1: user configuration is
parsed in an earlier, non-deferred pass, so every user bean is unconditionally "already
there" from an auto-configuration's point of view.

### Step 6 — how exclusion works

Three ways to remove a candidate:

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

```
# an application.yml under a specific profile, same key
```

Exclusions are applied to the candidate list *before* conditions run, and they appear in
their own section of the report. Excluding a class that is not a candidate is an error by
default (Boot fails fast rather than letting you think you excluded something).

### Step 7 — `@ImportAutoConfiguration` and test slices (forward reference)

Test slices such as `@WebMvcTest` and `@DataJpaTest` (Topic 60) do **not** load all
auto-configurations. They use `@ImportAutoConfiguration`, which reads a *different* set
of imports files keyed by the slice annotation, so only the auto-configurations relevant
to that layer are considered. That is exactly why a slice test starts in a fraction of
the time of `@SpringBootTest` — and why a bean your app has at runtime can be absent in a
slice test. Same registry-plus-predicate machinery, a smaller registry.

---

## `[BOOT 3.x DELTA]` — and the 2.x history you will meet in real repos

| Boot line | Registration file | Notes |
|---|---|---|
| 2.0 – 2.6 | `META-INF/spring.factories`, key `org.springframework.boot.autoconfigure.EnableAutoConfiguration` | Also used for dozens of other extension points. |
| 2.7 | **both** — `.imports` supported, `spring.factories` still read, deprecated | Migration window. |
| 3.x | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` **only** | `spring.factories` for auto-configuration is removed. |
| 4.x | same as 3.x | Unchanged. |

**Why the change:** `spring.factories` is a properties file, so a long list of class names
became one enormous line continued with backslashes — unreadable, merge-conflict-prone,
and it mixed auto-configuration with unrelated extension points. The `.imports` file is
one class per line and is used for nothing else.

**The symptom of getting this wrong on a 3.x/4.x app:** nothing. Absolute silence. Your
auto-configuration class is never read, so it is never a candidate, so it never appears in
the report — not even under "Negative matches". See Trap 1.

**Other Boot 4 deltas relevant here:**

- **Modularisation.** Boot 4 splits into many smaller jars. An auto-configuration class you
  remember from 3.x may have moved package or artifact. Do not memorise coordinates;
  resolve them. `./mvnw dependency:tree` is the authority on your machine.
- **Jackson 3 is standard, Jackson 2 deprecated.** Which means the `@ConditionalOnClass`
  guards inside Boot's own JSON auto-configuration now reference Jackson 3 types. If you
  write a starter that touches JSON, guard on the class you actually use and let the
  condition do its job — do not assume which Jackson is present.
  *(Flagged: verify the exact Jackson 3 package names against the Boot version you resolve.
  I am not going to quote them from memory.)*
- Never write a Spring artifact version number in your `pom.xml`. Inherit
  `spring-boot-starter-parent` or import `spring-boot-dependencies` as a BOM
  (Topic 32) and omit `<version>`.

---

## Example 1 — minimal

The smallest auto-configuration that demonstrates all four moving parts.

**The bean it contributes:**

```java
package com.orderflow.pricing;

public interface PriceRounder {
    long round(long amountInMinorUnits);
}
```

```java
package com.orderflow.pricing;

public class HalfUpRounder implements PriceRounder {
    @Override
    public long round(long amountInMinorUnits) {
        return amountInMinorUnits;   // already minor units; nothing to round
    }
}
```

**The auto-configuration:**

```java
package com.orderflow.pricing.autoconfigure;

import com.orderflow.pricing.HalfUpRounder;
import com.orderflow.pricing.PriceRounder;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(PriceRounder.class)
public class PricingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public PriceRounder priceRounder() {
        return new HalfUpRounder();
    }
}
```

**The registry file** — path matters character for character:

```
src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

containing exactly one line:

```
com.orderflow.pricing.autoconfigure.PricingAutoConfiguration
```

**What each piece does:**

| Line | Effect |
|---|---|
| the `.imports` file | makes the class a *candidate*. Without it, nothing happens, silently. |
| `@AutoConfiguration` | marks it as `@Configuration(proxyBeanMethods=false)` and enables the ordering attributes. |
| `@ConditionalOnClass(PriceRounder.class)` | the whole class is skipped if the pricing library is not on the classpath. |
| `@ConditionalOnMissingBean` | if the *application* declares its own `PriceRounder`, this method backs off. |

**The behaviour you should be able to predict without running it:**

- App with the starter, no `PriceRounder` bean → a `HalfUpRounder`.
- App with the starter and its own `@Bean PriceRounder` → the app's bean; the report shows
  the auto-configured one under **Negative matches** with the reason naming
  `OnBeanCondition`.
- App without the starter jar → the class is not a candidate at all; nothing in the
  report; no `PriceRounder`.

---

## Example 2 — production scenario: `orderflow-pricing-starter`

### The problem, with real constraints

`orderflow` is splitting into three deployables: `orderflow-api` (the REST service),
`orderflow-batch` (nightly reprice and settlement), and `orderflow-admin`. All three
compute prices. Pricing rules are a compliance surface — tax rounding, promotional
stacking order, currency minor-unit handling — and a divergence between services is a
finance incident, not a bug.

Concrete numbers to design against:

- `orderflow-api` serves **~1,800 requests/second at peak**, of which roughly 70% are
  catalogue reads that call the pricing engine.
- The catalogue holds **~120,000 products**, each with 0–4 active promotions.
- Price computation must stay under **2 ms p99** because it is inside the request path
  and the endpoint's own budget is 40 ms p99 (Topic 65's baseline).
- `orderflow-batch` runs the same engine over **~1.1M orders** overnight, with no HTTP
  and no web context at all.

So the starter must work in a web app and a plain app, must not force a web dependency,
must be overridable by any consumer, and must not add measurable startup cost to services
that do not use it.

### Module layout

```
orderflow-pricing/
  orderflow-pricing-core/                       <- domain: no Spring on the compile path
  orderflow-pricing-spring-boot-autoconfigure/  <- @AutoConfiguration + the .imports file
  orderflow-pricing-spring-boot-starter/        <- pom-only; depends on the two above
```

`orderflow-pricing-core` deliberately has no Spring dependency. That keeps the pricing
rules unit-testable with plain JUnit (Topic 58) and usable from a non-Spring context.

### The core types

```java
package com.orderflow.pricing;

import java.time.Duration;
import java.util.List;

public interface PricingEngine {
    /** All amounts are in minor units (pence). Never double — Topic 01. */
    long priceFor(long listPriceMinor, List<Promotion> promotions);
}
```

```java
package com.orderflow.pricing;

public enum StackingPolicy { BEST_SINGLE, SEQUENTIAL, ADDITIVE }
```

### The typed configuration

Forward-referencing Topic 43, which covers this properly:

```java
package com.orderflow.pricing.autoconfigure;

import com.orderflow.pricing.StackingPolicy;
import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.bind.DefaultValue;
import org.springframework.validation.annotation.Validated;

import java.time.Duration;

@Validated
@ConfigurationProperties(prefix = "orderflow.pricing")
public record PricingProperties(

        @DefaultValue("true")
        boolean enabled,

        @NotNull
        @DefaultValue("BEST_SINGLE")
        StackingPolicy stacking,

        /** Max promotions considered per line. Guards a pathological catalogue row. */
        @Min(1) @Max(16)
        @DefaultValue("4")
        int maxPromotionsPerLine,

        @DefaultValue("PT2S")
        Duration ruleRefreshInterval
) {}
```

Note `jakarta.validation.constraints`, **never** `javax.validation`. That move happened in
Boot 3 and is Topic 127's headline migration question.

### The auto-configuration

```java
package com.orderflow.pricing.autoconfigure;

import com.orderflow.pricing.PricingEngine;
import com.orderflow.pricing.StackingPricingEngine;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(PricingEngine.class)
@ConditionalOnProperty(prefix = "orderflow.pricing", name = "enabled",
                       havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(PricingProperties.class)
public class PricingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public PricingEngine pricingEngine(PricingProperties props) {
        return new StackingPricingEngine(props.stacking(), props.maxPromotionsPerLine());
    }

    /**
     * Only wires metrics if the consumer already has a MeterRegistry.
     * orderflow-batch has no Actuator, so this quietly does not apply there.
     */
    @Bean
    @ConditionalOnClass(MeterRegistry.class)
    @ConditionalOnBean(MeterRegistry.class)
    @ConditionalOnMissingBean
    public PricingMetrics pricingMetrics(MeterRegistry registry) {
        return new PricingMetrics(registry);
    }
}
```

**Read the design decisions, not just the code:**

| Decision | Why, in terms of the constraints above |
|---|---|
| `@ConditionalOnClass(PricingEngine.class)` at class level | if a service depends on the autoconfigure jar transitively but not on `-core`, the whole class is skipped instead of blowing up with `NoClassDefFoundError` during parsing. |
| `matchIfMissing = true` on the property | pricing is on by default; a consumer opts *out*. If it were opt-in, every one of the three services would need the same line, and someone would forget. |
| `@ConditionalOnMissingBean` on `pricingEngine` | `orderflow-admin` needs a rule-preview engine that records its decisions. It declares its own `@Bean PricingEngine` and this one backs off. No fork, no flag. |
| both `@ConditionalOnClass` **and** `@ConditionalOnBean` on the metrics bean | `@ConditionalOnClass` protects against Micrometer being absent entirely (`orderflow-batch`). `@ConditionalOnBean` protects against Micrometer being present but no registry configured. Class-presence and bean-presence are different questions and you need both. |
| `Duration ruleRefreshInterval` typed as `Duration`, not `long` | binding `PT2S` or `2s` is handled by the converter, and a typo fails at startup rather than at the first refresh. Topic 43. |
| no `@ConditionalOnWebApplication` | pricing is not web-specific. `orderflow-batch` must get the same engine. |

### The starter pom

```xml
<project>
  <artifactId>orderflow-pricing-spring-boot-starter</artifactId>
  <packaging>pom</packaging>

  <dependencies>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-pricing-core</artifactId>
    </dependency>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-pricing-spring-boot-autoconfigure</artifactId>
    </dependency>
  </dependencies>
</project>
```

`<packaging>pom</packaging>` — the starter contains no code. Its entire job is
dependency aggregation, which is exactly what Topic 31's scope and Topic 32's BOM
knowledge is for. Versions of the two internal modules come from the parent's
`dependencyManagement`; Spring's versions come from the Boot BOM. **No Spring version
number is typed anywhere in this repository.**

### What the consumer writes

```xml
<dependency>
  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-pricing-spring-boot-starter</artifactId>
</dependency>
```

```yaml
orderflow:
  pricing:
    stacking: SEQUENTIAL
    max-promotions-per-line: 6
```

One dependency, two properties. That is the payoff, and now you know exactly what it
costs to build.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the auto-configuration that never fires (the silent one)

**Wrong:** you write `PricingAutoConfiguration`, annotate it perfectly, and forget the
`.imports` file. Or you create it at the wrong path — `META-INF/spring.factories`
(the 2.x location), or `META-INF/spring/AutoConfiguration.imports` (missing the long
prefix), or under `src/main/java` instead of `src/main/resources`.

**Exact symptom:** **nothing happens at all.**
- No error, no warning, no log line.
- The application starts normally.
- `NoSuchBeanDefinitionException` for `PricingEngine` *only* at the point something
  injects it — and if nothing injects it, you get no signal whatsoever.
- Run with `--debug`. Your class name **does not appear anywhere in the report** — not
  under Positive matches, not under Negative matches, not under Exclusions.

That last bullet is the diagnostic. Learn it as a rule:

> **Absent from the report entirely = it was never a candidate = registry problem.**
> **Present under Negative matches = it was a candidate = predicate problem.**

Those are two completely different bugs and the report tells them apart in one look.

**Root cause:** Boot only ever asks the class loader for that one exact resource path.
There is no scanning fallback, no "did you mean", no validation that a class annotated
`@AutoConfiguration` is registered anywhere.

**Fix:**

```bash
# from the autoconfigure module
ls -l src/main/resources/META-INF/spring/
# expect exactly:
# org.springframework.boot.autoconfigure.AutoConfiguration.imports

# then prove it survived packaging
./mvnw -q clean package
unzip -l target/orderflow-pricing-spring-boot-autoconfigure-*.jar | grep -i imports
```

If the second command prints nothing, your resource is not in the jar and no amount of
annotation correctness will save you.

---

### Trap 2 — the auto-configuration that overrides the user's bean

**Wrong:**

```java
@AutoConfiguration
public class PricingAutoConfiguration {
    @Bean                                   // <-- no @ConditionalOnMissingBean
    public PricingEngine pricingEngine(PricingProperties props) { ... }
}
```

`orderflow-admin` declares its own audited `PricingEngine`.

**Exact symptom:** depends on the Boot version and is nasty either way.

| What you see | What it means |
|---|---|
| `BeanDefinitionOverrideException` at startup, naming `pricingEngine` | bean-definition overriding is disabled (the Boot 2.1+ default). Loud, and honestly the good outcome. |
| starts fine, but `orderflow-admin` has no audit trail on prices | someone set `spring.main.allow-bean-definition-overriding=true` to "fix" the exception. Now one definition silently replaces the other, and which one wins depends on registration order. |
| starts fine, audit works | you got lucky with ordering. It will break on an unrelated dependency bump. |

The middle row is the dangerous one, and it is common: the exception is annoying, someone
finds the property on Stack Overflow, and the loud failure becomes a silent one.

**Root cause:** without `@ConditionalOnMissingBean`, your library and your user are both
unconditionally claiming the same bean type.

**Fix:** `@ConditionalOnMissingBean` on every `@Bean` in every auto-configuration you
ship, without exception. Treat its absence as a review blocker. And never set
`allow-bean-definition-overriding=true` — it converts a compile-time-ish failure into a
runtime coin flip.

---

### Trap 3 — `@ConditionalOnMissingBean` in your own `@Configuration`

**Wrong:**

```java
package com.orderflow.config;   // your app, not a starter

@Configuration
public class PricingOverrides {
    @Bean
    @ConditionalOnMissingBean       // <-- looks reasonable, is not
    public PriceRounder rounder() { return new BankersRounder(); }
}
```

**Exact symptom:** works on your machine, fails in CI, or vice versa. The bean is present
in some builds and absent in others with no code change. Adding an unrelated
`@Configuration` class flips it. The condition evaluation report shows the outcome
changing between runs.

**Root cause:** your `@Configuration` classes are parsed in the *undeferred* pass, in an
order derived from component-scan discovery order, which is filesystem- and
classloader-dependent. "Is there already a bean of this type?" therefore has a
nondeterministic answer among your own classes.

Auto-configurations do not have this problem because (a) they run in the deferred pass,
after all of yours, and (b) they are explicitly sorted (Machine-level reality, step 4).

**Fix:** in application code, express intent directly instead of asking a question about
timing:

- `@Primary` to declare a winner among several candidates.
- `@Qualifier` at the injection point to name the one you want (Topic 39).
- `@ConditionalOnProperty` if the choice is genuinely configuration-driven — that
  question has a deterministic answer at any point in startup.

---

### Trap 4 — `@ConditionalOnClass` referencing a class you cannot load

**Wrong:**

```java
@AutoConfiguration
public class PricingMetricsAutoConfiguration {

    @Bean
    public PricingMetrics metrics(MeterRegistry registry) {   // Micrometer type in a signature
        return new PricingMetrics(registry);
    }
}
```

with `micrometer-core` declared `<optional>true</optional>` and absent from
`orderflow-batch`.

**Exact symptom:**

```
java.lang.NoClassDefFoundError: io/micrometer/core/instrument/MeterRegistry
```

thrown during context refresh, not at the point of use. The stack trace runs through
Spring's configuration parsing (`ConfigurationClassParser` / `ConfigurationClassBeanDefinitionReader`),
which makes it look like a Spring bug rather than a missing dependency.

**Root cause:** to read `@Bean` metadata Spring must resolve the method's parameter and
return types. `@ConditionalOnClass` works because the *condition* is evaluated from
annotation metadata as a **string**, without loading the class — but only if the risky
type does not appear in the enclosing class's own signatures before the condition runs.

**Fix:** move the optional-dependency beans into a nested `@Configuration` class guarded
by `@ConditionalOnClass`. The outer class stays loadable; the inner class is only
introspected once its condition passes.

```java
@AutoConfiguration
public class PricingMetricsAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(MeterRegistry.class)
    static class MicrometerBindings {
        @Bean
        @ConditionalOnBean(MeterRegistry.class)
        @ConditionalOnMissingBean
        PricingMetrics metrics(MeterRegistry registry) {
            return new PricingMetrics(registry);
        }
    }
}
```

This nested-class pattern is everywhere in Boot's own source. Now you know what it is for.

---

### Trap 5 — putting the auto-configuration inside your scanned packages

**Wrong:** you put `PricingAutoConfiguration` in `com.orderflow.pricing` while the app's
`@SpringBootApplication` lives in `com.orderflow` — so component scanning finds it *and*
the `.imports` file registers it.

**Exact symptom:** the class is applied **twice as two different things**, and it stops
behaving conditionally. Concretely:
- Its beans exist even when you set `orderflow.pricing.enabled=false`, because the
  scanned copy is a plain `@Configuration` processed in the undeferred pass.
- `@ConditionalOnMissingBean` inside it now sometimes wins against user beans, because in
  the undeferred pass it may be parsed *before* the user's configuration class.
- On some Boot versions you get a startup failure about the class being registered twice.

**Root cause:** `@AutoConfiguration` is meta-annotated with `@Configuration`, so a
component scan that reaches it will register it like any other configuration class,
completely bypassing the deferred-import ordering that makes conditions meaningful.

**Fix:** auto-configuration classes must live in a package that the consuming
application's component scan **cannot** reach. In practice: a separate module, under a
package that is not a sub-package of any consumer's application class. That is why the
`orderflow-pricing-spring-boot-autoconfigure` module is a separate artifact rather than a
package inside the service.

If you must keep them in one repository for now, use a package outside the app root — for
example app at `com.orderflow.api` and auto-config at `com.orderflow.pricing.autoconfigure`,
with the app's scan explicitly limited:

```java
@SpringBootApplication(scanBasePackages = "com.orderflow.api")
```

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM and no running `orderflow`, so I
will not print output and call it captured. What I will give you is the exact command, the
exact thing to look for, and a table mapping each possible result to its meaning.

### Setup

```bash
java --version          # expect 21 or 25
./mvnw --version
```

### Proof 1 — read the condition evaluation report

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug
```

or, against a built jar:

```bash
java -jar target/orderflow-api.jar --debug
```

> `--debug` here is a **Boot application argument**, not a JVM flag and not
> `mvn -X`. It sets `debug=true`, which switches on the
> `ConditionEvaluationReportLogger`. `mvn -X` gives you Maven's own debug output and
> tells you nothing about the context.

**The report has exactly four sections.** This is an illustration of the *shape* — it is
me drawing the format, not output I captured:

```
============================
CONDITION EVALUATION REPORT
============================

Positive matches:
-----------------

   DispatcherServletAutoConfiguration matched:
      - @ConditionalOnClass found required class '...DispatcherServlet' (OnClassCondition)
      - found 'session' scope (OnWebApplicationCondition)

Negative matches:
-----------------

   RedisAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class '...RedisOperations' (OnClassCondition)

Exclusions:
-----------

    org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

Unconditional classes:
----------------------

    org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration
```

**How to read each section:**

| Section | What it contains | What you use it for |
|---|---|---|
| **Positive matches** | candidates whose conditions all passed, with the reason each condition passed | "why *does* this bean exist?" Also: if you see something here you did not expect, this is where an unwanted dependency reveals itself. |
| **Negative matches** | candidates that were evaluated and rejected, with the **first failing condition** and its reason | "why is my bean missing?" This is the section you will read 90% of the time. |
| **Exclusions** | candidates removed by `spring.autoconfigure.exclude` or `@SpringBootApplication(exclude=)` before conditions ran | confirms an exclusion actually took effect and matched a real class name. |
| **Unconditional classes** | auto-configurations with no conditions at all — they always apply | usually infrastructure. If one of yours is here, you forgot a condition. |

**Now the diagnostic table for your own class:**

| What you see | What it means | Next action |
|---|---|---|
| `PricingAutoConfiguration matched:` under **Positive matches** | registry ✓, predicate ✓. The bean exists. | If behaviour is still wrong, the bug is in the bean, not the wiring. |
| `PricingAutoConfiguration:` under **Negative matches** with `@ConditionalOnClass did not find required class` | the candidate was found; the classpath does not have what it needs. | Check `./mvnw dependency:tree` for the named class's artifact. It is a Topic 32 resolution problem. |
| Under **Negative matches** with `@ConditionalOnMissingBean found beans of type` | working exactly as designed — a user bean won. | If you *wanted* your bean, remove theirs or add `@Primary` to yours. |
| Under **Negative matches** with `@ConditionalOnProperty ... did not find property` | the property is absent and `matchIfMissing` is false. | Set the property, or reconsider whether opt-out is the better default. |
| Under **Exclusions** | someone excluded it deliberately. | `grep -r "spring.autoconfigure.exclude" src/` and check every profile-specific file. |
| **Nowhere in the report** | it was never a candidate. **Registry problem, not predicate problem.** | Go to Proof 3. Check the `.imports` file path and that it is inside the jar. |

The last row is the single highest-value line in this document. Internalise it.

---

### Proof 2 — see the same report through Actuator

`--debug` dumps the whole thing once at startup and floods your terminal. Actuator gives
you the same data as queryable JSON.

`application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: conditions,configprops,env,beans,health
```

> Actuator endpoints are **not exposed by default** beyond `health`. If you skip this
> block, `/actuator/conditions` returns 404 and it is not a bug.
> `management.endpoints.web.exposure.include: "*"` is fine locally and must never reach
> production unauthenticated — Topic 121.

```bash
./mvnw spring-boot:run
```

then, in another terminal:

```bash
curl -s localhost:8080/actuator/conditions | jq '.contexts.application.positiveMatches | keys'
curl -s localhost:8080/actuator/conditions | jq '.contexts.application.negativeMatches | keys'

# the one you actually want, narrowed to your class:
curl -s localhost:8080/actuator/conditions \
  | jq '.contexts.application | {
        pos: (.positiveMatches | keys | map(select(test("Pricing")))),
        neg: (.negativeMatches | to_entries | map(select(.key | test("Pricing"))))
      }'
```

| What you see | What it means |
|---|---|
| `404` from `/actuator/conditions` | the endpoint is not exposed. Add it to `include`. Not a wiring problem. |
| `pos` contains `PricingAutoConfiguration`, `neg` is `[]` | it applied. |
| `pos` is `[]`, `neg` has an entry with a `notMatched` array | read `notMatched[0].message` — that string is the exact reason. |
| both empty | not a candidate. Registry problem. Proof 3. |

The Actuator route is better than `--debug` once you are past the first day, because you
can filter, and because you can hit it against a running container.

---

### Proof 3 — prove the registry file exists where Boot will look

This is the check that resolves Trap 1, and it takes ten seconds.

```bash
./mvnw -q clean package -DskipTests

# your own starter's autoconfigure jar
unzip -l orderflow-pricing-spring-boot-autoconfigure/target/*.jar \
  | grep -i "AutoConfiguration.imports"

# and read it
unzip -p orderflow-pricing-spring-boot-autoconfigure/target/*.jar \
  META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

| What you see | What it means |
|---|---|
| a line ending `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` | the file is packaged correctly. |
| grep prints nothing | the file is not in the jar. Check it is under `src/main/resources`, not `src/main/java`, and that the filename has no trailing space or `.txt`. |
| the file is there but `unzip -p` prints a class name with a typo | Boot silently ignores unresolvable class names in some versions and fails loudly in others. Either way, fix the typo — copy the FQN from your IDE, do not type it. |
| the file is there and correct, but the class still does not appear in the report | you are running against a stale jar. `./mvnw clean install` the starter, then rebuild the consumer. |

Now do the same to somebody else's jar to prove this is universal:

```bash
# find any Boot autoconfigure jar in your local repo
find ~/.m2/repository/org/springframework/boot -name "*autoconfigure*.jar" | head -3

# list the imports file inside it and count the candidates
unzip -p <that-jar> \
  META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports \
  | grep -v '^#' | grep -v '^$' | wc -l
```

**How to read it:** the number you get is how many candidate configurations that one jar
contributes. Seeing a plain text file with a hundred-odd class names in it is the moment
"magic" stops being a usable word. There is nothing else in there.

---

### Proof 4 — flip a condition and watch the report change

Make a prediction *before* each command. That is what makes this falsifiable rather than
observational.

```bash
# A. baseline
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug 2>&1 | grep -A5 "PricingAutoConfiguration"

# B. turn the property off
./mvnw spring-boot:run \
  -Dspring-boot.run.arguments="--debug --orderflow.pricing.enabled=false" 2>&1 \
  | grep -A5 "PricingAutoConfiguration"

# C. exclude it outright
./mvnw spring-boot:run \
  -Dspring-boot.run.arguments="--debug --spring.autoconfigure.exclude=com.orderflow.pricing.autoconfigure.PricingAutoConfiguration" 2>&1 \
  | grep -A5 "PricingAutoConfiguration"
```

| Run | Predicted section | Predicted reason text mentions |
|---|---|---|
| A | Positive matches | `OnClassCondition`, and `@ConditionalOnProperty` matching because `matchIfMissing` |
| B | Negative matches | `OnPropertyCondition` — property found with a different value |
| C | Exclusions | (no reason; exclusions are unconditional removals) |

**If your observed section differs from your prediction, your model is wrong and the
report is right.** Write down which one differed. That is the highest-value note you will
take in this topic.

---

### Proof 5 — prove `@ConditionalOnMissingBean` backs off

Add to `orderflow-api` only:

```java
package com.orderflow.api.config;

@Configuration
public class LocalPricingOverride {
    @Bean
    public PricingEngine pricingEngine() {
        return new RecordingPricingEngine();     // your own implementation
    }
}
```

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug 2>&1 \
  | grep -B2 -A6 "PricingAutoConfiguration#pricingEngine"
```

| What you see | What it means |
|---|---|
| `Did not match: @ConditionalOnMissingBean found beans of type 'com.orderflow.pricing.PricingEngine': pricingEngine` | correct back-off. Your library is well-behaved. |
| `BeanDefinitionOverrideException` at startup | you forgot `@ConditionalOnMissingBean` on the `@Bean` method. Trap 2. |
| starts fine but `RecordingPricingEngine` is not the injected instance | overriding is enabled somewhere. Find and remove `spring.main.allow-bean-definition-overriding`. |

Then confirm which object you actually got:

```java
@Bean
ApplicationRunner whichEngine(PricingEngine engine) {
    return args -> System.out.println("ENGINE = " + engine.getClass().getName());
}
```

Printing the runtime class is the same reflex Topic 40 taught you for spotting
`$$SpringCGLIB$$`. Do not reason about which bean you got. Print it.

---

### Proof 6 — measure the cost of the candidate list

```bash
# how many candidates does your app actually evaluate?
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug 2>&1 \
  | sed -n '/Positive matches:/,/Negative matches:/p' | grep -c "matched:"

./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug 2>&1 \
  | sed -n '/Negative matches:/,/Exclusions:/p' | grep -cE "^   [A-Z]"
```

**How to read it:** positive matches are configurations that were parsed and whose beans
were defined. Negative matches cost condition evaluation only — cheap, but not free.

This is the input to Topic 122's startup work: excluding auto-configurations you do not
use removes bean instantiation, which is usually a larger cost than the condition check
itself. Do not exclude anything on this evidence alone — measure the startup split first.
The count is context, not a conclusion.

---

## Practice exercises

### 1 — Easy: read the registry, then read the report

**Part A.** Without running your app, find three auto-configuration `.imports` files in
`~/.m2/repository` (Boot's own, plus any two others — a driver, a client library).
For each, report: how many candidates it lists, and pick one class name that surprises you.

**Part B.** Start `orderflow-api` with `--debug`. Find that surprising class in the
report. Which section is it in, and what is the exact reason string?

**Part C.** Answer in writing: for one auto-configuration in **Negative matches**, what is
the smallest change to `pom.xml` or `application.yml` that would move it to **Positive
matches**? Do not make the change — just state it and justify it from the reason string.

---

### 2 — Medium: build the starter (combines Topics 31, 32, 35, 36, 39, 41)

Build `orderflow-pricing-spring-boot-starter` as a real three-module Maven build.

Requirements:

1. **Topic 31:** a parent pom with a `<modules>` list. Explain in a comment why the
   reactor build order is not the order you wrote the modules in.
2. **Topic 32:** the starter module is `<packaging>pom</packaging>` and declares no
   version for any Spring artifact. Prove it: `./mvnw dependency:tree` and show where
   each Spring version came from.
3. **Topic 36:** put the auto-configuration in a package the consuming app's component
   scan cannot reach. Then *deliberately* move it into the scanned package, observe what
   changes, and write down the symptom. Move it back.
4. **Topic 39:** the `PricingEngine` bean must take `PricingProperties` by constructor
   injection, with `final` fields. State the two reasons this beats field injection here
   specifically (not in general).
5. **Topic 41:** add a second auto-configuration that contributes a timing aspect around
   `PricingEngine.priceFor`, guarded by `@ConditionalOnClass` on the AOP types and
   `@ConditionalOnProperty(name = "orderflow.pricing.metrics.enabled")`. Order it with
   `@AutoConfiguration(after = PricingAutoConfiguration.class)` and explain what breaks if
   you omit that.

**Deliverable:** the `unzip -p` output of your `.imports` file, and the four report
sections filtered to your two classes, for both `enabled=true` and `enabled=false`.

---

### 3 — Hard: production simulation — make back-off deterministic under a real constraint

`orderflow` now has three consumers of the starter, and a conflict.

- `orderflow-api` needs the default `StackingPricingEngine`.
- `orderflow-batch` needs a **bulk** engine that pre-resolves all promotions once per run,
  because at **1.1M orders per night** with a 90-minute window it cannot afford per-line
  promotion lookups. It has no web context and no Micrometer.
- `orderflow-admin` needs a **recording** engine that captures every rule decision for the
  compliance team, and it must win over anything the starter provides.

A second starter, `orderflow-bulk-pricing-spring-boot-starter`, ships the bulk engine and
is on `orderflow-batch`'s classpath alongside the base starter.

**Part A.** Both starters define a `PricingEngine` with `@ConditionalOnMissingBean`.
Predict, from the sorting rules in Machine-level reality step 4, which one wins in
`orderflow-batch` — and state your reasoning *before* you run anything. Then run it and
check the report. Were you right?

**Part B.** Make the outcome deterministic and correct, using `@AutoConfigureBefore` /
`@AutoConfigureAfter` / `@AutoConfigureOrder`. Explain why you chose the one you chose,
and why relying on alphabetical class-name ordering — even though it *is* deterministic —
would be a defect.

**Part C.** `orderflow-admin` must always get the recording engine. Implement that, then
prove with the report that both starters backed off. State which mechanism did the work:
`@ConditionalOnMissingBean`, or the undeferred/deferred parsing split, or both.

**Part D.** Now the honest engineering question. Given three consumers with three
different needs, argue the case **against** solving this with conditions at all. What
would `@Qualifier` plus explicit `@Bean` declarations in each service cost, and what would
it buy? State the condition under which your answer flips — for example, at what number
of consuming services does the starter stop being worth its complexity? There is no
single right answer here; there is a right *shape* of answer, and it names a threshold.

**Part E.** Measure. Record `orderflow-batch`'s startup time with both starters, then with
`spring.autoconfigure.exclude` removing everything it demonstrably does not use (from your
Negative/Positive match counts). Report the delta honestly. If it is under 100 ms, say so
and say that the complexity is not justified by startup alone.

---

## Interview questions

### Q1 — "How does Spring Boot's auto-configuration actually work?"

This is asked in nearly every senior Spring loop. Treat it as the set piece it is.

**Mid-level answer:** "Spring Boot looks at what's on your classpath and configures things
automatically. If it sees a database driver it sets up a `DataSource`. It's convention over
configuration — it just works so you don't have to write XML."

**Senior answer:** "It's a registry plus a predicate — there's nothing magic in it.

The registry: every jar can ship
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`, a
plain list of class names. `@EnableAutoConfiguration` imports
`AutoConfigurationImportSelector`, which reads every copy of that file from the class
loader and builds a candidate list.

The predicate: each candidate is annotated `@AutoConfiguration` plus conditions —
`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty` and so on.
Candidates that pass get their `@Bean` definitions registered; the rest are discarded.

Two mechanics make it dependable. First, the selector is a `DeferredImportSelector`, so it
runs *after* all user configuration is parsed — that's why `@ConditionalOnMissingBean`
reliably sees my beans and backs off. Second, candidates are sorted — alphabetically for
stability, then by `@AutoConfigureOrder`, then topologically by
`@AutoConfigureBefore`/`@AutoConfigureAfter` — so back-off between two auto-configurations
is deterministic rather than accidental.

And it's all falsifiable. Run with `--debug`, or hit `/actuator/conditions`, and you get
the condition evaluation report: Positive matches, Negative matches, Exclusions,
Unconditional classes, each with the exact reason. When a bean is missing I read that
report, and the first thing I check is whether the class appears in the report at all —
absent means it was never a candidate, which is a registry problem, versus present under
Negative matches, which is a predicate problem. Those are completely different bugs."

**What separates them:** the mid answer describes the *effect*; the senior answer describes
the *mechanism*, names the file and the selector, explains why deferred import and sorting
exist, and — critically — offers the falsification path. The refusal to say "magic" is the
signal.

**Interviewer's follow-up:** "Where does that file live in Boot 2.x?" They are checking
whether you have actually migrated something. Answer: `META-INF/spring.factories` under
the `EnableAutoConfiguration` key; 2.7 supported both; 3.x removed the old one. Then
volunteer *why*: `spring.factories` was one giant backslash-continued properties line
shared with unrelated extension points.

---

### Q2 — "You added a starter, but the bean it should provide isn't there. Walk me through your debugging."

**Mid-level answer:** "I'd check the dependency is in the pom, then check I've got
`@ComponentScan` covering the right package, then add `@ComponentScan` for the library's
package if not. Then maybe search for the class name."

**Senior answer:** "First, I would not add a `@ComponentScan` for a library package —
that's a workaround that bypasses the conditions and creates the double-registration
problem, and it hides the real cause.

I'd run with `--debug` and search the condition evaluation report for the class name.

If it's **not in the report at all**, it was never a candidate. That's a registry problem:
either the jar isn't on the classpath — `mvn dependency:tree` — or its `.imports` file is
missing or misnamed, which I'd confirm with `unzip -l` on the jar. That second case is
common with home-grown starters and it fails completely silently.

If it's under **Negative matches**, the report gives me the first failing condition and its
reason verbatim. `@ConditionalOnClass did not find` means a missing transitive dependency —
back to `dependency:tree`. `@ConditionalOnProperty did not find property` means a config
key, and I'd check `/actuator/env` for where the value is actually coming from, because
profile files and env vars beat what I'm staring at in `application.yml`.
`@ConditionalOnMissingBean found beans of type` means it worked correctly and one of my
own beans won.

If it's under **Exclusions**, someone excluded it — `grep` for
`spring.autoconfigure.exclude` across every profile-specific file, and check the
`@SpringBootApplication` annotation."

**What separates them:** the mid answer reaches for `@ComponentScan`, which is the
canonical wrong fix. The senior answer has a decision tree keyed on report sections, knows
that absence from the report is a categorically different bug from a negative match, and
names a specific command per branch.

**Interviewer's follow-up:** "The report says `@ConditionalOnClass did not find` a class
you're certain is a dependency. Now what?" Answer: it's a resolution problem, not a
Spring problem. Maven is flat and nearest-wins (Topic 32), so a transitive exclusion or a
`provided` scope somewhere removed it. `mvn dependency:tree -Dincludes=<groupId>` and
`mvn dependency:tree -Dverbose` show what was omitted and why.

---

### Q3 — "Why does `@ConditionalOnMissingBean` work in an auto-configuration but not in my own `@Configuration`?"

**Mid-level answer:** "It should work anywhere — it just checks whether the bean exists.
Maybe there's an ordering issue."

**Senior answer:** "'Ordering issue' is the right instinct but it's the whole answer, so
let me be precise.

The condition asks 'is a definition of this type registered *right now*'. So the answer
depends entirely on what has been processed before it.

`AutoConfigurationImportSelector` is a `DeferredImportSelector`. Deferred selectors run
after all regular `@Configuration` classes are parsed — including everything a component
scan found. So from an auto-configuration's viewpoint, every user bean is unconditionally
already registered. That's the guarantee that makes 'the user always wins' true.

Within my own configuration classes there's no such guarantee. They're all parsed in the
same undeferred pass, in an order that comes out of component-scan discovery — which is
filesystem- and classloader-dependent. So `@ConditionalOnMissingBean` between two of my
own classes is a race, and the nasty part is that it's a *stable* race: it produces the
same answer every run on my laptop and possibly a different one in CI, so it looks like
an environment bug.

In application code I express the intent directly instead: `@Primary` for a winner,
`@Qualifier` at the injection point, or `@ConditionalOnProperty` if the choice is genuinely
configuration-driven — because that question has a deterministic answer at any point in
startup."

**What separates them:** naming `DeferredImportSelector` and explaining the *two-pass*
consequence, plus recognising that a nondeterministic-in-principle bug presents as an
environment-specific one in practice.

**Interviewer's follow-up:** "So how do two auto-configurations that both want to provide
the same bean type resolve it?" Answer: the sort — alphabetical, then
`@AutoConfigureOrder`, then `@AutoConfigureBefore`/`@AutoConfigureAfter`. First one
processed wins; the second backs off. Relying on the alphabetical tier is a defect even
though it is deterministic, because a rename silently flips behaviour.

---

### Q4 — "What's the difference between a starter and an auto-configuration?"

**Mid-level answer:** "A starter is a dependency that brings in auto-configuration. They're
basically the same thing — `spring-boot-starter-web` gives you web stuff."

**Senior answer:** "They're separate artifacts with separate jobs, and Boot's own modules
keep them separate for a reason.

A starter is dependency aggregation — usually a pom-only module with no code. Its job is
to put a compatible set of jars on the classpath. `spring-boot-starter-web` contains
essentially no classes.

An auto-configuration is the code that reacts to what ended up on the classpath — the
`@AutoConfiguration` classes, their conditions, and the `.imports` file that registers
them. In Boot's case that lives in `spring-boot-autoconfigure`.

The split matters because they have different consumers. I might want the auto-config
without the starter's opinionated dependency set, or the dependencies without the
auto-config — and someone excluding one of my starter's transitive dependencies shouldn't
break the auto-configuration's ability to *back off* correctly, which is exactly what
`@ConditionalOnClass` is for.

For our own libraries we ship the pair:
`orderflow-pricing-spring-boot-autoconfigure` and
`orderflow-pricing-spring-boot-starter`. And the naming convention matters — a third-party
starter is `<name>-spring-boot-starter`, never `spring-boot-starter-<name>`, because that
prefix is reserved for the Spring team."

**What separates them:** knowing the artifacts are genuinely separate in Boot's own build,
articulating why the split has practical value, and knowing the naming convention without
being asked.

**Interviewer's follow-up:** "Where does the auto-configuration class need to live
relative to the application's packages?" Answer: outside anything the app's component scan
reaches. If a scan finds it, it is registered as a plain `@Configuration` in the undeferred
pass, which bypasses the ordering and makes its conditions unreliable — and on some
versions it fails outright as a duplicate registration.

---

### Q5 — "Boot starts too slowly. Would you exclude auto-configurations?"

**Mid-level answer:** "Yes — I'd exclude the ones we don't use with
`spring.autoconfigure.exclude`, and enable lazy initialisation. That should cut startup
time."

**Senior answer:** "Maybe, but not first, and not on this evidence.

Startup splits into JVM init, class loading, context refresh — which is where condition
evaluation and bean definition live — and bean instantiation. Those have completely
different fixes, so I'd measure the split before touching anything.

Condition *evaluation* is already cheap: Boot ships a build-time index in
`spring-autoconfigure-metadata.properties` that lets the class and bean filters eliminate
most candidates in bulk before the full condition machinery runs. So excluding a
configuration mainly saves you the *instantiation* of its beans, not the check.

If instantiation is the cost, the honest wins are usually: exclude auto-configurations
whose beans genuinely get created and are genuinely unused; check whether a connection
pool is warming up synchronously and blocking refresh, which is a config problem and not an
auto-config problem; and consider lazy initialisation, with the caveat that it moves
failures from startup to first request, which is worse for a Kubernetes readiness probe —
you pass readiness and then 500 on real traffic.

If class loading is the cost, that's AppCDS or the JDK 25 AOT cache territory, and
excluding auto-configurations does very little.

The failure mode I'd warn against: excluding a list someone copied from a blog post. Every
exclusion is a thing that will not be there when a future dependency needs it, and the
failure is a `NoSuchBeanDefinitionException` at an unrelated time."

**What separates them:** measuring before acting, knowing the metadata index exists so
"conditions are slow" is not assumed, naming the readiness-probe consequence of lazy init,
and treating each exclusion as a permanent liability.

**Interviewer's follow-up:** "How would you get the split?" They want to hear about
timing the phases rather than a total — Boot's startup metrics
(`ApplicationStartup` / `BufferingApplicationStartup` and the `startup` Actuator endpoint),
or a profiler. Topic 122 does this properly.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `AutoConfigurationImportSelector` is a `DeferredImportSelector` rather than a plain
   `ImportSelector`. Describe, concretely, what would break if it were plain — name a
   specific annotation that would stop working and explain the failure it would produce.

2. Auto-configuration candidates are sorted **alphabetically first**, before
   `@AutoConfigureOrder` and before the before/after graph. Why would Boot bother with a
   tier that no correct code should ever depend on? What does it buy that leaving the
   order unspecified would not?

3. `@ConditionalOnMissingBean` asks about *definitions*, not *instances*, because it runs
   before anything is constructed. Give a question you might *want* to ask in a condition
   that this design makes impossible — and say what you would do instead.

4. Boot's own docs steer you toward `@ConditionalOnMissingBean` and away from
   `@ConditionalOnBean`. Both look symmetric. Explain why absence is a safer question than
   presence, in terms of the sort order.

5. A colleague proposes replacing your starter with a plain `@Configuration` class in a
   shared library, imported explicitly with `@Import(PricingConfig.class)` in each service.
   That is closer to how NestJS works. Make the strongest case **for** their proposal, then
   the strongest case against. Which would you actually ship for three services? For thirty?

6. The condition evaluation report is generated whether or not you ask for it —
   `ConditionEvaluationReport` is a bean in every context. What does that cost, and why do
   you think the Spring team decided that cost was worth paying unconditionally?

7. Suppose Boot removed `@ConditionalOnClass` entirely and auto-configurations were
   expected to just fail with `NoClassDefFoundError` if their dependencies were absent.
   Describe the resulting developer experience precisely. Now argue that this would
   actually be *better* in at least one respect.

---

## Quick reference card

### The registry file

```
src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

One fully-qualified class name per line. `#` comments. Blank lines ignored.
`[BOOT 3.x DELTA]` — on Boot 2.6 and earlier this was `META-INF/spring.factories` under
the key `org.springframework.boot.autoconfigure.EnableAutoConfiguration`.

### The conditions

```java
@ConditionalOnClass(PricingEngine.class)              // classpath has it
@ConditionalOnMissingClass("com.orderflow.PricingEngine")     // classpath lacks it (String form)
@ConditionalOnBean(PricingEngine.class)               // a definition exists — order-sensitive
@ConditionalOnMissingBean                   // no definition exists — the back-off idiom
@ConditionalOnSingleCandidate(PricingEngine.class)    // exactly one, or one @Primary
@ConditionalOnProperty(prefix = "orderflow.pricing", name = "enabled",
                       havingValue = "true", matchIfMissing = true)
@ConditionalOnResource(resources = "classpath:pricing-rules.json")
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnNotWebApplication
@ConditionalOnExpression("${a:false} and ${b:false}")   // last resort
@Conditional(MyCondition.class)             // your own, implements Condition
```

### Ordering

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@AutoConfiguration(before = SomeOtherAutoConfiguration.class)
@AutoConfiguration(afterName = "com.other.ThingAutoConfiguration")
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
```

Sort order: alphabetical → `@AutoConfigureOrder` → before/after topological sort.

### Exclusion

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```
```yaml
spring.autoconfigure.exclude: org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### The report

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug
java -jar app.jar --debug
curl -s localhost:8080/actuator/conditions | jq .    # needs exposure config
```

Sections: **Positive matches** · **Negative matches** · **Exclusions** ·
**Unconditional classes**.

### The one-look diagnosis

| Where your class appears | Category of bug |
|---|---|
| nowhere in the report | **registry** — the `.imports` file or the classpath |
| Negative matches | **predicate** — read the reason string, it is exact |
| Exclusions | someone excluded it deliberately |
| Positive matches | wiring is fine; the bug is in the bean |

### Starter naming and layout

```
orderflow-pricing-core                          plain Java, no Spring
orderflow-pricing-spring-boot-autoconfigure     @AutoConfiguration + .imports
orderflow-pricing-spring-boot-starter           <packaging>pom</packaging>, no code
```

Never `spring-boot-starter-<yours>`. That prefix belongs to the Spring team.

### Checklist for shipping an auto-configuration

- [ ] `.imports` file exists at the exact path, under `src/main/resources`
- [ ] verified inside the built jar with `unzip -l`
- [ ] the class lives outside any consumer's component-scan reach
- [ ] `@ConditionalOnMissingBean` on **every** `@Bean` method
- [ ] `@ConditionalOnClass` guarding any optional dependency, with risky types confined to
      a nested `@Configuration`
- [ ] a `@ConfigurationProperties` type with defaults and validation, not scattered `@Value`
- [ ] ordering stated with `@AutoConfigureBefore`/`After` if another auto-config could conflict
- [ ] no Spring version numbers in any pom — the BOM supplies them
- [ ] the report checked in all three states: default, property off, user bean present

---

## When would I use this at work?

**1. A bean is missing and three people are guessing in the incident channel.**
You run `--debug`, grep the report, and post the exact reason string within a minute.
"`@ConditionalOnClass did not find required class io.lettuce.core.RedisClient` — the
Redis client was excluded transitively by the bump in PR 4412." That reframes a
speculative discussion into a resolved one, and it is the single most common way this
topic pays off.

**2. Shipping shared behaviour across several services without copy-paste.**
Pricing rules, the correlation-ID filter, the standard `ProblemDetail` error contract
(Topic 46), the metrics conventions (Topic 118) — all of these want to be one starter that
each service adds with one dependency line. Writing that starter well, with proper
back-off, is a mid-to-senior boundary in most Java shops. A team that has one person who
can do this stops having six divergent copies of the same filter.

**3. Reviewing a dependency upgrade before it reaches production.**
Diff the condition evaluation report before and after a Boot minor upgrade. New entries
under Positive matches are new beans you did not ask for; entries that moved from Positive
to Negative are behaviour you just lost, silently. This catches the class of upgrade bug
where nothing errors and something quietly stops happening — which is the worst kind to
find in production.

---

## Connected topics

**Prerequisites — you should have these before this topic makes sense:**

- **31 — Maven fundamentals**: modules, `<packaging>pom</packaging>`, the reactor, and why
  `provided` vs `runtime` scope changes what is on the classpath a condition inspects.
- **32 — Dependency resolution and BOMs**: `@ConditionalOnClass` asks a question about the
  flat, nearest-wins classpath. When a condition fails unexpectedly, the answer is almost
  always in `mvn dependency:tree`. Also why you never type a Spring version number.
- **35 — `ApplicationContext` two-phase startup**: definitions registered, *then* singletons
  instantiated. **This is why conditions can be evaluated at all.** If you are hazy on this,
  re-read it before continuing.
- **36 — Bean definition and component scanning**: what gets scanned, and therefore why an
  auto-configuration class must live outside it.
- **37 — Bean lifecycle**: the instantiation phase that conditions run *before*.
- **40 — Proxying**: why `@AutoConfiguration` sets `proxyBeanMethods = false`, and what you
  give up by doing so.
- **41 — AOP and ordering**: `@Order` for advice vs `@AutoConfigureOrder` for
  configurations — same idea, different axis.

**This unlocks:**

- **43 — Configuration and property precedence**: `@ConditionalOnProperty` is only as
  predictable as your understanding of *which* source supplied the property. The next
  topic, and they are read as a pair.
- **44 — `@RestController`**: every bean in the MVC request path
  (`DispatcherServlet`, the `HandlerMapping`s, the `HttpMessageConverter`s) arrives via an
  auto-configuration you can now read.
- **45 — Bean Validation**: the `Validator` bean is auto-configured, guarded by
  `@ConditionalOnClass` on the Jakarta Validation API. If validation "isn't working", the
  report tells you whether the validator exists.
- **46 — Error handling**: `ErrorMvcAutoConfiguration` provides the default whitelabel
  error page. Knowing where it comes from is how you replace it cleanly rather than
  fighting it.
- **47 — Spring Data JPA**: repository proxies, `DataSourceAutoConfiguration`,
  `HibernateJpaAutoConfiguration` — a long chain of conditions you can now trace.
- **60 — Test slices and context caching**: `@WebMvcTest` and friends use
  `@ImportAutoConfiguration` against slice-specific registry files. Same machinery,
  smaller candidate list.
- **61 — Testcontainers**: `@ServiceConnection` is auto-configuration reacting to a
  container bean.
- **121 — Actuator**: `/actuator/conditions`, `/actuator/beans`, `/actuator/configprops`,
  and the exposure rules you must set before any of them answer.
- **122 — Docker and startup time**: the measured consequence of how many candidates apply,
  and where excluding them does and does not help.
- **127 — Migration planning**: `spring.factories` → `.imports` is the concrete migration
  step in every Boot 2 → 3 upgrade, and Boot 4's modularisation moves artifacts again.

---

*Java baseline 21, runtime JDK 25. Spring Boot 4.1 / Framework 7.0 / Jakarta EE 11.
The registry-plus-predicate model in this document has been stable since Boot 1.0; only
the location of the registry file changed (2.7 / 3.0). Every artifact coordinate in your
build should come from the Boot BOM — I have deliberately not quoted version numbers, and
where an artifact may have moved under Boot 4's modularisation I have said so rather than
guessed.*
