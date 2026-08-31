# 83 — GraalVM native-image / AOT: What You Gain, What You Lose

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: build `orderflow` as a native binary, hit the first runtime reflection failure, fix it with reachability metadata, and then run the Topic 65 baseline against the native build — 100k products, 1M orders, 5M order lines, k6 at 70/20/10 — and compare **startup, RSS and steady-state throughput** against the recorded JVM baseline in `/docs/java/baselines/`. Be honest about the third number, because steady-state throughput is usually **worse**, and the whole professional value of this topic is being the person who says so with evidence instead of enthusiasm.

---

## R0 — READ THIS BEFORE ANY OTHER LINE IN THIS DOCUMENT

**I do not have GraalVM, a build machine, or a running `orderflow`. Nothing in this
document is captured tool output.**

Specifically, you will not find here:

- a startup time in milliseconds, an RSS figure, or a throughput number,
- a binary size, a container image size, or a build duration,
- "native-image starts in 50 ms and uses 60 MB" or anything shaped like it,
- a percentage by which steady-state throughput regresses.

**Why this rule matters especially here:** native-image is the most marketing-adjacent
topic in Phase 8. The internet is full of confident numbers from benchmarks whose
conditions are not stated, comparing a hello-world binary against a Spring Boot fat jar,
on hardware nobody names. If I add another set of invented numbers, I make the problem
worse and I rob you of the only defensible position available: **"here are the three
numbers I measured on our service, on our hardware, with our load."**

What you get instead, everywhere:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see" → "what it means"** table covering plausible and surprising outcomes.

### The labelled exception

To read reachability metadata you must know its shape. In two places I show the
**structure** of a JSON configuration file — key names and nesting — with every value
replaced by `<fully.qualified.Class>` or `<name>`, carrying the inline label:

> *illustration of the format, not captured output*

Structure only. No sizes, no timings, no counts.

### Spec-level facts I state plainly, each with a confirming command

1. **native-image performs closed-world static analysis at build time and emits a binary
   containing only reachable code.** Confirm: build with a build report enabled and read
   the reachable-methods section (Proof 4).
2. **Anything resolved at run time — reflection, dynamic proxies, resource loading,
   `Class.forName`, `ServiceLoader`, serialization — must be declared in metadata, or it
   fails at run time rather than at build time.** Confirm: the failure drill, which
   produces exactly that failure deliberately.
3. **There is no C2 at run time in a native image.** Confirm: `native-image --version` and
   the absence of any JIT-related `-XX` flags on the produced binary; and the
   throughput measurement in the drill.

### THE RULE

> **If your build output or your measurements disagree with anything in this document,
> YOUR OUTPUT IS THE TRUTH.** GraalVM moves fast; flags, file names and defaults change
> between releases. Where I am unsure I say so and give you the command, and there are
> more of those in this document than in any other in Phase 8 — deliberately.

---

## Mechanical statement

Read this three times.

> **`native-image` does closed-world static analysis at build time and emits a binary
> containing only the code it proved reachable.**
>
> "Closed world" means: **the set of classes, methods and fields that can ever execute is
> decided at build time and is final.** The builder starts from entry points — your `main`,
> plus anything explicitly registered — and performs a **points-to analysis**: for every
> reachable call site it computes the set of types that can flow to the receiver, which
> tells it which method implementations are reachable, which reaches more call sites, and
> so on, iterating to a fixed point.
>
> Whatever that analysis does not reach **is not in the binary at all.** Not "loaded
> lazily". Not "there but unused". Absent. That is where the small binary, the fast
> startup and the low memory come from, and it is not a heuristic — it is the whole design.
>
> **Therefore anything resolved at run time defeats the analysis.** `Class.forName(s)`
> where `s` is computed. A `Method` looked up by name. A JDK dynamic proxy built from an
> interface list assembled at run time. A resource loaded by a path from configuration. A
> `ServiceLoader` provider. Deserialization reconstructing a graph of types chosen by the
> input. In every case the analysis cannot see the target, so the target is not in the
> binary.
>
> **And the failure moves.** On the JVM these all work, because the JVM has the whole
> classpath and loads lazily on demand (Topic 67). In a native image the same code throws
> `ClassNotFoundException`, `NoSuchMethodException` or a missing-resource error — **at run
> time, in production, on the code path nobody exercised during the build.** The compiler
> did not warn you. It could not: from its point of view the code was unreachable, which
> is exactly what you asked it to assume.
>
> The fix is **reachability metadata**: JSON declarations that tell the builder "include
> this class, this method, this field, this resource, this proxy interface list, even
> though you cannot prove anyone calls it."
>
> **And the thing you give up is C2.** There is no JIT in the produced binary. There is no
> profile, no speculative devirtualization on an observed monomorphic call site, no
> uncommon trap, no recompilation when an assumption breaks. The compiler must be
> conservative because it has no run-time evidence. **So a long-running service typically
> reaches a LOWER peak throughput than the same code on HotSpot** — you traded the
> optimiser that learns for a binary that starts instantly.

The one-line version:

*"Native image makes Java **start** fast and use less memory. It does not make Java fast.
For a service that runs for weeks behind a load balancer, that is usually the wrong
trade."*

---

## The bridge from what you know

### There is NO ANALOGUE for this in your Node experience. Say it out loud.

This is the second topic in Phase 8 with no bridge (Topic 73 was the first), and it is
worth being precise about *why*, because you will reach for three candidates and all three
are misleading in ways that will cost you.

**Candidate 1: esbuild / webpack / Rollup bundling and tree-shaking.**

Superficially this looks like the same idea: a build-time analysis of what is reachable,
which discards what is not, producing a smaller artefact.

Three differences, and each one matters:

| | Bundler tree-shaking | native-image closed-world |
|---|---|---|
| **Failure mode when the analysis is wrong** | You ship a **bigger bundle** than necessary. A dynamic `require()` defeats tree-shaking and the bundler includes more, or warns | You ship a binary **missing a class**, and it throws in production on a path nobody tested |
| **Severity** | A performance nuisance | A **correctness failure**, at run time, discovered by a user |
| **What runs afterwards** | JavaScript, on V8, **with a JIT** | Ahead-of-time compiled machine code, **with no JIT** |
| **Escape hatch** | `require.context`, dynamic import hints, or just accept the bigger bundle | Explicit JSON metadata, which you must be exhaustive about |

The failure asymmetry is the whole point. A bundler that guesses wrong makes your download
slower. A closed-world analysis that guesses wrong takes down your payment endpoint.

**Verdict: NOT an analogue.** Structurally similar analysis, incomparable consequences.

