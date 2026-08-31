# 122 — Dockerising Spring Boot: Layered Jars, AppCDS, and Startup Time

## Phase: 11 — Distributed Systems & Production
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: the `orderflow` image is rebuilt with layered jars and an AppCDS archive, and the startup time is **decomposed and recorded before and after** — JVM init, class loading, context refresh, bean instantiation — so the improvement is attributed to a named phase rather than claimed.

---

## Mechanical statement

**A fat jar in one Docker layer invalidates that whole layer on every code change; layered
jars separate rarely-changing dependencies from your classes. AppCDS memory-maps a
pre-parsed class archive, cutting class-loading time — usually the largest slice of Spring
startup.**

Three mechanical facts:

1. **A Docker layer is content-addressed.** Change one byte of a layer's input and that
   layer's digest changes, and so does every layer after it. A 60 MB fat jar copied in one
   `COPY` is one layer whose content changes on every commit — so every build pushes,
   stores and pulls the whole thing, including the dependencies that did not change.
2. **A layered jar is the same jar with an index.** Boot's build plugin writes a
   `layers.idx` describing which entries belong to which layer, and a tool extracts them in
   dependency-stability order. Nothing about the application changes; the *packaging*
   changes so Docker's cache can do its job.
3. **AppCDS trades build-time work for start-time work.** The JVM normally reads each class
   file, parses it, verifies it, and builds internal metadata — every time, in every
   process. A CDS archive contains that metadata already built, in a form the JVM can
   `mmap` into memory. Loading becomes a memory mapping rather than a parse.

---

## The bridge from what you know

### Layered jars ≈ Docker layer caching, which you already understand

You have written this Node Dockerfile a hundred times:

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci --omit=dev          # <- cached unless the lockfile changed
COPY . .                       # <- invalidated on every source change
```

The whole point is that `node_modules` is a separate, earlier layer from your source, so a
one-line change to a route handler does not reinstall 400 packages.

**Java's default packaging destroys that split, and this is the entire first half of the
topic.** `mvn package` produces one executable "fat jar" (Boot calls it an *uber jar*)
containing your compiled classes *and* every dependency jar, in one file. Copy it into an
image and you have the Node equivalent of:

```dockerfile
COPY . .                       # source AND node_modules in one layer, every time
```

Every commit changes the file, so every commit changes the layer digest, so every deploy
pushes and pulls the full artefact.

**Layered jars restore the split you already know.** Boot's build plugin can index the jar
into four layers, ordered from least to most volatile:

| Layer | Contents | Changes when |
|---|---|---|
| `dependencies` | released third-party jars | you edit `pom.xml` |
| `spring-boot-loader` | the loader classes | you change the Boot version |
| `snapshot-dependencies` | `-SNAPSHOT` dependencies | those snapshots rebuild |
| `application` | **your** classes and resources | every commit |

Copy each into its own `COPY` and Docker caches the first three across builds. The layer
that changes every commit is the small one.

**HONEST ANALOGUE.** If you understand `COPY package.json` before `COPY .`, you understand
layered jars completely. The only new thing is the command that produces the layers.

### AppCDS has NO Node analogue — this is the genuinely new idea

There is nothing in Node that corresponds to Class Data Sharing. V8 has snapshots and code
caching, but they are internal to the engine and you do not configure them per application.
So build the mental model from scratch.

Every time a JVM loads a class it must: read the bytes, parse the constant pool, build the
internal `InstanceKlass` metadata, verify the bytecode, and link it. For a Spring Boot
application that is **thousands to tens of thousands of classes**, and the work is
identical on every start of every pod. Ten pods restarting after a deploy do the same
parsing ten times.

**CDS says: do it once, write the result to a file, and `mmap` that file at startup.** The
archive holds already-parsed class metadata in a layout the JVM can map directly into
memory. Loading a class from the archive skips parsing and most of verification.

Two names you will meet:

- **Default CDS** — a base archive of JDK classes, shipped with the JDK, on by default.
  You are already getting this and did not notice.
- **AppCDS (Application CDS)** — an archive you generate that *also* contains your
  application's and framework's classes. This is the one that matters, because the JDK's
  own classes are a minority of what Spring loads.

**Verdict: NO ANALOGUE.** This is a JVM-specific optimisation with no equivalent in your
Node experience, and it is the highest-leverage startup change available that does not
require rearchitecting anything.

### The third thing you know that transfers with a twist

You know that a container image should be small, pinned, and non-root. All true in Java.
The twist is that **a Java image's size is dominated by two things you do not control by
being careful**: the JRE (tens of megabytes) and the dependency layer (often larger than
your JRE). Micro-optimising your own classes is pointless; choosing a slim base image and
letting the dependency layer be cached is the whole game.

And the crucial reframing for someone with your background:

> **In Node, "startup time" is a developer-experience concern. In Java, it is a
> *production availability* concern**, because it is the length of time a pod is consuming
> resources and serving nothing during every rolling deploy, every autoscale event, every
> node eviction, and every crash recovery. That is why this topic sits in Phase 11 and not
> in a build-tooling chapter.

---

## What is this?

### What is inside a Boot fat jar

```
orderflow.jar
├── META-INF/
│   └── MANIFEST.MF                 # Main-Class: org.springframework.boot.loader.launch.JarLauncher
├── org/springframework/boot/loader/  # the loader: knows how to read nested jars
├── BOOT-INF/
│   ├── classes/                    # YOUR compiled classes and resources
│   ├── lib/                        # every dependency, as nested .jar files
│   └── layers.idx                  # present only if layered packaging is enabled
```

The loader exists because **the standard JVM class loader cannot read a jar inside a jar**.
Boot ships its own `URLClassLoader` subclass that can. This has a cost: an extra
indirection on every class load, which is one reason exploding the jar (below) starts
faster.

### Three ways to get a Boot application into an image

| Approach | Command | When |
|---|---|---|
| **Hand-written Dockerfile with layered extraction** | `docker build` | You want control, and you want to add CDS. **What this topic teaches** |
| **Cloud Native Buildpacks** | `./mvnw spring-boot:build-image` | No Dockerfile at all; produces a well-built, layered, CDS-capable image. Excellent default |
| **Jib** (Google's Maven/Gradle plugin) | `./mvnw jib:dockerBuild` | Daemonless, reproducible, very fast; layered by design |

Buildpacks are genuinely good and a legitimate answer in an interview. **You should still be
able to write the Dockerfile**, because when the image misbehaves you need to know what a
buildpack did on your behalf — the same argument as Topic 42 for auto-configuration.

### `jarmode` — and the command that changed

This is the part the brief specifically wants stated precisely, because getting it wrong
wastes twenty minutes:

```bash
# Boot 3.3 and later (including Boot 4.x) — the `tools` jarmode:
java -Djarmode=tools -jar orderflow.jar list-layers
java -Djarmode=tools -jar orderflow.jar extract --destination extracted/
java -Djarmode=tools -jar orderflow.jar extract --layers --launcher --destination extracted/

# Boot 2.3 through roughly 3.2 — the older `layertools` jarmode:
java -Djarmode=layertools -jar orderflow.jar list
java -Djarmode=layertools -jar orderflow.jar extract
```

- **`layertools`** was introduced with layered jars in Boot 2.3 and is the command in every
  older tutorial and Dockerfile you will find.
- **`tools`** replaced and generalised it in Boot 3.3; it also handles extracting the jar
  for CDS and for a plain (non-nested) launch.
- **Boot 4.1 uses `tools`.** `layertools` is deprecated or removed; if you meet it in an
  inherited Dockerfile, that Dockerfile predates 3.3.

> **Flagged uncertainty, one line:** I am confident about the 3.3 transition and that 4.x
> uses `tools`; I am **not** certain of the exact release in which `layertools` was finally
> removed, nor of every sub-command flag on 4.1. Settle it on your own build with
> `java -Djarmode=tools -jar target/orderflow.jar` with no arguments, which prints the
> available sub-commands, and `--help` on the one you want.

### What "startup time" actually consists of

You cannot optimise a number you have not decomposed. Four phases, in order:

| Phase | What happens | Typical lever |
|---|---|---|
| **1. JVM init** | process start, `mmap` the JDK CDS archive, initialise the heap, start GC and JIT threads | heap sizing, `AlwaysPreTouch`, CPU quota (Topic 82) |
| **2. Class loading** | read, parse, verify, link thousands of classes | **AppCDS**, fewer dependencies, exploded jar |
| **3. Context refresh** | classpath scanning, condition evaluation, bean *definition* registration | fewer auto-configurations, no broad component scans, Spring AOT |
| **4. Bean instantiation** | construct singletons, build the Hibernate `SessionFactory`, open the connection pool, warm caches | lazy init, deferred pool warm-up, fewer eager beans |

**The discipline this topic exists to install: measure the split before you touch anything.**
The four levers are completely different, and the wrong one is not merely useless — it costs
you weeks. That is the "12 seconds" interview question, and it is the Measurement section.

---

## Why does it matter?

**1. Because startup time is availability, not convenience.**

Count the places `orderflow` pays it. A rolling deploy of six pods pays it six times.
Horizontal autoscaling pays it every time a pod is added — and autoscaling triggers when you
are *already* under load, so the delay lands exactly when you need capacity. Node eviction,
spot reclamation, and OOMKill recovery all pay it. If a pod takes 40 seconds to become
ready, your effective recovery time from any single-pod failure is 40 seconds, and your
autoscaler is 40 seconds behind demand forever.

**2. Because a fat-jar layer makes every deploy move the whole artefact.**

The dependency set of a Spring Boot service is usually several times larger than the
application's own classes. With one layer, every commit pushes all of it to the registry and
every node pulls all of it. That is CI minutes, registry storage and egress, and — most
visibly — **rollout time**, because a node cannot start a pod until it has pulled the image.
Layering makes the per-commit delta the small layer.

**3. Because class loading is usually the biggest slice and the least obvious one.**

The intuition from Node is that framework *initialisation* is slow. In Java, a very large
share of the time before your first bean is constructed is spent turning class files into
JVM metadata. It is invisible in a profiler that samples your application code, because it
is not your application code. AppCDS attacks it directly, and it is a build-and-flag change
rather than an architectural one.

**4. Because the wrong fix is expensive and popular.**

"Startup is slow, let us go native-image" is the single most common expensive mistake in
this area. GraalVM native-image is a genuine tool (Topic 83) with a genuine cost: a
closed-world assumption that breaks reflection, dynamic proxies (Topic 40) and resource
loading, long build times, a different debugging story, and a different runtime profile.
Spending a month on it to fix a problem that AppCDS and one lazy-initialisation flag would
have halved is how teams lose a quarter. **Measure first** is not a platitude here; it is the
difference between a two-hour change and a two-month project.

---

## Machine-level reality

### How Docker layer hashing decides your deploy cost

An image is an ordered list of filesystem layers plus a manifest. Each layer is
content-addressed by the digest of its contents (a `sha256:`). The build cache works on a
simple rule:

> A build step is cache-hit if **the step's instruction** and **the digest of its inputs**
> both match a previous build — and only if **every preceding step also hit**.

Two consequences that explain almost everything about Java image builds:

1. **Cache invalidation cascades forward.** Change something in step 3 of ten and steps
   4–10 rebuild regardless of whether their own inputs changed. This is why ordering
   instructions from least to most volatile is not a style preference.
2. **`COPY` hashes the *content* of what it copies**, plus the path. `COPY . .` hashes your
   entire build context — including, unless you exclude them, `target/`, `.git/`, IDE
   files, and log files. A `.git` directory whose contents change on every commit will
   invalidate that layer on every build even if no source file changed.

`docker history` shows the layers and their sizes; the build output shows `CACHED` per step.
Both are in Hands-on proof.

### The Boot layered-jar format

With layering enabled the plugin writes `BOOT-INF/layers.idx`, a small YAML-ish index
mapping layer names to the jar entries they contain:

```
- "dependencies":
  - "BOOT-INF/lib/spring-core-<version>.jar"
  - "BOOT-INF/lib/hibernate-core-<version>.jar"
