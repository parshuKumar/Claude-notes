# 20 — JPMS modules and `jlink`

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21, 25
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Imagine a shared office with one enormous unlabelled drawer.

Everyone throws their stuff in it. Everyone can reach into it and take anything.
There is no rule about what belongs to whom. If two people put in a folder with the
same name, whoever reached in first wins and nobody is told.

That drawer is the **classpath**. It has been Java's model since 1996.

Now imagine replacing it with labelled boxes. Each box has a card on the front saying:

- **my name** — so people can ask for me specifically
- **what I need** — the other boxes I cannot work without
- **what you may take** — the shelves inside me that are open to visitors
- **what I keep private** — everything else, and a guard actually stops you reaching it

That is a **module**. The guard is real: it stops you at build time *and* at run time,
including if you try to sneak in with a crowbar (reflection).

And `jlink` is the last step: once everything is in labelled boxes, you can build a
custom office containing **only** the boxes this particular job needs, and throw the
rest of the building away. That office is smaller, starts faster, and has less surface
to attack.

The catch — and this is why you have probably never seen a modular Java application —
is that the labelled-box scheme only works if **everyone** uses it. One person still
using the big drawer drags the whole floor back to the old model.

---

## The bridge from what you know

### ES modules versus JPMS — **PARTIAL analogue**

You already think in modules. That part transfers.

```ts
// pricing/index.ts
export { calculatePrice } from "./api/calculator";   // public
// ./internal/rounding.ts is not exported -> not part of the public API
import { Decimal } from "decimal.js";                // an explicit dependency
```

```java
// module-info.java
module com.orderflow.pricing {
    requires com.orderflow.money;                    // an explicit dependency
    exports com.orderflow.pricing.api;               // public
    // com.orderflow.pricing.internal is NOT exported -> genuinely unreachable
}
```

The shapes rhyme. Both declare dependencies explicitly and both distinguish public from
internal. But four differences matter, and each one changes how you use it.

| ES modules | JPMS | The difference that matters |
|---|---|---|
| The unit is a **file** | The unit is a **package** | You export `com.orderflow.pricing.api`, not `Calculator.java`. Package structure becomes your API boundary, which changes how you lay out code. |
| Non-exported symbols are merely *not imported*; a bundler or a determined caller can still reach them | Non-exported packages are **inaccessible at compile time and enforced by the JVM at run time**, including via reflection | This is the actual feature. It is the first time in Java's history that `public` has not meant "public to everyone with a classpath". |
| Cycles are allowed (with caveats) | Cycles between modules are a **hard error** at resolution | You cannot have two modules that require each other. This forces a layered design and is a real migration blocker on old codebases. |
| npm resolves versions; two versions can coexist | JPMS has **no version selection at all** | The module system deliberately does not do versions. That stays with Maven/Gradle (Topic 32). A `requires` names a module, never a version. |
| Adopting ESM in one package is possible; the ecosystem interoperates (messily) | Modularising is **all-or-nothing across the dependency graph** | This is why adoption stalled. See below. |

### Why adoption stalled — say this plainly

To compile your module, **every** module you `requires` must itself be a module. A
plain jar placed on the module path becomes an **automatic module**: it gets a name
derived from its `Automatic-Module-Name` manifest entry, or — failing that — **from its
filename**. A filename-derived module name is not a stable identifier. Change the jar's
version and the derived name may change. Your `requires` clause breaks for a reason
that has nothing to do with your code.

And `jlink`, the payoff, **refuses to work with automatic modules at all**. So the
reward for modularising is withheld until your entire transitive dependency graph is
modular — which, for a Spring Boot service with 120 jars, it is not.

The result: the JDK itself is fully modular, some libraries ship `module-info.class`,
and almost no applications are modular.

**But you are still affected by JPMS every single day**, because the JDK's own modules
are strongly encapsulated whether or not your code is modular. That is where
`--add-opens` comes from, and it is why this topic matters even though you will probably
never write a `module-info.java` for `orderflow`.

---

## What is this?