**Candidate 2: `pkg`, `nexe`, Deno/Bun `compile`, or Node's Single Executable Applications.**

These produce a single binary containing your script and the runtime. That *looks* like
what native-image does.

**It is not, and the reason is the whole topic: those binaries embed V8, including its
JIT.** You get packaging convenience. You do not get faster startup in any meaningful
sense — V8 still parses and compiles your JavaScript at startup — and you certainly do not
give up the optimiser. Nothing about the peak-throughput trade is present.

**Verdict: NOT an analogue.** Same shape of artefact, none of the same trade-offs.

**Candidate 3: "ahead-of-time compilation" as a general idea.**

You have used AOT-compiled languages, or at least run programs written in them. But you
have never made *this particular trade*, because it has never been offered to you:

> **Node ships a JIT either way.** There has never been a decision point in your career
> where the question was "shall we give up profile-guided run-time optimisation in exchange
> for startup time and memory footprint?" That decision does not exist in the JavaScript
> ecosystem.

**That is the honest statement of the gap.** Not "this is like X but harder". There is no
X. The mental model you need — *a compiler that must decide everything now, without ever
seeing the program run* — has no counterpart in your experience, and you should build it
from scratch rather than by analogy.

### The one thing that does transfer, and it is small

**The instinct that a build-time analysis needs escape hatches for dynamism.** You have
written a `webpack.config.js` entry to force something into a bundle that the analysis
could not see. Reachability metadata is that instinct, applied with far more rigour and far
worse consequences for getting it wrong.

Use that instinct to accept the *existence* of metadata as normal engineering rather than
as a smell. Do not use it to estimate how much work it is, or how bad it is when you miss
something.

### What you must build fresh

| Concept | Why nothing transfers |
|---|---|
| Points-to analysis to a fixed point | Bundlers do syntactic reachability; this is a semantic type-flow analysis |
| Build-time class initialization and the image heap | Nothing in JS runs your module initialisers at build time and snapshots the resulting object graph into the binary |
| Losing the JIT | V8 always has one |
| Losing JVMTI, and therefore your profiler and your APM agent | No equivalent loss exists |
| Metadata as a maintained artefact that must track your dependencies | The closest thing is a bundler config, which is far smaller and far less consequential |

---

## What is this?

**GraalVM's `native-image` is an ahead-of-time compiler that takes your compiled Java
classes and dependencies and produces a standalone native executable.**

The executable contains:

- your reachable code, compiled to machine code for one OS/architecture pair,
- a **substrate runtime**: a garbage collector, a thread scheduler shim, and the minimum
  runtime services the code needs,
- an **image heap**: a pre-built snapshot of objects created during build-time class
  initialization, mapped into memory at startup rather than constructed.

It does **not** contain: a bytecode interpreter, C1, C2, a class loader capable of loading
arbitrary new classes, or JVMTI.

### The build pipeline, in order

1. **Compile normally.** `javac` produces ordinary class files. Nothing special.
2. *(Spring only)* **AOT processing.** Spring Boot runs an AOT phase that evaluates the
   bean definitions at build time and generates source code — bean registrations,
   proxy classes, and reachability metadata — so that the container does not need to do
   reflection-heavy component scanning at run time. **This is why Spring on native-image
   works at all**, and it is a large piece of engineering you get for free.
3. **Points-to analysis.** Starting from entry points, compute the reachable universe.
4. **Class initialization.** Classes marked for build-time initialization have their static
   initialisers **run during the build**, and the resulting objects are written into the
   image heap.
5. **Compilation.** Reachable methods are compiled to machine code.
6. **Link.** Everything is linked into one executable.

### The vocabulary

| Term | Definition |
|---|---|
| **Closed-world assumption** | The set of reachable classes, methods and fields is fixed at build time and cannot grow at run time |
| **Points-to analysis** | The type-flow analysis that computes what is reachable, iterated to a fixed point |
| **Reachability metadata** | JSON declarations telling the builder to include things it could not prove reachable — reflection targets, resources, proxy interface lists, serialization types, JNI |
| **Tracing agent** | A JVM agent that runs your application **on HotSpot** and records every dynamic access, emitting metadata. Its coverage is exactly your run's coverage |
| **Build-time initialization** | Running a class's static initialiser during the build and snapshotting the result into the image heap |
| **Run-time initialization** | Deferring a class's static initialiser to executable startup — necessary for anything that must be per-run |
| **Image heap** | The pre-initialised object graph baked into the binary and mapped at startup. A major source of the startup win |
| **Substitution** | A build-time replacement of a method or field, used to work around code that cannot be analysed or must behave differently in a native image |
| **PGO (profile-guided optimisation)** | Instrument a build, run a representative workload, feed the profile back into a second build. The AOT answer to C2's run-time profile |

### What you gain, what you lose — the honest table

| | Native image | HotSpot JVM |
|---|---|---|
| **Startup** | Dramatically faster — no class loading, no verification, no interpretation, image heap mapped rather than built | Slow, and gets slower as the application grows |
| **Memory (RSS)** | Substantially lower — no JIT, no code cache, no profiling data, no metaspace of the same shape | Higher, and the floor is high even when idle |
| **Peak throughput** | **Usually lower.** No run-time profile, no speculative optimisation, no deoptimization safety net | **Usually higher**, once warm — C2 optimises against what actually happened |
| **Warm-up** | None. First request is representative | Real. The first thousands of requests are slower (Topic 74) |
| **Build time** | Long. Minutes, and it scales with the reachable universe | Seconds |
| **Peak build memory** | High — the analysis holds the whole type-flow graph | N/A |
| **Reflection / dynamic proxies / resources** | Must be declared, or fail at run time | Just work |
| **Observability** | **Degraded.** No JVMTI, so no async-profiler and no bytecode-rewriting APM agent (Topics 78, 81). JFR support exists but is narrower | Full |
| **GC choices** | Fewer, and distribution-dependent | Serial, Parallel, G1, ZGC, Shenandoah |
| **Debugging** | Native debugging with generated debug info; not your normal workflow | Standard |
| **Portability** | One binary per OS/architecture | One jar, everywhere |
| **Attack surface** | Smaller — less code, no dynamic class loading | Larger |

### Where it clearly wins

- **Serverless / FaaS.** You pay for wall-clock time and cold starts are user-visible.
- **Scale-to-zero** deployments (Knative, KEDA) where pods start on a request.
- **CLI tools.** Nobody tolerates a JVM's startup for `mytool --help`.
- **Very high pod density**, where RSS per instance is the binding constraint and each
  instance is lightly loaded.