- "spring-boot-loader":
  - "org/springframework/boot/loader/"
- "snapshot-dependencies":
- "application":
  - "BOOT-INF/classes/"
  - "META-INF/"
```

*Illustration of the format, not captured output.*

The `tools extract` command reads that index and writes each layer to its own directory. Your
Dockerfile then does one `COPY` per directory, in that order — and Docker's cache does the
rest. **Layer ordering in the Dockerfile must match volatility order**, or you have the index
and none of the benefit.

You can customise the layer definition in the build plugin's configuration — for example,
splitting your own company's internal libraries into a layer of their own because they change
weekly rather than per-commit. Worth doing in a monorepo, over-engineering in a single
service.

### What a CDS archive contains and how it is mapped

The pipeline has three steps:

```
1. TRAINING RUN:  start the app with -XX:ArchiveClassesAtExit=app.jsa, let it initialise,
                  exit cleanly. The JVM dumps the metadata of every class it loaded.
2. ARCHIVE:       app.jsa — pre-parsed class metadata in a mappable layout.
3. PRODUCTION:    start with -XX:SharedArchiveFile=app.jsa. The JVM mmaps the archive and
                  resolves classes from it instead of parsing class files.
```

What is actually in the archive: the JVM's internal representations — `InstanceKlass`
structures, constant pool entries, method metadata, and some pre-resolved constant-pool
references. Not your bytecode as source-of-truth, but the *result* of reading it.

Why mapping is fast, in one paragraph: the archive is laid out so it can be `mmap`ed into
the process at a known address, and the internal pointers inside it are already correct for
that address. There is no parse step, no verification step for archived classes, and — the
underrated part — **the mapping is read-only and shareable**, so several JVMs on the same
host can share the same physical pages. On a node running ten pods of the same image, that
is real memory saved, not just time.

**The validation rules are strict, and violating them is silent.** The JVM refuses to use an
archive when the runtime does not match the dump-time environment closely enough:

| Must match between dump and use | Why |
|---|---|
| JDK version and build | internal metadata layouts change between builds |
| Classpath (same entries, same order, and the runtime classpath may not be a *prefix* mismatch) | archived classes record which classpath entry they came from |
| Relevant JVM options (e.g. compressed oops settings) | metadata layout depends on them |
| The GC in some combinations | archived heap objects depend on heap layout |

If any of these fail, **the JVM by default logs nothing at the default log level and starts
normally without the archive.** You get zero benefit and no error. That is Trap 3, and it is
why `-Xshare:on` and `-Xlog:cds` exist.

### `[JAVA 25]` The AOT cache — what I am and am not sure of

JDK 24 and 25 ship work from Project Leyden that generalises CDS into an **AOT cache**: a
single artefact that can hold not only class metadata but also pre-linked state and,
progressively, more of the work a JVM repeats on every start. The ergonomics are a
record-then-use pair of commands, conceptually the same shape as the CDS training run.

**What I am confident about:**
- The direction is real and is in the JDK 25 LTS.
- It is a superset of, and intended successor to, the AppCDS workflow described here.
- The AppCDS flags in this document (`-XX:ArchiveClassesAtExit`, `-XX:SharedArchiveFile`)
  continue to work.

**What I am not confident about and will not guess:**
- The exact spelling of the AOT-cache flags on your build.
- Which JEPs landed in 24 versus 25 versus a later release, and what is still preview.
- Whether Spring Boot's build tooling wires it for you on 4.1.

**Settle it from primary sources rather than from me** — this is exactly Topic 125's skill:

```bash
# 1. The JEP index. Search for "AOT" and read the JEPs targeted to your JDK release.
#    https://openjdk.org/jeps/0

# 2. What your actual JDK supports:
java -XX:+PrintFlagsFinal -version | grep -i aot
java --help-extra 2>&1 | grep -i -A2 aot

# 3. Your JDK's release notes for the exact build you ship.
java -version
```

Ship AppCDS today; it works on every JDK you might run and the workflow is identical. Revisit
the AOT cache with the flags your JDK actually prints, not the flags a blog post remembers.

### Startup, decomposed — what each phase leaves as evidence

| Phase | Evidence | Command |
|---|---|---|
| JVM init | the gap between process start and the JVM handing control to `main` | Boot's own log line: `Started ... in X seconds (process running for Y)` → **JVM init ≈ Y − X** |
| Class loading | one line per class loaded | `-Xlog:class+load=info:file=classes.log` then `wc -l` |
| Context refresh | per-step timings inside `refresh()` | `BufferingApplicationStartup` + `/actuator/startup` |
| Bean instantiation | per-bean timings | the same `/actuator/startup` output, `spring.beans.instantiate` steps |

That `Started ... in X seconds (process running for Y)` line is doing more work than most
people realise. `X` is measured from the start of the Spring application; `Y` is measured
from JVM start using the process start time. **`Y − X` is your JVM-init-plus-class-loading
overhead before Spring even begins**, and if that number is large, no amount of Spring tuning
will help you.

### Why an exploded layout starts faster than a fat jar

Running `java -jar orderflow.jar` goes through Boot's `JarLauncher`, which opens the outer
jar, indexes the nested jars, and installs a class loader that can read entries inside them.
Every class load then goes through that indirection.

Running the **extracted** application instead:

```dockerfile
ENTRYPOINT ["java", "-cp", "BOOT-INF/classes:BOOT-INF/lib/*", "com.orderflow.OrderflowApplication"]
```

…or, more robustly, using the launcher that `tools extract` writes for you, uses ordinary
class loading over ordinary jars. It is also **the configuration AppCDS wants**, because a
stable, ordinary classpath is much easier to keep identical between the training run and
production than a nested-jar classpath is.

So "extract the jar in the image" buys you two things at once: a small startup win by itself,
and the precondition for the larger CDS win.

---

## Example 1 — minimal

Two Dockerfiles for the same application, so you can measure the difference yourself.

### The naive version (what most Java Dockerfiles look like)

```dockerfile
# Dockerfile.naive — one layer for everything. DO NOT SHIP THIS.
FROM eclipse-temurin:25-jre
WORKDIR /app
COPY target/orderflow.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

One `COPY` of the whole fat jar. Every code change changes that layer.

### The layered version

```dockerfile
# Dockerfile — layered, cache-friendly.
# STAGE 1: extract the jar into layer directories.
FROM eclipse-temurin:25-jdk AS builder
WORKDIR /build
COPY target/orderflow.jar orderflow.jar
# Boot 3.3+ / Boot 4.x. On Boot <= 3.2 this is: java -Djarmode=layertools -jar orderflow.jar extract
RUN java -Djarmode=tools -jar orderflow.jar extract --layers --launcher --destination extracted

# STAGE 2: assemble the runtime image, least-volatile layer first.
FROM eclipse-temurin:25-jre
WORKDIR /app
COPY --from=builder /build/extracted/dependencies/ ./
COPY --from=builder /build/extracted/spring-boot-loader/ ./
COPY --from=builder /build/extracted/snapshot-dependencies/ ./
COPY --from=builder /build/extracted/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**The ordering is the whole design.** `dependencies` first because it changes least;
`application` last because it changes every commit.

> **Note on the entry point:** the launcher class name moved package between Boot 2.x
> (`org.springframework.boot.loader.JarLauncher`) and Boot 3.2+
> (`org.springframework.boot.loader.launch.JarLauncher`). Rather than hard-coding either,
> `extract --launcher` writes the correct launch layout for your version — check what it
> produced with `ls extracted/` before committing the `ENTRYPOINT`.

### Prove the layering does what you think

```bash
# 1. Build both.
docker build -f Dockerfile.naive -t orderflow:naive .
docker build -f Dockerfile        -t orderflow:layered .

# 2. Change ONE line of Java. Rebuild both. Watch the build output.
./mvnw -q package -DskipTests
docker build -f Dockerfile.naive -t orderflow:naive .    | tee naive-build.log
docker build -f Dockerfile        -t orderflow:layered . | tee layered-build.log