JPMS — the Java Platform Module System, introduced in Java 9 (JEP 261, "Project
Jigsaw") — adds a layer above packages.

It exists for **two** stated goals, and "smaller jars" is not one of them:

**1. Reliable configuration.** The classpath's failure mode is `NoClassDefFoundError`
at runtime, on the first code path that happens to need the missing class — possibly
weeks after deploy. The module system resolves the whole module graph at **startup** and
fails immediately if something is missing, duplicated or cyclic. You find out in one
second, not on Black Friday.

**2. Strong encapsulation.** Before JPMS, `public` meant "callable by anyone who can put
this jar on their classpath". There was no way to have a class that is public across
your own packages but invisible outside your library. Package-private (**Topic 03**) is
the only tool, and it forces everything into one package. Modules fix this: a package is
accessible outside the module only if the module `exports` it.

A third thing came along for the ride, and it is the one you will actually use:

**3. `jlink`.** Because the JDK is now a set of modules with declared dependencies, you
can compute the minimal set of JDK modules a program needs and build a custom runtime
image containing only those. A full JDK is roughly 300 MB; a `jlink` image for a typical
service is commonly in the 50–100 MB range, and can go much lower. This matters for
container images and for startup.

### The four kinds of module

You will meet all four, and confusing them is the source of most JPMS error messages.

| Kind | What it is | How it behaves |
|---|---|---|
| **Named (explicit)** | A jar with a `module-info.class`, on the **module path** | Full rules apply. Reads only what it `requires`. Exports only what it `exports`. |
| **Automatic** | A plain jar with no `module-info`, on the **module path** | Gets a name (from `Automatic-Module-Name`, else from the filename). Reads *everything*. Exports *everything*. A bridge, not a destination. |
| **Unnamed** | Anything on the **classpath** | The old world. One giant unnamed module. Reads everything on the module path that is exported. Cannot be `requires`d by a named module. |
| **Platform** | The JDK's own modules: `java.base`, `java.sql`, `java.desktop`, … | Named modules, always present. `java.base` is required implicitly by everything. |

The single most useful rule to memorise:

> **Module path or classpath is decided by the flag you use, not by the jar.**
> `-cp` / `--class-path` puts it in the unnamed module. `-p` / `--module-path` makes it
> a named or automatic module. The same jar behaves differently depending on which
> flag you used.

---

## Why does it matter?

**1. You will hit `--add-opens` within your first month.** Any Java 17+ service running
a library that reflects into JDK internals — older Hibernate, older Jackson, some
mocking and serialization libraries, anything touching `sun.misc.Unsafe` — throws
`InaccessibleObjectException` at runtime. The fix is a JVM flag that half the industry
copies from Stack Overflow without knowing what it does. After this topic you will know.

**2. Container image size and startup time.** `jlink` is how you get a Java service into
a small image. The realistic pattern for `orderflow` is not "modularise the application"
— it is "use `jdeps` to compute which JDK modules we need, `jlink` a runtime containing
only those, and run our ordinary classpath application on it". That works today, with
zero source changes, and it is what production Java containers actually do
(**Topic 122**).

**3. Library design.** If you ever publish a library with a real API surface, a
`module-info.java` is how you stop consumers depending on your internals — which is how
you stop your internals from becoming a support obligation you never agreed to.
**Topic 03** told you package-private is the only boundary before JPMS. This is what
comes after.

**4. It is a standard interview question**, and the answer that impresses is not the
feature list. It is knowing *why adoption stalled* and *when you would still use it*.

---

## Syntax breakdown

`module-info.java` is a genuinely new file with genuinely new keywords. It lives at the
**root of your source tree**, not inside a package directory.

```
src/main/java/module-info.java
src/main/java/com/orderflow/pricing/api/PriceCalculator.java
src/main/java/com/orderflow/pricing/internal/RoundingRules.java
```

```java
module com.orderflow.pricing {

    // ---- what I need ----
    requires com.orderflow.money;              // I need it, and my consumers do not
    requires transitive com.orderflow.model;   // I need it, AND anyone who requires me gets it too
    requires static org.jspecify;              // needed at COMPILE time only (annotations)
    requires java.sql;                         // a JDK module

    // ---- what I offer ----
    exports com.orderflow.pricing.api;                       // public to everyone
    exports com.orderflow.pricing.spi to com.orderflow.orders; // public to ONE named module

    // ---- reflective access ----
    opens com.orderflow.pricing.model;                       // reflection allowed at runtime
    opens com.orderflow.pricing.entity to org.hibernate.orm.core;  // to one module only

    // ---- services (ServiceLoader) ----
    uses com.orderflow.pricing.spi.TaxStrategy;              // I look these up
    provides com.orderflow.pricing.spi.TaxStrategy
        with com.orderflow.pricing.internal.UkVatStrategy;   // I supply this implementation
}
```

Keyword by keyword. These are **restricted keywords** — they are only keywords inside
`module-info.java`, so a variable named `module` elsewhere in your code still compiles.

| Directive | What it means | The mistake people make |
|---|---|---|
| `requires X` | I read module X. Its exported packages are on my compile and run paths. | Forgetting it and getting `package ... is not visible` even though the jar is right there. |
| `requires transitive X` | I read X, **and** anybody who reads me also reads X. | Under-using it: if X's types appear in your **public API signatures**, it must be `transitive`, or your consumers cannot use your methods. |
| `requires static X` | Needed to compile, optional at run time. | Used for annotations (`@Nullable`) and optional integrations. If it is genuinely needed at runtime, this hides a `NoClassDefFoundError` until later. |
| `exports p` | Package `p`'s **public** types are accessible outside the module at compile and run time. | Assuming `exports` also permits reflection. It does not. That is `opens`. |
| `exports p to A, B` | A "qualified export": only modules A and B see it. | Using it for a public library — it hard-codes your consumers' names. Fine inside one application. |
| `opens p` | Deep reflective access at **run time**: `setAccessible(true)` works on private members. Does *not* grant compile-time access. | Confusing it with `exports`. `exports` is for `import`; `opens` is for frameworks. |
| `opens p to A` | Qualified open. **Prefer this.** | Opening everything to everybody, which throws away the encapsulation you just bought. |
| `open module M { ... }` | Every package is open. An escape hatch for migration. | Shipping it. It is a migration state, not a design. |
| `uses S` | I will look up implementations of service interface `S` via `ServiceLoader`. | Forgetting it, then getting an empty `ServiceLoader` with no error message at all. |
| `provides S with Impl` | I supply `Impl` as an implementation of `S`. | The `META-INF/services` file no longer works for a named module; you must use this directive. |

### The command-line escape hatches

These are the flags you will actually use, probably before you ever write a
`module-info.java`.

```bash
--add-exports  java.base/sun.nio.ch=ALL-UNNAMED    # grant compile/run access to a package
--add-opens    java.base/java.lang=ALL-UNNAMED     # grant DEEP REFLECTIVE access
--add-modules  java.sql,jdk.crypto.ec              # add modules not resolved by default
--add-reads    my.module=ALL-UNNAMED               # let a named module read the classpath
--limit-modules java.base                          # restrict the observable module set
--enable-native-access ALL-UNNAMED                 # [JAVA 25] see the note below
```

Read the syntax as `<source-module>/<package>=<target-module>`. `ALL-UNNAMED` means
"everything on the classpath", which for a normal Spring Boot application is your whole
program.

> **`--illegal-access` is gone.** In Java 9–15 the default was `--illegal-access=permit`,
> which allowed reflection into JDK internals with a warning. Java 16 changed the default
> to `deny`. **Java 17 removed the option entirely** (JEP 403, "Strongly Encapsulate JDK
> Internals"). So on Java 21 there is no global escape hatch: you grant access
> package-by-package with `--add-opens`, or you fix the library. If you find
> `--illegal-access=permit` in a startup script, it is being silently ignored — verify
> with `java --illegal-access=permit -version` and see whether your JDK errors or warns.

### Where flags can live

You cannot always control the `java` command line — a container entrypoint, a Maven
plugin, an IDE run configuration. Three other places work:

```bash
# 1. Environment variable, honoured by the `java` launcher (Java 9+)
export JDK_JAVA_OPTIONS="--add-opens java.base/java.lang=ALL-UNNAMED"

# 2. An @argfile
echo "--add-opens java.base/java.lang=ALL-UNNAMED" > jvm.args
java @jvm.args -jar orderflow.jar
```

```
# 3. JAR manifest attributes, honoured for `java -jar` (MANIFEST.MF)
Add-Opens: java.base/java.lang java.base/java.util
Add-Exports: java.base/sun.nio.ch
```

I am confident about `JDK_JAVA_OPTIONS` and `@argfile`. I am **reasonably** confident
that the `Add-Opens` / `Add-Exports` manifest attributes are honoured for executable
jars launched with `java -jar`, but the exact conditions are worth confirming on your
build rather than trusting. Proof 5 settles it.

`JAVA_TOOL_OPTIONS` also works but applies to **every** JVM started in that environment,
including Maven and your IDE. Prefer `JDK_JAVA_OPTIONS`, which only affects the `java`
launcher.

---

## Example 1 — minimal

Two modules. One exports an API and hides its implementation. The second uses it, and
is physically prevented from reaching the internals.

```
~/java-lab/20/minimal/
  pricing/src/module-info.java
  pricing/src/com/orderflow/pricing/api/PriceCalculator.java
  pricing/src/com/orderflow/pricing/internal/RoundingRules.java
  app/src/module-info.java
  app/src/com/orderflow/app/Main.java
```

`pricing/src/module-info.java`:
```java
module com.orderflow.pricing {
    exports com.orderflow.pricing.api;
    // com.orderflow.pricing.internal is deliberately NOT exported
}
```

`pricing/src/com/orderflow/pricing/api/PriceCalculator.java`:
```java
package com.orderflow.pricing.api;

import com.orderflow.pricing.internal.RoundingRules;

public final class PriceCalculator {
    public long totalMinor(long unitPriceMinor, int quantity) {
        return RoundingRules.roundToPenny(unitPriceMinor * quantity);
    }
}
```

`pricing/src/com/orderflow/pricing/internal/RoundingRules.java`:
```java
package com.orderflow.pricing.internal;

public final class RoundingRules {                 // NOTE: public
    public static long roundToPenny(long minor) { return minor; }
}
```

`app/src/module-info.java`:
```java
module com.orderflow.app {
    requires com.orderflow.pricing;
}
```

`app/src/com/orderflow/app/Main.java`:
```java
package com.orderflow.app;

import com.orderflow.pricing.api.PriceCalculator;
// import com.orderflow.pricing.internal.RoundingRules;   // <-- uncomment this

public class Main {
    public static void main(String[] args) {
        System.out.println(new PriceCalculator().totalMinor(1999L, 3));
    }
}
```

Build and run:
```bash
cd ~/java-lab/20/minimal
javac -d out/pricing $(find pricing/src -name '*.java')
javac --module-path out -d out/app $(find app/src -name '*.java')
java --module-path out --module com.orderflow.app/com.orderflow.app.Main
```

Now uncomment the second import in `Main.java` and rebuild.

`RoundingRules` is `public`. It is in a jar that is on the module path. And it is
**unreachable**, with a compile error saying the package is not visible because the
module does not export it.

That is the feature. In the classpath world there is no way to express this at all —
`public` has always meant public to everyone, and **Topic 03**'s package-private is the
only alternative, which would force `PriceCalculator` and `RoundingRules` into the same
package.

---

## Example 2 — production scenario

`orderflow` runs in Kubernetes. The container image is 420 MB, dominated by a full JDK.
Startup is 9 seconds. Security keeps flagging CVEs in JDK components the service does
not use — the AWT/desktop stack, the RMI registry, the built-in HTTP server.

### The approach that looks right and stalls

"Let's modularise `orderflow`." Somebody adds `module-info.java` to the application.

Within an hour:

```
error: module not found: spring.boot.autoconfigure
error: the unnamed module reads package com.orderflow.orders from both ...
error: automatic module cannot be used with jlink: spring.core from file:///.../spring-core-6.2.0.jar
```

The application depends on ~120 jars. Some have `module-info`, some have only
`Automatic-Module-Name`, and some have neither — so their module names come from their
filenames and change with their versions. Spring Boot's fat jar layout is not a module
path layout. Two libraries share a package (a "split package"), which JPMS forbids
outright.

The effort is abandoned after a week. This is the normal outcome and it is not a
failure of skill.

### The approach that ships

**Do not modularise the application. Modularise the runtime.**

`jdeps` tells you which JDK modules your code and its dependencies actually use.
`jlink` builds a runtime containing only those. Then you run your perfectly ordinary
classpath application on that trimmed runtime. No `module-info.java`, no source changes,
no fight with your dependency graph.

```bash
# Step 1 — one fat jar, or the exploded Boot layout. Either works.
./mvnw -q clean package
JAR=target/orderflow-1.0.0.jar

# Step 2 — ask jdeps which JDK modules are actually reachable.
#   --print-module-deps  -> a comma-separated list ready to paste into jlink
#   --ignore-missing-deps -> tolerate optional dependencies that are absent.
#                            Read the caveat below before using it.
#   --multi-release 21   -> pick the right classes from multi-release jars
DEPS=$(jdeps --multi-release 21 \
             --print-module-deps \
             --ignore-missing-deps \
             "$JAR")
echo "JDK modules required: $DEPS"

# Step 3 — build the runtime image.
jlink --add-modules "$DEPS" \
      --strip-debug \
      --no-header-files \
      --no-man-pages \
      --compress=zip-6 \
      --output ./orderflow-runtime

# Step 4 — run the ordinary application on the trimmed runtime.
./orderflow-runtime/bin/java -jar "$JAR"

# Step 5 — measure what you gained.
du -sh "$JAVA_HOME" ./orderflow-runtime
./orderflow-runtime/bin/java --list-modules | wc -l
```

The Dockerfile that results — a two-stage build where the second stage carries no JDK at
all:

```dockerfile
# ---- stage 1: build the app and the runtime ----
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /build
COPY . .
RUN ./mvnw -q clean package -DskipTests
RUN DEPS=$(jdeps --multi-release 21 --print-module-deps --ignore-missing-deps \
                 target/orderflow-1.0.0.jar) && \
    jlink --add-modules "$DEPS" \
          --strip-debug --no-header-files --no-man-pages --compress=zip-6 \
          --output /javaruntime

# ---- stage 2: the runtime image ----
FROM debian:bookworm-slim
ENV JAVA_HOME=/opt/java
ENV PATH="${JAVA_HOME}/bin:${PATH}"
COPY --from=builder /javaruntime  $JAVA_HOME
COPY --from=builder /build/target/orderflow-1.0.0.jar /app/orderflow.jar
ENTRYPOINT ["java", "-jar", "/app/orderflow.jar"]
```

What you gain, and what it costs:

| Gain | Honest caveat |
|---|---|
| A much smaller image — commonly 40–60% off the base layer | The application jar and its dependencies are unchanged. If your fat jar is 90 MB, `jlink` cannot help with that. |
| A smaller attack surface: no AWT, no RMI, no built-in HTTP server unless you need them | Some scanners flag the *whole* JDK version regardless of which modules are present. You may still have to argue the case. |
| Slightly faster startup — fewer modules to resolve | The win is small compared to class loading. AppCDS is the bigger lever (**Topic 122**). |
| No source changes | You must rebuild the runtime when you change dependencies, or you get `NoClassDefFoundError` at runtime for a JDK class you newly need. |

**The `--ignore-missing-deps` caveat, stated honestly.** That flag tells `jdeps` to
proceed when it cannot resolve some references. It is frequently necessary with Spring
because of optional integrations, and it is also exactly how you end up with a runtime
missing a module you need on a code path that runs once a month. Two defences: add
modules you know you need explicitly, and — non-negotiably — run your full integration
test suite against the `jlink`ed runtime in CI, not against the build JDK.

A reasonable belt-and-braces line:

```bash
jlink --add-modules "$DEPS",jdk.crypto.ec,jdk.unsupported,java.management \
      --output ./orderflow-runtime
```

`jdk.crypto.ec` is needed for some TLS cipher suites and is a classic omission that
surfaces as a handshake failure in production. `jdk.unsupported` provides
`sun.misc.Unsafe`, which many libraries still use. `java.management` is needed by JMX
and by most monitoring agents.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — reflection into JDK internals on Java 17+

**Wrong:** a library — an older Hibernate, an older Jackson, a mocking framework, or
anything reaching for `sun.misc.Unsafe` — calls `setAccessible(true)` on a JDK internal.

**Exact symptom:**
```
java.lang.reflect.InaccessibleObjectException: Unable to make field
  private final java.lang.Object[] java.util.ArrayList.elementData accessible:
  module java.base does not "opens java.util" to unnamed module @0x2f4d3709
	at java.base/java.lang.reflect.AccessibleObject.throwInaccessibleObjectException(...)
```

It appears at **runtime**, not at build time. Often it appears only on one code path, so
it survives your test suite and fails in production. It typically arrives on the day you
upgrade from Java 11 to 17 or 21, alongside twenty other changes, which makes it hard to
attribute.

**Root cause:** Java 16 made strong encapsulation of JDK internals the default and Java
17 removed the `--illegal-access` opt-out (JEP 403). Before that, reflecting into
`java.base` printed a warning and worked. Now it throws.

**Fix, in order of preference:**

1. **Upgrade the library.** Almost every widely-used library fixed this years ago. If
   you are hitting it, you are probably several major versions behind, and the flag is
   treating a symptom.
2. **Open only the exact package to the exact target:**
   ```bash
   --add-opens java.base/java.util=ALL-UNNAMED
   ```
   Never a blanket open of `java.base`. Read the exception message: it names the exact
   module and package you need, so the flag can be constructed mechanically from the
   error.
3. **Record it where it will not be lost.** Put it in `JDK_JAVA_OPTIONS`, an `@argfile`,
   or the jar manifest — with a comment naming the library and a link to the upstream
   issue, so it can be removed when the library is upgraded. An `--add-opens` with no
   comment is permanent.

**The diagnostic that saves you an hour:** the exception message contains the exact flag
you need. `module java.base does not "opens java.util"` translates directly into
`--add-opens java.base/java.util=ALL-UNNAMED`. Read the message; do not search for it.

---

### Trap 2 — a `requires` on an automatic module, whose name comes from the filename

**Wrong:**
```java
module com.orderflow.pricing {
    requires commons.lang3;      // derived from commons-lang3-3.12.0.jar
}
```

**Exact symptom:** it compiles today. Then a dependency bump changes the jar's filename
and you get:
```
error: module not found: commons.lang3
```
Or, more confusingly, the build works locally and fails in CI because the two
environments resolved slightly different artifact filenames.

**Root cause:** for a jar with no `module-info` and no `Automatic-Module-Name` in its
manifest, JPMS derives a module name from the **filename**: strip the extension, strip a
trailing version, replace non-alphanumerics with dots. That derived name is not a
contract. Nobody promised it would stay the same.

**Fix:**

1. Check what the name actually is before depending on it:
   ```bash
   jar --describe-module --file=commons-lang3-3.12.0.jar
   ```
   The output tells you whether the name is declared (`Automatic-Module-Name` in the
   manifest, which *is* a promise from the library author) or derived from the filename
   (which is not). Only depend on the former.
2. If the name is filename-derived, do not modularise against that library. Wait for it
   to ship a real module name, or keep that part of your code on the classpath.
3. Never rename jars to make a module name work. It is a fix that lasts until the next
   person runs `mvn clean`.

---

### Trap 3 — split packages

**Wrong:** two jars both contain classes in `com.orderflow.common.util`. On the
classpath this "worked" — whichever jar came first won, silently.

**Exact symptom:**
```
error: module com.orderflow.app reads package com.orderflow.common.util
  from both common.core and common.legacy
```
or at runtime:
```
java.lang.LayerInstantiationException: Package com.orderflow.common.util
  in both module common.legacy and module common.core
```

**Root cause:** JPMS requires that each package belong to exactly one module. This is
not a restriction it invented arbitrarily — it is the fix for a real classpath defect
where two jars silently shadow each other's classes and the winner depends on the order
of `-cp`, which can differ between your laptop and production.

**Fix:** rename one package. There is no flag for this and there should not be. The
split package was always a bug; JPMS is the first thing that told you about it.

**The wider lesson:** many of the errors you hit while migrating to modules are the
module system reporting pre-existing defects. That is genuinely valuable, and it is also
why migration takes longer than people budget — you are not adding modules, you are
paying down a decade of accumulated classpath ambiguity.

---

### Trap 4 — `jlink` refusing to run

**Wrong:**
```bash
jlink --module-path target/libs --add-modules com.orderflow.app --output ./runtime
```

**Exact symptom:**
```
Error: automatic module cannot be used with jlink: spring.core from
  file:///home/dev/orderflow/target/libs/spring-core-6.2.0.jar
```

**Root cause:** `jlink` produces a *linked* runtime image, which requires knowing the
complete, resolved module graph. An automatic module reads everything and exports
everything, so its dependencies are unknown by construction. `jlink` cannot link against
"unknown", so it refuses rather than producing a broken image.

This is the wall that ends most application-modularisation efforts, and it is worth
understanding as a design constraint rather than a limitation: an automatic module is a
promise-free bridge, and `jlink` needs promises.

**Fix:** stop trying to `jlink` the application. `jlink` only the **JDK** modules —
Example 2's pattern — and run the application from the classpath on top. You get the
image-size and attack-surface benefits with none of the modularisation cost.

```bash
jlink --add-modules java.base,java.sql,java.naming,java.management,jdk.unsupported \
      --output ./runtime
./runtime/bin/java -jar orderflow.jar        # ordinary classpath app, trimmed runtime
```

Confirm before you commit to it:
```bash
./runtime/bin/java --list-modules
```

---

### Trap 5 — `exports` when you needed `opens`

**Wrong:**
```java
module com.orderflow.orders {
    exports com.orderflow.orders.entity;      // Hibernate needs to reflect on these
    requires jakarta.persistence;
}
```

**Exact symptom:** it compiles cleanly. At runtime, on the first database access:
```
java.lang.reflect.InaccessibleObjectException: Unable to make field
  private java.lang.Long com.orderflow.orders.entity.Order.id accessible:
  module com.orderflow.orders does not "opens com.orderflow.orders.entity"
  to module org.hibernate.orm.core
```

**Root cause:** `exports` and `opens` grant different things and people conflate them
constantly.

| | Compile-time access to public types | Runtime reflective access to private members |
|---|---|---|
| `exports p` | **yes** | no |
| `opens p` | no | **yes** |
| both | yes | yes |

Hibernate, Jackson, Spring and every other reflective framework need `opens`. They set
private fields, invoke private constructors, and read private methods. `exports` does
nothing for them.

**Fix:** open the package, and open it to the specific module that needs it rather than
to the world:

```java
module com.orderflow.orders {
    exports com.orderflow.orders.api;                              // my API
    opens   com.orderflow.orders.entity to org.hibernate.orm.core; // for the ORM only
    requires jakarta.persistence;
}
```

**The reason `opens ... to X` beats a bare `opens`:** a bare `opens` lets *anything* in
the JVM reflect into your entities, which gives back the encapsulation you modularised
to obtain. A qualified open documents exactly which framework needs the privilege, which
is also the thing a security reviewer will ask about.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM and will not print invented
output. Each proof gives the command, what to look for, and how to read each result.

### Setup

```bash
mkdir -p ~/java-lab/20 && cd ~/java-lab/20
java --version
echo "JAVA_HOME=$JAVA_HOME"
```

### Proof 1 — the JDK is already modular

```bash
java --list-modules | wc -l
java --list-modules | head -20
java --describe-module java.sql
java --describe-module java.base | head -20
```

**What to look for:** the number of modules, and the structure of a `describe-module`
listing.

| What you see | What it means |
|---|---|
| Roughly 70 modules listed, each with a version | The JDK is fully modularised. This is the only large-scale modular Java codebase in existence, and it is the proof that the system works when everyone participates. |
| `java.sql` shows `requires java.base`, `requires transitive java.logging`, `requires transitive java.xml`, and a list of `exports` | Read the `transitive` entries: anything requiring `java.sql` automatically gets `java.logging` and `java.xml`, because those types appear in `java.sql`'s public signatures. That is the rule from the syntax table, in the wild. |
| `java.base` exports a long list including `java.lang`, `java.util`, `java.io` — but **not** `sun.*` or `jdk.internal.*` | This is exactly why `--add-opens` exists. Those internal packages are present and unexported. |

Now find what is *not* exported:

```bash
java --describe-module java.base | grep -c "exports"
java --describe-module java.base | grep "sun\.\|jdk\.internal" | head
```

| What you see | What it means |
|---|---|
| Lines with `-> ` after a package name | A **qualified** export: that package is exported only to the named modules. The JDK uses these heavily to share internals between its own modules without opening them to you. |

### Proof 2 — build the two-module example and see encapsulation fail

Use Example 1's source tree.

```bash
cd ~/java-lab/20/minimal
javac -d out/pricing $(find pricing/src -name '*.java')
javac --module-path out -d out/app $(find app/src -name '*.java')
java --module-path out --module com.orderflow.app/com.orderflow.app.Main
```

Then uncomment the `RoundingRules` import in `Main.java` and recompile the app.

**What to look for:** the compile error, and its exact wording.

| What you see | What it means |
|---|---|
| `error: package com.orderflow.pricing.internal is not visible` / `(package ... is declared in module com.orderflow.pricing, which does not export it)` | The core feature demonstrated. A `public` class is unreachable because its package is not exported. This cannot be expressed on the classpath at all. |
| It compiles fine | You compiled with `-cp` instead of `--module-path`, so both are in the unnamed module and no rules apply. This is itself the most important lesson: **the flag decides**, not the jar. |

Now prove the runtime enforcement separately, by compiling against the module path and
then running everything on the **classpath**:

```bash
java -cp out/pricing:out/app com.orderflow.app.Main
```

| What you see | What it means |
|---|---|
| It runs | Correct and important: on the classpath there are no modules, so there is no encapsulation. `module-info.class` is simply ignored. Your library's boundaries only exist if your *consumer* uses the module path. |

That asymmetry is worth sitting with. It is a large part of why JPMS did not take over:
the guarantee is opt-in by the consumer, not enforced by the producer.

### Proof 3 — reflection versus `opens`

`ReflectIn.java` (put it in the `app` module, package `com.orderflow.app`):
```java
package com.orderflow.app;

import java.lang.reflect.Field;

public class ReflectIn {
    public static void main(String[] args) throws Exception {
        // 1. reflect into a JDK internal
        try {
            Field f = java.util.ArrayList.class.getDeclaredField("elementData");
            f.setAccessible(true);
            System.out.println("JDK internal: ACCESSIBLE");
        } catch (Exception e) {
            System.out.println("JDK internal: " + e.getClass().getSimpleName());
            System.out.println("  " + e.getMessage());
        }
    }
}
```

```bash
java --module-path out --module com.orderflow.app/com.orderflow.app.ReflectIn
java --add-opens java.base/java.util=ALL-UNNAMED \
     --module-path out --module com.orderflow.app/com.orderflow.app.ReflectIn
```

Note that `ALL-UNNAMED` is wrong for a named module — try the correct target too:

```bash
java --add-opens java.base/java.util=com.orderflow.app \
     --module-path out --module com.orderflow.app/com.orderflow.app.ReflectIn
```

**What to look for:** which of the three runs prints `ACCESSIBLE`.

| What you see | What it means |
|---|---|
| Run 1: `InaccessibleObjectException` with a message naming `java.base does not "opens java.util"` | Strong encapsulation of JDK internals, working as designed since Java 16/17. Note that the message names the exact flag you need. |
| Run 2 still fails when running as a **named module** | Expected. `ALL-UNNAMED` targets the classpath. Running via `--module` puts you in a named module, so the target must be your module's name. This distinction wastes a lot of people's afternoons. |
| Run 3 succeeds | Confirms the correct form: `--add-opens <source>/<package>=<target-module>`. |
| Run 2 succeeds | You are running from the classpath (`-cp`), not as a named module. Check your command. |

### Proof 4 — `jdeps` and `jlink` on a real jar

Use any jar you have. If you have none, build the minimal example into one:

```bash
cd ~/java-lab/20/minimal
jar --create --file pricing.jar -C out/pricing .
jar --describe-module --file=pricing.jar
```

**What to look for** in `jar --describe-module`:

| What you see | What it means |
|---|---|
| `com.orderflow.pricing` followed by `exports ...` and `requires java.base mandated` | An **explicit** module. Its name is a contract. Safe to `requires`. |
| `automatic` on the first line | An **automatic** module. Check whether the name came from `Automatic-Module-Name` (a promise) or from the filename (not a promise) — the tool tells you which. |
| `No module descriptor found. Derived automatic module.` | Filename-derived. This is Trap 2 waiting to happen, and `jlink` will refuse it. |

Now the real workflow:

```bash
jdeps --print-module-deps pricing.jar
jdeps --multi-release 21 --print-module-deps --ignore-missing-deps pricing.jar
jdeps -s pricing.jar                     # summary: which packages depend on which
jdeps --module-path out --check com.orderflow.pricing    # check the declared module graph
```

| What you see | What it means |
|---|---|
| A comma-separated list like `java.base` | Ready to paste into `jlink --add-modules`. This is the whole point of `--print-module-deps`. |
| `Error: ... not found` without `--ignore-missing-deps` | Some referenced class is not on the analysed path. Either add it to `--class-path`, or use `--ignore-missing-deps` and accept the caveat from Example 2 — you may be omitting a module you need. |
| `--check` reports transitive requires that could be reduced | `jdeps --check` will tell you when a `requires transitive` is unnecessary or when a `requires` is unused. Useful for tidying a `module-info` you inherited. |

Then build and measure a runtime:

```bash
jlink --add-modules java.base --output ./tiny-runtime
du -sh ./tiny-runtime "$JAVA_HOME"
./tiny-runtime/bin/java --list-modules
./tiny-runtime/bin/java --version

jlink --add-modules java.base,java.sql,java.naming,java.management,jdk.unsupported \
      --strip-debug --no-header-files --no-man-pages --compress=zip-6 \
      --output ./service-runtime
du -sh ./service-runtime
```

| What you see | What it means |
|---|---|
| `tiny-runtime` is a small fraction of `$JAVA_HOME` | The size win, measured on your own machine rather than quoted from a blog post. Record both numbers. |
| `--list-modules` on the tiny runtime shows only `java.base` | Confirms `jlink` linked only what you asked for. Everything else — AWT, RMI, the built-in HTTP server — is physically absent, not merely unused. |
| `--compress=zip-6` is rejected | Compression option syntax changed across JDK versions (older builds used numeric levels like `--compress=2`). Run `jlink --help` and use whatever your build accepts. This is a real version difference and worth checking rather than guessing. |

### Proof 5 — where `--add-opens` can live

Test each mechanism against Proof 3's `ReflectIn` and record which ones work on your
build.

```bash
# a) command line
java --add-opens java.base/java.util=ALL-UNNAMED -cp out/app:out/pricing com.orderflow.app.ReflectIn

# b) environment variable
JDK_JAVA_OPTIONS="--add-opens java.base/java.util=ALL-UNNAMED" \
  java -cp out/app:out/pricing com.orderflow.app.ReflectIn

# c) argfile
echo "--add-opens java.base/java.util=ALL-UNNAMED" > jvm.args
java @jvm.args -cp out/app:out/pricing com.orderflow.app.ReflectIn

# d) jar manifest
cat > manifest.txt <<'EOF'
Main-Class: com.orderflow.app.ReflectIn
Add-Opens: java.base/java.util
EOF
jar --create --file app.jar --manifest manifest.txt -C out/app . -C out/pricing .
java -jar app.jar
```

**What to look for:** which of the four print `ACCESSIBLE`.

| What you see | What it means |
|---|---|
| (a) and (c) work | Certain. These are the documented mechanisms. |
| (b) works, and prints a line to stderr echoing the picked-up options | Expected — the `java` launcher echoes `JDK_JAVA_OPTIONS` so the behaviour is not invisible. Note this: it means the flag is discoverable in logs, unlike `JAVA_TOOL_OPTIONS`. |
| (d) works | Confirms the `Add-Opens` manifest attribute is honoured for `java -jar` on your build. This is the one I flagged as only reasonably certain — you have now settled it. |
| (d) does not work | Then the attribute is not honoured here. Note your JDK version and use (b) or (c) instead. Do not conclude the attribute does not exist; conclude it does not work in this configuration. |

**Why this matters practically:** in a container you often cannot change the `java`
command line, because the entrypoint comes from a base image or a Helm chart. Knowing
which of these four work on your JDK decides how you ship a required `--add-opens`.

### Proof 6 — `[JAVA 25]` native-access warnings

Java 24 and 25 began warning when code calls restricted native methods — JNI and the FFM
API — unless the module is granted access with `--enable-native-access`. The direction is
that these will become errors in a future release.

```bash
java --enable-native-access=ALL-UNNAMED -jar app.jar
java -XX:+PrintFlagsFinal -version | grep -i native   # not the right tool, but try it
java --help-extra 2>&1 | grep -i native-access
```

| What you see | What it means |
|---|---|
| `--help-extra` lists `--enable-native-access` | The option exists on your JDK. Its enforcement level differs between releases, so read your JDK's release notes for whether it warns or errors. |
| The option is unrecognised | You are on a JDK older than it. On **Java 21 there is no such enforcement**, which is the 21 fallback: native access is unrestricted and unwarned. |

I am flagging this rather than asserting details, because the exact release in which each
warning became a warning, and the planned release for it becoming an error, is precisely
the kind of thing that moves. The settling command is above; the JEP index at
openjdk.org/jeps is authoritative.

The same honest caveat applies to `sun.misc.Unsafe`: its memory-access methods have been
progressively deprecated and warned about across recent releases, with removal intended.
`jdk.unsupported` is the module that still provides it, which is why Example 2 adds it to
the `jlink` set. Check your JDK's release notes rather than trusting a version number
from me.

---

## Practice exercises

### 1 — Easy: measure a runtime image

**Part A.** Run `java --list-modules | wc -l` and `du -sh $JAVA_HOME`. Record both.

**Part B.** Build three runtimes with `jlink`: `java.base` alone; a "typical service"
set (`java.base,java.sql,java.naming,java.management,jdk.unsupported,jdk.crypto.ec`);
and `java.se`. Record the size of each and the module count of each.

**Part C.** Write a `HelloOrderflow` class that opens a JDBC connection to nothing (just
call `DriverManager.getDrivers()`) and prints the count. Run it on all three runtimes.
Note which fail, and capture the **exact** exception from the failures.

**Part D.** Now do it properly: `jdeps --print-module-deps` on your class, `jlink` with
exactly that list, and confirm it runs. Then answer: how much smaller is your minimal
runtime than the full JDK, as a percentage — and what did you give up to get there?

### 2 — Medium: modularise a small library (combines Topics 03, 05, 13, 16, 17)

Build `orderflow-money`, a real library module.

**Part A.** Write it with a genuine API/internal split:
- `com.orderflow.money.api` — a `Money` value type (immutable, `long` minor units, an
  explicit currency `enum`; Topics 01, 16, 17) and a `MoneyFormatter` interface.
- `com.orderflow.money.internal` — the formatter implementations and a
  `RoundingRules` helper. All `public` classes, none of them exported.
- `equals`/`hashCode` correct per Topic 13, with `Money` usable as a `HashMap` key.

**Part B.** Write `module-info.java` exporting only the API package. Then write a second
module that consumes it and **prove**, with a captured compile error, that it cannot
reach `internal`.

**Part C.** Add a `ServiceLoader` layer: declare `uses com.orderflow.money.api.MoneyFormatter`
in the consumer and `provides ... with ...` in the library. Prove the consumer finds the
implementation without ever naming its class. Then delete the `uses` directive and
capture what happens — note whether you get an error or a silently empty `ServiceLoader`.

**Part D.** Compare with **Topic 03**. Write 8–12 lines answering: what does
`module-info.java` give you that package-private does not, and what does it cost? Would
you actually ship this `module-info` in a library you published to Maven Central? Justify
either answer.

### 3 — Hard: production simulation — shrink the `orderflow` container

Take any Spring Boot application. If you do not have one yet, generate a minimal one from
start.spring.io with web, JPA and PostgreSQL. (You will build the real `orderflow` from
Topic 35; this exercise is about the packaging, so any Boot app will do.)

**Part A — baseline.** Build a conventional image (`FROM eclipse-temurin:21-jre`), record:
image size, cold start time to first successful request, and RSS after startup
(`docker stats`).

**Part B — attempt the honest failure.** Try to add `module-info.java` to the
application and build it as a named module. Timebox it to 45 minutes. **Capture every
error you hit, in order.** Then stop and write down which of them are fixable in your own
code and which are properties of your dependency graph. This part is not busy-work: it is
how you will recognise the wall in a real project instead of spending a week on it.

**Part C — the approach that works.** Use `jdeps --print-module-deps` plus `jlink` to
build a trimmed runtime and the two-stage Dockerfile from Example 2. Record the same
three numbers as Part A.

**Part D — break it deliberately.** Remove `jdk.crypto.ec` from your module list and
rebuild. Make an outbound HTTPS call to a host that requires an EC cipher suite. Capture
the failure. This is exactly the class of bug `--ignore-missing-deps` produces, and
meeting it in a lab is much cheaper than meeting it in production.

**Part E — the safety net.** Add a CI step that runs your integration tests **against the
jlinked runtime**, not the build JDK. Show it failing on the Part D configuration and
passing once you restore the module. Write the two-line explanation of why this step is
mandatory rather than nice-to-have.

**Part F — the recommendation.** Write the paragraph you would put in a PR:
what changed, the three numbers before and after, what it cost in build complexity, and
what would make you revert it. Then compare it honestly against the alternative of a
plain `-jre` base image with a distroless final stage — is `jlink` actually worth it for
your numbers? Forward-reference **Topic 122**, where AppCDS attacks the startup half of
the same problem and is usually the larger lever.

---

## Interview questions

### Q1 — "What is JPMS and why did they add it?"

**Mid-level answer:** "It's the module system from Java 9. You declare modules with
`module-info.java` and it lets you split your application into modules and make jars
smaller."

**Senior answer:** "Two goals, and smaller jars isn't one of them. First, reliable
configuration: the classpath's failure mode is a `NoClassDefFoundError` at runtime on
whichever code path first needs the missing class, which can be weeks after deploy. The
module system resolves the whole graph at startup and fails immediately on anything
missing, duplicated or cyclic. Second, strong encapsulation: before JPMS, `public` meant
'callable by anyone who can put this jar on their classpath', so a library had no way to
have a type that's public across its own packages but invisible to consumers.
Package-private was the only boundary, and it forces everything into one package.
`exports` fixes that, and the enforcement is at compile time *and* by the JVM at run
time, including through reflection. The size benefit is real but it's downstream — the
JDK being modular is what makes `jlink` possible, and `jlink` is where the size win
actually comes from."

**What separates them:** naming reliable configuration at all — most people only know
encapsulation — and correctly placing size as a consequence rather than a goal.

**Follow-up:** "So why doesn't anyone use it?" That is Q2, and volunteering it here is
better than waiting to be asked.

---

### Q2 — "Why did modularisation adoption stall?"

**Mid-level answer:** "It's complicated and most libraries don't support it."

**Senior answer:** "Because it's all-or-nothing across the dependency graph, and the
payoff is withheld until you finish. To compile a named module, every module you
`requires` must be a module. A plain jar on the module path becomes an automatic module
whose name comes from `Automatic-Module-Name` if the author supplied one, and otherwise
from the *filename* — which is not a stable identifier, so a version bump can break your
`requires`. And `jlink`, which is the reward, refuses automatic modules outright. So for
a service with a hundred-odd dependencies you do a lot of work, hit split packages and
cycles that were latent classpath bugs, and get no runtime image at the end.
The migration also surfaces genuine pre-existing defects — split packages, circular
dependencies — which is valuable but is not what the team signed up for that sprint.
Meanwhile the JDK itself is fully modular and everyone benefits from that without doing
anything, which removes most of the pressure to migrate. So the equilibrium is: modular
JDK, some modular libraries, almost no modular applications."

**What separates them:** the specific mechanics — automatic module naming, `jlink`'s
refusal — rather than a vague "it's hard", and the observation that the JDK being modular
removed the incentive.

**Follow-up:** "Then when would you actually use it?" A library with a real API surface,
where hiding internals stops them becoming a support obligation. And `jlink` for the
runtime image, which needs no application modules at all.

---

### Q3 — "What does `--add-opens java.base/java.lang=ALL-UNNAMED` do, and why do so many services need it?"

**Mid-level answer:** "It allows reflection into `java.lang`. You add it when you get an
`InaccessibleObjectException`."

**Senior answer:** "It grants deep reflective access — `setAccessible(true)` on private
members — of the `java.lang` package in the `java.base` module, to the unnamed module,
which is everything on the classpath. Services need it because Java 16 made strong
encapsulation of JDK internals the default and Java 17 removed the `--illegal-access`
opt-out entirely, so libraries that used to reflect into the JDK with a warning now
throw. Note the distinction from `--add-exports`: exports grants compile and run access
to public types, opens grants runtime reflective access to private members. Frameworks
need opens; they set private fields.
The honest position on the flag is that it's a symptom. It almost always means a library
is several major versions behind — most fixed this years ago — so the first fix is the
upgrade, and the flag is the stopgap. And it should be the narrowest possible: read the
exception message, which names the exact module and package, and open only that one, to
the specific target rather than `ALL-UNNAMED` where you can. I'd also record it somewhere
with a comment naming the library, because an `--add-opens` with no comment is
permanent — nobody will ever dare remove it."

**What separates them:** the exports/opens distinction, treating the flag as debt rather
than a fix, and knowing the exception message contains the answer.

**Follow-up:** "Where do you put it if you can't change the entrypoint?"
`JDK_JAVA_OPTIONS`, an `@argfile`, or the jar manifest's `Add-Opens` attribute.

---

### Q4 — "How would you reduce a Java service's container image size?"

**Mid-level answer:** "Use a JRE base image instead of a JDK, and multi-stage builds."

**Senior answer:** "Measure first — I'd want to know how the current image splits between
base layer, dependencies and application classes, because the answer changes the fix.
If the base layer dominates, `jlink`: run `jdeps --print-module-deps` on the fat jar to
find which JDK modules are actually reachable, `jlink` a runtime with exactly those plus
`jdk.crypto.ec`, `jdk.unsupported` and `java.management` which are classic omissions,
then a two-stage build where the final stage is a slim base plus that runtime plus the
jar. That commonly takes 40–60% off the base layer with zero source changes — I don't
need to modularise the application, which is the mistake people make and which will fail
because of automatic modules.
If dependencies dominate, `jlink` won't help at all and the work is dependency pruning.
Two things I'd insist on: `--ignore-missing-deps` on `jdeps` is often necessary and is
exactly how you end up missing a module on a monthly code path, so the integration suite
must run against the jlinked runtime in CI, not the build JDK. And if the actual goal was
startup time rather than size, `jlink` is a small lever — AppCDS or the JDK's AOT cache
is the bigger one, and I'd want to see the startup breakdown before choosing."

**What separates them:** measuring before choosing, knowing you must not modularise the
app, naming the specific commonly-missed modules, and the CI safety net. Also separating
the size goal from the startup goal, which are usually conflated.

**Follow-up:** "What's the risk of the jlink approach?" A missing module that only
surfaces on a rare code path, and a runtime that must be rebuilt when dependencies
change.

---

### Q5 — "ES modules and JPMS. Same idea?"

**Mid-level answer:** "Pretty much — both let you declare what a module exports and
imports."

**Senior answer:** "The declaration shape is similar and that part transfers, but four
things differ and each one changes how you use it. The unit is a package rather than a
file, so package layout becomes your API boundary. Encapsulation is enforced by the JVM
at run time, including against reflection, which ES modules have no equivalent of — a
determined caller in JS can generally still get at things. Cycles between modules are a
hard error in JPMS, where ESM tolerates them, which is a real migration blocker for old
codebases. And JPMS deliberately has no version selection at all — `requires` names a
module, never a version — so resolution stays with Maven or Gradle, unlike npm where the
module system and the version resolver are the same thing and two versions can coexist.
That last one matters because it means adopting JPMS doesn't solve any of the dependency
problems people hope it will."

**What separates them:** naming the version-selection difference. Most candidates assume
a module system implies version management because npm's does, and that assumption
produces wrong expectations about what JPMS is for.

**Follow-up:** "Does JPMS solve the diamond dependency problem?" No — that is Maven's
nearest-wins resolution (Topic 32), and JPMS makes it *stricter* by refusing duplicate
packages rather than solving it.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `module-info.class` is ignored entirely when a jar is on the classpath. What does
   that tell you about who actually controls whether encapsulation happens — the library
   author or the consumer? What does that imply about publishing a modular library?

2. JPMS forbids cycles between modules but Java has always permitted cycles between
   packages and between classes. Why draw the line at modules? What would break if cycles
   were allowed?

3. `exports` and `opens` are separate directives. Argue that this separation is essential,
   then argue it is unnecessary complexity that contributed to adoption failure. Which
   argument do you find stronger?

4. `jlink` refuses automatic modules. Explain precisely *why* it cannot work with them —
   not "because it's not allowed", but what information `jlink` needs that an automatic
   module cannot supply.

5. The JDK is fully modular, which means every Java developer benefits from JPMS without
   writing a `module-info.java`. Does that make JPMS a success or a failure? Defend your
   answer against the opposite position.

6. Strong encapsulation of JDK internals broke a great deal of working software when it
   became the default in Java 16/17. Was that the right call? What would the cost have
   been of never enforcing it?

7. You are designing a new internal library that six teams will consume. Would you ship
   a `module-info.java`? State the condition under which your answer flips, and say what
   you would do instead to get the same benefit.

---

## Quick reference card

### `module-info.java` directives

```java
module com.example.name {
    requires other.module;              // I read it
    requires transitive other.module;   // I read it, and so do my consumers
    requires static other.module;       // compile-time only

    exports com.example.api;            // public types accessible outside
    exports com.example.spi to a.b, c.d;// qualified: only to these modules

    opens com.example.model;            // deep reflection at runtime
    opens com.example.entity to org.hibernate.orm.core;  // qualified: prefer this

    uses com.example.spi.Strategy;                       // I ServiceLoader these
    provides com.example.spi.Strategy with com.example.internal.Impl;
}
open module com.example.name { }        // every package opened — migration only
```

### `exports` versus `opens`

| | compile-time access to public types | runtime reflective access to private members |
|---|---|---|
| `exports p` | yes | no |
| `opens p` | no | yes |

Frameworks (Hibernate, Jackson, Spring) need `opens`. Callers need `exports`.

### Module kinds

| Kind | Where | Reads | Exports |
|---|---|---|---|
| named | module path, has `module-info` | what it `requires` | what it `exports` |
| automatic | module path, no `module-info` | everything | everything |
| unnamed | **classpath** | everything exported | everything |

### Command-line flags

```bash
-p / --module-path <dirs>              # module path
-m / --module <module>/<MainClass>     # run a module's main class
--add-modules m1,m2                    # add modules not otherwise resolved
--add-exports  src/pkg=target          # grant compile+run access
--add-opens    src/pkg=target          # grant deep reflective access
--add-reads    mymod=ALL-UNNAMED       # let a named module read the classpath
ALL-UNNAMED                            # target: everything on the classpath
```

Where flags can live: the command line, `JDK_JAVA_OPTIONS`, an `@argfile`, or the jar
manifest's `Add-Opens` / `Add-Exports` attributes.

### Diagnostic commands

```bash
java --list-modules                       # every observable module
java --describe-module java.sql           # one module's requires/exports
jar  --describe-module --file=x.jar       # is it named, automatic, or filename-derived?
jdeps -s x.jar                            # package-level dependency summary
jdeps --print-module-deps x.jar           # module list, ready for jlink
jdeps --generate-module-info out/ x.jar   # draft a module-info for a plain jar
jdeps --check com.example.name            # find unused or over-broad requires
jlink --add-modules "$DEPS" --output ./runtime
jlink --add-modules java.base --strip-debug --no-header-files --no-man-pages --output ./tiny
```

### The realistic production pattern

> Do **not** modularise the application. `jdeps --print-module-deps` on the fat jar,
> `jlink` a runtime with those JDK modules plus `jdk.crypto.ec`, `jdk.unsupported` and
> `java.management`, then run the ordinary classpath application on it. Test against the
> jlinked runtime in CI.

### Gotchas

- [ ] Module path versus classpath is decided by the **flag**, not the jar.
- [ ] `module-info.class` is ignored on the classpath, so encapsulation is opt-in by the
      consumer.
- [ ] `exports` is not `opens`. Frameworks need `opens`.
- [ ] `opens p to X` beats a bare `opens p`.
- [ ] If a type appears in your public API signatures, its module must be
      `requires transitive`.
- [ ] Never `requires` a filename-derived automatic module name.
- [ ] Split packages are a hard error. Rename one.
- [ ] `jlink` refuses automatic modules — do not try to link the application.
- [ ] `jdeps --ignore-missing-deps` can silently omit a module you need. Test the image.
- [ ] `--illegal-access` was removed in Java 17. It is silently ignored, not honoured.
- [ ] `META-INF/services` does not work for a named module. Use `provides ... with`.

---

## When would I use this at work?

**1. The morning after a Java version upgrade.**
A service that ran on 11 or 17 throws `InaccessibleObjectException` on one endpoint.
You read the message, extract the module and package from it, add the narrowest possible
`--add-opens`, file a ticket to upgrade the offending library, and note the flag with a
comment. Fifteen minutes instead of an afternoon — and, importantly, you do not paste in
a blanket `--add-opens java.base/java.lang=ALL-UNNAMED` from a search result and move on.

**2. When someone asks why the container image is 400 MB.**
`jdeps --print-module-deps` plus `jlink` is a measurable, source-free win that you can
deliver in an afternoon and defend with numbers. Knowing that you must *not* modularise
the application to get it is the difference between shipping this and burning a week.
This becomes real work at **Topic 122**.

**3. Publishing a library that other teams depend on.**
The moment your internals are reachable, they are load-bearing, and removing them becomes
a breaking change you never agreed to. A `module-info.java` — even if consumers use the
classpath and ignore it — documents the boundary and lets the ones who care enforce it.
This is the direct continuation of **Topic 03**'s point that everything public is a
support obligation.

---

## Connected topics

**Prerequisites:**
- **03 — Access modifiers and packages**: package-private is the only encapsulation
  boundary *before* JPMS. This topic is what comes after, and the reason
  `public`-but-hidden was previously impossible.
- **05 — Generics**: `requires transitive` exists because types leak through public
  signatures — including generic type parameters.
- **18 — Strings**: Proof 5 of that topic needed
  `--add-opens java.base/java.lang=ALL-UNNAMED`. This is where that flag comes from.
- **19 — Serialization**: reflective deserialization frameworks are the single biggest
  consumer of `opens`, and the reason most `module-info` files have `opens` directives at
  all.

**This unlocks:**
- **31 / 32 — Maven and dependency resolution**: JPMS does **no** version selection.
  Understanding that split — the module system resolves *presence*, Maven resolves
  *versions* — prevents a common category of wrong expectations.
- **34 — Supply chain**: a smaller runtime is a smaller attack surface, and a defensible
  answer to "why is this CVE not applicable to us".
- **36 — Component scanning**: Spring's classpath scanning assumes the classpath. This is
  a concrete reason a modular Spring Boot application is difficult.
- **49 / 53 — Hibernate**: entity classes need `opens ... to org.hibernate.orm.core`.
  Trap 5 is the error you will actually see.
- **83 — GraalVM native image**: the other closed-world approach. Native image does
  whole-program static analysis; `jlink` does module-graph linking. Comparing them
  honestly — what each costs and buys — is a strong interview answer.
- **122 — Dockerising Spring Boot**: where Example 2 becomes the real `orderflow`
  container, alongside layered jars and AppCDS. AppCDS is usually the larger startup
  lever; `jlink` is the larger size lever.
- **127 — Migration planning**: the Java 8→17+ jump is where strong encapsulation of JDK
  internals breaks things, and sequencing that break is a migration-plan concern.

---

*Java baseline 21. JPMS (Java 9), strong encapsulation of JDK internals (Java 16 default,
Java 17 with `--illegal-access` removed) and `jlink` are all stable on 21 and 25. Three
things in this document are deliberately flagged rather than asserted: whether the
`Add-Opens`/`Add-Exports` jar-manifest attributes work on your build (Proof 5 settles it),
the exact `jlink --compress` syntax your JDK accepts (`jlink --help` settles it), and the
`[JAVA 25]` native-access and `sun.misc.Unsafe` enforcement levels, which have been
changing release to release — check openjdk.org/jeps and your JDK's release notes rather
than trusting a summary. On Java 21 there is no native-access enforcement at all, which
is the fallback.*