- **Short-lived batch jobs** that finish before HotSpot would have warmed up.

### Where it clearly loses

- **A long-running service behind a load balancer.** `orderflow` starts once and runs for
  weeks. Startup is amortised to nothing. Peak throughput is what you pay for, all day.
- **Anything reflection-heavy that you do not control** — a plugin architecture, a scripting
  engine, dynamic class generation.
- **Anywhere your observability depends on a JVMTI agent**, which is most production Java
  shops.

**`orderflow` is squarely in the second list.** The drill still makes you build it, because
you should measure the trade yourself rather than take my word — and because the
measurement is what makes you credible when someone proposes this in a design review.

---

## Why does it matter?

### Because "should we go native?" is a real architecture question you will be asked

And the mid-level answer — *"it makes Java fast, and startup is much better"* — is half
wrong in a way that leads teams into a six-month migration with a throughput regression at
the end.

The senior answer starts by asking what problem is being solved. If the answer is "our
pods take 40 seconds to become ready and it hurts during a rolling deploy", there are three
cheaper interventions before native-image, and **you must have measured where those 40
seconds go** before choosing any of them.

### Because the failure mode is uniquely nasty

Most Java build problems fail at build time. This one fails **at run time, in production,
on the code path that was not exercised during the tracing-agent run**. It is
`ClassNotFoundException` on your payment-gateway fallback at 02:00 on the one night the
primary gateway is down — because that fallback is instantiated by name from configuration,
and nobody triggered it in CI.

That failure shape is why the drill in this document is specifically "find the first
runtime reflection failure". It is the experience that calibrates you.

### Because it is where honesty is tested

The steady-state throughput number is usually the unflattering one. Plenty of migration
write-ups quietly omit it, or report only p50 (which looks fine, because there is no
warm-up) and not sustained throughput under load. **The professional habit is to report all
three numbers — startup, RSS, steady-state throughput — even when the third one argues
against the thing you just spent two weeks building.**

Being that person is worth more, over a career, than any individual optimisation.

### Because it teaches what the JIT is actually worth

You cannot really appreciate what C2 does until you run the same code without it. Topics
74 and 75 told you about profile-guided speculation, inlining, escape analysis and scalar
replacement. Building `orderflow` natively and measuring the throughput difference makes
those topics concrete in a way no amount of reading will.

---

## Machine-level reality

Four mechanisms: the closed-world analysis, reachability metadata, build-time versus
run-time initialization, and why peak throughput drops without C2.

### 1. Closed-world analysis — what the builder is actually computing

The builder performs a **points-to analysis**. In plain terms:

1. Start with a set of **roots**: your `main` method, plus every entry point explicitly
   registered (including everything reachability metadata names).
2. For each reachable method, walk its instructions. For every call site, compute the set
   of **types that can flow to the receiver**, from the assignments and returns the analysis
   has seen so far.
3. For each type in that set, the corresponding method implementation becomes reachable.
4. Reachable methods create new call sites and new type flows. **Iterate until nothing
   changes** — a fixed point.
5. Fields are similarly tracked: a field is reachable if something reads or writes it, and
   the types that can be stored in it feed the flow.

**Three consequences you must internalise:**

**Consequence A — an interface with one implementation is devirtualised for free.** The
analysis knows exactly one type can flow there. This is the good news, and it is why some
native-image code is genuinely fast.

**Consequence B — an interface whose implementation is chosen at run time by name is a
hole.** The analysis sees `Class.forName(gatewayClassName)` and can determine nothing. The
class is not reachable, so it is not in the binary, so the call throws.

**Consequence C — the analysis is CONSERVATIVE within what it can see, and BLIND to what it
cannot.** That combination is why the build succeeds and the run fails. There is no "I am
not sure" state — a class is reachable or absent.

**Look at what your build actually reached.** GraalVM can emit a build report; the flag
name and the output format have changed across releases, so check yours:

```bash
native-image --help | grep -i -E "report|diagnostic|dashboard"
native-image --expert-options-all 2>&1 | grep -i -E "PrintAnalysis|Report|Dashboard" | head -20
```

> **Flagged uncertainty.** I know a build report exists and includes reachable types,
> methods and the image-heap breakdown. I am **not** confident about the exact flag
> spelling on your GraalVM version (`--emit build-report`, `-H:+BuildReport`, and a
> dashboard option have all existed at different times). **Take the flag from your own
> `--help`, not from this document.**

**WHAT TO LOOK FOR in a build report:**

| Section | What it tells you |
|---|---|
| Reachable types / methods / fields counts | The size of your closed world. Large counts mean a large binary and a long build |
| **Image heap breakdown by class** | What got baked in at build time. **A surprise here is a build-time-initialization bug** — see §3 |
| Top contributors to image size | Where the binary's bytes went. Often a surprise: a resource bundle, a large static table |
| Reflective/JNI registrations | What metadata pulled in. If this is huge, your metadata is over-broad |

### 2. Reachability metadata — the escape hatch, and its shape

Metadata is JSON, placed on the classpath under
`META-INF/native-image/<groupId>/<artifactId>/`. Historically it was several files
(`reflect-config.json`, `resource-config.json`, `proxy-config.json`,
`serialization-config.json`, `jni-config.json`); more recent GraalVM versions consolidate
these into a single `reachability-metadata.json`.

> **Flagged uncertainty, and it matters for what you write.** I am confident both the
> split-file form and the consolidated form exist and that recent releases prefer the
> consolidated one. I am **not** confident about which your GraalVM version expects, nor
> about the exact top-level key names in the consolidated schema. **Generate metadata with
> the tracing agent and look at what it writes** — that is authoritative for your version,
> and it is one command:
>
> ```bash
> ls -la src/main/resources/META-INF/native-image/**/
> ```

The reflection declaration's structure:

```json
[
  {
    "name": "<fully.qualified.Class>",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "fields":  [ { "name": "<fieldName>" } ],
    "methods": [ { "name": "<methodName>", "parameterTypes": ["<fully.qualified.Class>"] } ]
  }
]
```

*illustration of the format, not captured output — key names and nesting only*

Resources:

```json
{
  "resources": { "includes": [ { "pattern": "<regex-matching-a-resource-path>" } ] },
  "bundles":   [ { "name": "<resource.bundle.BaseName>" } ]
}
```

*illustration of the format, not captured output — key names and nesting only*

**The three ways metadata gets into your build, in order of preference:**

