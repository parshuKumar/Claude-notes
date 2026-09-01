# 83 — GraalVM native-image and AOT: What You Gain, What You Lose

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is where you decide whether `orderflow` should ship as a native binary — and, most likely, prove with evidence that it should not.

---

## Before anything else — what is and is not in this document

**I do not have a JVM, I do not have GraalVM, and I have never built `orderflow` as a
native image. Nothing in this document is captured output, and I will never present
anything as if it were.**

Specifically, you will not find here:

- a startup time, in milliseconds or otherwise,
- an RSS figure, a binary size, or a build duration,
- a throughput number, a requests-per-second figure, or a latency percentile,
- "native image starts 50× faster and uses a third of the memory" or any claim of that
  shape presented as measured,
- a `native-image` build log presented as a transcript of something I ran.

**Why the rule is unusually load-bearing on this topic.** Native image is the most
oversold technology in the modern Java ecosystem. The internet is full of benchmarks with
enormous, exciting numbers, almost all of them from a hello-world HTTP handler, almost none
of them measuring steady-state throughput on a real service with a real dataset. **The
professional value of this document is not teaching you to build a native image. It is
teaching you to be the person in the room who says "our steady-state throughput will
probably be worse, and here is how we find out" instead of the person who says "native
image will make it fast."** If I invent a number here, I have made you the second person.

What you get instead, everywhere:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in the output *you* generate,
- a **"what you see" → "what it means"** table covering the plausible outcomes *and* the
  surprising ones.

### The labelled exception

To read a build report and a failure message you must know their shape. In three places I
show the **structure** of native-image output — field names and the shape of an error —
with every value replaced by `<n>`, `<name>` or `xxx`, carrying the inline label:

> *illustration of the format, not captured output*

Placeholders only. No plausible-looking numbers, ever.

### The direction I am confident about, and the magnitude I am not

There are two different kinds of claim in this document and I keep them separate:

| Claim | Confidence | Why |
|---|---|---|
| Native image **starts much faster** than the JVM | High — it is a structural consequence of not doing class loading, verification, or JIT warm-up | You still measure it |
| Native image uses **less resident memory** at startup | High — no JIT, no profiling metadata, no code cache | You still measure it |
| Native image has **lower peak throughput** on a long-running service | **Directionally high, magnitude unknown** — see below | You must measure it |
| Native image builds are **much slower** than JVM builds | High — whole-program static analysis is expensive | You still measure it |
| Reflection that works on the JVM **fails at runtime** on a native image unless declared | Certain — this is the defining property of the technology | The drill proves it |

**On the throughput claim specifically.** GraalVM has invested heavily in closing this
gap: profile-guided optimisation (PGO) in Oracle GraalVM, the G1 collector option, and
year-on-year improvements to the compiler. So "native image is always slower at steady
state" is **not** a safe absolute. What I will assert is narrower and defensible: *a
long-running JVM service with a warm C2 has profile information a static compiler does not
have, and closing that gap requires deliberate work (PGO with a representative profile) that
most teams do not do.* **The honest engineering position is: assume it will be worse, prove
otherwise with your own numbers on your own workload, and never quote someone else's.**

### On Project Leyden and the JDK 25 AOT cache

I will say plainly what I am and am not sure of.

**Sure:** Project Leyden exists, its goal is to improve JVM startup and warm-up time, and
JDK 24 delivered an AOT class-loading-and-linking cache (JEP 483) with further refinement
in the JDK 25 line. It is a *different design point* from native image: it keeps the full
JVM, keeps the JIT, keeps dynamic class loading, and caches work that would otherwise be
repeated at every startup.

**Not sure, and therefore not stating:** the exact flag names and their spelling in your
JDK, whether the flags are experimental or product, the precise workflow (training run then
production run) in the release you have, and what has changed between JDK 24 and JDK 25.
These have moved, and a wrong flag from a teaching document costs you an afternoon.

**What to do instead of trusting me:** go to the OpenJDK JEP index at
`https://openjdk.org/jeps/0`, search for "Leyden" and for "Ahead-of-Time", and read the
JEPs targeted at *your* JDK. Then confirm on your machine:

```bash
java -XX:+PrintFlagsFinal -version | grep -i -E 'aot|cds|archive'
java -XX:+PrintFlagsFinal -version | grep -i -E 'CacheDataStore|SharedArchiveFile'
java -Xshare:off -version && java -Xshare:auto -version   # CDS is the older, stable relative
```

**The engineering point survives the uncertainty**, and it is the one that matters for
`orderflow`: AppCDS and the AOT cache buy you a meaningful share of the startup win *while
keeping the JIT*, at a fraction of the operational cost of native image. For a
long-running service behind a load balancer, that is the better trade, and it is the
recommendation you should be prepared to defend.

---

## Mechanical statement

> **`native-image` does closed-world static analysis at build time and emits a binary
> containing only reachable code. Anything resolved at runtime — reflection, dynamic
> proxies, resource loading, `Class.forName` — must be declared in configuration, or it
> fails at runtime rather than at build time.**

Four mechanical consequences, each of which the drill will show you:

1. **"Closed world" means the set of classes is fixed at build time.** There is no class
   loader waiting to load something new. If a class is not proven reachable from a root,
   it is not in the binary. Not lazily loaded — *not present*.

2. **Static analysis is a reachability proof, and reflection breaks the proof.**
   `Class.forName(someString)` has no statically-knowable target. The analysis cannot
   follow it, so the target is not marked reachable, so it is not in the binary. The build
   succeeds. The failure happens on the code path that executes the reflection — which may
   be in production, on a rare endpoint, weeks later.

3. **The error moves from build time to run time, which is the wrong direction.** This is
   the single most important sentence in the topic. On the JVM, a missing class is found at
   the moment you touch it, and the classpath is inspectable. On a native image, the class
   was silently dropped from the binary and cannot be recovered without a rebuild.

4. **No JIT means no profile-guided optimisation, unless you build one.** C2 compiles based
   on what your service actually did (Topic 74). A static compiler compiles based on what it
   can prove. Those are not the same information, and the difference is peak throughput.

And the operational corollary:

5. **You give up most of your Phase 8 toolkit.** No `jcmd`, no `jstack`, no heap dump in the
   familiar format, no `PrintCompilation`, no JFR by default, and async-profiler's Java-aware
   modes do not apply. **Everything you spent Topics 66–82 learning becomes unavailable or
   different.** That cost is rarely mentioned in the benchmarks and it is enormous.

---

## The bridge from what you know

### NO ANALOGUE — and the tempting comparison is actively misleading

The comparison people reach for is **esbuild or webpack bundling and tree-shaking**. It is
wrong in the way that matters, and reasoning from it will cost you.

Where it looks similar:

- Both do whole-program analysis at build time.
- Both eliminate unreachable code.
- Both make the output smaller and the start faster.
- Both are defeated by dynamic access: `require(someVariable)` defeats a bundler's
  static analysis exactly as `Class.forName(someString)` defeats native-image's.

Where the comparison breaks, decisively:

| | esbuild / webpack | `native-image` |
|---|---|---|
| What is removed | Source modules from a bundle | **Machine-code paths and whole classes from the runtime** |
| What runs afterwards | Node, with V8's full JIT | A binary with **no JIT at all** |
| Peak performance effect | None — V8 still profiles and optimises at runtime | **Lower** — the compiler had only static information |
| Failure mode | A module is missing; you get a `require` error you can fix without rebuilding the runtime | A class is missing from the binary; **you must rebuild the whole image** |
| Build time cost | Seconds | Minutes, sometimes many |
| Debugging afterwards | Source maps; the normal debugger | A different toolchain entirely |

**The sentence to hold onto: Node ships a JIT either way.** Bundling changes how your code
is *delivered*; it does not change the engine. `native-image` changes the engine. You are
not choosing a build optimisation — you are choosing a different runtime with different
performance characteristics, different failure modes, and different observability.

### The closest thing to a real analogue, and its limits

If you have ever shipped a Node CLI as a single self-contained executable — `pkg`, or
Node's single-executable-application support — you have touched the *deployment* half of
the idea: one artefact, no runtime to install, fast to start. What you have not touched is
the *compilation* half. Those tools still embed a full V8. **They change packaging; native
image changes compilation.** Half the analogy is honest and the more important half is not.

### What actually transfers

- **"Dynamic access defeats static analysis"** transfers exactly, and it is the single most
  useful intuition you already have. Every reflection failure in this topic is the shape of
  a `require(variable)` bug you have already debugged.
- **"Scale-to-zero rewards fast startup"** transfers from serverless Node. The workload
  where native image genuinely wins is the workload where you already know startup matters.
- **Your instinct that a build step which takes minutes will damage the inner development
  loop** transfers, and is correct, and is under-weighted by most native-image advocacy.

---

## What is this?

### The two things sold under one name

**GraalVM** is several products. Keep them separate or you will have confused conversations:

| Thing | What it is | Relevant here |
|---|---|---|
| **GraalVM native-image** | Ahead-of-time compiler producing a standalone native binary. Closed world. No JIT. | **This document** |
| **The Graal JIT compiler** | A JIT written in Java, usable as a C2 replacement inside a normal JVM | Different topic; keeps the JVM, keeps warm-up |
| **Truffle / polyglot** | Language implementation framework | Not relevant |
| **AppCDS / CDS** | JVM feature (not GraalVM) that memory-maps a pre-parsed class archive | The cheap startup win — Topic 82's neighbour |
| **JDK AOT cache (Leyden)** | JVM feature caching class loading/linking work across runs | The other cheap startup win. Check the JEP index for your JDK |

**Native image and the AOT cache are answers to the same question with opposite trade-offs.**
Native image gives up the JVM to get the fastest possible start. The AOT cache keeps the JVM
and gets a smaller but still substantial start improvement with essentially no behavioural
change. **For a service that runs for weeks, the second is almost always right.**

### The closed-world assumption, precisely