# 3. Count cache hits.
grep -c CACHED naive-build.log
grep -c CACHED layered-build.log

# 4. Look at the layers and their sizes.
docker history orderflow:naive   --format '{{.Size}}\t{{.CreatedBy}}' | head -20
docker history orderflow:layered --format '{{.Size}}\t{{.CreatedBy}}' | head -20
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `docker history` on `naive` shows one very large `COPY` layer | Everything is in one blob. Every commit reships all of it |
| `docker history` on `layered` shows four `COPY` layers, one large and three small | Correct split. Only the small one changes per commit |
| The layered rebuild reports `CACHED` for the dependency `COPY` | The cache is working — this is the deploy-time saving, made visible |
| The layered rebuild rebuilds *all* four `COPY` steps | The order is wrong, or the extraction ran after something volatile, or the jar timestamp changed the digest |
| Both builds take the same wall-clock time | Layering helps *push and pull*, not necessarily local build time. Measure the right thing (see Measurement) |

**A gotcha worth knowing now:** if your build produces a jar whose bytes differ on every
build even when the source is identical (embedded timestamps, non-deterministic ordering),
the dependency layer's *content* is stable but the extraction step's *input* is not, so the
cache may still miss. Reproducible builds (Topic 34) and layering are related problems.

---

## Example 2 — production scenario (on the project spine)

### The constraints

- `orderflow` at the Topic 65 baseline, deployed to Kubernetes as multiple pods.
- **CPU limits from Topic 82 apply during startup too** — and startup is the most
  CPU-hungry phase the process ever has, because class loading and JIT compilation are both
  running flat out. A tight CPU quota makes startup disproportionately slower.
- **Every rolling deploy pays startup once per pod**, and Topic 123's zero-dropped-request
  deploy depends on new pods becoming ready promptly.
- **Topic 121's probes are already configured**, including a startup probe. Startup time and
  probe configuration are coupled; changing one without the other causes crash loops.
- **Topic 74's JIT warm-up** means "started" and "fast" are different moments. A pod that
  passes readiness is not yet at steady-state latency.
- The image must be non-root, pinned by digest, and contain no secrets (Topic 123).

### The production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.7

########################################################################
# STAGE 1 — build. Cached on the POM, exactly like package.json in Node.
########################################################################
FROM eclipse-temurin:25-jdk@sha256:<pinned-digest> AS build
WORKDIR /build

# Dependency resolution is its own cached layer: it changes only when the POM does.
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B dependency:go-offline

# Now the source. This layer changes every commit; the one above does not.
COPY src/ src/
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B -DskipTests package

########################################################################
# STAGE 2 — extract into layers.
########################################################################
FROM eclipse-temurin:25-jdk@sha256:<pinned-digest> AS extract
WORKDIR /extract
COPY --from=build /build/target/orderflow.jar orderflow.jar
RUN java -Djarmode=tools -jar orderflow.jar extract --layers --launcher --destination .

########################################################################
# STAGE 3 — the AppCDS training run.
# Start the application against the SAME classpath layout the runtime image will
# use, let the context refresh, and exit. The archive is the by-product.
########################################################################
FROM eclipse-temurin:25-jdk@sha256:<pinned-digest> AS cds
WORKDIR /app
COPY --from=extract /extract/dependencies/ ./
COPY --from=extract /extract/spring-boot-loader/ ./
COPY --from=extract /extract/snapshot-dependencies/ ./
COPY --from=extract /extract/application/ ./

# spring.context.exit=onRefresh starts the context fully, then exits cleanly, so the
# archive covers everything loaded up to a refreshed context. Verify this property on
# your Boot version; if it is unavailable, a training run with a short-lived profile
# and a scripted shutdown is an equivalent, if uglier, substitute.
RUN java -XX:ArchiveClassesAtExit=/app/orderflow.jsa \
         -Dspring.context.exit=onRefresh \
         -Dspring.profiles.active=cds-training \
         org.springframework.boot.loader.launch.JarLauncher

########################################################################
# STAGE 4 — the runtime image.
########################################################################
FROM eclipse-temurin:25-jre@sha256:<pinned-digest>

RUN groupadd --system --gid 10001 orderflow \
 && useradd  --system --uid 10001 --gid orderflow orderflow
USER 10001:10001
WORKDIR /app

COPY --from=extract --chown=10001:10001 /extract/dependencies/           ./
COPY --from=extract --chown=10001:10001 /extract/spring-boot-loader/     ./
COPY --from=extract --chown=10001:10001 /extract/snapshot-dependencies/  ./
COPY --from=extract --chown=10001:10001 /extract/application/            ./
COPY --from=cds     --chown=10001:10001 /app/orderflow.jsa               ./orderflow.jsa

EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:SharedArchiveFile=/app/orderflow.jsa", \
  "-XX:MaxRAMPercentage=70", \
  "-XX:+ExitOnOutOfMemoryError", \
  "-XX:+HeapDumpOnOutOfMemoryError", \
  "-XX:HeapDumpPath=/tmp/heapdump.hprof", \
  "-Xlog:gc*:file=/tmp/gc.log:time,uptime,level,tags:filecount=5,filesize=10M", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

**Six decisions in that file worth being able to defend:**

1. **The training run is in the image build, not a manual step.** An archive generated by a
   human on a laptop is an archive that will silently stop matching. Building it in the same
   pipeline, from the same layers, with the same JDK, is what makes the classpath and version
   constraints hold by construction.
2. **The training-run stage copies the *same* extracted layers as the runtime stage.** This is
   not tidiness — a classpath mismatch invalidates the archive silently.
3. **JDK image for training, JRE image for runtime.** The archive must be generated by a
   compatible JDK; keep the *version* identical across all stages by pinning the same digest
   family. (If the JRE and JDK images differ in build, verify the archive is actually used —
   Proof 4.)
4. **`-XX:SharedArchiveFile` without `-Xshare:on`.** In production, a missing or invalid
   archive should be a *slow start*, not a *failed start*. `-Xshare:on` makes mismatch fatal;
   that is exactly what you want in **CI**, and exactly what you do not want at 3am.
5. **Non-root with an explicit numeric UID.** `USER orderflow` alone is not enough for some
   Kubernetes `runAsNonRoot` policies, which need a numeric UID they can verify without
   resolving the image's `/etc/passwd`.
6. **Digest-pinned base images.** A tag is mutable. `eclipse-temurin:25-jre` today and next
   month are different images, which breaks both reproducibility (Topic 34) and — relevant
   here — your CDS archive, because the JDK build changed underneath you.

### The CI gate that keeps the archive honest

The archive failing silently is the whole hazard, so make the build fail instead:

```bash
# In CI, after building the image: assert the archive is actually usable.
docker run --rm --entrypoint java orderflow:${TAG} \
  -Xshare:on \
  -XX:SharedArchiveFile=/app/orderflow.jsa \
  -Xlog:cds=info \
  -version
# -Xshare:on makes a mismatch FATAL, so a broken archive fails the pipeline
# rather than silently costing you the benefit in production.
```

### The interaction with probes and warm-up

Startup time and Topic 121's probes are one system, not two.

```yaml
# The startup probe's budget must exceed your MEASURED worst-case cold start,
# with margin for a cold node pulling the image and a CPU-contended node.
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 8081 }
  periodSeconds: 5
  failureThreshold: 30          # 30 x 5s = 150s budget -- derive from measurement
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8081 }
  periodSeconds: 5
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8081 }
  periodSeconds: 10
```

**And the part almost everyone gets wrong:** passing readiness means "I can serve a request",
not "I can serve a request *fast*". A freshly started JVM is running interpreted or
lightly-compiled code (Topic 74). The moment it joins the load balancer it receives its full
share of traffic and answers slowly, so **your p99 spikes on every deploy** — and the spike is
in the tail, so an average-based dashboard will not show it (Topic 118, Trap 2).

Two honest mitigations, and the trade-off between them:

```java
/**
 * Warm the hot paths BEFORE reporting readiness. Costs startup time; buys a flat p99
 * across the deploy. Whether that trade is right depends on which one your SLO is
 * written against -- state the reasoning, do not just pick one.
 */
@Component
class StartupWarmup {

    private final ApplicationAvailability availability;

    @EventListener(ApplicationReadyEvent.class)
    void warmUp() {
        // Exercise the paths Topic 65's load hits, enough times to trigger C1/C2
        // compilation of the hot methods and to fill the connection pool.
        for (int i = 0; i < 200; i++) {
            productService.findBySku(WARMUP_SKUS.get(i % WARMUP_SKUS.size()));
            orderQueryService.recentOrders(PageRequest.of(0, 20));
        }
        // Only NOW publish readiness (Topic 121's AvailabilityChangeEvent).
        AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC);
    }
}
```

The alternative is a slow ramp at the load balancer or service mesh, which shifts the problem
out of the application. Both are defensible; the one thing that is not defensible is being
surprised by the p99 spike at every deploy.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the fat jar in one layer

**Wrong approach**