**(a) It is already there.** Spring Boot's AOT processing generates it for your beans. Many
libraries ship their own under `META-INF/native-image/`. And the **GraalVM Reachability
Metadata Repository** is a community-maintained collection for popular libraries, which the
Native Build Tools plugins can consume automatically. **Check what you already have before
writing anything:**

```bash
# What metadata is on the classpath, from all dependencies?
mvn -q dependency:build-classpath -Dmdep.outputFile=/tmp/cp.txt
tr ':' '\n' < /tmp/cp.txt | while read -r j; do
  case "$j" in *.jar) unzip -l "$j" 2>/dev/null | grep -q "META-INF/native-image" && echo "$j";; esac
done
```

**(b) The tracing agent generates it.** Run your application **on a normal JVM** with the
agent attached; it records every reflective access, resource load, proxy creation and
serialization, and writes the metadata out.

```bash
mkdir -p src/main/resources/META-INF/native-image/com.orderflow/orderflow

java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image/com.orderflow/orderflow \
     -jar target/orderflow.jar
```

To merge several runs rather than overwrite:

```bash
java -agentlib:native-image-agent=config-merge-dir=src/main/resources/META-INF/native-image/com.orderflow/orderflow \
     -jar target/orderflow.jar
```

**The single most important fact about the tracing agent, and it is the source of most
production failures:**

> **The agent records what your run DID, not what your application CAN do.**
>
> If the agent run never exercises the payment-gateway fallback, that class is not in the
> metadata, so it is not in the binary, so it throws in production the first time the
> primary gateway fails.
>
> **Therefore: the agent must run against your FULL test suite and your FULL Topic 65 load
> scenario, plus every error path, admin endpoint, scheduled job and failover branch you can
> trigger.** Metadata coverage is a coverage problem, and you should treat it with the same
> seriousness as test coverage — arguably more, because a missing branch here is a
> production outage rather than an untested line.

**(c) You write it by hand.** For the paths the agent cannot reach. Also available:
programmatic registration via a `RuntimeHintsRegistrar` in Spring, or GraalVM's `Feature`
API, both of which keep the declaration next to the code that needs it — which is much more
maintainable than a JSON file nobody remembers.

**Merging config files:**

```bash
native-image-configure --help
native-image-configure generate --input-dir=<dir1> --input-dir=<dir2> --output-dir=<merged>
```

> **Flagged.** `native-image-configure` is a separate tool that may need installing
> (historically via `gu install`, though `gu` itself has changed across distributions).
> Check with `which native-image-configure` and your distribution's documentation.

### 3. Build-time versus run-time initialization — and the bug it creates

**The mechanism.** A class marked for **build-time initialization** has its static
initialiser executed **during the build**, on the build machine. The objects it creates are
written into the **image heap**, which is a region of the binary mapped into memory at
startup.

**This is a large part of the startup win.** Instead of running static initialisers, loading
configuration and building object graphs at startup, the binary maps a pre-built graph.

**And it is a rich source of bugs**, because a value captured at build time is baked into
every instance of the binary, forever.

```java
// A CLASSIC build-time-initialization bug.
public final class InstanceIdentity {
    // If this class is initialised at BUILD time, every deployed instance of the
    // binary has the SAME id - the one generated on the build machine.
    public static final String INSTANCE_ID = UUID.randomUUID().toString();

    // Same problem, worse consequences.
    public static final SecureRandom RNG = new SecureRandom();   // seeded at build time?

    // The build machine's clock, not the deployment's.
    public static final Instant BUILT_AT = Instant.now();

    // The build container's hostname.
    public static final String HOST = System.getenv("HOSTNAME");
}
```

**WHAT TO LOOK FOR — the symptoms of a build-time capture bug, which are unmistakable once
you know them:**

| Symptom | What was captured |
|---|---|
| Every pod reports the same instance id in logs and metrics | A `UUID` or counter generated at build time |
| A "uptime since" or "build time" that predates deployment | An `Instant.now()` at build time |
| Correlated "random" values across instances | A `Random`/`SecureRandom` seeded at build time |
| A file path or hostname from the CI container appearing at run time | Environment read at build time |
| The build **fails** with an error about an unsupported object in the image heap | The builder caught you: some objects (open file descriptors, sockets, threads, native handles) **cannot** be snapshotted, and the builder refuses. **This is the good outcome** — a build failure instead of a production surprise |

The controls:

```bash
--initialize-at-build-time=<comma-separated classes or packages>
--initialize-at-run-time=<comma-separated classes or packages>
```

> **Flagged.** The **default** initialization policy (which classes are initialised at build
> time unless told otherwise) has changed across GraalVM releases, and framework plugins set
> their own policies. Do not assume mine matches yours. Inspect what your build did:
>
> ```bash
> native-image --expert-options-all 2>&1 | grep -i "initialize" | head -20
> ```
>
> and read the image-heap section of your build report.

**The rule:** anything whose value must be per-process — identity, randomness, time,
environment, network state, open resources — must be initialised **at run time**, and you
should assert it in a test.

### 4. Why peak throughput drops without C2

This is the part that separates people who have read a blog post from people who understand
the trade. Recall Topics 74 and 75.

**What C2 does that an AOT compiler cannot, absent a profile:**

| C2, at run time | native-image, at build time |
|---|---|
| **Observes** that a call site has seen exactly one receiver type across a million calls, and inlines that target directly, guarded by an uncommon trap | Must prove monomorphism statically. Where the type set has more than one member, it emits a real dispatch |
| **Observes** that a branch is never taken and compiles the cold path out of line, or not at all | Has no frequency information; lays out both paths as if equally likely |
| **Speculates** aggressively and **deoptimizes** to the interpreter if the speculation is violated | Has **no interpreter to fall back to**. Every speculation must be sound for all executions, or not made |
| **Recompiles** when conditions change — a new class loaded, a profile shifted | The binary is fixed |
| Inlines based on observed hot paths, spending its budget where it pays | Inlines on static heuristics — size, depth — with no frequency data |
| Escape analysis benefits from inlining decisions driven by real profiles (Topic 75) | Escape analysis still happens, but on a less well-inlined graph |

**The deoptimization point is the deep one.** C2's power comes from being *allowed to be
wrong*: it compiles an optimistic version and keeps a safety net. Native image has no
interpreter and no deopt path, so **every optimisation must be sound unconditionally**. That
is a strictly weaker position, and it is why the gap exists at all.

**The mitigation is PGO**, which reconstructs the missing profile:

```bash
# 1. Build an instrumented binary.
native-image --pgo-instrument -cp <cp> <MainClass> -o app-instrumented

# 2. Run it under a REPRESENTATIVE workload - for orderflow, the Topic 65 k6 scenario.
./app-instrumented &
k6 run loadtest/baseline.js
# a profile file is written on exit

# 3. Rebuild using the profile.
native-image --pgo=<profile-file> -cp <cp> <MainClass> -o app-optimized
```