The build performs a **points-to analysis** from a set of roots — your `main`, plus
everything declared reachable in configuration. It computes the transitive closure of
methods that can be invoked and classes that can be instantiated. Everything outside that
closure is **not in the binary**.

What this buys:

- No class loading at runtime. No verification. No linking. Startup is essentially "map the
  binary and run `main`".
- Aggressive whole-program optimisation: the compiler knows every implementation of every
  interface, so a call site with one implementation is a direct call with no guard.
- A smaller artefact containing only what is used.

What this costs:

- **Anything the analysis cannot follow must be declared.** Reflection, JNI, dynamic
  proxies, resource loading by name, service loaders, serialization.
- **No dynamic class loading, ever.** Not an agent, not a plugin, not a script engine
  compiling classes at runtime.
- **No JIT.** Which means no deoptimization, no speculation, no profile-guided inlining, no
  escape analysis informed by real behaviour (Topic 75). The compiler had to be
  conservative because it had to be correct for every possible execution.

### Build-time versus run-time initialisation

This is the concept that produces the most surprising bugs, so learn it before you need it.

Native image can run a class's **static initialiser at build time** and snapshot the
resulting heap into the binary. That is where a lot of the startup win comes from — work
you would otherwise do on every boot is done once, at build.

It is also a correctness hazard:

| Initialised at | What happens | Hazard |
|---|---|---|
| **Build time** | Static initialiser runs during the build; resulting objects are baked into the image heap | **Anything captured is frozen.** A timestamp taken at build time is the build's timestamp, forever. An open file handle, a socket, a thread, or a random seed captured at build time is wrong or invalid at run time |
| **Run time** | Static initialiser runs on first use in the running binary, as on the JVM | Slower start for that class; correct semantics |

The classic failure: a class that caches a `SecureRandom`, a hostname, an environment
variable, or a `System.currentTimeMillis()` in a static field. **Initialised at build time,
every deployed instance shares the build machine's value.** For `orderflow`, imagine an
idempotency-key generator seeded at class-init: every pod generating the same sequence.
That is a correctness incident, not a performance one.

```bash
# The flags. Read them as "which side of the line is this class on".
--initialize-at-build-time=com.orderflow.catalog.PriceTable
--initialize-at-run-time=com.orderflow.security.IdempotencyKeyFactory
```