```dockerfile
FROM eclipse-temurin:25-jre
COPY target/orderflow.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

**Exact symptom**

- Every CI build pushes the full image size to the registry, even for a one-character
  change. The push step's duration is roughly constant regardless of the diff.
- Rolling deploys are slow to *start*, because each node must pull the changed layer before
  it can start the pod — and the changed layer is everything.
- Registry storage grows by roughly the full artefact size per build. On a busy repository
  this becomes a visible cost line and a retention-policy argument.
- Rollback is equally slow, which is the worst time for it to be slow.
- `docker history` shows one enormous `COPY` layer and nothing else of consequence.

**Root cause**

Layer digests are content-addressed. A fat jar contains dependencies and application classes
in one file, so any application change changes the file, changes the digest, and invalidates
the layer. Docker cannot know that 95% of the bytes are identical to last time; the unit of
caching is the layer, not the byte.

**Fix**

Extract into layers and `COPY` them in volatility order (Example 1). Then verify the fix
rather than assuming it:

```bash
docker build -t orderflow:new . 2>&1 | grep -E 'CACHED|COPY'
# The dependency COPY must say CACHED after a source-only change.
```

**A second-order fix worth mentioning in a review:** if your dependency layer is very large,
split it further in the build plugin's layer configuration — for example internal company
libraries into their own layer, since those change on a different cadence than third-party
releases. Do this when you have measured that it matters, not by default.

---

### Trap 2 — `COPY . .` and no `.dockerignore`

**Wrong approach**

```dockerfile
FROM eclipse-temurin:25-jdk AS build
WORKDIR /build
COPY . .                      # <- the whole repository, including .git and target/
RUN ./mvnw package -DskipTests
```

**Exact symptom**

- The dependency-resolution step **never** hits the cache, even when `pom.xml` has not
  changed. Every build downloads every dependency again.
- Build times are constant and slow regardless of how small the change is, which trains the
  team to believe "Java builds are just slow".
- The build context upload itself is slow — Docker reports sending a large context before the
  build even begins.
- Occasionally, a locally built `target/` directory gets copied into the image and *shadows*
  the artefact the build produces, causing a container that runs stale code. This one is
  genuinely maddening to diagnose.
- **A secret in a local `.env` or `application-local.yml` ends up in an image layer**, where
  it survives even if a later step deletes the file — layers are additive, and `docker
  history` plus a layer extract will show it (Topic 123).

**Root cause**

`COPY . .` hashes the entire build context. `.git` changes on every commit; `target/` changes
on every build. So the layer's input digest changes every time, invalidating that step and
everything after it — including the dependency resolution you carefully put first.

**Fix**

```
# .dockerignore
.git
.gitignore
target/
build/
*.iml
.idea/
.vscode/
**/*.log
**/.env
**/application-local.yml
docs/
load/results/
```

…and copy the build inputs explicitly rather than wholesale:

```dockerfile
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw -B dependency:go-offline      # cached on the POM alone
COPY src/ src/                            # only now the volatile part
RUN ./mvnw -B -DskipTests package
```

**Verify:**

```bash
touch src/main/java/com/orderflow/orders/OrderService.java
docker build -t orderflow:test . 2>&1 | grep -E 'CACHED|RUN|COPY'
# dependency:go-offline MUST be CACHED. If it is not, the .dockerignore is incomplete.
```

---

### Trap 3 — the AppCDS archive that is silently ignored

**Wrong approach**

The archive is generated once, by hand, on a developer machine, and committed or copied into
the image:

```dockerfile
COPY orderflow.jsa /app/orderflow.jsa
ENTRYPOINT ["java", "-XX:SharedArchiveFile=/app/orderflow.jsa", "-jar", "/app/app.jar"]
```

**Exact symptom**

- Startup time is **unchanged**. Not worse — exactly the same as without the flag.
- No error, no warning, no log line at the default logging level. The flag appears to be
  accepted.
- It worked when it was first measured, then quietly stopped after a base-image bump, a
  dependency addition, or a JDK patch release — and nobody connected the two events.
- A colleague concludes "AppCDS doesn't really help for Spring apps" and the team abandons a
  technique that works.

**Root cause**

The JVM validates the archive against the current runtime: JDK version and build, classpath
composition and order, and certain JVM options. Any mismatch and it **falls back to normal
class loading silently**, because the default `-Xshare:auto` mode is explicitly designed to
degrade gracefully rather than fail. Graceful degradation is right for production and
terrible for feedback.

The three common triggers, in order of frequency:
1. **Adding or removing a dependency** — the classpath changed.
2. **A base image bump** — the JDK build changed.
3. **Generating the archive with a different launch layout than production uses** — fat jar
   at training time, exploded at runtime, or vice versa.

**Fix**

1. **Generate the archive in the image build**, from the exact layers the runtime stage uses
   (Example 2 stage 3). This makes drift structurally impossible rather than a discipline
   problem.
2. **Assert it in CI with `-Xshare:on`**, which turns a mismatch into a hard failure:

```bash
docker run --rm --entrypoint java orderflow:${TAG} \
  -Xshare:on -XX:SharedArchiveFile=/app/orderflow.jsa -Xlog:cds=info -version
```

3. **Log the outcome at startup in every environment**, so you can answer "is it on right
   now?" from a log search rather than an experiment:

```
-Xlog:class+load=info:file=/tmp/classload.log:uptime
-Xlog:cds=info:stdout
```

4. **Keep `-Xshare:auto` (the default) in production.** CI fails loudly; production degrades
   quietly. That asymmetry is deliberate and is the right pairing.

**How to check, right now, whether it is working:**

```bash
# Classes loaded FROM the archive are marked "shared file" in class+load output.
docker run --rm orderflow:latest \
  java -Xlog:class+load=info -XX:SharedArchiveFile=/app/orderflow.jsa -version \
  | grep -c 'shared file'
```

Zero means the archive is not being used, whatever the flag says.

---

### Trap 4 — reaching for native-image before measuring

**Wrong approach**

"Startup takes 12 seconds. Let's go GraalVM native."

**Exact symptom**

The symptom here is organisational, and it is worth describing precisely because you will
recognise it:

- Two to six weeks of engineering time disappears.
- Reflection-based code fails **at runtime, in a deployed environment, not at build time**
  (Topic 83's closed-world analysis) — Jackson deserialization of a class it did not know
  about, a dynamic proxy (Topic 40), a resource loaded by a path built at runtime.
- Every one of those needs a reachability-metadata hint, discovered one production failure at
  a time.
- Build times go from tens of seconds to many minutes, which slows every subsequent change.
- The debugging story changes entirely: no `jcmd`, no heap dump in the familiar format, no
  JFR in the way you learned in Phase 8. **Every diagnostic skill from Phase 8 becomes
  partially inapplicable.**
- And often, at the end: startup is dramatically faster, peak throughput is *lower* (no
  profile-guided JIT), memory is lower, and **the original problem was a 4-second connection
  pool warm-up inside the context refresh** that one property would have fixed.

**Root cause**

Optimising a number nobody decomposed. "12 seconds" is not a problem statement; it is a sum
of four unrelated phases with four unrelated fixes:

| If the time is in… | The fix is… | Effort |
|---|---|---|
| JVM init | heap sizing, CPU quota (Topic 82) | minutes |
| Class loading | **AppCDS**, fewer dependencies | hours |
| Context refresh | fewer auto-configurations, narrower scanning, Spring AOT | hours to days |
| Bean instantiation | lazy init, deferred pool/cache warm-up | hours |
| **All of the above, and you need sub-second** | native-image | weeks |

Native-image is the right answer for a genuine cold-start-per-request workload — a function
platform, a CLI, a scale-to-zero service. It is an expensive answer to "our rolling deploy
takes a while".

**Fix**

Decompose first. The Measurement section is the procedure, in order, and it takes about
thirty minutes. Then:

1. Fix the largest slice with the cheapest lever available for that slice.
2. Re-measure.
3. Repeat until the number meets the *requirement* — and write the requirement down first,
   because "faster" is not a target and cannot be met.
4. **Only then** ask whether native-image is worth its cost, with a decomposed number in hand
   and a written statement of what you would give up (Topic 83's list, and Topic 126's
   build-vs-buy framing).

---

### Trap 5 — startup time and the probes disagree

**Wrong approach**

The startup probe is copied from a Node service's manifest:

```yaml
livenessProbe:
  httpGet: { path: /actuator/health, port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
# no startupProbe at all
```

**Exact symptom**

- Pods enter `CrashLoopBackOff` **immediately after a deploy**, with no application exception
  in the logs — the logs simply stop mid-startup.
- `kubectl describe pod` shows `Liveness probe failed: ... connection refused` and
  `Killing container`.
- It works on a warm laptop and fails in the cluster, or works in staging and fails in
  production, because the cluster's CPU quota (Topic 82) makes startup slower there.
- It is intermittent: it fails on a busy node and succeeds on an idle one, which sends people
  looking for a nondeterminism bug that does not exist.
- The subtler version: the probes are fine, pods start, and **p99 spikes on every single
  deploy** because pods take traffic before the JIT has warmed (Topic 74).

**Root cause**

Two distinct errors that look the same from outside.

First: `initialDelaySeconds: 10` plus `failureThreshold: 3` at `periodSeconds: 5` gives a JVM
about 25 seconds before it is declared dead. A Spring Boot service on a CPU-quota-limited pod
can legitimately need more, especially on a cold node where the image pull is also competing.
There is no startup probe to suppress liveness during initialisation, so liveness kills the
pod mid-startup — and the restart is *slower*, because the node is now busier. That is a
self-reinforcing loop.

Second: readiness answers "can I serve", not "can I serve at target latency". The JVM is
correct to say yes and still be slow.

**Fix**

1. **Measure your cold start under production-shaped CPU limits**, not on your laptop. The
   number you want is the worst case, not the median.
2. **Add a startup probe** with a budget generously above that, and let liveness be
   aggressive only *after* it passes:

```yaml
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 8081 }
  periodSeconds: 5
  failureThreshold: 30           # derive from your measurement; this is an example shape
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8081 }   # NOT the aggregate endpoint
  periodSeconds: 10
  failureThreshold: 3
```

3. **Reduce the startup time itself** (that is the rest of this document), which shrinks the
   budget you need and therefore shrinks how long a genuinely wedged JVM survives.
4. **Decide explicitly about warm-up**: either warm before publishing readiness, or ramp
   traffic at the load balancer. Then look at the p99 panel across a deploy and confirm.

**Connect it back to Topic 121:** liveness must point at a liveness *group*, never the
aggregate `/actuator/health`. A slow database plus a liveness probe that checks it turns a
dependency blip into a cluster-wide restart cascade — and a long startup makes that cascade
much harder to recover from, because every restarting pod takes 30+ seconds to come back.

---

## Hands-on proof

### Setup

```bash
cd /path/to/orderflow
./mvnw -q -DskipTests package
ls -la target/orderflow.jar
```

### Proof 1 — is the jar layered at all?

```bash
# Boot 3.3+ / 4.x
java -Djarmode=tools -jar target/orderflow.jar list-layers