> **Flagged, and this one has a licensing dimension.** PGO has historically been an **Oracle
> GraalVM** feature rather than a Community Edition one, and the flag spellings have
> changed. Check `native-image --help | grep -i pgo` on your distribution before planning
> around it. **If PGO is unavailable in the distribution you can actually ship, the
> throughput gap is what it is** — and that is a licensing question feeding directly into an
> architecture decision, which is exactly the kind of thing a senior engineer is expected to
> surface early.

**The garbage collector is the other half of the throughput story.** A native image ships a
GC from the substrate runtime, and the available choices are fewer and
distribution-dependent — a Serial collector is the baseline; a G1-class collector is
available in some distributions. **Check yours**, because `orderflow`'s allocation profile
(Topic 70) was measured against G1 and a serial collector at that allocation rate would
change the p99 story completely:

```bash
native-image --help | grep -i -E "gc|--gc="
# and, on the produced binary:
./orderflow -XX:+PrintFlagsFinal 2>&1 | grep -i -E "gc|Heap" | head -20
```

> **Flagged.** Which collectors are available in which GraalVM distribution and version is
> exactly the sort of thing that changes, and getting it wrong in a design document is
> embarrassing. **Take it from `--help` on the build you will actually ship.**

### 5. What you lose in observability, mechanically

There is no JVMTI in a native image. That single fact removes:

- **async-profiler** and every other `AsyncGetCallTrace`/JVMTI-based profiler (Topic 78);
- **bytecode-rewriting APM agents** — Datadog, New Relic, OpenTelemetry auto-instrumentation
  (Topic 81) — because there is no class loading to intercept and no bytecode at run time;
- **`jcmd` diagnostic commands** in their familiar form, and therefore `jcmd GC.heap_dump`
  and `jcmd VM.native_memory` as you learned them in Topics 79 and 80.

JFR support in native image exists but has historically been narrower than on HotSpot.
**Check what your build supports and, critically, check it against the events you actually
depend on:**

```bash
native-image --help | grep -i -E "jfr|monitoring"
# then, on the binary:
./orderflow -XX:StartFlightRecording=filename=/dumps/native.jfr,duration=60s
jfr summary /dumps/native.jfr
```

**WHAT TO LOOK FOR:** whether the event types your dashboards and runbooks depend on are
present. **This is a migration blocker that teams discover after the migration**, and
raising it before is a genuinely senior contribution.

---

## Example 1 — minimal

**The goal:** produce the closed-world failure in five minutes, with code small enough that
there is nothing to argue about.

### The program

```java
package com.orderflow.demo;

/**
 * The whole topic, in twenty lines.
 * On HotSpot this works: the class loader resolves the name lazily at run time.
 * In a native image the class is not reachable from any static path, so it is
 * NOT IN THE BINARY, and this throws.
 */
public final class ClosedWorldDemo {

    public interface PricingRule {
        long applyMinor(long amountMinor);
    }

    public static final class TenPercentOff implements PricingRule {
        @Override public long applyMinor(long amountMinor) { return amountMinor * 9 / 10; }
    }

    public static void main(String[] args) throws Exception {
        // The class name comes from OUTSIDE the program. The analysis can see nothing.
        String ruleClass = args.length > 0
                ? args[0]
                : "com.orderflow.demo.ClosedWorldDemo$TenPercentOff";

        PricingRule rule = (PricingRule) Class.forName(ruleClass)
                .getDeclaredConstructor()
                .newInstance();

        System.out.println("result=" + rule.applyMinor(12_499L));
    }
}
```

### Step 1 — confirm it works on the JVM

```bash
javac -d out ClosedWorldDemo.java
java -cp out com.orderflow.demo.ClosedWorldDemo
```

**WHAT TO LOOK FOR:** it prints a result. This is the control, and it matters: the failure
you are about to see is **not a bug in your code**.

### Step 2 — verify your GraalVM installation. Do not take versions from this document.

```bash
java -version                 # should identify a GraalVM distribution
native-image --version
native-image --help | head -40
echo "$JAVA_HOME"
```

> **I will not name a GraalVM version.** Distributions (Oracle GraalVM, GraalVM Community,
> Liberica NIK, Mandrel) differ in features, and the master plan's stack line mentions
> GraalVM 25 alongside Spring Framework 7. **`--version` and `--help` on your installation
> are the authority for every flag in this document.**

### Step 3 — build it, and watch the build succeed

```bash
native-image -cp out --no-fallback -o closedworld com.orderflow.demo.ClosedWorldDemo
ls -l closedworld
```

`--no-fallback` matters and you should always use it while learning. Without it, when the
builder detects it cannot fully analyse your program it may produce a **fallback image** —
a binary that quietly launches a JVM. That "works", which is worse than failing, because it
hides the very problem you are trying to find and delivers none of the benefits.

**WHAT TO LOOK FOR during the build:**

| What you see | What it means |
|---|---|
| The build succeeds, with warnings about reflection | Expected. **The build succeeding is the trap**, not the reward |
| The build fails saying it would produce a fallback image | `--no-fallback` doing its job on a program with more dynamism than this one |
| The build fails on an unsupported object in the image heap | A build-time-initialization problem (§3). The good failure |
| The build takes far longer than you expected | Normal. Build time scales with the reachable universe |

### Step 4 — run it and get the failure

```bash
./closedworld
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| `ClassNotFoundException: com.orderflow.demo.ClosedWorldDemo$TenPercentOff` | **The lesson.** The class exists in your source and in `out/`, and is absent from the binary because nothing statically reachable referenced it |
| It works anyway | The analysis found the class through some static path. Change the argument to a class genuinely referenced nowhere and try again. Also check you built with `--no-fallback` |
| `NoSuchMethodException` on the constructor | The class made it in but the constructor was not registered. Reflection metadata is granular — class, constructor, method and field are separate declarations |
| An error about a fallback image | Rebuild with `--no-fallback` and read the message; it names what it could not analyse |

**The sentence to internalise:**

> *"The build succeeded and the program is broken. That is not a bug in the tool; it is the
> closed-world assumption doing exactly what it promises. The compiler cannot warn me,
> because from its point of view I never referenced that class."*

### Step 5 — fix it with metadata

```bash
mkdir -p META-INF/native-image
cat > META-INF/native-image/reflect-config.json <<'JSON'
[
  {
    "name": "com.orderflow.demo.ClosedWorldDemo$TenPercentOff",
    "allDeclaredConstructors": true,
    "allPublicMethods": true
  }
]
JSON