Frameworks (Spring Boot's AOT support, Quarkus, Micronaut) make a large number of these
decisions for you. **That is genuinely valuable and it is also why native-image failures in
a framework app are hard to debug: the configuration you need is generated, not written.**

### Reachability metadata — the thing you will actually spend your time on

"Reachability metadata" is the modern umbrella term for the JSON configuration that tells
the build about things the analysis cannot see. Historically these were separate files;
current tooling consolidates them, and the exact file names and layout have changed across
GraalVM versions.

| Kind | Declares | Typical trigger in `orderflow` |
|---|---|---|
| Reflection | Classes, methods, fields accessed reflectively | Jackson deserialising a DTO; Hibernate touching an entity; Spring instantiating a bean by name |
| Dynamic proxies | Interface sets for `java.lang.reflect.Proxy` | Spring Data repository interfaces; `@Transactional` JDK proxies (Topic 40) |
| Resources | Files loaded from the classpath by name | `application.yml`, Flyway migrations, message bundles, `logback-spring.xml` |
| Serialization | Classes crossing Java serialization | Session state; some cache libraries |
| JNI | Native access | Rare in `orderflow`; common in crypto and native libraries |

**Where the files live and what they are called: check your GraalVM version's
documentation.** I am not going to give you a path that has changed three times, and you
will look at it once and never again because the tooling generates it.

**The generator you will actually use** is the tracing agent, which runs your app **on a
normal JVM**, records everything dynamic that happens, and writes the configuration:

```bash
# Run on the ORDINARY JVM with the agent attached. Exercise the app properly.
java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image \
     -jar target/orderflow.jar
```

**The critical limitation, and it is the whole drill:** the agent records what your run
*did*. Code paths you did not exercise produce no configuration. **Your test suite's
coverage becomes your native image's correctness.** An endpoint that only fires on a
payment failure, or a Jackson subtype only used for one payment method, is exactly the thing
that will fail in production at 3am. This is why Topic 63's mutation testing and Topic 65's
load profile matter here: they are how you make the agent's run representative.

### What you give up operationally

Say this part out loud in the design review, because nobody else will:

| On the JVM | On a native image |
|---|---|
| `jcmd`, `jstack`, `jmap`, `jinfo` | Not applicable. Different tooling, less of it |
| Heap dump → MAT dominator tree (Topic 79) | Support exists in some builds/flags; **verify for your version** rather than assuming |
| JFR (Topic 78) | Partial support, improving; **verify** |
| async-profiler Java modes (Topic 78) | Not applicable in the same way; native perf tooling instead |
| `-XX:+PrintCompilation`, deopt traces (Topic 74) | Nothing to print — no JIT |
| GC tuning across G1/ZGC/Shenandoah (Topics 71–72) | A smaller set of collector options; **check what your GraalVM edition offers** |
| Attach an agent at runtime (Topic 81) | No |
| Change behaviour with a JVM flag on restart | Many decisions are baked into the binary |

**This is the hidden cost, and it is the one that bites during an incident**, when you most
need the tools and least want to discover they are gone.

---

## Why does it matter?

**Because the decision gets made badly, expensively, and in public.** "Should we go native?"
arrives at most Java teams eventually, usually driven by a conference talk. The cost of
getting it wrong is measured in months: reflection configuration, a broken inner loop,
retrained operations, and a rollback.

**Because for `orderflow` specifically, the answer is probably no, and you need to be able
to say why with evidence.** `orderflow` runs 24/7 behind a load balancer. It starts once and
serves for weeks. Its scaling events are gradual. It has a warm C2, a tuned collector, a
measured baseline, and a Phase 8 toolkit built around JVM observability. **Startup time is
not on its critical path. Steady-state throughput and p99 are.** Native image optimises the
thing that does not matter here and risks the things that do.

**Because the workloads where it wins are real and you should recognise them.** Serverless
functions billed per-invocation with cold starts. CLI tools. Scale-to-zero on Knative. A
sidecar that must be tiny. Batch jobs that run for four seconds. In those, startup and RSS
*are* the business metric, and the throughput loss is irrelevant because there is no steady
state to lose.

**Because being the person who says "worse, with evidence" is a career skill.** Anyone can
be enthusiastic. Producing the comparison table that kills a fashionable proposal — and
being right — is what a senior engineer is for.

---

## Machine-level reality

### What the build actually does

```
your jars + JDK classes
  -> points-to analysis from roots (main + declared reachable)
     iterated to a fixed point: which methods can run, which types can exist
  -> class initialisation: build-time (snapshot into image heap) or run-time
  -> BUILD-TIME HEAP SNAPSHOT: objects created during build-time init are serialised
     into the binary's data section
  -> ahead-of-time compilation of every reachable method (Graal compiler)
  -> a substrate runtime is linked in: GC, thread scheduling, monitors, exception
     handling - a JVM's *runtime services* without the JVM's *dynamic* machinery
  -> one native executable
```

**"SubstrateVM" is the name of that embedded runtime.** It is worth knowing because error
messages and stack traces mention it, and because it explains what remains: you still have a
garbage collector, still have threads, still have monitors and the Java Memory Model. **The
JMM applies unchanged** (Topics 86–88). What you lost is the dynamic half — loading,
linking, profiling, JIT.

### Why peak throughput is lower

This is the part to be able to explain mechanically, because it is where the argument is
won.

C2 in a long-running JVM has information a static compiler cannot have:

| C2 has | Static compiler has | Consequence |
|---|---|---|
| A **receiver-type profile** per call site — this site saw one type a million times | The class hierarchy | C2 inlines with a cheap guard, or with no guard at all via CHA. The static compiler emits a virtual call unless it can prove uniqueness |
| **Branch frequencies** — this branch is taken 0.01% of the time | Static heuristics | C2 lays out code so the hot path is straight-line and the cold path is out of the way |
| **The ability to speculate and be wrong** — deoptimize and recompile (Topic 74) | No escape hatch; must be correct for all executions | C2 can compile the optimistic version. A static compiler cannot |
| **Escape analysis on real inlined graphs** (Topic 75) | Escape analysis on statically-inlined graphs | Less inlining upstream means weaker escape analysis, means more allocation |
| **The option to recompile** when the profile changes | One shot | A workload that shifts over the day is served by one fixed compilation |

**The mitigation is PGO**: run the binary (or an instrumented build) under a representative
load, collect a profile, and rebuild using it. This genuinely narrows the gap. Two honest
caveats: **profile-guided optimisation is an Oracle GraalVM feature rather than a Community
Edition one in the versions most teams encounter — verify for your distribution and
licence**; and it requires you to produce a representative load, which is exactly the Topic
65 load profile. Teams that will not build a load profile will not do PGO, and those are the
teams most likely to be disappointed.

### Memory, honestly

The RSS win is real and comes from several places at once: no JIT compiler threads, no
profiling metadata, no code cache, no interpreter, a smaller runtime, and a heap sized for
the actual live set rather than the JVM's defaults.

**Two honest caveats that get omitted:**

1. **The comparison is often unfair.** A JVM at default `MaxRAMPercentage` in a container
   (Topic 82) reserves far more than it needs. Compare a *tuned* JVM — heap sized from the
   measured live set — against the native image, not a default one. Half of the memory
   "win" in public benchmarks is a JVM nobody configured.
2. **Build-time memory is enormous.** The static analysis holds the whole program graph. It
   is common for a `native-image` build of a Spring service to need several gigabytes of RAM
   and multiple cores. **Your CI runners may not have it**, and discovering that is part of
   the evaluation.

### The failure shape you must recognise on sight

When something dynamic was not declared, the binary throws at the point of use. The message
shape is roughly:

```
Exception in thread "<name>" java.lang.ClassNotFoundException: <fully.qualified.Name>
	at ...

  -- or --

com.oracle.svm.core.jdk.UnsupportedFeatureException: <feature> is not supported ...

  -- or, the one that names the fix directly --

... registered for reflection ... To resolve this, add <name> to reflect-config
```

*illustration of the format, not captured output*

| What you see | What it means |
|---|---|
| `ClassNotFoundException` for a class you can see in your source | **The class was not proven reachable and is not in the binary.** Declare it in reachability metadata and rebuild |
| A message naming reflection or `reflect-config` explicitly | The best case — the tooling is telling you the fix. Add the entry |
| `UnsupportedFeatureException` | A feature genuinely unsupported at run time in the image (dynamic class definition, some `MethodHandle` shapes). Not a configuration problem — a design problem |
| A `NullPointerException` deep inside a framework with no obvious cause | Often a resource that was not included: a properties file, a `logback` config, a `META-INF/services` entry |
| Behaviour differing from the JVM with no exception | **The worst case.** Usually build-time initialisation capturing something it should not have. Check `--initialize-at-build-time` |
| The build itself fails with an analysis error | Better than all of the above — a build-time failure is a gift |

**Internalise the asymmetry: on the JVM this class of bug is a build-time or first-touch
error with a readable classpath. Here it is a runtime error in a binary you cannot inspect
the same way.** That asymmetry is the technology's defining cost.

---

## Example 1 — minimal

Before you touch `orderflow`, prove the mechanism on twenty lines. **If you cannot explain
the failure in a tiny program, you will not be able to explain it inside Spring.**

```java
package com.orderflow.lab.aot;

import java.lang.reflect.Constructor;

/**
 * A payment-method handler chosen by name at runtime.
 * This is the shape of every plugin/strategy registry in every real service,
 * and it is exactly what closed-world analysis cannot follow.
 */
public class PaymentHandlerRegistry {

    public interface PaymentHandler { String describe(); }

    public static class CardPaymentHandler implements PaymentHandler {
        public CardPaymentHandler() { }
        public String describe() { return "card"; }
    }

    public static class WalletPaymentHandler implements PaymentHandler {
        public WalletPaymentHandler() { }
        public String describe() { return "wallet"; }
    }

    public static void main(String[] args) throws Exception {
        // The class name comes from the ARGUMENT. Nothing static can follow this.
        String name = args.length > 0
            ? args[0]
            : "com.orderflow.lab.aot.PaymentHandlerRegistry$CardPaymentHandler";

        Class<?> type = Class.forName(name);
        Constructor<?> ctor = type.getDeclaredConstructor();
        PaymentHandler handler = (PaymentHandler) ctor.newInstance();
        System.out.println("handler: " + handler.describe());
    }
}
```

### Step 1 — confirm it works on the JVM

```bash
mkdir -p ~/java-lab/83 && cd ~/java-lab/83
javac -d out com/orderflow/lab/aot/PaymentHandlerRegistry.java
java -cp out com.orderflow.lab.aot.PaymentHandlerRegistry
java -cp out com.orderflow.lab.aot.PaymentHandlerRegistry \
     'com.orderflow.lab.aot.PaymentHandlerRegistry$WalletPaymentHandler'
```

**WHAT TO LOOK FOR:** both succeed. **This is the control**, and it is the whole point:
the code is correct Java and the JVM has no difficulty with it.

### Step 2 — install GraalVM and record what you have

```bash
# SDKMAN is the least painful route on macOS. Pick a current GraalVM for JDK 21+.
sdk list java | grep -i graal
sdk install java <the-identifier-you-just-saw>
sdk use java <the-identifier-you-just-saw>

java --version          # should mention GraalVM
native-image --version  # record this exactly; behaviour differs across versions
uname -m                # aarch64 on Apple Silicon
```

**Record the GraalVM version, the JDK version, `uname -m`, and whether you are on
Community or Oracle GraalVM.** Every result in this topic is conditional on all four, and
the Community/Oracle distinction determines whether PGO is available to you at all.

### Step 3 — build it, and time the build yourself

```bash
time native-image -cp out \
     --no-fallback \
     -o payment-registry \
     com.orderflow.lab.aot.PaymentHandlerRegistry
```

`--no-fallback` is **not optional for learning**. Without it, native-image may silently
produce a "fallback image" — a launcher that requires a JVM — which appears to work and
teaches you nothing. **`--no-fallback` turns a silent degradation into an honest failure.**

**WHAT TO LOOK FOR in the build output:** the build prints a progress report with sections
covering analysis, building the image, and a summary of what went in. Its column structure
is roughly:

```
[<n>/<n>] <Phase name>                    (<n>.<n>s @ <n>.<n>GB)
   <n> reachable types    <n> reachable fields    <n> reachable methods
   <n> types, <n> fields, <n> methods registered for reflection
Finished generating '<name>' in <n>m <n>s.
```

*illustration of the format, not captured output*

| Line | What to note |
|---|---|
| Reachable types/fields/methods | The size of the closed world. Compare between builds when you add configuration |
| Registered for reflection | **How much you had to declare.** Growth here is your configuration burden made visible |
| Peak build memory (`@ <n>GB`) | Whether your CI runner can do this at all |
| Total build time | The number that determines whether your inner loop survives |

Write these four down. They are the honest cost side of the ledger and they are the numbers
that never appear in a conference talk.

### Step 4 — run it, and find the failure

```bash
./payment-registry
./payment-registry 'com.orderflow.lab.aot.PaymentHandlerRegistry$WalletPaymentHandler'
```

| What you see | What it means |
|---|---|
| Both fail with a reflection or `ClassNotFoundException` | Expected. Neither handler was reachable — the only reference is a string |
| The default succeeds, the argument one fails | The analysis constant-folded the default string literal and followed it. **A genuinely interesting outcome** — it shows the analysis is smarter than "reflection always fails", and that your bug appears only on the path with a real dynamic value |
| Both succeed | Check you passed `--no-fallback` and are running the binary, not a fallback launcher. Verify with `file payment-registry` |
| The build failed before you got here | Read the analysis error; it usually names the unsupported construct precisely |

**The second outcome is the one to internalise.** It is why native-image bugs are found in
production and not in tests: the tested path had a literal, and the production path had a
value from a database.

### Step 5 — fix it with reachability metadata

Generate the configuration with the tracing agent, on the **ordinary JVM**:

```bash
mkdir -p native-config
java -agentlib:native-image-agent=config-output-dir=native-config \
     -cp out com.orderflow.lab.aot.PaymentHandlerRegistry \
     'com.orderflow.lab.aot.PaymentHandlerRegistry$WalletPaymentHandler'

# Look at what it recorded. This file IS the lesson.
cat native-config/*.json
```

**WHAT TO LOOK FOR:** an entry naming `WalletPaymentHandler` and its constructor — and **no
entry for `CardPaymentHandler`**, because you did not exercise that path. Now rebuild:

```bash
time native-image -cp out \
     --no-fallback \
     -H:ConfigurationFileDirectories=native-config \
     -o payment-registry \
     com.orderflow.lab.aot.PaymentHandlerRegistry

./payment-registry 'com.orderflow.lab.aot.PaymentHandlerRegistry$WalletPaymentHandler'  # works
./payment-registry 'com.orderflow.lab.aot.PaymentHandlerRegistry$CardPaymentHandler'    # ?
```

**That last line is the whole topic in one command.** The path you exercised under the
agent works. The path you did not still fails. **The agent's coverage is your binary's
correctness**, and no build-time check will tell you what you missed.

*(Flag spelling note: `-H:ConfigurationFileDirectories` is the long-standing form, and
current versions also pick configuration up automatically from
`META-INF/native-image/**`. Confirm against `native-image --help` and
`native-image --expert-options-all` on your version rather than trusting this line.)*

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised, Spring Boot |
| Dataset | 100k products, 1M orders, 5M order lines |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m`, G1 (confirm with `jcmd VM.flags -all`) |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Baseline | `/docs/java/baselines/` — p50/p95/p99/p999 per endpoint, plus throughput |
| Deploy shape | Long-running pods behind a load balancer; rolling deploys; no scale-to-zero |

### How the proposal arrives

A staff engineer returns from a conference. `orderflow` takes twelve seconds to start.
Rolling deploys are slow, and during a deploy the cluster runs briefly under-capacity. The
proposal: **build `orderflow` as a native image**. Startup drops to milliseconds, memory
drops, deploys get fast, and the container image gets smaller.

Every one of those is plausible. The proposal is still probably wrong, and the reason is
that **nobody has measured where the twelve seconds goes.**

### The first move: measure the twelve seconds

This is Topic 82's territory and it is the step that gets skipped.

```bash
# 1. Where does startup time go? Spring tells you, if you ask.
java -jar orderflow.jar --debug 2>&1 | grep -E 'Started|JVM running for'

# 2. Boot's startup tracking - the actual breakdown by step.
#    Enable the ApplicationStartup buffering recorder and read /actuator/startup.
curl -s localhost:8080/actuator/startup | jq '.timeline.events
       | sort_by(.duration) | reverse | .[0:20]'

# 3. Is it the JVM, or is it your code? Compare bare JVM start with app start.
time java -version
time java -jar orderflow.jar --spring.main.web-application-type=none

# 4. Class loading volume - the thing AppCDS and the AOT cache actually address.
java -Xlog:class+load:file=/tmp/classload.txt -jar orderflow.jar
wc -l /tmp/classload.txt

# 5. Is the container starving the JIT and the startup path? (Topic 82)
docker run --cpus=2 --memory=2g orderflow:latest   # vs --cpus=0.5
```

**WHAT TO LOOK FOR, and what each answer implies:**

| What you find | What it means | What to do |
|---|---|---|
| Several seconds in Hibernate/JPA metamodel building | Entity scanning and validation dominate | Native image would help — but so would fewer entities scanned, and `spring.jpa.*` tuning |
| Several seconds in classpath scanning / auto-configuration | Boot's component scan (Topics 36, 42) | **AppCDS or the AOT cache directly targets this**, at a fraction of the cost |
| Several seconds waiting on a database or a downstream at startup | Not a JVM problem at all | **Native image would change nothing.** Fix the readiness/startup ordering (Topic 121) |
| Several seconds of Flyway migrations | Not a JVM problem | Move migrations out of the app's startup path |
| Time spread evenly across thousands of classes being loaded and verified | The JVM's own startup cost | **This is the case AppCDS and the AOT cache were built for** |
| The container is CPU-throttled during boot | Topic 82 | Raise the startup CPU request; a `startupProbe` with a generous threshold |

**In a typical Spring service, a large share of the twelve seconds is class loading,
verification and context building.** AppCDS addresses exactly that, keeps the JVM, keeps
every tool you own, and takes an afternoon.

### The cheaper alternative, spelled out

```bash
# 1. AppCDS, the stable and well-documented option. Two steps.
java -XX:ArchiveClassesAtExit=orderflow.jsa -jar orderflow.jar   # training run
java -XX:SharedArchiveFile=orderflow.jsa -jar orderflow.jar      # production run

# 2. Confirm it is actually being used, rather than silently ignored.
java -Xshare:on -XX:SharedArchiveFile=orderflow.jsa -jar orderflow.jar
#    -Xshare:on FAILS LOUDLY if the archive cannot be mapped. Use it to verify,
#    then decide whether production should use :on (strict) or :auto (tolerant).

# 3. Measure the same thing you measured before, the same way.
curl -s localhost:8080/actuator/startup | jq '.timeline.events | length'

# 4. The JDK 25 AOT cache is the newer, more capable relative of this.
#    Flags have moved between releases. Read the JEPs for YOUR JDK:
#      https://openjdk.org/jeps/0    (search "Ahead-of-Time" and "Leyden")
java -XX:+PrintFlagsFinal -version | grep -i -E 'aot|CacheDataStore'
```

**The honest comparison to put in the design doc:**

| | JVM, untuned | JVM + AppCDS / AOT cache | Native image |
|---|---|---|---|
| Startup | baseline | meaningfully better | **dramatically better** |
| RSS | baseline | slightly better | **much better** |
| Steady-state throughput | baseline | **unchanged** | **likely worse — measure** |
| Build time | baseline | +one training run | **much worse** |
| Reflection configuration work | none | none | **substantial and ongoing** |
| Observability (Topics 74–82) | full | **full** | **substantially reduced** |
| Team retraining | none | almost none | real |
| Risk of a runtime-only failure | none new | none new | **new and permanent** |

**Fill in the first two columns with your own measurements. Fill in the third only after
the drill.** The table's shape is the argument; the numbers are yours.

### The recommendation, and how to phrase it

> "`orderflow` starts once and runs for weeks. Startup is not on our critical path — our
> deploy pain is a rolling-update configuration problem plus twelve seconds of class
> loading, and AppCDS addresses the second in an afternoon without changing our runtime,
> our tooling, or our on-call procedures. Native image optimises startup at the cost of
> steady-state throughput, which is the number our SLO is written against. I built it,
> here are the three columns, and here is the reflection configuration it needed. I'd
> revisit this if we move to scale-to-zero or if we ship a CLI."

**Note the structure: you built it, you measured it, and you left the door open.** That is
what makes the "no" credible rather than conservative.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — reaching for native image on a 12-second startup without measuring where the time goes

**Wrong approach.** Startup is slow → native image starts fast → adopt native image. The
logic is valid and the premise is unexamined.

**Exact symptom.** Two engineers spend six weeks on reflection configuration. The binary
finally boots. Startup is indeed dramatically faster. **And the deploy is barely quicker,
because eight of the twelve seconds were Flyway migrations and a downstream health check
that native image does not touch.** Meanwhile steady-state throughput is down against the
Topic 65 baseline and nobody can profile it any more.

**Root cause.** "Startup is slow" is a symptom, not a diagnosis. Startup time in a Spring
service decomposes into JVM start, class loading and verification, context building, and
your own initialisation — plus anything you do synchronously at boot. **Native image
addresses the first three. It addresses none of the fourth.**

**Fix.** Measure first, every time:

```bash
curl -s localhost:8080/actuator/startup | jq '.timeline.events | sort_by(.duration) | reverse | .[0:20]'
java -Xlog:class+load:file=/tmp/cl.txt -jar orderflow.jar && wc -l /tmp/cl.txt
time java -version                      # the JVM's own floor
```

Then apply the cheapest thing that targets the actual dominant cost. **In practice the
order is: (1) move migrations and blocking downstream checks out of the startup path, (2)
AppCDS or the AOT cache, (3) reduce component scanning, (4) only then consider native
image.** Steps 1–3 keep the JVM and cost days rather than months.

**This trap connects Topics 82 and 122 directly.** A `startupProbe` with a sane threshold
and a fixed rolling-update `maxUnavailable` solves the actual deploy pain in most cases.

---

### Trap 2 — a native image failing at runtime on reflection that worked on the JVM

**Wrong approach.** Build it, see it start, see the smoke test pass, ship it.

**Exact symptom.** Everything works. Then, days later, one endpoint fails —
`ClassNotFoundException` for a class that is unambiguously in your source tree, or a
Jackson error deserialising a subtype that has always worked. The failure is **specific to
one code path** and did not appear in any test.

**Root cause.** Closed-world analysis dropped the class because nothing statically reachable
referenced it. The tracing agent never saw that path, so it never wrote the configuration.
**Your test coverage silently became your binary's correctness.** The path that fails is
usually a rare one: a payment-failure branch, an admin endpoint, a polymorphic Jackson
subtype for a payment method that only 2% of traffic uses.

**Fix, in order.**

```bash
# 1. Read the exception. Modern tooling often names the exact fix.
#    "add <name> to reflect-config" is a gift; take it.

# 2. Regenerate config with the agent, exercising MORE paths.
#    Run your FULL test suite under the agent, not a smoke test.
java -agentlib:native-image-agent=config-output-dir=native-config -jar target/orderflow.jar
mvn test -DargLine="-agentlib:native-image-agent=config-merge-dir=native-config"

# 3. Run the Topic 65 LOAD PROFILE under the agent. This is the highest-value step
#    and almost nobody does it - it exercises the realistic mix, not the happy path.
java -agentlib:native-image-agent=config-merge-dir=native-config -jar target/orderflow.jar &
k6 run --vus 200 --duration 15m load/orderflow-baseline.js

# 4. Diff the configuration before and after. The delta is what you were missing.
git diff native-config/
```

**The durable fix is a process one:** the native binary must be exercised by the *same*
load profile and the *same* integration suite as the JVM build, in CI, before it can ship.
**"It started" is not a test.** And the honest thing to tell your team: **this obligation
never goes away.** Every new Jackson subtype, every new Spring Data repository, every new
resource file is a potential runtime failure, forever.

---

### Trap 3 — believing the throughput number from a hello-world benchmark

**Wrong approach.** Citing a blog post: "native image handled 40k requests per second".
Building a proof-of-concept with one endpoint returning a fixed string, measuring it, and
generalising.

**Exact symptom.** The proof-of-concept looks excellent. The real service, with Hibernate,
a connection pool, Jackson over real DTOs, and the Topic 65 dataset, does not reproduce it.
Steady-state throughput is below the JVM baseline, and there is no obvious reason, and no
profiler to find one.

**Root cause.** A hello-world handler has almost no hot code. There is nothing for C2 to
optimise, so the JVM's advantage — profile-guided optimisation of hot paths — never
materialises, and the comparison collapses to startup and RSS, where native image wins.
**Real services are the opposite case: lots of hot code, deeply polymorphic framework
layers, and exactly the situation where a receiver-type profile is worth a great deal.**
The published benchmark measured the case least like yours.

**Fix.** Only ever compare like with like:

```bash
# 1. Same dataset. 100k products, 1M orders, 5M lines. Not a seeded 100-row database.
# 2. Same load profile. The unchanged Topic 65 k6 script.
k6 run --vus 200 --duration 30m load/orderflow-baseline.js

# 3. Same container limits, both sides.
docker run --cpus=2 --memory=2g orderflow-jvm:latest
docker run --cpus=2 --memory=2g orderflow-native:latest

# 4. STEADY STATE, not the first minute. The JVM needs warm-up (Topic 74);
#    measuring it cold is measuring the thing you already know is slower.
#    Discard the first 10 minutes on the JVM side. Say that you did.

# 5. Compare against the RECORDED baseline, not against a fresh run of the JVM
#    on a machine that is also running a native-image build.
cat /docs/java/baselines/orderflow-p99.json
```

**And if native image does win on your workload, say so.** The point is not that native
image is bad. The point is that the number must be yours.

---

### Trap 4 — build-time initialisation silently capturing state

**Wrong approach.** Accepting the framework's build-time initialisation defaults, or adding
`--initialize-at-build-time` broadly to make an analysis error go away.

**Exact symptom.** **No exception at all.** The binary works. Then something is subtly
wrong: every pod reports the same hostname; a cache is pre-populated with data from the
build machine; a "generated at" timestamp is the build date; an idempotency key sequence
repeats across pods; a `SecureRandom` produces the same first values everywhere.

**Root cause.** A static initialiser ran during the build and its results were snapshotted
into the image heap. **Everything it captured is frozen at build time and shared by every
instance of the binary.** For anything security- or identity-related this is a correctness
and security incident, not a performance quirk.

**Fix.**

```bash
# Force run-time initialisation for anything that captures environment, entropy,
# time, network identity, or an open resource.
native-image ... \
  --initialize-at-run-time=com.orderflow.security.IdempotencyKeyFactory \
  --initialize-at-run-time=com.orderflow.config.HostIdentity

# When the analysis complains about a class being initialised at build time
# unexpectedly, ask WHY rather than adding the flag that silences it:
native-image ... -H:+TraceClassInitialization
```

**The review rule that catches this:** any static field holding a value derived from time,
randomness, the environment, the network, the filesystem, or a connection is a
run-time-initialisation candidate. **Write that rule down; it is short and it prevents a
category of incident that produces no stack trace.**

---

### Trap 5 — discovering during an incident that the tools are gone

**Wrong approach.** Evaluating native image purely on startup, RSS and throughput, and never
asking what happens at 3am.

**Exact symptom.** A latency incident on the native build. Your instincts, built over
Topics 66–82, all fail in sequence: `jcmd` does not apply, there is no thread dump in the
form you know, `PrintCompilation` has nothing to print because there is no JIT, and
async-profiler's Java modes do not work the way you have practised. **You are debugging a
production incident with tooling you have never used.**

**Root cause.** The observability cost of native image is real, version-dependent, and
routinely omitted from evaluations because it does not show up in a benchmark.

**Fix — make it an explicit, tested column in the evaluation:**

```bash
# Before adopting anything, answer each of these ON YOUR VERSION, with a command,
# and write the answer in the design doc:
#   - Can I get a thread dump from a hung native orderflow?
#   - Can I get a heap dump, and can MAT open it?           (Topic 79)
#   - Does JFR work, and which events?                       (Topic 78)
#   - Can I profile CPU, and with what?
#   - Can I see GC activity, and with which flags?           (Topics 70-72)
#   - How do I change a runtime setting without a rebuild?
native-image --help
native-image --expert-options-all | grep -i -E 'jfr|heapdump|monitoring'
```

**Then rehearse an incident on the native build before you ship it.** If nobody on the team
can diagnose a simulated hang on the binary, you are not ready, regardless of the
throughput numbers. **Support for monitoring in native images has improved substantially
and continues to; check your version rather than trusting either the pessimism or the
optimism of any document, including this one.**

---

## Hands-on proof

### Setup, and the four facts every result depends on

```bash
mkdir -p ~/java-lab/83 && cd ~/java-lab/83
java --version               # GraalVM version and JDK version
native-image --version
uname -m                     # aarch64 on Apple Silicon
# Community Edition or Oracle GraalVM? This determines PGO availability.
```

**Write those four down.** A native-image result without them is not reportable, and the
CE/Oracle distinction changes what options you have.

### Proof 1 — what does the build actually do?

```bash
time native-image -cp out --no-fallback -o payment-registry \
     com.orderflow.lab.aot.PaymentHandlerRegistry 2>&1 | tee build.log
```

| What to record | Where from | Why it matters |
|---|---|---|
| Wall-clock build time | `time` | Determines whether your inner loop survives |
| Peak build memory | the `@ <n>GB` annotations | Determines whether CI can run it |
| Reachable types / methods | the analysis summary | The size of the closed world |
| Entries registered for reflection | the summary | Your configuration burden, quantified |
| Binary size | `ls -lh payment-registry` | The deployment artefact |

### Proof 2 — confirm it is a real image, not a fallback

```bash
file payment-registry
./payment-registry --version 2>/dev/null || echo "no JVM launcher behaviour"
ls -lh payment-registry
```

**WHAT TO LOOK FOR:** `file` should report a Mach-O (macOS) or ELF (Linux) executable for
your architecture. **If the build printed anything about a fallback image, the result is
worthless for this topic** — rebuild with `--no-fallback`.

### Proof 3 — measure startup properly

```bash
# Naive: one run each. WRONG, and you should be able to say why.
time java -cp out com.orderflow.lab.aot.PaymentHandlerRegistry
time ./payment-registry

# Better: many runs, look at the distribution, and use a tool built for it.
hyperfine --warmup 3 --runs 50 \
  'java -cp out com.orderflow.lab.aot.PaymentHandlerRegistry' \
  './payment-registry'
```

`hyperfine` reports mean, standard deviation, min and max. **A single `time` measurement of
a process start is dominated by filesystem cache state, other load on the machine, and
scheduling.** One number is not a measurement.

| What you see | What it means |
|---|---|
| A large, consistent gap favouring the binary | Expected. Record it as *your* number, with the four setup facts attached |
| A small gap | Your JVM case is trivial — a tiny program starts a JVM quickly. Real services are the interesting case |
| High variance on either side | Machine noise, or thermal throttling on a laptop. Close everything, re-run, report the spread |
| The binary slower | Very surprising. Check `file`, check `--no-fallback`, check you are not on a first run reading from cold storage |

### Proof 4 — measure resident memory, fairly

```bash
# macOS
/usr/bin/time -l ./payment-registry 2>&1 | grep -i 'maximum resident'
/usr/bin/time -l java -cp out com.orderflow.lab.aot.PaymentHandlerRegistry 2>&1 | grep -i 'maximum resident'

# Linux / in a container
/usr/bin/time -v <cmd> 2>&1 | grep -i 'Maximum resident'
docker stats --no-stream <container>
```

**The fairness rule:** measure the JVM with **realistic flags**, not defaults. A JVM at
default `MaxRAMPercentage` in a 2 GiB container reserves far more than it needs (Topic 82).
Compare `-Xmx` sized from the measured live set (Topic 68) against the native image, and
say in your write-up which JVM configuration you used. **Anything else is a rigged
comparison, and someone will notice.**

### Proof 5 — the reflection failure and its fix

Run Example 1's Steps 4 and 5. This is the mechanism, and it takes ten minutes.

### Proof 6 — what survives in the binary

```bash
# Which classes made it in? The build can tell you.
native-image ... -H:+PrintAnalysisCallTree
# and, for the reachability question directly:
native-image ... --diagnostics-mode

# What does the binary link against?
otool -L payment-registry     # macOS
ldd payment-registry          # Linux
```

**WHAT TO LOOK FOR:** a short list of system libraries. **This is the deployment win** — a
`FROM scratch` or distroless container image with no JDK in it. Weigh it honestly against
the fact that your container base image was probably not your problem.

---

## Failure drill

### The assignment, restated from the master plan

> Build `orderflow` as a native image. Find the first reflection failure at runtime. Fix it
> with reachability metadata. Compare startup, RSS and **steady-state throughput** against
> the JVM baseline.

**What the drill proves:** that closed-world analysis moves errors from build time to run
time, and that the steady-state comparison — the one that decides the question — usually
does not favour the binary.

**The honesty rule that governs this drill:** you are going to spend a week on this, and
the most likely correct conclusion is "we should not do this". **A drill whose conclusion
is negative is a successful drill.** Budget your ego accordingly.

### Step 0 — record the control

```bash
java --version && native-image --version && uname -m
cat /docs/java/baselines/orderflow-p99.json     # the numbers you must beat
docker images orderflow:latest --format '{{.Size}}'
```

Also record, from the current JVM build: startup time, RSS at steady state under load, and
throughput at the baseline load mix. **If you do not have these recorded from Topic 65, get
them before you build anything.** You cannot compare against a baseline you do not have.

### Step 1 — make the build work at all

For a Spring Boot service use the framework's native support rather than raw
`native-image`; it generates a large amount of reachability metadata for you.

```bash
# Spring Boot's native profile. Confirm the plugin and goal names against the
# Spring Boot reference documentation for YOUR Boot version - these have changed.
mvn -Pnative native:compile -DskipTests

# Time it. This number goes in the report.
time mvn -Pnative native:compile -DskipTests
```

| What you see | What it means |
|---|---|
| The build succeeds | Good. Note the time and the peak memory; both are costs |
| The build fails on an unsupported feature | Read it carefully — this may be a hard blocker (a library doing dynamic class definition), which is a *finding*, not a setback |
| The build runs out of memory | Raise the build container's memory. Note the requirement; your CI may not have it |
| The build takes longer than your patience | **That is data.** Record it. It is the inner-loop cost |

### Step 2 — start it, and find the first failure

```bash
./target/orderflow

# If it starts, exercise it - do not stop at "it started".
curl -s localhost:8080/actuator/health
curl -s localhost:8080/api/products?page=0\&size=20
curl -s localhost:8080/api/orders/1
curl -s -XPOST localhost:8080/api/orders -H 'content-type: application/json' \
     -d '{"customerId":1,"lines":[{"productId":42,"quantity":2}]}'
```

**WHAT TO LOOK FOR:** the first endpoint that behaves differently from the JVM build.
Capture the **exact** exception and stack trace. **This artefact is the point of the
drill** — it is the evidence that a build-time-clean binary fails at run time.

| Where it typically first breaks | Why |
|---|---|
| A Jackson polymorphic DTO | Subtypes resolved by name at runtime |
| A Spring Data repository method | Dynamic proxy over an interface (Topic 40) |
| A Hibernate entity or a lazy proxy | Runtime-generated subclass (Topic 49) |
| A resource load — a migration, a message bundle, a logging config | Loaded by string path |
| A `@ConfigurationProperties` binding | Reflective setter/record-component access (Topic 43) |

### Step 3 — fix it with reachability metadata

```bash
# 1. Run the JVM build under the tracing agent, exercising the failing path.
java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image \
     -jar target/orderflow.jar &
# ... exercise the failing endpoint ...

# 2. Better: run the FULL integration suite under the agent, merging config.
mvn verify -DargLine="-agentlib:native-image-agent=config-merge-dir=src/main/resources/META-INF/native-image"

# 3. Best: run the Topic 65 LOAD PROFILE under the agent. Realistic mix, realistic paths.
java -agentlib:native-image-agent=config-merge-dir=src/main/resources/META-INF/native-image \
     -jar target/orderflow.jar &
k6 run --vus 50 --duration 10m load/orderflow-baseline.js

# 4. Inspect and commit the generated config. Read it - do not just commit it.
git diff src/main/resources/META-INF/native-image/

# 5. Rebuild and re-test.
mvn -Pnative native:compile -DskipTests
```

**Record how many rounds of build → fail → configure → rebuild it took, and how long each
round was.** That loop time is one of the most important findings in the drill and it is
never in a benchmark.

### Step 4 — the comparison, run honestly

Both artefacts, same container limits, same dataset, same k6 script.

```bash
docker run --rm --cpus=2 --memory=2g -p 8080:8080 orderflow-jvm:latest
docker run --rm --cpus=2 --memory=2g -p 8080:8080 orderflow-native:latest

# STARTUP: many runs, not one.
hyperfine --warmup 2 --runs 20 \
  'docker run --rm --cpus=2 --memory=2g orderflow-jvm:latest --exit-after-start' \
  'docker run --rm --cpus=2 --memory=2g orderflow-native:latest --exit-after-start'

# RSS: at steady state under load, not at boot.
docker stats --no-stream

# STEADY-STATE THROUGHPUT: the number that decides the question.
k6 run --vus 200 --duration 30m load/orderflow-baseline.js
```

**The measurement rule that makes or breaks this drill: discard the JVM's warm-up window.**
The JVM is slower for its first minutes by design (Topic 74). Including that window makes
native image look better and answers a question nobody asked — you already know the JVM
starts cold. **Run for 30 minutes, report the last 20, and say in the write-up that you did
it.** If you do not say it, your reader cannot trust the number.

### Step 5 — what to capture

| Artefact | Command | Why |
|---|---|---|
| GraalVM / JDK / arch / edition | `native-image --version`, `uname -m` | Every result is conditional on these |
| Build time and peak build memory | `time mvn -Pnative native:compile` | The inner-loop and CI cost |
| Binary and container image size | `ls -lh`, `docker images` | The deployment win |
| **The first runtime reflection failure, verbatim** | copy the stack trace | **The core artefact of the drill** |
| Number of build→fail→configure rounds | your notes | The configuration burden, quantified |
| The generated reachability metadata | `git diff META-INF/native-image/` | What the analysis could not see |
| Startup distribution, both builds | `hyperfine --runs 20` | Not one `time` run |
| RSS at steady state, both builds | `docker stats` | With a **tuned** JVM heap, not defaults |
| **Throughput and p50/p95/p99, last 20 min of 30** | k6, vs `/docs/java/baselines/` | **The decision number** |
| The observability answers | Trap 5's checklist | The cost nobody scores |

### Step 6 — how to read it

| What you see | What it means |
|---|---|
| Startup dramatically better, RSS better, **steady-state throughput worse** | **The expected result.** Write it up exactly like that. This is the professional deliverable |
| Startup better, throughput **equal** | Plausible — `orderflow` may be database-bound rather than CPU-bound at your load. **Say so**, and note that the JVM's advantage is invisible when the bottleneck is elsewhere. This weakens the argument against native image, honestly |
| Throughput **better** on native | Surprising. Check: did you discard JVM warm-up? Is the JVM heap tuned? Are both under the same CPU limit? If it survives those checks, **report it** — it is a real finding about your workload |
| Startup barely better | Your startup was not JVM-bound. **Go back to Trap 1** — you measured the wrong thing at the start |
| RSS barely better | Compare against a *default* JVM to see the difference; the gap you did not find is the gap a public benchmark would have shown you |
| p99 worse but throughput equal | Interesting and worth digging into — but note that you now have far fewer tools to dig with. **That itself is a finding** |

### Step 7 — the write-up

One page. Sections: what I measured, on what, how; three columns (JVM tuned / JVM+AppCDS /
native); the reflection configuration burden with round count; the observability answers;
recommendation; and **what would change my recommendation**.

**The last section is what makes it engineering rather than opinion.** For `orderflow` the
honest triggers are: moving to scale-to-zero, a per-invocation billing model, shipping a
CLI, or GraalVM PGO becoming available and cheap enough to close the throughput gap on your
workload.

---

## Measurement

### The instruments, and what each is authoritative for

| Question | Instrument | Notes |
|---|---|---|
| Startup time | `hyperfine --warmup 3 --runs 20+` | **Never a single `time` run** |
| Where startup time goes (JVM) | `/actuator/startup`, `-Xlog:class+load` | Do this **before** deciding anything |
| RSS | `/usr/bin/time -l` (macOS), `docker stats` | Compare against a **tuned** JVM |
| Steady-state throughput | The Topic 65 k6 profile, 30 min, report last 20 | **The decision number** |
| Latency percentiles | k6 summary vs `/docs/java/baselines/` | Same dataset, same limits |
| Build cost | `time`, peak-memory annotations in the build log | The cost that never appears in talks |
| Configuration burden | round count + `git diff` of the metadata | Quantify it or it will be dismissed |
| Binary/image size | `ls -lh`, `docker images` | Real, and usually the least important win |

### Why a naive `System.nanoTime()` measurement is WRONG here

Two distinct measurements are at stake, and the naive approach ruins both differently.

**For startup**, the naive approach is a single `time java -jar app.jar` versus a single
`time ./app`. It is wrong because:

1. **One sample is not a measurement.** Process start is dominated by filesystem cache
   state, other load, and scheduler luck. Use `hyperfine` with warm-up and 20+ runs, and
   report the spread.
2. **You may be measuring cold page cache.** The first run of a large binary reads from
   disk. `--warmup 3` exists for this.
3. **"Started" is ambiguous.** JVM start, Spring context ready, and first-request-served are
   three different events. **Pick one, define it, and measure the same one on both sides.**
   Most disagreements about startup numbers are two people measuring different events.

**For throughput**, an in-process `System.nanoTime()` loop is wrong for every reason Topic
77 gives, plus one specific to this topic:

4. **It measures the wrong runtime state.** On the JVM the first executions are interpreted
   and then C1-compiled (Topic 74); on the native image there is no such ramp. A naive
   timing loop averages the JVM's warm-up into its result and systematically flatters the
   binary. **The comparison you want is warm JVM versus native image**, and only a
   long-running load test at steady state gives you that.
5. **Dead-code elimination and constant folding** apply on the JVM side and not identically
   on the native side, so the same broken microbenchmark is broken *differently* in the two
   runtimes — which can produce a large, entirely artificial difference.

**The correct instrument for the throughput comparison is the Topic 65 load profile against
the containerised service, run long enough to reach steady state, with the JVM's warm-up
window explicitly discarded and that discarding stated in the write-up.** JMH is the right
tool for a method; it is the wrong tool for this question.

### What to track if you do ship a native image

| Number | Where from | Why |
|---|---|---|
| Startup time per pod | Deploy pipeline / k8s events | The thing you bought |
| RSS per pod | k8s metrics | The other thing you bought |
| Throughput and p99 vs baseline | Your APM, vs `/docs/java/baselines/` | The thing you risked |
| **Runtime reflection failures** | Error-rate alert on `ClassNotFoundException` and reflection errors | **The new failure class you introduced.** Alert on it explicitly, forever |
| Build time and build memory | CI metrics | The inner-loop tax; watch it grow |
| Reachability metadata size | A CI check on the file | Growth is your maintenance burden accumulating |

**The fourth row is not optional.** Adopting native image means accepting a permanent new
category of production failure. If you do not alert on it specifically, you will find it
via a customer.

---

## Practice exercises

### 1 — Easy: build the decision fact sheet, without building an image

Produce `~/java-lab/83/decision.md` containing only things you measured or read:

- GraalVM/JDK/arch/edition on your machine, from `--version`.
- `orderflow`'s startup time decomposed from `/actuator/startup` — the top ten steps by
  duration.
- The class-load count from `-Xlog:class+load`.
- `time java -version` — the JVM's own floor.
- Which of the following your twelve seconds is actually made of: JVM start, class loading,
  context building, Flyway, downstream health checks, your own initialisation.
- One sentence: **which intervention targets the dominant cost**, chosen from (a) moving
  work out of startup, (b) AppCDS/AOT cache, (c) less component scanning, (d) native image.

**Success criterion:** you can answer "where do the twelve seconds go?" with a
`/actuator/startup` extract rather than a guess — and in most cases you will find the
answer is not (d).

### 2 — Medium: the AppCDS comparison (combines Topics 65, 67, 82, 122)

Do the cheap intervention properly and measure it, before touching GraalVM.

1. Produce an AppCDS archive with a training run that exercises the real startup path.
2. Verify the archive is actually used with `-Xshare:on` (which fails loudly if not).
3. Measure startup with `hyperfine --runs 20`, with and without.
4. Measure RSS and steady-state throughput with and without. **Confirm throughput is
   unchanged** — that is the point of the option.
5. Containerise both (Topic 122) and repeat inside the container at `--cpus=2 --memory=2g`.
6. Investigate whether your JDK offers the newer AOT cache, using the JEP index rather than
   guessing flags, and record what you found — including "the flags in my JDK do not match
   what I expected", if that is the answer.

**Success criterion:** a two-column table showing what AppCDS bought and what it cost, on
your machine, for `orderflow` — and enough confidence to say "we should do this first"
regardless of the native-image decision.

### 3 — Hard: production simulation — the full native evaluation

The failure drill, executed end to end and written up as a decision document.

1. Build `orderflow` native. Record build time, build memory, round count.
2. Capture the **first runtime reflection failure verbatim**. Fix it with reachability
   metadata generated from the Topic 65 load profile, not from a smoke test.
3. Run both builds under identical container limits and the identical k6 profile for 30
   minutes; report the last 20.
4. Produce the three-column table: JVM tuned / JVM + AppCDS / native. Rows: startup, RSS,
   throughput, p50/p95/p99, build time, image size, configuration burden, observability,
   new failure classes.
5. Answer Trap 5's observability checklist **with commands you actually ran** on the native
   build.
6. Write a one-page recommendation with a "what would change my mind" section.
7. **Then present it to someone and let them attack it.** The most common successful attack
   is "your JVM wasn't tuned" — pre-empt it by stating your `-Xmx` and how you chose it.

**Success criterion:** a document that would survive a staff-engineer review, whose
conclusion is supported by numbers you generated, and which is comfortable saying "worse"
about the fashionable option. **Reaching the opposite conclusion is equally a pass if the
numbers support it. Fabricating either is the only failure.**

---

## Interview questions

### Q1 — "Should we move to native image?"

**MID-LEVEL.** "Yes — it starts in milliseconds and uses much less memory, so our pods are
cheaper and deploys are faster."

**SENIOR.** "It depends entirely on the workload, and for a long-running service my prior is
no. Native image buys startup time and resident memory. It costs peak throughput, because
you lose C2's profile-guided optimisation — the static compiler has the class hierarchy but
not the receiver-type profile or branch frequencies, and it can't speculate and deoptimize.
It also costs build time, an ongoing reflection-configuration burden, and a large part of
our observability: no `jcmd`, no thread dumps in the form we know, no `PrintCompilation`,
and JFR and heap-dump support that varies by version and needs verifying. So the first
question is what our workload is. If we're serverless, or scale-to-zero, or shipping a CLI,
startup is the business metric and I'd evaluate it seriously. `orderflow` runs 24/7 behind
a load balancer and its SLO is written against p99 and throughput — the things native image
puts at risk — so I'd instead measure where our startup time actually goes and probably
find that AppCDS or the JDK's AOT cache gets a large share of the win in an afternoon,
while keeping the JIT and every tool we own. I'd build the native image anyway, so the
comparison is ours rather than a blog post's."

**What separates them:** the senior answer names the **cost** side of the ledger in
specifics, ties the decision to a workload shape, and proposes the cheaper alternative
before the expensive one. It also commits to building it — the recommendation is not
laziness dressed as caution.

**Follow-up:** *"What would change your answer?"* → Moving to scale-to-zero or
per-invocation billing; shipping a CLI or an operator; a memory-constrained edge
deployment; or PGO becoming cheap enough on our distribution and licence to close the
throughput gap on our own workload. **Not** a benchmark someone else ran.

---

### Q2 — "It works on the JVM and fails at runtime in the native image. What happened, and how do you prevent it?"

**MID-LEVEL.** "Native image doesn't support reflection, so you have to add it to a config
file."

**SENIOR.** "It supports reflection; it just can't *discover* it. The build does closed-world
points-to analysis from a set of roots, and anything not proven reachable is not in the
binary — not lazily loaded, absent. `Class.forName` on a computed string, a dynamic proxy, a
resource loaded by path: the analysis can't follow any of them, so the target isn't marked
reachable and gets dropped, and the build still succeeds. That's the defining property of
the technology — **the error moves from build time to run time**, which is the wrong
direction. The fix for the immediate failure is reachability metadata, usually generated
with the tracing agent on a JVM run. The important part is *how* you generate it: the agent
only records what your run did, so if you generate it from a smoke test, your binary is
correct only on the paths the smoke test touched. I'd run the full integration suite and the
Topic 65 load profile under the agent, so the configuration reflects the realistic traffic
mix rather than the happy path. And I'd make it permanent: the native build has to pass the
same integration suite and the same load profile in CI, and we'd alert specifically on
reflection failures in production, because we've introduced a failure class that didn't
exist before and it never goes away — every new Jackson subtype is a candidate."

**What separates them:** the senior answer corrects the misconception ("supports but cannot
discover"), names the mechanism, and — decisively — identifies that **the agent's coverage
becomes the binary's correctness**, which is the non-obvious operational consequence. It
also treats the fix as a permanent process obligation rather than a one-time file.

**Follow-up:** *"What's worse than a `ClassNotFoundException` here?"* → A build-time
initialisation problem, which produces **no exception at all**. A static initialiser runs
during the build and snapshots its result into the image heap, so every deployed instance
shares the build machine's timestamp, hostname or random seed. That is a correctness and
sometimes security incident with no stack trace.

---

### Q3 — "Why would peak throughput be lower without a JIT? Isn't ahead-of-time compilation strictly better?"

**MID-LEVEL.** "AOT compiles everything up front so there's no interpretation, which should
be faster — maybe the Graal compiler is just not as good as C2."

**SENIOR.** "It's not about compiler quality, it's about information. C2 compiles a method
after watching it run thousands of times, so it has a receiver-type profile per call site,
branch frequencies, and a record of which paths never execute. It uses that to inline
aggressively behind a cheap guard — or with no guard at all when class-hierarchy analysis
says one implementation exists — to lay hot code out straight-line, and to speculate on
things it can't prove, because it has an escape hatch: if the speculation is violated it
deoptimizes and recompiles. A static compiler has none of that. It has the hierarchy but not
the profile, and it has no escape hatch, so it must be correct for every possible execution.
Less inlining means weaker escape analysis downstream, which means allocations that would
have been scalar-replaced now happen for real. That's the mechanism. The mitigation is
profile-guided optimisation — run the binary under a representative load, collect a profile,
rebuild with it — which genuinely narrows the gap. Two caveats: PGO availability depends on
your GraalVM distribution and licence, and it requires a representative load profile, which
is the same artefact as our load test. Teams that won't build a load profile won't do PGO,
and those are the teams most disappointed by the result."

**What separates them:** the senior answer locates the difference in **information rather
than compiler quality**, names the specific optimisations lost, connects it to escape
analysis from Topic 75, and knows the mitigation *and* why the mitigation is often not
applied. The mid-level answer's "Graal isn't as good as C2" is a wrong model that leads to
wrong decisions.

**Follow-up:** *"So would you expect native image to be slower on every workload?"* → No. On
a workload with little hot code — a short-lived function, a CLI — there is nothing for C2 to
optimise, so the JVM's advantage never materialises and the comparison collapses to startup
and memory, where native image wins outright. **The advantage of a JIT scales with how long
and how hot you run.**

---

### Q4 — "Our startup is twelve seconds. Walk me through what you'd do."

**MID-LEVEL.** "I'd look at native image, or trim dependencies to reduce classpath scanning."

**SENIOR.** "I'd measure where the twelve seconds goes before proposing anything, because
'startup is slow' decomposes into at least five different problems with five different
fixes. Boot's `/actuator/startup` endpoint gives a per-step timeline; `-Xlog:class+load`
gives the class-loading volume; `time java -version` gives the JVM's own floor. Then the
answer follows the finding. If it's Flyway migrations or a blocking downstream health check
at boot, that's not a JVM problem at all and native image would change nothing — move it out
of the startup path. If it's class loading, verification and context building, that's exactly
what AppCDS was built for: two commands, keeps the JIT, keeps every tool, verify with
`-Xshare:on` because it fails loudly if the archive isn't used. The JDK's AOT cache is the
newer relative of that — I'd read the JEPs for our JDK rather than guess flags, because
they've moved between releases. If it's component scanning, narrow the scan. Native image is
last, and separately I'd ask whether twelve seconds is even the problem — if the pain is
slow rolling deploys, a properly configured `startupProbe` and `maxUnavailable` may fix it
without touching startup at all."

**What separates them:** the senior answer refuses to accept the framing. "Startup is slow"
is a symptom; the answer starts by decomposing it, and half the plausible findings make the
proposed solution irrelevant. It also connects to the deploy-shape question, which is
frequently the real problem behind the request.

**Follow-up:** *"AppCDS gave us a modest improvement and the team still wants native image.
What now?"* → Then build it and measure it, with the same dataset, the same container
limits, the same load profile, and the JVM's warm-up window explicitly discarded. Put
throughput, p99, build time, configuration burden and observability in the table alongside
startup and RSS. **Let the numbers decide, and be genuinely willing to be wrong** — but
insist the numbers are ours.

---

### Q5 — "What do you lose operationally, and how would you find out before committing?"

**MID-LEVEL.** "Debugging is harder because there's no JVM, so you'd rely more on logging."

**SENIOR.** "The specific losses are the ones I'd list in the design doc, each with a
question I'd answer by running a command on the actual version rather than reading a blog:
can I get a thread dump from a hung native `orderflow`, can I get a heap dump that MAT can
open, does JFR work and which events, how do I profile CPU, how do I see GC activity, and
how do I change a runtime setting without a rebuild. Support for several of these has
improved substantially and keeps improving, so the answers are version-specific and I
wouldn't trust either the pessimism or the optimism of any secondhand source. Beyond the
tooling, there are two structural losses: I can't attach an agent at runtime, and many
decisions are baked into the binary, so 'restart with a different flag' — which is how we
resolve a lot of incidents — is often not available. The way I'd find out before committing
is to rehearse an incident on the native build: simulate a hang and a memory problem in
staging and have someone who isn't me diagnose them using only what the native build offers.
If we can't do that, we're not ready to run it in production regardless of the throughput
numbers."

**What separates them:** the senior answer converts a vague concern into a **checklist of
answerable questions**, insists on verifying per version rather than generalising, and
proposes an *incident rehearsal* as the acceptance test. That last idea — that operational
readiness is something you test, not something you assert — is the mark of someone who has
been on call.

**Follow-up:** *"Isn't that over-cautious for a technology this mature?"* → The maturity
question is real and the answer is version- and workload-specific, which is why the
checklist is a set of commands rather than an opinion. **The rehearsal costs a day. An
incident you cannot diagnose costs considerably more, and it happens at the worst possible
time.**

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Closed-world analysis means the set of classes is fixed at build time. **Derive from
   that alone** why a `ClassNotFoundException` in a native image is fundamentally different
   from one on the JVM, and what that implies about which environment you must find it in.

2. The tracing agent records what your run *did*. Derive the consequence for a payment
   subtype used by 2% of traffic, and design the process change that closes the gap.

3. A static compiler has the class hierarchy but not the receiver-type profile. **Name three
   specific optimisations that become impossible without the profile**, and for each say what
   the observable consequence is on `orderflow`'s hot path. (Topics 74 and 75 are the source;
   do this without re-reading them.)

4. Build-time class initialisation snapshots the resulting heap into the binary. Construct
   the `orderflow` scenario in which this produces a **security incident with no exception**,
   and state the review rule that would have caught it.

5. AppCDS keeps the JIT; native image does not. Derive from that single difference why
   AppCDS cannot help steady-state throughput and cannot hurt it either — and why that makes
   it a fundamentally lower-risk change.

6. Your native image shows better startup, better RSS, and **equal** throughput to the JVM
   build. **List everything that rules in and everything it rules out**, and name the first
   thing you would check.

7. A published benchmark shows native image beating the JVM on requests per second. Without
   reading it, name the three most likely ways the comparison is not like-for-like, and the
   one question you would ask the author first.

---

## Quick reference card

### The model

```
native-image:
  points-to analysis from roots -> reachable set -> AOT compile -> one binary
  + SubstrateVM runtime (GC, threads, monitors, exceptions - the JMM still applies)
  - no class loading, no verification, no linking, NO JIT

CLOSED WORLD: not reachable => NOT IN THE BINARY.
  reflection / proxies / resources / Class.forName / serialization / JNI
    => must be DECLARED in reachability metadata
    => otherwise: BUILD SUCCEEDS, RUNTIME FAILS.   <- the whole topic

BUILD-TIME INIT: static initialiser runs at BUILD, heap snapshotted into the binary.
  captured time / hostname / entropy / handles are FROZEN and SHARED.
  fails with NO EXCEPTION.  <- the worst failure mode

WINS:   startup, RSS, image size, no JDK in the container
LOSSES: peak throughput (no profile), build time, reflection config forever,
        most of Topics 66-82's tooling

FOR A LONG-RUNNING SERVICE: AppCDS / AOT cache is usually the better trade.
```

### Commands

```bash
# Versions - every result is conditional on these four.
java --version && native-image --version && uname -m   # + CE or Oracle?

# Build. --no-fallback is not optional: it turns silent degradation into honest failure.
native-image -cp out --no-fallback -o <name> <MainClass>
time mvn -Pnative native:compile -DskipTests            # Spring Boot; confirm goal names

# Generate reachability metadata - ON THE ORDINARY JVM.
java -agentlib:native-image-agent=config-output-dir=<dir> -jar app.jar
mvn verify -DargLine="-agentlib:native-image-agent=config-merge-dir=<dir>"
# Best: run the Topic 65 LOAD PROFILE under the agent.

# Initialisation control.
--initialize-at-build-time=<class>
--initialize-at-run-time=<class>
-H:+TraceClassInitialization

# Is it a real image or a fallback launcher?
file <binary> && ls -lh <binary> && otool -L <binary>

# Startup, measured properly. NEVER a single `time` run.
hyperfine --warmup 3 --runs 20 'java -jar app.jar' './app'

# RSS. Compare against a TUNED JVM, not a default one.
/usr/bin/time -l ./app 2>&1 | grep -i 'maximum resident'
docker stats --no-stream

# The decision number: steady-state throughput, JVM warm-up discarded.
k6 run --vus 200 --duration 30m load/orderflow-baseline.js

# The cheaper alternative, first.
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar
java -Xshare:on -XX:SharedArchiveFile=app.jsa -jar app.jar   # :on fails loudly
curl -s localhost:8080/actuator/startup | jq '.timeline.events | sort_by(.duration) | reverse | .[0:20]'

# The AOT cache: read the JEPs for YOUR JDK, do not guess flags.
#   https://openjdk.org/jeps/0   (search "Ahead-of-Time", "Leyden")
java -XX:+PrintFlagsFinal -version | grep -i -E 'aot|cds|CacheDataStore'
```

### When native image wins / loses

| Wins | Loses |
|---|---|
| Serverless, per-invocation billing | Long-running services behind a load balancer |
| Scale-to-zero (Knative, KEDA) | Anything whose SLO is throughput or p99 |
| CLI tools and operators | Anything using heavy reflection or dynamic proxies |
| Short batch jobs (seconds) | Teams with a mature JVM observability practice |
| Hard memory ceilings at the edge | Teams that cannot afford a slow inner loop |
| Distroless/scratch container requirements | Anything needing runtime-configurable behaviour |

### Gotchas checklist

- [ ] Measure where startup time goes **before** proposing native image.
- [ ] `--no-fallback`, always, or you may be measuring a launcher.
- [ ] The agent only records paths you exercised. Run the **load profile** under it.
- [ ] Build-time initialisation freezes captured state and throws **no exception**.
- [ ] Compare against a **tuned** JVM heap, not a default one.
- [ ] Discard the JVM's warm-up window, and **say that you did**.
- [ ] Same dataset, same container limits, same k6 script, or it is not a comparison.
- [ ] Answer the observability checklist with commands **on your version**.
- [ ] Alert on reflection failures in production, permanently.
- [ ] AppCDS / AOT cache first: cheaper, keeps the JIT, cannot hurt throughput.
- [ ] Read the Leyden/AOT JEPs for your JDK rather than trusting any flag list, mine included.

---

## When would I use this at work?

**1. When someone proposes native image after a conference.**

You have a response that is neither dismissive nor credulous: *"Let's measure where our
startup time goes first — it takes twenty minutes — and let's price AppCDS, which keeps the
JIT and every tool we own. Then, if we still want to, I'll build the native image and we'll
compare steady-state throughput on our own dataset."* That reply is impossible to argue
with, costs almost nothing, and in most cases resolves the question in the first twenty
minutes. **Being the person who reframes an enthusiasm into an experiment is a large part of
what senior means.**

**2. When you genuinely do have a startup-sensitive workload.**

Scale-to-zero, a per-invocation billing model, a CLI your team ships, a Kubernetes operator,
a short batch job. Here native image is the right answer and you can now say *why* it is
right for this and not for the main service — which is a far more credible position than
being for or against the technology in general. You also know what it will cost you: the
configuration burden, the build time, and the observability gap, each of which you can plan
for instead of discovering.

**3. When you are writing the capacity or cost model.**

Someone will claim native image reduces the cluster bill because RSS is lower. You can check
it properly: is the JVM's memory actually tuned, or is the comparison against an
unconfigured default? Does the throughput change mean you need *more* pods, cancelling the
per-pod memory saving? **The interaction between lower memory per pod and lower throughput
per pod is the whole cost question, and it is routinely answered by looking at only one of
the two numbers.** Topic 129 is where that model lives; this topic gives you the inputs.

---

## Connected topics

**Prerequisites:**

- **19 — Serialization**: classes crossing Java serialization need explicit declaration in
  the image, and the CVE history is a reason to have removed it before you get here.
- **20 — JPMS and `jlink`**: the other way to ship a smaller runtime, keeping the JVM
  entirely. Worth pricing alongside native image; it is frequently forgotten.
- **40 — Proxying**: JDK dynamic proxies and CGLIB subclasses are generated at runtime,
  which is exactly what closed-world analysis cannot follow. Spring Data repositories and
  `@Transactional` are the first things to break.
- **42 — Auto-configuration**: conditional evaluation over the classpath is a build-time
  decision in a native image, which is why Spring's AOT support has to do so much work.
- **49 — Hibernate lazy proxies**: runtime-generated subclasses, same problem.
- **63 — Mutation testing**: your test coverage becomes the binary's correctness, which
  gives coverage a consequence it did not have before.
- **65 — The load baseline**: the only legitimate basis for the throughput comparison, and
  the right input to the tracing agent. `/docs/java/baselines/`.
- **67 — Class loading**: everything this topic removes. Read it again and note how much of
  it simply does not happen.
- **68–72 — Heap, GC, collectors**: the collector options available in a native image are a
  smaller set; check what your edition offers before assuming your tuning transfers.
- **74 — JIT I**: the profile-guided optimisation you are giving up. **This is the core of
  the throughput argument** and you cannot make it without Topic 74.
- **75 — JIT II**: escape analysis depends on inlining, which depends on the profile. Less
  profile means less inlining means more allocation — the second-order cost.
- **78–79 — Profiling and heap dumps**: the tooling whose availability you must verify per
  version. Most of the operational cost lives here.
- **81 — Instrumentation agents**: you cannot attach one at runtime. If your APM depends on
  a Java agent, that is a blocking question to answer early.
- **82 — JVM tuning and containers**: the fair-comparison rules, and the AppCDS alternative.
  **Trap 1 is a Topic 82 failure wearing a Topic 83 costume.**

**This unlocks:**

- **86–88 — The JMM**: unchanged in a native image. Same threads, same monitors, same
  happens-before edges, same publication bugs. **Nothing in Phase 9 becomes easier.**
- **92 / 94 — Concurrent collections and explicit locks**: same semantics, same correctness
  obligations, fewer tools to diagnose them with.
- **99 — jcstress**: still the right way to verify concurrency, and worth running against
  the same code you intend to ship natively.
- **101 — Virtual threads**: support is version-dependent; verify rather than assume, and
  note that a large part of virtual threads' value is throughput under blocking I/O, which
  interacts with everything in this document.
- **121 — Actuator and k8s probes**: a correctly configured `startupProbe` is frequently the
  actual fix for the pain that motivated the native-image proposal.
- **122 — Docker and startup**: the container-level view — layers, base images, and the
  `FROM scratch` deployment that native image enables.
- **129 — Capacity, cost and latency budgets**: where the memory-versus-throughput trade
  becomes a number on a cluster bill. **Do not let anyone model only the memory half.**

---

*Java baseline 21, running on JDK 25. Several things here are deliberately hedged rather
than asserted: the exact Leyden/AOT-cache flag names and workflow in your JDK (read the JEPs
at openjdk.org/jeps/0 — I will not guess flags that have moved), the current file names and
layout of reachability metadata, the precise state of JFR, heap-dump and monitoring support
in native images on your GraalVM version, whether PGO is available under your distribution
and licence, and the magnitude of every trade-off in this document. Each has a command or a
primary source that settles it. **No startup time, RSS figure, binary size, build duration,
throughput number or latency percentile in this document was measured — I have no JVM and no
GraalVM.** The build-report and error-message shapes shown are format illustrations with
`<n>` placeholders, labelled as such. The direction I will defend, and the reason this
document exists: native-image does closed-world static analysis at build time and emits a
binary containing only reachable code, so anything resolved at runtime must be declared or it
fails at runtime rather than at build time; it buys startup and memory; it costs peak
throughput, build time, ongoing configuration and most of your observability; and for a
long-running service the honest recommendation is usually AppCDS or the AOT cache instead —
a conclusion you should reach with your own numbers, and be willing to have overturned by
them.*