# Boot <= 3.2
java -Djarmode=layertools -jar target/orderflow.jar list

# Is the index actually in the jar?
unzip -l target/orderflow.jar | grep layers.idx

# What is the jar made of, by size?
unzip -l target/orderflow.jar | sort -rn -k1 | head -20
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| Four layer names printed | Layering is enabled. Your Dockerfile can use it |
| `Unknown jarmode` or an error | Wrong command for your Boot version — try the other one. If both fail, layering is disabled in the plugin config |
| No `layers.idx` in the jar listing | The build plugin has layering turned off. Enable it in the plugin configuration |
| `BOOT-INF/lib/` dominating the size listing | Expected, and it is the argument for layering in one line |

### Proof 2 — the layer cache, demonstrated

```bash
docker build -t orderflow:v1 . 2>&1 | tee build1.log

# Change one line of application code only.
echo "// $(date)" >> src/main/java/com/orderflow/orders/OrderService.java
./mvnw -q -DskipTests package

docker build -t orderflow:v2 . 2>&1 | tee build2.log

grep -E 'CACHED|COPY|RUN' build2.log
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `CACHED` on the dependency `COPY` and on `dependency:go-offline` | Correct. Your per-commit delta is the application layer only |
| No `CACHED` anywhere | Something volatile is copied early — missing `.dockerignore`, or the build context includes `.git`/`target` |
| `CACHED` on dependencies but not `spring-boot-loader` | Harmless: the loader layer is tiny. Check the `COPY` order anyway |
| Everything cached including `application` | Your change did not reach the jar — check the `package` actually ran |

### Proof 3 — measure the image, not your feelings

```bash
docker images orderflow --format '{{.Tag}}\t{{.Size}}'
docker history orderflow:v2 --format '{{.Size}}\t{{.CreatedBy}}' | head -20

# The size that actually matters on a deploy: the layers that CHANGED.
docker inspect orderflow:v1 --format '{{json .RootFS.Layers}}' | jq -r '.[]' > l1.txt
docker inspect orderflow:v2 --format '{{json .RootFS.Layers}}' | jq -r '.[]' > l2.txt
diff l1.txt l2.txt
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| One differing layer digest between v1 and v2 | Only the application layer changed. This is the whole objective |
| All layer digests differ | No caching is happening at all — revisit Proof 2 |
| Total size dominated by the base image | Normal for Java. Consider a slimmer base or `jlink` (Topic 20) if it matters |

### Proof 4 — is the CDS archive actually being used?

```bash
# Classes resolved from the archive are annotated "shared file".
java -Xlog:class+load=info \
     -XX:SharedArchiveFile=orderflow.jsa \
     -jar target/orderflow.jar 2>&1 | grep -c 'shared file'

# Total classes loaded, for the ratio.
java -Xlog:class+load=info -jar target/orderflow.jar 2>&1 | grep -c 'source:'

# Make a mismatch fatal, so you SEE it.
java -Xshare:on -XX:SharedArchiveFile=orderflow.jsa -version

# The CDS subsystem's own view.
java -Xlog:cds=info -XX:SharedArchiveFile=orderflow.jsa -version
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| A large count of `shared file` loads | The archive is in use. The ratio to total loads is your coverage |
| Zero `shared file` loads | The archive is being ignored. `-Xshare:on` will tell you why |
| `-Xshare:on` fails with a mismatch message | **The useful outcome.** It names the reason: JDK version, classpath, or an option |
| `-Xshare:on` succeeds but startup is unchanged | Class loading was not your bottleneck. Go decompose properly before optimising further |

### Proof 5 — decompose the startup, which is the point of the whole topic

```bash
# 1. The two numbers Boot itself gives you.
docker run --rm orderflow:v2 2>&1 | grep 'Started'
#    "Started OrderflowApplication in X seconds (process running for Y)"
#    JVM init + pre-Spring class loading  ≈  Y - X
#    Spring context refresh + beans        =  X

# 2. Class loading volume.
docker run --rm --entrypoint java orderflow:v2 \
  -Xlog:class+load=info:file=/tmp/cl.log \
  org.springframework.boot.loader.launch.JarLauncher &
# then, in the container or via a bind mount:
wc -l /tmp/cl.log
grep -c 'shared file' /tmp/cl.log

# 3. Per-step Spring timings.
```

For step 3, enable Boot's startup tracking:

```java
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(OrderflowApplication.class);
    app.setApplicationStartup(new BufferingApplicationStartup(4096));
    app.run(args);
}
```

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus,startup
```

```bash
# The endpoint DRAINS the buffer, so the first call gets the data and the second is empty.
curl -s -X POST localhost:8080/actuator/startup > startup.json

# The 20 slowest steps.
jq -r '.timeline.events
       | sort_by(-.duration)
       | .[0:20]
       | .[] | "\(.duration)\t\(.startupStep.name)\t\(.startupStep.tags[]?.value // "")"' startup.json
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `Y − X` is large relative to `X` | The time is *before Spring*. AppCDS, JVM flags, CPU quota — Spring tuning will not help |
| `X` is large and `spring.beans.instantiate` dominates | Bean construction. Look at the specific slow beans by name |
| One bean taking a large share, often a `DataSource` or a cache warmer | A blocking initialisation. Defer it, or do it after readiness |
| `spring.context.config` / condition evaluation dominating | Auto-configuration and scanning. Narrow `@ComponentScan`, exclude unused auto-configurations, consider Spring AOT |
| `/actuator/startup` returns an empty timeline | You called it twice (it drains), or `BufferingApplicationStartup` was not set |

---

## Failure drill

**The drill for this topic has two parts. Part 1 breaks the layer cache; Part 2 breaks the
CDS archive.** Both failures are *silent* — no error, no warning, just a cost you pay
forever.

### Part 1 — the fat-jar layer, and what a deploy really costs

**Setup.** Build `orderflow` two ways: `Dockerfile.naive` (single `COPY` of the fat jar) and
the layered `Dockerfile`. Push both to a local registry so push and pull are measurable.

```bash
docker run -d -p 5000:5000 --name registry registry:2
docker build -f Dockerfile.naive -t localhost:5000/orderflow:naive-v1 .
docker build -f Dockerfile        -t localhost:5000/orderflow:layered-v1 .
docker push localhost:5000/orderflow:naive-v1
docker push localhost:5000/orderflow:layered-v1
```

**Break it.** Change one line of Java. Rebuild and push both again as `-v2`.

```bash
echo "// change $(date)" >> src/main/java/com/orderflow/orders/OrderService.java
./mvnw -q -DskipTests package
time docker build -f Dockerfile.naive -t localhost:5000/orderflow:naive-v2 .
time docker build -f Dockerfile        -t localhost:5000/orderflow:layered-v2 .
time docker push localhost:5000/orderflow:naive-v2
time docker push localhost:5000/orderflow:layered-v2

# Now the pull, which is what a node does during a rollout.
docker rmi localhost:5000/orderflow:naive-v2 localhost:5000/orderflow:layered-v2
time docker pull localhost:5000/orderflow:naive-v2
time docker pull localhost:5000/orderflow:layered-v2
```

**Fill in — layer cache (blank template):**

| Measurement | Naive (fat jar) | Layered |
|---|---|---|
| Total image size | | |
| Number of layers | | |
| Layers whose digest changed v1 → v2 | | |
| Bytes transferred on push v2 | | |
| Push wall-clock (v2) | | |
| Pull wall-clock (v2, cold local cache) | | |
| Pull wall-clock (v2, v1 already present locally) | | |
| Build wall-clock (v2) | | |
| `CACHED` step count in the v2 build | | |

**How to read it**

| What you see | What it means |
|---|---|
| Naive pull time with v1 present ≈ pull time with nothing present | Nothing was reusable. Every node re-downloads everything, every deploy |
| Layered pull with v1 present is much shorter | The dependency layers were reused — this is the rollout speedup, quantified |
| Both builds take similar wall-clock time locally | Expected. Layering helps *distribution*, not compilation. Do not oversell it |
| Layered push transfers nearly as much as naive | Your layer ordering is wrong, or the jar is non-reproducible so the dependency layer changes anyway |

Multiply the pull delta by your pod count to get the per-deploy cost. That is the number for
Topic 129's cost model and Topic 124's review.

### Part 2 — the archive that stops working

**Setup.** Build the image with the CDS stage (Example 2) and confirm the archive is used:

```bash
docker run --rm --entrypoint java orderflow:cds \
  -Xshare:on -XX:SharedArchiveFile=/app/orderflow.jsa -Xlog:cds=info -version