native-image -cp out:. --no-fallback -o closedworld com.orderflow.demo.ClosedWorldDemo
./closedworld
```

**WHAT TO LOOK FOR:** it now works. And then the important follow-up experiment: **add a
second implementation and do not add it to the metadata.** Run with that class name as the
argument. It fails, and now you can feel the maintenance obligation directly: *every future
implementation must be declared, and nothing in your build will remind you.*

### Step 6 — do it properly, with the tracing agent

```bash
mkdir -p META-INF/native-image
java -agentlib:native-image-agent=config-output-dir=META-INF/native-image \
     -cp out com.orderflow.demo.ClosedWorldDemo
ls -la META-INF/native-image/
cat META-INF/native-image/*.json
```

**WHAT TO LOOK FOR:** the agent wrote metadata covering **exactly the class you exercised
in that run**. Run it again with a different argument and `config-merge-dir`, and watch the
file grow. **That is the whole story about metadata coverage, demonstrated in two commands**
— and it is the thing to remember when someone proposes running the agent against the unit
test suite and calling it done.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, Spring Boot 4.1 / Framework 7, containerised |
| Dataset | 100k products, 1M orders, **5M order lines** |
| Container | 2 vCPU, 2 GiB memory limit |
| JVM baseline heap | `-Xmx1200m -Xms1200m`, G1 |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Baseline artefacts | `/docs/java/baselines/<date>-run-01/` — p50/p95/p99/p999, throughput, error rate per endpoint |
| Gate rule | a re-run must land within ±10% on every recorded percentile |

### The proposal that arrives

> *"Our pods take too long to become ready. During a rolling deploy we're degraded for
> several minutes. Let's go native — startup drops to milliseconds and we halve our memory
> footprint, so we can run more replicas."*

**This is a reasonable-sounding proposal and it may well be the wrong answer.** Before
touching GraalVM, one question: **where do those startup seconds actually go?** That is
Trap 1, and it is the first thing to do.

### Step 0 — measure the startup you are trying to fix (do this FIRST)

```bash
# What does the app itself say? Spring Boot logs its own startup time.
docker logs orderflow 2>&1 | grep -i "Started .* in .* seconds"

# Break it down: which beans are slow?
# application-load.yml:
#   logging.level.org.springframework.boot.autoconfigure: DEBUG
docker logs orderflow 2>&1 | grep -i "condition\|Bean.*took"

# Class loading volume - the part native-image actually eliminates.
docker exec orderflow jcmd 1 VM.class_hierarchy | wc -l
docker exec orderflow jcmd 1 PerfCounter.print | grep -i -E "classes|loaded|time"

# JIT compilation during startup.
# Run once with: -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation
# and count how many methods compile before readiness.

# And the real number nobody measures: time to first SUCCESSFUL request.
time (until curl -sf localhost:8080/actuator/health/readiness > /dev/null; do sleep 0.1; done)
```

**WHAT TO LOOK FOR — and each answer points somewhere different:**

| Where the startup time goes | native-image helps? | The cheaper fix |
|---|---|---|
| Class loading, verification, bean scanning, interpretation before JIT | **Yes, substantially** | **AppCDS or the JDK's AOT cache first** (Topic 82) — much cheaper to adopt |
| Flyway/Liquibase migrations against a 5M-row table | **No.** Identical in native | Run migrations as a separate job, not at startup |
| Warm-up cache load (Topic 37's catalogue preload) | **No.** Identical | Load it lazily, or in the background after readiness |
| Waiting on a downstream dependency at startup | **No** | Fix the readiness semantics (Topic 121) |
| HikariCP filling `minimum-idle` connections | **No** | Lower `minimum-idle`; connect lazily |
| JIT warm-up *after* startup (first requests slow) | **Yes** — no warm-up in native | Or accept it; or the AOT cache; or a warm-up request in the readiness gate |

> **This table is the most valuable thing in this document.** If your 40 seconds are 30
> seconds of Flyway and 10 seconds of Spring, native-image buys you 10 seconds and costs you
> a migration, a throughput regression and your profiler. **Measure first.**

### Step 0b — the cheaper alternatives, honestly

Before native-image, in increasing order of cost:

1. **Move migrations and cache warm-up out of the startup path.** Free, and often the whole
   answer.
2. **AppCDS (Class Data Sharing).** Dump the loaded-class archive once, then map it at
   startup instead of loading and verifying each class. Requires no code changes, no
   metadata, and keeps C2. Topic 82.
3. **The JDK 25 AOT cache (Project Leyden's first shipped pieces).** See the note below.
4. **Spring Boot AOT processing without native-image** (`spring-boot:process-aot`). Moves
   bean-definition work to build time and still runs on HotSpot with the JIT intact. **This
   is an under-used middle ground** and it costs you almost nothing.
5. **Then, and only then, native-image.**

> ### What I am and am not sure of about Leyden and the JDK 25 AOT cache
>
> **What I am confident of:** Project Leyden's goal is to shift work earlier — to build time
> or to a cached artefact — and its first shipped pieces are an **AOT cache** that stores
> loaded-and-linked class data, and in JDK 25 additional pieces around ergonomics and
> ahead-of-time method profiling. Conceptually: **much of native-image's startup win without
> the closed-world cost, and with C2 still present at run time.** For a long-running service
> like `orderflow`, that is a strictly better shape of trade than native-image.
>
> **What I am NOT confident enough about to have you copy from here:** the exact flag
> spellings (`-XX:AOTCache`, `-XX:AOTMode`, and the two-step record-then-use workflow), which
> JEP numbers landed in which release, and how the AOT cache interacts with AppCDS on JDK 25.
> **I am not going to guess.**
>
> **Find out from primary sources:**
>
> ```bash
> java -XX:+PrintFlagsFinal -version | grep -i -E "aot|cds|shared"
> java -Xshare:help 2>&1 | head -20
> java --help-extra 2>&1 | grep -i -E "aot|cds"
> ```
>
> and read the JEP index at **`https://openjdk.org/jeps/0`**, filtering for Leyden. That
> index is the authority; a document written months earlier is not.

### Step 1 — build `orderflow` natively

With Spring Boot's Maven plugin and the GraalVM Native Build Tools:

```bash
# Verify the toolchain first.
java -version && native-image --version && mvn -version

# Spring Boot's AOT processing on its own (still HotSpot - the middle-ground option).
mvn -q clean package spring-boot:process-aot

# The native build.
mvn -Pnative clean native:compile

# Or a container image via buildpacks, which handles the GraalVM toolchain for you.
mvn -Pnative spring-boot:build-image
```

> **Flagged.** Profile name (`native`), goal names (`native:compile`), and whether the
> plugin is bound automatically vary with Spring Boot and Native Build Tools versions.
> **Check your generated `pom.xml`** and `mvn help:describe -Dplugin=native` rather than
> copying from here.

**WHAT TO LOOK FOR during the build:**

| What you see | What it means |
|---|---|
| The build runs for many minutes and uses a large amount of memory | Normal. Budget for it: native builds are a CI capacity question, not a nuisance |
| Warnings listing classes registered for reflection | Spring's AOT processing at work. Expected and good |
| A failure about an unsupported feature or an object in the image heap | The builder caught a real problem at build time. **This is the good failure** — read the message; it names the class |
| The build succeeds cleanly | **Do not relax.** Closed-world failures are run-time failures. A clean build tells you nothing about reflection coverage |
| A warning that a fallback image would be produced | Add `--no-fallback` and read what it could not analyse |
| An out-of-memory failure in the builder itself | Give the builder more heap; the analysis holds the whole type-flow graph |

### Step 2 — the first runtime failure, and it will not be in a Spring bean

Spring Boot's AOT processing handles the framework's own reflection well. So the failure
you actually hit is in **your** code, or in a library the framework does not know about.
The realistic `orderflow` candidates, in rough order of likelihood:

**(a) A `PaymentGateway` implementation chosen by class name from configuration.** This is
`orderflow`'s Topic 39 `@Qualifier` pattern taken one step further by someone who wanted it
configurable:

```java
package com.orderflow.payments;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class PaymentGatewayFactory {

    @Value("${orderflow.payments.gateway-class}")
    private String gatewayClassName;

    public PaymentGateway create() throws Exception {
        // Invisible to the closed-world analysis. Works on the JVM. Fails natively.
        return (PaymentGateway) Class.forName(gatewayClassName)
                .getDeclaredConstructor()
                .newInstance();
    }
}
```

**And note the timing:** the primary gateway is configured everywhere. The **fallback**
gateway is instantiated only when the primary is down. So this fails the first time your
payment provider has an outage — which is precisely the worst moment for a second,
unrelated failure.

**(b) A resource loaded by a path from configuration.**

```java
var stream = getClass().getResourceAsStream("/pricing/" + region + "-rules.json");
```

The pattern is dynamic, so the resource is not included. `getResourceAsStream` returns
`null` and you get an NPE somewhere unhelpful, far from the cause.

**(c) A JDBC driver, Hibernate dialect, or similar resolved by name.** Usually handled by
the framework's metadata, but a non-standard driver or an explicitly-configured dialect
class name is not.

**(d) Jackson polymorphic deserialization.** `@JsonTypeInfo`/`@JsonSubTypes` on a sealed
`PaymentResult` hierarchy (Topic 28) instantiates subtypes reflectively by a discriminator
in the JSON.

**(e) A `ServiceLoader` provider** from a library that does not ship metadata.

**WHAT TO LOOK FOR at run time:**

| Exception | Missing metadata | Fix |
|---|---|---|
| `ClassNotFoundException` | Reflection: the class itself | Register the class |
| `NoSuchMethodException` / `NoSuchFieldException` | The class is present; the member is not | Register the constructor/method/field — registration is granular |
| `getResourceAsStream` returning `null` | Resource | `resources.includes` pattern |
| `MissingResourceException` | Resource bundle | `bundles` entry |
| `IllegalArgumentException` from `Proxy.newProxyInstance` | Dynamic proxy interface list | Proxy configuration |
| A serialization failure on a class that deserialises fine on the JVM | Serialization | Serialization configuration |
| `UnsatisfiedLinkError` | JNI | JNI configuration |
| It works in dev and fails in prod | **The tracing agent never covered that path.** The most common failure of all | Extend the agent run — see Step 3 |

### Step 3 — fix it properly: the agent against the FULL load scenario

Do **not** fix it by hand-writing one JSON entry and moving on. Hand-fixing one path leaves
every other unexercised path broken and gives you false confidence.

```bash
mkdir -p src/main/resources/META-INF/native-image/com.orderflow/orderflow

# 1. Run the JVM build with the agent, under the FULL Topic 65 load scenario.
java -agentlib:native-image-agent=config-merge-dir=src/main/resources/META-INF/native-image/com.orderflow/orderflow \
     -jar target/orderflow.jar &
APP=$!
k6 run loadtest/baseline.js

# 2. THEN exercise everything k6 does not: error paths, admin endpoints,
#    scheduled jobs, the payment fallback, every configured gateway class.
curl -X POST localhost:8080/admin/reconciliation/run
curl -X POST localhost:8080/orders -d @fixtures/invalid-order.json    # validation errors
curl -X POST localhost:8080/orders -d @fixtures/insufficient-funds.json
curl localhost:8080/actuator/{health,info,metrics,prometheus}
# force the payment fallback path:
docker compose stop payment-gateway-primary
curl -X POST localhost:8080/orders -d @fixtures/valid-order.json

# 3. Run the whole test suite under the agent too.
mvn -q test -DargLine="-agentlib:native-image-agent=config-merge-dir=src/main/resources/META-INF/native-image/com.orderflow/orderflow"

kill $APP
git diff --stat src/main/resources/META-INF/native-image/
```

**WHAT TO LOOK FOR:** the metadata files growing with each additional scenario. **That
growth is the visible evidence that coverage is the whole game.** If the file stopped
growing after the k6 run, you did not exercise anything new — which means every branch you
did not touch is still missing.

**And the durable fix**, better than a JSON file nobody maintains: register the hints next
to the code that needs them, in Spring, so the declaration cannot drift away from the
requirement.

```java
package com.orderflow.payments;

import org.springframework.aot.hint.MemberCategory;
import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;

/**
 * Registered via META-INF/spring/aot.factories (or @ImportRuntimeHints).
 * Lives next to the code whose dynamism it declares, so a new gateway
 * implementation and its registration are reviewed in the same PR.
 */
public class PaymentGatewayHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        for (Class<?> impl : new Class<?>[]{
                StripeGateway.class, AdyenGateway.class, FallbackGateway.class}) {
            hints.reflection().registerType(impl,
                    MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                    MemberCategory.INVOKE_PUBLIC_METHODS);
        }
        hints.resources().registerPattern("pricing/*-rules.json");
    }
}
```

> **Flagged.** `MemberCategory` constant names and the exact `RuntimeHints` API have
> changed between Spring versions. **Check the API on your Spring Framework 7 build**; the
> pattern — a `RuntimeHintsRegistrar` beside the dynamic code — is what is stable.

### Step 4 — the three measurements, and report all three

**This is the deliverable, and it is the entire point of the drill.**

#### Measurement 1 — startup

Do **not** measure `time ./orderflow`. That measures process start, not usefulness.

```bash
# The honest number: time to first SUCCESSFUL request.
measure_startup() {
  local start end
  start=$(date +%s%N)
  "$@" > /tmp/startup.log 2>&1 &
  local pid=$!
  until curl -sf http://localhost:8080/actuator/health/readiness > /dev/null 2>&1; do
    sleep 0.01
  done
  end=$(date +%s%N)
  echo "time-to-ready-ms: $(( (end - start) / 1000000 ))"
  kill "$pid"
}