```

Record a cold start with and without `-XX:SharedArchiveFile`, five runs each, taking the
**median** (a single run is noise; see Topic 77's discipline).

**Fill in — CDS baseline (blank template):**

| Configuration | Run 1 | Run 2 | Run 3 | Run 4 | Run 5 | Median |
|---|---|---|---|---|---|---|
| No archive: "Started in X" | | | | | | |
| No archive: "process running for Y" | | | | | | |
| With archive: "Started in X" | | | | | | |
| With archive: "process running for Y" | | | | | | |
| Classes loaded (total) | | | | | | |
| Classes loaded from `shared file` | | | | | | |

**Break it — three ways, one at a time, re-measuring after each:**

1. **Add a dependency** to `pom.xml` (anything unused; the point is the classpath change).
   Rebuild the *application* layers but **reuse the previously built archive** — simulate this
   by copying the old `.jsa` into the new image instead of regenerating it.
2. **Bump the base image** to a different JDK patch build, again keeping the old archive.
3. **Change the launch layout**: generate the archive from `java -jar app.jar` (nested) and
   run the container with the exploded classpath (or the reverse).

**Fill in — what silently broke (blank template):**

| Mutation | Startup (median) | Classes from `shared file` | Any error at default log level? | `-Xshare:on` message |
|---|---|---|---|---|
| Baseline (archive valid) | | | | |
| Dependency added | | | | |
| Base image bumped | | | | |
| Launch layout changed | | | | |

**How to read it**

| What you see | What it means |
|---|---|
| Startup returns to the no-archive number | The archive was rejected. It is now pure overhead in your image |
| `shared file` count drops to zero | Confirms rejection at the class-loading level |
| Nothing in the logs at default level | **The headline.** This is why it survives in production for months |
| `-Xshare:on` prints a specific mismatch reason | That message is what your CI gate should be catching |

**Fix and re-prove.** Regenerate the archive inside the image build (Example 2 stage 3), add
the `-Xshare:on` CI assertion, and re-run all three mutations. Every one should now either
produce a fresh valid archive or fail the build.

**Write it up** (Topic 133's form):

- **Contributing factors:** the archive was produced outside the build; nothing asserted it
  was valid; the failure mode is graceful degradation by design; no startup-time metric to
  notice the regression.
- **Detection:** what would have caught it, and how long after the change.
- **Prevention that scales:** generate in-build, assert with `-Xshare:on` in CI, and graph
  startup time as a metric per deploy so a regression is visible without anyone looking for
  it.

---

## Measurement

### The rule: decompose before you optimise

This is the section the topic exists for. **Do these four steps in order, before changing
anything.** They take about half an hour and they routinely change what people do next.

#### Step 0 — write down the requirement

Not "faster". A number, with a reason:

**Fill in — the requirement (blank template):**

| Question | Your answer |
|---|---|
| What is the current cold start, measured under production CPU limits (median of 5)? | |
| What is the worst case you have observed? | |
| What breaks because of it? (rollout duration / autoscale lag / recovery time / probe budget) | |
| What number would make that stop being a problem? | |
| What is that improvement worth, in engineering days? | |

If you cannot fill in rows 3 and 4, **stop**. You do not have a performance problem; you have
a number you do not like. That distinction is most of the senior judgement in this topic.

#### Step 1 — split JVM init from Spring

```bash
docker run --rm --cpus=1 --memory=1g orderflow:current 2>&1 | grep 'Started'
```

`Started ... in X seconds (process running for Y)`:

- **`X`** = Spring: context refresh + bean instantiation.
- **`Y − X`** = everything before Spring: process start, JVM init, and the class loading that
  happens getting to `main`.

**Run it under the same CPU limit as production.** Startup is the most CPU-intensive phase the
JVM ever has (class loading and JIT compilation together), so a laptop measurement is not a
prediction — this is Topic 82's cascade applied to startup.

#### Step 2 — measure class loading

```bash
docker run --rm --cpus=1 orderflow:current \
  java -Xlog:class+load=info:file=/tmp/cl.log \
       org.springframework.boot.loader.launch.JarLauncher

wc -l /tmp/cl.log                       # total classes loaded
grep -c 'shared file' /tmp/cl.log       # how many came from a CDS archive
awk '{print $NF}' /tmp/cl.log | sort | uniq -c | sort -rn | head   # by source
```

#### Step 3 — measure inside Spring

`BufferingApplicationStartup` + `POST /actuator/startup` (Proof 5), then sort by duration.

#### Step 4 — attribute the total

**Fill in — startup decomposition (blank template — every cell measured, none estimated):**

| Phase | How measured | Duration | % of total |
|---|---|---|---|
| JVM init + pre-Spring class loading | `Y − X` from the Boot log line | | |
| Spring context refresh (scanning, conditions, definitions) | `/actuator/startup`, non-bean steps | | |
| Bean instantiation | `/actuator/startup`, `spring.beans.instantiate` steps | | |
| — of which, the single slowest bean | `/actuator/startup`, sorted | | |
| — of which, `DataSource` / pool initialisation | `/actuator/startup` | | |
| **Total (`Y`)** | Boot log line | | 100% |
| Classes loaded (count) | `wc -l` on `class+load` | | — |
| Classes from CDS archive | `grep -c 'shared file'` | | — |

**Now, and only now, choose the lever:**

| Largest slice | Lever | Effort |
|---|---|---|
| JVM init | heap sizing, `-XX:TieredStopAtLevel=1` for short-lived processes, CPU request | minutes |
| Class loading | **AppCDS**, exploded layout, fewer dependencies | hours |
| Context refresh | exclude unused auto-configurations, narrow `@ComponentScan`, Spring AOT | hours–days |
| Bean instantiation | `spring.main.lazy-initialization`, defer pool warm-up, remove eager caches | hours |
| Everything, and you need sub-second | native-image (Topic 83) | weeks |

### The cheap levers, with their honest costs

| Lever | Change | Cost / risk |
|---|---|---|
| AppCDS | `-XX:ArchiveClassesAtExit` then `-XX:SharedArchiveFile` | build complexity; silent invalidation (Trap 3) |
| Exploded layout | run the extracted classpath, not the nested jar | slightly more complex image |
| Lazy initialisation | `spring.main.lazy-initialization=true` | **moves failures from startup to first request** — a genuinely bad trade for a service that must fail fast. Prefer `@Lazy` on specific beans |
| Exclude auto-configurations | `spring.autoconfigure.exclude=...` | you must know what you are removing |
| Defer pool warm-up | reduce `minimumIdle`, or open the pool after readiness | first requests pay connection setup |
| Fewer dependencies | delete unused starters | requires knowing what is unused |
| `-XX:TieredStopAtLevel=1` | JIT stops at C1 | **faster startup, lower peak throughput.** Correct for a short-lived job; wrong for `orderflow` |

**`spring.main.lazy-initialization=true` deserves the warning it gets.** It genuinely speeds
startup and it converts "the application refuses to start because a bean is misconfigured"
into "the first request that touches that bean fails in production". Trading a loud
build-time failure for a quiet runtime one is usually the wrong direction, and it is worth
saying so out loud when someone proposes it.

### Make startup a permanent metric, not a one-off experiment

```java
@Component
class StartupTimeMetric {

    StartupTimeMetric(MeterRegistry registry, ApplicationContext context) {
        long jvmStart = ManagementFactory.getRuntimeMXBean().getStartTime();
        long ready = System.currentTimeMillis();

        // A TimeGauge recorded once at startup: queryable per deploy, per version.
        TimeGauge.builder("orderflow.startup.duration",
                          () -> (double) (ready - jvmStart),
                          TimeUnit.MILLISECONDS)
                .description("JVM start to application ready")
                .register(registry);
    }
}
```

With Topic 118's `version` common tag, this gives you startup time **per release**, so a
regression — a new dependency, a broken CDS archive, an eager bean someone added — is visible
in a graph rather than discovered during an incident.

**Fill in — startup over time (blank template):**

| Version / commit | Median cold start | JVM init share | Spring share | CDS active? | What changed |
|---|---|---|---|---|---|
| | | | | | |

---

## Practice exercises

### 1 — Easy: layer your image and prove the cache works

1. Confirm whether your jar is layered (`list-layers` / `list`).
2. Write the two-stage layered Dockerfile from Example 1.
3. Build, change one line of Java, rebuild.
4. Show which layers changed, using `docker inspect` digests.
5. Record image size, layer count, and changed-layer count for both variants.

**Deliverable:** the two Dockerfiles and a filled-in comparison table.
**Acceptance:** exactly one layer digest changes between builds for an application-only
change, and you can point at the `CACHED` lines that prove it.

### 2 — Medium: decompose and attack the largest slice (combines Topics 74, 82, 109, 121)

1. Measure cold start **under production CPU limits**, median of five runs.
2. Fill in the startup decomposition table completely.
3. State, in one sentence, which phase dominates and why.
4. Apply **one** lever appropriate to that phase.
5. Re-measure with the same method and the same number of runs.
6. Confirm the improvement is in the phase you targeted — not just in the total.
7. Check the side effects: peak throughput at the Topic 65 baseline (did
   `TieredStopAtLevel=1` cost you?), pool behaviour on first requests, and p99 across a
   simulated deploy.

**Acceptance:**
- [ ] The decomposition table is complete before any change is made.
- [ ] The improvement is attributed to a named phase with a measurement.
- [ ] Peak throughput at the Topic 65 baseline is unchanged, or the regression is stated and
      justified.
- [ ] The probe budget is updated to match the new measured worst case.

### 3 — Hard: production simulation — a zero-surprise rolling deploy

**Goal:** make a deploy of `orderflow` fast *and* boring.

1. Build with layered jars **and** an in-build CDS archive, with the `-Xshare:on` CI gate.
2. Deploy under the Topic 65 load.
3. Roll out a one-line change and measure, across the whole rollout:
   - image pull time per node,
   - time from pod start to readiness, per pod,
   - **p99 during the rollout versus the steady-state baseline**,
   - error rate during the rollout,
   - total rollout wall-clock (`kubectl rollout status --timeout`).
4. Identify the largest contributor to rollout duration and attack it.
5. If p99 spikes during the rollout, decide between warm-before-ready and load-balancer ramp,
   implement one, and re-measure.

**Fill in — rollout (blank template):**

| Measurement | Before optimisation | After |
|---|---|---|
| Image pull time (cold node) | | |
| Image pull time (previous version cached) | | |
| Pod start → ready (median) | | |
| Pod start → ready (worst) | | |
| p99 during rollout | | |
| p99 at steady state | | |
| Error rate during rollout | | |
| Total `kubectl rollout status` duration | | |

**Acceptance:** rollout duration and the p99 delta both improve, both are measured the same
way before and after, and you can name which change produced which improvement. A total that
improved for reasons you cannot attribute is not an acceptance — it is a coincidence you have
not investigated.

---

## Interview questions

### Q1 — "Startup takes 12 seconds. What do you do?"

**MID-LEVEL ANSWER**

"I'd look at what's slow during startup — probably bean creation. We could enable lazy
initialisation, or look at GraalVM native image, which starts in milliseconds."

**SENIOR ANSWER**

"First I would ask what breaks because of it, because that decides how much it is worth
fixing. Twelve seconds is fine for a service that deploys weekly and is painful for one that
autoscales under bursty load, and those are different problems with different budgets.

Then I decompose before touching anything, because 'startup' is four unrelated phases with
four unrelated fixes. Spring's own log line gives me the first split for free: 'Started in X
seconds, process running for Y'. `Y − X` is JVM init plus class loading before Spring even
begins; `X` is context refresh and bean instantiation. If most of it is in `Y − X`, no amount
of Spring tuning helps, and I would be wasting my time in the wrong file.

Then I go finer. `-Xlog:class+load` tells me how many classes are being loaded and whether any
are coming from a CDS archive. `BufferingApplicationStartup` with the `/actuator/startup`
endpoint gives me per-step timings inside the refresh, sorted by duration, so I can see the
individual slow beans by name.

In practice, for a Spring service, class loading is usually the largest single slice, and
AppCDS attacks it directly for a few hours of build work — a training run that dumps an
archive, and a flag to use it. Bean instantiation is usually the second, and it is normally
one or two specific beans: a connection pool warming eagerly, a cache preload, a client doing
a blocking handshake in its constructor.

And I would measure under the production CPU limit, not on a laptop. Startup is the most
CPU-hungry phase the JVM ever has — class loading and JIT compilation at once — so a CPU quota
changes the number substantially.

Native image I would consider only after that decomposition, and only if the requirement is
genuinely sub-second. It is weeks of work, it breaks reflection and dynamic proxies with
runtime failures rather than build failures, it costs peak throughput because there is no
profile-guided JIT, and it invalidates most of the JVM diagnostic toolkit. It is the right
answer for a scale-to-zero or function workload. It is a very expensive answer to 'our rolling
deploy feels slow'."

**WHAT SEPARATES THEM**

The mid-level answer jumps to solutions. The senior answer establishes the requirement first,
gives a concrete decomposition procedure with named instruments and what each one tells you,
knows which slice is typically largest for Spring specifically, insists on measuring under
production constraints, and treats native-image as a costed decision with named trade-offs
rather than a faster option.

**FOLLOW-UP:** *"You decompose and it is bean instantiation. Now what?"* — Look at which beans,
by name, from the `/actuator/startup` timeline. It is almost always something doing I/O in a
constructor: a connection pool opening its minimum-idle connections, a cache preloading the
catalogue, a client fetching configuration. The fix is to defer that work past readiness
rather than to lazily initialise the whole application, because global lazy initialisation
converts startup failures into first-request failures — trading a loud, safe failure for a
quiet, production one.

---

### Q2 — "What are layered jars and why do they matter in Docker?"

**MID-LEVEL ANSWER**

"They split the jar into layers so Docker can cache the dependencies separately from the
application code, which makes builds faster."

**SENIOR ANSWER**

"They matter for *distribution*, more than for build time, and that distinction is the point.

A Docker layer is content-addressed, so changing one byte changes the layer digest and every
layer after it. A Boot fat jar puts your classes and every dependency in one file, so a
one-character change makes the whole artefact a new layer. Every build then pushes it all to
the registry, and every node pulls it all before it can start a pod. That lands directly on
rollout duration and on rollback duration — which is exactly when you least want to wait.

Layered jars write a `layers.idx` into the jar, and `java -Djarmode=tools -jar app.jar extract
--layers` splits it into dependencies, the Boot loader, snapshot dependencies, and your
application. You `COPY` them in that order, least volatile first, so a source change
invalidates only the last, smallest layer.

The command name is worth knowing precisely because half the tutorials are stale: it was
`-Djarmode=layertools ... extract` from Boot 2.3, and it became `-Djarmode=tools ...
extract` in Boot 3.3, which is what Boot 4 uses.

It is exactly the reasoning behind `COPY package.json` before `COPY .` in a Node image, and
Java's default packaging happens to destroy that split unless you ask for it back.

Two things I would check beyond the plugin flag. First, `.dockerignore` — `COPY . .` with a
`.git` directory in the context invalidates the cache on every commit no matter how good your
layering is. Second, whether the build is reproducible enough that the dependency layer's
bytes are actually stable; if the jar embeds timestamps, the layer digest churns anyway.

And buildpacks — `spring-boot:build-image` — do all of this correctly with no Dockerfile at
all. I would still want the team to be able to read the Dockerfile version, for the same reason
I want them to be able to read the auto-configuration report."

**WHAT SEPARATES THEM**

Naming distribution rather than build speed as the benefit; explaining content-addressed
layers and cascading invalidation; getting the `jarmode` history right; knowing `.dockerignore`
and reproducibility are prerequisites; and knowing buildpacks exist and when they are the
right call.

**FOLLOW-UP:** *"How would you verify the layering is actually working?"* — Build, change one
line of Java, rebuild, and diff the layer digests with `docker inspect --format
'{{json .RootFS.Layers}}'`. Exactly one should differ. Then push to a registry and time it —
the transferred bytes are the number that matters, and the build output's `CACHED` lines are
the explanation.

---

### Q3 — "What is AppCDS and when would you use it?"

**MID-LEVEL ANSWER**

"Class Data Sharing — it caches loaded classes so the JVM starts faster. You generate an
archive and point the JVM at it."

**SENIOR ANSWER**

"It attacks the largest slice of Spring startup, which is class loading, by moving the parse
work from every start to one build step.

Normally the JVM reads each class file, parses the constant pool, builds its internal metadata,
verifies and links it — thousands of times for a Spring service, identically on every start of
every pod. A CDS archive contains that metadata already built, in a layout the JVM can `mmap`.
Loading a class from the archive becomes a memory mapping rather than a parse. And because the
mapping is read-only, multiple JVMs on the same host share the same physical pages, so it saves
memory as well as time.

The workflow is a training run with `-XX:ArchiveClassesAtExit=app.jsa` — start the app, let the
context refresh, exit cleanly — then run production with `-XX:SharedArchiveFile=app.jsa`. Boot
3.3+ helps: `-Djarmode=tools ... extract` gives you a stable exploded layout, and
`spring.context.exit=onRefresh` makes the training run terminate cleanly at the right moment.

The trap is that the archive is validated against the runtime — JDK build, classpath
composition and order, some JVM options — and if it does not match, the JVM **silently falls
back** to normal class loading. No error, no warning at default log level. So the archive
stops working when someone adds a dependency or bumps the base image, and nobody notices for
months while the flag sits there looking reassuring.

Two things fix that. Generate the archive inside the image build, from the same layers the
runtime stage uses, so drift is structurally impossible. And assert it in CI with
`-Xshare:on`, which makes a mismatch fatal. I keep `-Xshare:auto` in production deliberately:
CI should fail loudly, production should degrade quietly.

Going forward, JDK 24 and 25 generalise this into an AOT cache from Project Leyden, which is a
superset of the same idea. I would read the JEPs for the exact JDK we ship rather than trust a
remembered flag name — the ergonomics have been changing release to release."

**WHAT SEPARATES THEM**

Explaining the mechanism including the memory-sharing benefit; naming the exact workflow and
Boot's supporting features; knowing the silent-failure mode and that it is the actual risk;
proposing structural prevention with the CI/production asymmetry; and being honest about the
AOT-cache direction without inventing flags.

**FOLLOW-UP:** *"How do you know the archive is being used right now?"* —
`-Xlog:class+load=info` and count lines marked `shared file`. Zero means it is not being used,
whatever the flag says. `-Xlog:cds=info` gives the CDS subsystem's own view, and `-Xshare:on`
turns any mismatch into a message naming the reason.

---

### Q4 — "Your pods CrashLoopBackOff after a deploy, with no exception in the logs. Go."

**MID-LEVEL ANSWER**

"I'd check the logs for errors and look at `kubectl describe pod` for the reason. Maybe it's
running out of memory."

**SENIOR ANSWER**

"'No exception, logs just stop' is a strong signal on its own: the process is being killed, not
failing. So I am looking for an external killer, and there are three candidates.

`kubectl describe pod` distinguishes them immediately. `OOMKilled` in the last state is the
kernel — the container exceeded its memory limit, which for a JVM usually means the heap plus
metaspace plus thread stacks plus direct buffers exceeded the limit, and that is Topic 82's
arithmetic. `Liveness probe failed` followed by `Killing container` is the kubelet — the probe's
budget is shorter than the JVM's startup.

The probe case is the one that specifically bites Java teams, because probe settings get copied
from a Node service where startup is a few hundred milliseconds. With
`initialDelaySeconds: 10`, `periodSeconds: 5` and `failureThreshold: 3`, the JVM has about
twenty-five seconds before it is declared dead — and a Spring service on a CPU-limited pod, on
a cold node that is also pulling the image, can legitimately need more. Then it gets killed
mid-startup, restarts, and the node is now busier, so it is slower. Self-reinforcing.

The fix has two halves. A startup probe with a budget derived from the measured worst case
under production CPU limits — not the laptop number — which suppresses liveness until the app
is up. And reducing the actual startup time, so the budget you need is smaller and a genuinely
wedged JVM does not survive for two minutes.

I would also check that liveness points at the liveness *group*, not the aggregate
`/actuator/health`. If it includes a database check, a slow database restarts every pod
simultaneously — which is Topic 121's cascade, and a long startup makes recovery from it much
worse.

And the version of this that does not crash-loop but still hurts: pods start, pass readiness,
take full traffic immediately, and answer slowly because the JIT has not warmed. That shows up
as a p99 spike on every deploy and is invisible on an average-based dashboard."

**WHAT SEPARATES THEM**

Reading "logs stop with no exception" as external termination; distinguishing the three killers
by the exact `describe` output; explaining the self-reinforcing loop; fixing both the budget and
the underlying time; connecting to the liveness-group rule; and naming the non-crashing variant
that most teams never notice.

**FOLLOW-UP:** *"How do you pick the startup probe's `failureThreshold`?"* — From measurement,
not from a round number. Take the cold-start distribution under production CPU limits including
image pull on a cold node, take the worst case, add margin for a contended node, and divide by
`periodSeconds`. Then graph startup time as a metric per release so the budget can be revisited
when it drifts, rather than being set once and forgotten until it crash-loops.

---

### Q5 — "Would you use buildpacks or write a Dockerfile?"

**MID-LEVEL ANSWER**

"Buildpacks are easier — `mvn spring-boot:build-image` and you get an optimised image without
maintaining a Dockerfile."

**SENIOR ANSWER**

"For most teams, buildpacks, and I would say so without embarrassment. They produce a layered,
non-root, sensibly configured image with a memory calculator and CDS support, and they are
maintained by people who think about this full-time. A hand-written Dockerfile is a thing your
team now owns, and most hand-written Java Dockerfiles I have reviewed are a single `COPY` of a
fat jar running as root on a `:latest` base.

I would write a Dockerfile when I need something the buildpack does not give me: an unusual base
image for compliance reasons, a specific CDS or AOT workflow, a native library, or a corporate
base image mandate. And I would write one for a team that needs to *understand* what is
happening, because a buildpack image you cannot explain is the same problem as auto-configuration
you cannot explain — fine until it misbehaves, and then you have no purchase on it.

Either way the same properties have to hold, and I would review for them explicitly: layered so
the per-commit delta is small; base image pinned by digest, not tag, because a tag is mutable
and moves your JDK build underneath a CDS archive; non-root with a numeric UID so
`runAsNonRoot` can verify it; no secrets in any layer, remembering that layers are additive so
deleting a file in a later step does not remove it; and a startup time that is measured and
tracked per release rather than assumed.

The thing I would not accept from either approach is 'it works, do not ask'. The image is
production infrastructure, and if nobody on the team can explain why it has the layers it has,
that is a bus-factor problem, not a tooling preference."

**WHAT SEPARATES THEM**

Recommending the simpler option confidently while naming the conditions under which it stops
being right; listing the properties that must hold regardless of tool; the digest-versus-tag
point and its specific interaction with CDS; the additive-layers secret hazard; and framing
comprehension as a requirement rather than a nicety.

**FOLLOW-UP:** *"A security scan flags a CVE in your base image. What is your process?"* —
Rebuild on a patched base and redeploy, which is fast if the base is a pinned digest you can
bump deliberately. Then triage properly, per Topic 34: is the vulnerable code on our classpath,
is the affected path reachable from our entry points, is the input attacker-controlled. And
note the second-order effect specific to this topic — a base image change invalidates the CDS
archive, so the rebuild must regenerate it, which is another argument for generating it in the
build rather than by hand.

---

## Mental model checkpoint

1. **A Docker layer is content-addressed.** Explain, from that fact alone, why a fat jar makes
   every deploy transfer the dependencies you did not change — and why fixing it is a
   *packaging* change rather than a code change.

2. **Name the four phases of Spring Boot startup in order**, and for each, name the instrument
   that measures it and one lever that changes it.

3. **`Started in X seconds (process running for Y)`.** What is `Y − X`, and what does it tell
   you about whether Spring tuning will help?

4. **AppCDS gives no improvement and no error.** List three specific things that could have
   invalidated the archive, and the two commands that would tell you which.

5. **Why does `-Xshare:on` belong in CI and `-Xshare:auto` in production?** State the asymmetry
   and the reason for it.

6. **Your pods start successfully but p99 spikes on every deploy.** What is happening, why does
   readiness not prevent it, and what are the two mitigations?

7. **`spring.main.lazy-initialization=true` cuts startup meaningfully.** What exactly are you
   trading away, and in what kind of service would you accept that trade?

---

## Quick reference card

### Layering

```bash
# Boot 3.3+ / 4.x
java -Djarmode=tools -jar app.jar list-layers
java -Djarmode=tools -jar app.jar extract --layers --launcher --destination extracted/