# Repeat at least ten times each and record the DISTRIBUTION, not one run.
for i in $(seq 1 10); do measure_startup java -jar target/orderflow.jar; done
for i in $(seq 1 10); do measure_startup ./target/orderflow; done
```

**WHAT TO LOOK FOR:** the native binary should be dramatically faster, and if it is not,
your startup time was never class loading — go back to Step 0's table.

#### Measurement 2 — memory

Measure **RSS**, not heap. Heap is not the footprint (Topics 80, 82).

```bash
# At idle, after readiness:
ps -o pid,rss,vsz,comm -p <pid>
cat /proc/<pid>/status | grep -E "VmRSS|VmHWM"          # VmHWM is the peak

# In the container, under load - this is the number that sizes your pods:
docker stats --no-stream orderflow
docker exec orderflow cat /sys/fs/cgroup/memory.peak 2>/dev/null || \
  docker exec orderflow cat /sys/fs/cgroup/memory.max_usage_in_bytes
```

**Measure at idle AND at steady state under the Topic 65 load.** The idle figure is what
marketing quotes; the under-load figure is what sizes your pods. They are different numbers
and only the second one matters for capacity (Topic 129).

#### Measurement 3 — steady-state throughput (the honest one)

```bash
# The JVM baseline, re-run today so the comparison is same-day, same-machine.
k6 run loadtest/baseline.js --out json=/tmp/k6-jvm.json

# The native build, same scenario, same dataset, same container limits.
k6 run loadtest/baseline.js --out json=/tmp/k6-native.json
```

**Non-negotiable conditions**, or the comparison is worthless:

- Same machine, same day, same dataset, same container CPU/memory limits.
- Both at **steady state** — for the JVM that means after warm-up; running the JVM cold
  against a native binary is the classic dishonest benchmark.
- Same k6 scenario, **open arrival model** (Topic 65), so coordinated omission does not
  flatter either side.
- Both measured for long enough that a GC cycle is included.

**WHAT TO LOOK FOR, and what each outcome means:**

| What you see | What it means | What to write |
|---|---|---|
| Native starts far faster, uses much less RSS, throughput measurably **lower** | **The expected result.** The documented trade, confirmed on your hardware | Report all three. This is the honest outcome and the one to lead with |
| Native throughput **similar** to the JVM | Possible if the workload is I/O-bound — if you are waiting on Postgres, C2's advantage has little to work on. **`orderflow` is I/O-bound, so this is genuinely plausible** | A strong result for native. Verify by checking CPU utilisation: if both are low, the trade barely applies here |
| Native throughput **higher** | Surprising. Check for a GC difference (the substrate collector versus G1 at your allocation rate), and check the JVM run was actually warm | Investigate before reporting. A surprising result you cannot explain is not a result |
| Native p50 better, p99 **worse** | Often the GC. A serial collector at `orderflow`'s allocation rate will show exactly this shape (Topics 70, 71) | Report the percentile breakdown, not just throughput. This is where the decision actually lives |
| Native RSS **not** much lower under load | Your footprint was dominated by the live set, not by JIT and metadata. The live set is the same either way (Topic 70) | An important negative result: the memory argument does not apply to this service |
| Native fails under load with an error absent from the JVM run | An unexercised reflective path. Back to Step 3 with a wider agent run | **Report it.** It is evidence about metadata coverage risk, which is a migration cost |

### Step 5 — the recommendation, written honestly

```markdown
# orderflow native-image evaluation

## What we were trying to fix
Pod readiness took <n>s, degrading rolling deploys.

## Where that time actually went (measured BEFORE evaluating native-image)
| Phase | Time | Does native-image help? |
|---|---|---|
| Flyway migrations | <n>s | No |
| Spring context + class loading | <n>s | Yes |
| Catalogue cache warm-up | <n>s | No |
| HikariCP minimum-idle fill | <n>s | No |

## The three numbers (same machine, same day, same dataset, both at steady state)
| Metric | JVM baseline | Native | Delta |
|---|---|---|---|
| Time to first successful request (median of 10) | <n> ms | <n> ms | <n> |
| RSS at idle | <n> MB | <n> MB | <n> |
| RSS at steady state under load | <n> MB | <n> MB | <n> |
| Throughput at the baseline scenario | <n> rps | <n> rps | <n> |
| GET /orders p50 / p95 / p99 / p999 | <n>/<n>/<n>/<n> ms | <n>/<n>/<n>/<n> ms | <n> |

## Costs we would take on
- Build time: <n> minutes per build, <n> GB peak builder memory. CI capacity impact: <...>
- **Observability: no JVMTI. We lose async-profiler (Topic 78) and our APM agent
  (Topic 81). JFR support in native image is narrower - the events our dashboards depend
  on are: <list>, of which <n> are available.** This is a migration blocker unless resolved.
- Reachability metadata becomes a maintained artefact tracking every dependency upgrade.
- A missed reflective path is a RUNTIME failure in production, not a build failure.
  Our agent coverage is currently <...>.

## Recommendation
<...>

## What I recommend instead, if native-image is not justified
1. Move Flyway migrations to a separate job: removes <n>s at zero risk.
2. Load the catalogue cache in the background after readiness: removes <n>s.
3. AppCDS or the JDK AOT cache (Topic 82): removes some of the class-loading time,
   keeps C2, requires no metadata and no code change.
4. Spring Boot AOT processing WITHOUT native-image: build-time bean definitions,
   still on HotSpot with the JIT intact.
Combined, these address <n>s of the <n>s at a fraction of the cost and risk.
```

**That document is what a senior engineer produces.** Note that it may well conclude "no",
and note that the "instead" section is what makes the "no" useful rather than obstructive.

---