# Boot 2.3 - 3.2
java -Djarmode=layertools -jar app.jar list
java -Djarmode=layertools -jar app.jar extract
```

Copy order in the Dockerfile — least volatile first:
`dependencies` → `spring-boot-loader` → `snapshot-dependencies` → `application`.

### AppCDS

```bash
# Train
java -XX:ArchiveClassesAtExit=app.jsa -Dspring.context.exit=onRefresh \
     org.springframework.boot.loader.launch.JarLauncher

# Use (production: auto = degrade quietly)
java -XX:SharedArchiveFile=app.jsa org.springframework.boot.loader.launch.JarLauncher

# Verify (CI: on = fail loudly)
java -Xshare:on -XX:SharedArchiveFile=app.jsa -Xlog:cds=info -version
java -Xlog:class+load=info ... | grep -c 'shared file'
```

### Startup decomposition

```bash
# 1. The split
grep 'Started' app.log            # Started in X (process running for Y); JVM init ≈ Y - X

# 2. Class loading
-Xlog:class+load=info:file=/tmp/cl.log     # then: wc -l ; grep -c 'shared file'

# 3. Inside Spring
app.setApplicationStartup(new BufferingApplicationStartup(4096));
curl -s -X POST localhost:8080/actuator/startup | jq '.timeline.events | sort_by(-.duration)[0:20]'
```

### Docker diagnostics

```bash
docker history <image> --format '{{.Size}}\t{{.CreatedBy}}'
docker inspect <image> --format '{{json .RootFS.Layers}}' | jq -r '.[]'
docker build . 2>&1 | grep -E 'CACHED|COPY|RUN'
docker images <repo> --format '{{.Tag}}\t{{.Size}}'
kubectl rollout status deployment/orderflow --timeout=300s
kubectl describe pod <pod> | sed -n '/Last State/,/Ready/p'
```

### Reading the evidence

| Symptom | Cause |
|---|---|
| Every build pushes the full image | Fat jar in one layer, or a broken `.dockerignore` |
| Dependency step never cached | `COPY . .` before dependency resolution; `.git`/`target` in context |
| CDS flag present, no improvement | Archive rejected — check with `-Xshare:on` |
| `Y − X` large, `X` small | Time is before Spring; tune the JVM, not the context |
| One bean dominating `/actuator/startup` | Blocking I/O in a constructor — defer it |
| CrashLoopBackOff, logs stop mid-startup | Probe budget shorter than startup, or OOMKilled |
| p99 spikes on every deploy | JIT warm-up (Topic 74); warm before readiness or ramp traffic |

### Gotchas checklist

- [ ] `.dockerignore` excludes `.git`, `target/`, local config, logs.
- [ ] Base images pinned by digest, not tag.
- [ ] `COPY` order matches volatility order.
- [ ] CDS archive generated **in the build**, from the runtime layers.
- [ ] `-Xshare:on` asserted in CI; `-Xshare:auto` in production.
- [ ] Non-root with a numeric UID.
- [ ] Secrets never in any layer — layers are additive; deletion does not remove.
- [ ] Startup probe budget derived from measured worst case under production CPU limits.
- [ ] Startup time exported as a metric, tagged by version.
- [ ] Global lazy initialisation understood as moving failures, not removing them.

---

## When would I use this at work?

**1. The first time you watch a rolling deploy take longer than you expected.**
The instinct is to blame Kubernetes settings. The measurement is: how long does a node take to
pull the image, and how long from pod start to ready? Layering attacks the first, startup
decomposition attacks the second, and they are independent — so measure both before changing
either. This is usually a half-day of work with a visible, defensible result.

**2. When someone proposes native-image.**
The useful contribution is not "no". It is the decomposition table: here is where the twelve
seconds actually goes, here is the cheap lever for the largest slice, here is what it gets us,
and here is what native-image would additionally get us and what it would cost — the reflection
failures at runtime, the build times, the lost JVM tooling, the throughput trade. That turns an
enthusiasm into a decision, which is what Topic 126 is about.

**3. When you are asked why the cloud bill has a large registry line.**
Image storage and egress scale with how much of your image changes per build times how many
builds you do. Layering can change that by a large factor, and it is one of very few
infrastructure optimisations that is purely upside — no latency cost, no correctness risk,
no ongoing maintenance beyond keeping the `COPY` order right.

---

## Connected topics

**Backwards:**

- **20 — JPMS and `jlink`.** The other way to shrink a Java runtime image: build a custom JRE
  with only the modules you use. Complementary to layering, not a substitute.
- **34 — reproducible builds and SBOM.** A non-reproducible jar defeats layer caching, because
  the dependency layer's bytes change even when the dependencies do not.
- **42 — auto-configuration.** Every auto-configuration evaluated is context-refresh time.
  Excluding unused ones is a startup lever, and the condition-evaluation report tells you which
  are active.
- **65 — the load baseline.** The throughput number you must not regress when you reach for
  `TieredStopAtLevel=1` or lazy initialisation.
- **67 — class loading.** The mechanism AppCDS optimises. Understanding parse/verify/link is
  what makes the archive comprehensible rather than magic.
- **74 — JIT warm-up.** Why "started" and "fast" are different moments, and why a cold JVM
  behind a load balancer produces a p99 spike on every deploy.
- **77 — JMH.** The measurement discipline: medians over repeated runs, not a single
  stopwatch reading.
- **82 — JVM tuning in containers.** CPU quota changes startup time substantially, because
  startup is the most CPU-hungry phase the JVM has. Measure under production limits.
- **83 — native image.** The expensive alternative. Know its cost model before proposing it.
- **121 — Actuator and probes.** The startup probe's budget is a function of the number this
  topic measures; the two must be changed together.

**Forwards:**

- **123 — configuration, secrets and graceful shutdown.** Secrets must not be in image layers;
  the container's entry point and signal handling determine whether shutdown is graceful at all.
- **124 — the production-readiness gate.** Startup time, rollout duration, image provenance and
  the CDS validity gate are all review rows.
- **127 — migration planning.** A JDK or Boot upgrade changes the CDS archive, the `jarmode`
  command, and possibly the launcher class — all of which live in this Dockerfile.
- **129 — capacity and cost.** Registry storage, egress, and the CPU cost of repeated startup
  across a fleet are line items derived from these measurements.
- **133 — postmortems.** "The deploy took twenty minutes and rollback took twenty more" is a
  contributing factor with a known, cheap fix.
