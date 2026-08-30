# 65 — **GATE** — `orderflow` Running Under Load With Recorded Baselines

## Phase: 7 — GATE
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (running on JDK 25)
## Project spine: `orderflow` itself — the whole service, containerised, against a 100k-product / 1M-order / 5M-line dataset, under an open-model load generator, with every number written down and committed. This document does not add a feature. It produces the **artefact that every remaining phase of this curriculum profiles.**

---

## THE GATE RULE — read this first

> **If you cannot re-run the baseline and land within ±10% on every recorded
> percentile, Phase 8 is not startable.**

That is not a motivational slogan. It is a statement about what the next seventy
topics are able to teach you.

Phase 8 asks you to change a GC flag and see whether p99 improved. Phase 9 asks you
to move to virtual threads and see whether throughput improved. Topic 101 asks you to
re-run *this exact test* on virtual threads and compare. Topic 72 asks you to re-run
it under ZGC and compare. Topic 109 asks you to change the HikariCP pool size and
compare.

Every one of those is a **difference between two measurements**. If your measurement
has more than 10% of run-to-run noise in it, a real 8% improvement is invisible and a
random 12% fluctuation looks like a discovery. You will spend Phase 8 drawing
conclusions from noise, and you will not know you are doing it.

So the deliverable of this topic is not "a load test". It is **a measurement
instrument you trust**, plus a written-down number it produced.

Everything from Topic 66 onward profiles this artefact. There is no substitute step.

---

## Mastery line (from the master plan)

> You have a reproducible load test, a realistic dataset, and numbers you trust — and
> you can explain why a single-number average latency is useless and what p99
> actually measures.

## Mid → Senior (from the master plan)

> "We load tested it, it did 2000 rps" → "at what concurrency, what latency
> percentile, against what dataset size, with what cache state, and was the load
> generator itself saturated? Coordinated omission means most naive load tests
> under-report tail latency by an order of magnitude."

---

## ELI5 anchor

You want to know how fast a car is.

**The wrong experiment.** You drive it round an empty car park at 20 mph for ten
seconds, note the fuel gauge barely moved, and write "excellent economy" in a
notebook. Every part of that is a measurement. None of it tells you anything about
the motorway.

**The right experiment** has four parts, and skipping any one of them invalidates the
rest:

1. **A real road.** Not a car park. Your database needs five million rows in it,
   because a table that fits in memory is a car park.
2. **A stopwatch that does not lie.** If you only start timing when the car is
   already moving, you have hidden the slowest part. That is what a warmed-up JVM
   versus a cold one does to you, and it is why the first minute of every run gets
   thrown away.
3. **Traffic that keeps arriving whether or not you are ready.** Real cars do not
   politely wait for you to finish before joining the motorway. If your test only
   sends the next request after the previous one comes back, then when the car stalls,
   *no cars arrive during the stall* — and the stall never appears in your numbers.
   This is the single most important idea in this document.
4. **A stopwatch operator who is not also driving.** If the person timing the run is
   out of breath, their timings are wrong. If your load generator's own CPU is
   maxed out, its timings are wrong, and it will blame the car.

Then you write the number down, put it in version control, and drive the same route
again next week to check the stopwatch still agrees with itself. That last step is the
gate.

---

## The bridge from what you know

### What you already have — I am not going to re-teach it

You come from strong system design. You already know, and I am taking as given:

- What p50, p95, p99 and p999 mean, and why an average latency is a number about
  nobody.
- Why tail latency compounds across a fan-out: if a page makes 10 backend calls, a
  p99 per call is roughly a p90 per page.
- What an SLO is, what an error budget is, and why "fast" is not a requirement.
- Open versus closed system models, at least conceptually.
- Little's Law: concurrency = arrival rate × latency.
- That you should test with realistic data.

**None of that is repeated below.** If you want the percentile refresher, you do not
need it.

### What is genuinely different because this is the JVM

Here are the three things that will make your first Java load test wrong, all of
which have no equivalent in your Node experience.

---

#### Hazard 1 — JIT warm-up: the first thousand requests are a different program

Node's V8 also has a JIT, and you have lived with it. But two things make the JVM's
version far more consequential for a load test:

- The JVM starts by **interpreting** bytecode. Not compiling it — interpreting it,
  one instruction at a time.
- It then compiles hot methods in **tiers**: C1 (fast to compile, moderate code) and
  then C2 (slow to compile, aggressively optimised). A method has to be executed
  thousands of times before C2 touches it, and C2 then makes *speculative*
  optimisations — inlining a branch it has only ever seen go one way — which get
  **deoptimised and recompiled** the first time reality disagrees.

The practical consequence: **your service is a materially slower program for the first
part of the run, and then it becomes a faster program.** Not slightly. The difference
between interpreted and C2-compiled code is large, and I am deliberately not putting a
multiplier here because you are going to measure it yourself in the Hands-on section.

There is more on top of the JIT: class loading is lazy (Topic 67), so the first
request through a code path pays to load and link every class it touches; the
connection pool is empty until first use; Hibernate builds its query plan cache on
first execution of each query; the OS page cache and Postgres's `shared_buffers` are
cold.

**So: the first N seconds of every run must be excluded from the measurement window,
and N is something you measure, not guess.** The protocol is in the Hands-on section.
The mechanism is Topic 74.

---

#### Hazard 2 — GC pauses land in the tail, and only in the tail

Your Node service has a garbage collector too. But you have almost certainly never
tuned it, never read its log, and never had to reason about a stop-the-world pause,
because V8's heaps are typically small and single-threaded.

`orderflow` will run with a heap in the gigabytes, a collector you chose, and pauses
that stop every application thread.

The arithmetic that matters, and it is simple: **a pause of `P` milliseconds at an
arrival rate of `R` requests per second delays roughly `R × P / 1000` requests.**

Work an arbitrary example through it — these are made-up inputs to exercise the
formula, not numbers from any service. *If* R were 500 and P were 200, the pause
would delay about 100 requests, and each of those would have a latency floor of up to
200 ms regardless of how fast your code is. In a run of 300,000 requests, 100 requests
is the top 0.03% — so an effect of that shape shows up in **p999 and p9999**, becomes
visible in p99 if pauses are frequent, and is completely invisible in p50.

Substitute your own R from the baseline and your own P from `gc.log`. Both are numbers
you will measure in this topic; neither is a number I can give you.

This is why:

- You **must** capture a GC log during the baseline run, in the same window, so that
  when a tail spike appears you can check whether it was a pause before you go
  looking for a slow query.
- p999 is a mandatory recorded metric in this topic and not an optional extra. It is
  where GC lives.

Topics 71 and 72 are where you learn to read that log and change the collector.
Today you only have to **capture it**.

---

#### Hazard 3 — coordinated omission, which will make you off by an order of magnitude

This is the one that separates a senior answer from a mid-level one, so read it
carefully.

**The closed-loop (fixed-VU) model.** You configure 50 virtual users. Each one does:
send a request, wait for the response, send the next request. This is what almost
every naive load test does, and it is what `k6 run --vus 50 --duration 60s` does.

Now suppose the server stalls for one second — a GC pause, a lock, a pool wait.

What happens in the closed-loop model? All 50 VUs are blocked waiting. **They send no
requests during the stall.** When the stall ends, they each get their response, record
a latency of about 1000 ms, and carry on.

You recorded **50 slow requests.**

What actually happened in production? Requests kept arriving during that second,
because real users do not coordinate with your server. At 500 rps, **500 requests
arrived during the stall**, and each one experienced not just the stall but the queue
of everything ahead of it — so their latencies ranged from ~1000 ms down through
whatever the drain time was. Some of them saw 1500 ms.

You recorded 50 slow samples. Reality produced 500-plus, and worse ones. **The
requests that would have been slowest are precisely the ones your test never sent.**
Your test omitted them because it coordinated with the server it was measuring. Hence
the name.

Effect on your numbers: the tail is under-reported, often by an order of magnitude,
and the worse the stall, the more the test hides it. **The measurement gets more wrong
exactly when the system is behaving worst.** That is the property that makes it
dangerous rather than merely inaccurate.

**The open-model fix.** An open model starts new requests **on a schedule**,
independent of whether previous ones have completed. In k6 this is the
`constant-arrival-rate` executor: "start 500 iterations per second, whatever is going
on". If the server stalls, iterations pile up, and every piled-up iteration records
the full latency including its queueing time. That is what a user experiences.

**This is why an open-model arrival rate is mandatory for the gate and a fixed-VU
loop is not acceptable.**

**The catch you must also know**, because it re-introduces the same bug through the
back door: k6's arrival-rate executors need a pool of VUs to run those scheduled
iterations. You declare `preAllocatedVUs` and `maxVUs`. **If the pool runs out, k6
drops the iteration** — it does not run it, and it does not record it. Dropped
iterations are coordinated omission wearing a different hat.

k6 counts them in a metric called `dropped_iterations`. **Any run with
`dropped_iterations > 0` is invalid and must be discarded.** There is a threshold in
the script below that enforces exactly this.

---

### Summary of the bridge

| You already know | The JVM-specific thing you do not |
|---|---|
| p50/p95/p99/p999 | p999 is specifically where GC pauses live, so it is mandatory here |
| Averages hide the tail | JIT warm-up hides a *different program* in the first part of the run |
| Realistic data matters | A 100-row table sits entirely in `shared_buffers` and exercises no index |
| Open vs closed models | The specific k6 mechanism, and the `dropped_iterations` back door |
| Load generators can be a bottleneck | The exact five checks, and that you run them **first**, not last |
| Aggregate metrics across instances | Percentiles cannot be averaged; you must aggregate histogram buckets |

---

## What is this?

This topic produces **six artefacts**, all of which are mandatory before Phase 8, plus
one gate check.

| # | Deliverable | Committed to |
|---|---|---|
| 1 | A `docker compose` stack: `orderflow` + Postgres + Redis + Kafka, with explicit CPU and memory limits | repo root, `load/docker-compose.yml` |
| 2 | A seed dataset of ≥100,000 products, ≥1,000,000 orders, ≥5,000,000 order lines, with realistic cardinality and a few hot products | `load/seed/*.sql` |
| 3 | A **k6** load script using an **open-model arrival rate** (`constant-arrival-rate`), not a fixed-VU loop | `load/k6/orderflow.js` |
| 4 | A scenario mix of **70% catalogue read / 20% order read / 10% order placement** | inside the k6 script |
| 5 | **Recorded baselines**: p50, p95, p99, p999, throughput and error rate **per endpoint** | `/docs/java/baselines/<date>-run-<n>/` |
| 6 | **Recorded JVM configuration**: heap size, collector, container CPU and memory limits | the same baseline directory |
| ✅ | **Gate check**: re-run and land within ±10% on every recorded percentile | `baselines/.../gate-check.md` |

Gatling is an acceptable substitute for k6 if you prefer Scala/Java DSLs. This document
uses k6 because the script is short, the arrival-rate executor is a first-class
concept, and the generator's own overhead is low — which matters, because the
generator is the first thing you have to rule out.

---

## Why does it matter?

### 1. Because every remaining phase is a comparison, and comparisons need a baseline

Look at what the curriculum is going to ask you to do:

| Topic | The question it asks |
|---|---|
| 71 — G1 in depth | Drive up the allocation rate. Which pause cause appears? |
| 72 — ZGC / Shenandoah | Re-run the baseline under `-XX:+UseZGC`. Did p99 improve? Did throughput drop? |
| 74 — JIT | How long is warm-up, and what did C2 do to your hot path? |
| 77 — JMH | Micro-benchmark the method the profiler pointed at. |
| 78 — Profiling | Where does CPU actually go under baseline load? |
| 79 — Heap dumps | What is retained at steady state? |
| 101 — Virtual threads | Re-run the baseline on virtual threads. Compare. |
| 109 — HikariCP | Change the pool size. Compare. |
| 118 — Metrics | Export what you measured here as RED metrics. |

Every row is "compare two runs". A baseline with 30% noise makes all of them
unanswerable. That is the entire justification for the ±10% gate rule.

### 2. Because the failures you were promised in Phase 5 only appear under load

Three of them, specifically, and you should expect to trip over at least one during
this topic:

- **Topic 55's transaction-holding-a-connection failure.** A `@Transactional` method
  that makes an HTTP call, or simply takes longer than you thought, pins a HikariCP
  connection for its whole duration. At low concurrency this is invisible. At the
  arrival rate you are about to apply, the pool drains and **every endpoint fails,
  including ones that never touch the database**. You cannot see this without load.
- **Topic 52's oversell.** Two concurrent placements against one unit of stock is a
  unit test. Two thousand concurrent placements against a hot product is a different
  experiment, and it is the one that finds the retry storm and the lock convoy.
- **Topic 50's N+1.** You fixed it and asserted the statement count. Under load, the
  cost of an unfixed N+1 is not "slower" — it is pool exhaustion, because each request
  holds a connection for 250 round trips instead of 3.

### 3. Because "we load tested it, it did 2000 rps" is a claim with no content

Six questions turn that sentence into either an engineering statement or an admission:

1. At what **arrival rate** — and was it open-model or a fixed VU count?
2. Which **percentile**, and over what window?
3. Against what **dataset size** and what **skew**?
4. In what **cache state** — cold, warm, or undefined?
5. Was the **load generator** saturated?
6. Was the JVM **warm**?

By the end of this topic you will be able to answer all six about your own service,
which is a different professional position from being able to ask them.

### 4. Because a number you cannot reproduce is not a measurement

This is the part people skip. A one-off number is an anecdote. A number you can
produce again, on demand, within a tolerance, is an instrument. The difference is the
whole of experimental discipline, and the ±10% check is the cheapest possible way to
find out which one you have.

---

## Syntax breakdown

### `docker-compose.yml` — the parts that carry meaning

```yaml
name: orderflow-load          # project name; prefixes container and volume names

services:
  postgres:
    image: postgres:16-alpine # PINNED tag. Never :latest. Record the digest too.
    healthcheck: { ... }      # what "ready" means, so depends_on can wait for it
    deploy:
      resources:
        limits:
          cpus: "4.0"         # hard CPU limit
          memory: 4g          # hard memory limit
volumes:
  pgdata:                     # named volume: survives `down`, dies on `down -v`
```

| Key | What it does | Why it matters here |
|---|---|---|
| `name:` | Names the compose project. | Volumes become `orderflow-load_pgdata`. You need that name to snapshot the seeded data. |
| `image:` with a pinned tag | Fixes the version. | A silently-updated `latest` is a change in the system under test that your baseline does not record. |
| `healthcheck:` | A command Docker runs to decide readiness. | Same idea as a Testcontainers wait strategy (Topic 61). Without it, `depends_on` means "started", which is not "ready". |
| `depends_on: {x: {condition: service_healthy}}` | Waits for the healthcheck. | Stops the app crash-looping against a Postgres that has not finished initialising. |
| `deploy.resources.limits.cpus` | Caps CPU. | **This is the number `Runtime.availableProcessors()` returns inside the container**, which sets GC thread counts, the ForkJoinPool common pool size, and Tomcat's defaults. An unlimited container gives the JVM your whole laptop and makes the baseline meaningless. |
| `deploy.resources.limits.memory` | Caps memory. | `-XX:MaxRAMPercentage` is a percentage **of this**. Without a limit, the JVM sizes its heap from the host's RAM. |
| `volumes:` named volume | Persistent data. | You seed once and reuse. `docker compose down` keeps it; `down -v` destroys it. Know which one you typed. |
| `JAVA_TOOL_OPTIONS` env var | JVM flags without changing the image. | The JVM prints `Picked up JAVA_TOOL_OPTIONS: ...` on startup, which is your proof the flags applied. |

> **On `deploy.resources` (honest note).** `deploy:` was originally a Swarm-only
> section, and in old Compose v1 it was ignored outside Swarm. Compose v2 does honour
> `cpus` and `memory` limits. **Do not trust that, or me — verify it**, with
> `docker stats` and by reading the cgroup limit from inside the container. Both
> commands are in the Hands-on section. If your Compose version ignores it, the
> top-level `cpus:` and `mem_limit:` keys are the fallback.

### k6 script structure

A k6 script has exactly four things in it, and understanding the shape makes the long
script in Example 2 readable.

```js
import http from 'k6/http';

// 1. options -- the test PLAN. Read once, before anything runs.
export const options = {
  scenarios: { ... },
  thresholds: { ... },
};

// 2. setup() -- runs ONCE, before all scenarios. Its return value is passed
//    as the argument to every VU function. Use it for auth tokens.
export function setup() { return { token: '...' }; }

// 3. one exported function per scenario -- the VU code. Runs once per ITERATION.
export function catalogueRead(data) { http.get(...); }

// 4. handleSummary(data) -- runs ONCE at the end, with all aggregated metrics.
//    Return an object mapping file paths (and 'stdout') to content.
export function handleSummary(data) { return { 'stdout': '...', 'summary.md': '...' }; }
```

### `options.scenarios` — and why the executor choice is the whole ballgame

```js
scenarios: {
  catalogue_read: {
    executor: 'constant-arrival-rate',   // <-- OPEN MODEL. This is mandatory.
    exec: 'catalogueRead',               // which exported function this scenario runs
    rate: 350,                           // start 350 iterations...
    timeUnit: '1s',                      // ...per second
    duration: '600s',
    startTime: '120s',                   // begin AFTER the warm-up scenario ends
    preAllocatedVUs: 200,                // VUs created up front
    maxVUs: 2000,                        // ceiling; beyond this, iterations are DROPPED
    gracefulStop: '30s',                 // let in-flight iterations finish
    tags: { endpoint: 'catalogue' },     // tags every metric this scenario emits
  },
}
```

| Executor | Model | Verdict for this gate |
|---|---|---|
| `constant-vus` | Closed loop. N VUs, each looping. | **Forbidden.** Coordinated omission. |
| `ramping-vus` | Closed loop with a changing N. | **Forbidden**, same reason. Useful only for finding a breaking point, never for a baseline. |
| `shared-iterations` / `per-vu-iterations` | Closed loop, fixed work. | Not applicable. |
| `constant-arrival-rate` | **Open.** Fixed iterations/sec. | **This is the one.** |
| `ramping-arrival-rate` | **Open**, with a changing rate. | Correct for a capacity search (find the knee). Use `constant` for the recorded baseline. |

Three parameters that decide whether the run is valid:

- **`rate` + `timeUnit`** — the arrival rate. This is your independent variable.
- **`preAllocatedVUs`** — how many VUs exist before the scenario starts. Creating VUs
  mid-run costs generator CPU, so allocate enough that k6 does not have to.
- **`maxVUs`** — the ceiling. By Little's Law, the VUs you need is roughly
  `rate × latency_seconds`. Worked with arbitrary illustrative inputs: a rate of 500
  and a latency of 0.1 s needs about 50 VUs — but if latency degrades to 2 s under
  stress the same rate needs about 1000. **Size `maxVUs` from the degraded latency, not
  the healthy one**, and be generous: a dropped iteration invalidates the run, and idle
  VUs in k6 are cheap.

### `options.thresholds` — pass/fail, and the trick that creates sub-metrics

```js
thresholds: {
  // A threshold on a TAGGED metric creates a "sub-metric" that k6 tracks
  // separately -- and that handleSummary can then read. A trivially-true
  // condition is a legitimate way to force the sub-metric into existence.
  'http_req_duration{endpoint:catalogue}': ['p(99)>=0'],   // always true; creates the sub-metric
  'http_reqs{endpoint:catalogue}':        ['count>=0'],
  'http_req_failed{endpoint:catalogue}':  ['rate>=0'],

  // Structural guards. These are POLICY choices, not measurements.
  'dropped_iterations': ['count==0'],   // any drop invalidates the run
  'http_req_failed':    ['rate<0.01'],  // an error budget you chose
}
```

**Note on numbers in thresholds.** `count==0` and `rate<0.01` are *policy*: you are
declaring what you will accept. They are not claims about performance. **Latency
thresholds are different** — a `p(99)<200` is only meaningful once you know what your
service does, so in the script below they are read from environment variables and are
absent on your first run. You fill them in from **your own** measured baseline. I have
not put a latency number anywhere in this document, because I have never run your
service.

### `summaryTrendStats` — you will not get p999 without this

```js
export const options = {
  summaryTrendStats: ['avg', 'min', 'med', 'p(50)', 'p(90)', 'p(95)', 'p(99)', 'p(99.9)', 'max', 'count'],
};
```

k6's default summary does **not** include p(99.9). The gate requires p999. Set this or
you will finish the run and not have the number.

---

## Example 1 — minimal

Before the full stack, prove the four moving parts work: the app answers, k6 runs, the
open model behaves like an open model, and you can tell the difference.

Assume `orderflow` is running on `localhost:8080` against any database at all.

`load/k6/smoke.js`:

```js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    // OPEN model: 20 iterations start every second regardless of completion.
    smoke: {
      executor: 'constant-arrival-rate',
      rate: 20,
      timeUnit: '1s',
      duration: '30s',
      preAllocatedVUs: 20,
      maxVUs: 200,
    },
  },
  thresholds: {
    'dropped_iterations': ['count==0'],
    'http_req_failed': ['rate<0.01'],
  },
  summaryTrendStats: ['avg', 'med', 'p(95)', 'p(99)', 'p(99.9)', 'max', 'count'],
};

export default function () {
  const res = http.get('http://localhost:8080/actuator/health', {
    tags: { name: 'GET /actuator/health' },
  });
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

```bash
k6 version
k6 run load/k6/smoke.js
```

**What to look for**, in three places:

| Where | What to look for | What it means |
|---|---|---|
| `http_reqs` | The `rate` should be very close to 20/s, and `count` close to 600 | The open model is delivering the arrival rate you asked for. |
| `dropped_iterations` | Absent from the summary, or zero | The VU pool was sufficient. If it is non-zero, raise `maxVUs`. |
| `http_req_duration` | The `p(99.9)` column exists | `summaryTrendStats` took effect. If p(99.9) is missing, you did not set it. |

### The five-minute experiment that makes coordinated omission real

Do this once. It is the best money-per-minute in the whole topic.

**Step 1.** Add a deliberately stalling endpoint to `orderflow`, behind a profile so it
can never reach production:

```java
package com.orderflow.support;

import org.springframework.context.annotation.Profile;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.atomic.AtomicLong;

/**
 * Deliberate stall generator. Every 200th request sleeps for one second.
 * Exists ONLY to demonstrate coordinated omission. Never enabled outside
 * the 'load-demo' profile.
 */
@RestController
@Profile("load-demo")
public class StallController {

    private final AtomicLong counter = new AtomicLong();

    @GetMapping("/demo/stall")
    public String maybeStall() throws InterruptedException {
        if (counter.incrementAndGet() % 200 == 0) {
            Thread.sleep(1000);
        }
        return "ok";
    }
}
```

**Step 2.** Hit it with a **closed** loop, then with an **open** model, at the same
throughput:

```bash
# CLOSED: 20 VUs looping. This is what a naive load test does.
k6 run --vus 20 --duration 60s \
       --summary-trend-stats 'p(50),p(95),p(99),p(99.9),max,count' \
       -e URL=http://localhost:8080/demo/stall load/k6/closed.js

# OPEN: 200 iterations per second, arriving on a schedule.
k6 run --summary-trend-stats 'p(50),p(95),p(99),p(99.9),max,count' \
       -e URL=http://localhost:8080/demo/stall load/k6/open.js
```

**Step 3.** Fill in your own two columns:

| Metric | Closed loop (`constant-vus`) | Open model (`constant-arrival-rate`) |
|---|---|---|
| `http_reqs` count | | |
| p50 | | |
| p95 | | |
| p99 | | |
| **p99.9** | | |
| max | | |
| `dropped_iterations` | | |

**How to read your table.** The stall is identical in both runs — the same server,
the same `Thread.sleep`. If the two tail columns are meaningfully different, the
difference is entirely an artefact of **how you measured**, not of the system. That
is coordinated omission, in your own numbers, on your own machine.

Write the two columns down. When someone tells you their service does 2000 rps, this
table is why your first question is "open or closed?".

---

## Example 2 — production scenario (on the project spine)

Now the real gate artefact, in five parts: the stack, the schema readiness, the seed,
the k6 script, and the run protocol.

### Directory layout

```
load/
  docker-compose.yml
  Dockerfile                 # if you are not already building an image
  prometheus.yml
  seed/
    00-pre-load.sql
    01-products.sql
    02-inventory.sql
    03-wallets.sql
    04-orders.sql
    05-order-lines.sql
    99-post-load.sql
  k6/
    orderflow.js
    bucket-percentiles.py
  logs/                      # bind-mounted; gc.log and JFR land here
docs/java/baselines/
  2026-08-29-run-01/
    README.md
    environment.md
    baseline.md
    summary.json
    results.json.gz
    gc.log
```

---

### Part 1 — `load/docker-compose.yml`

```yaml
name: orderflow-load

services:

  # ---------------------------------------------------------------------------
  # Postgres. This is the system under test as much as the app is.
  # ---------------------------------------------------------------------------
  postgres:
    image: postgres:16-alpine
    container_name: orderflow-postgres
    environment:
      POSTGRES_DB: orderflow
      POSTGRES_USER: orderflow
      POSTGRES_PASSWORD: orderflow
      # Speeds up the initial seed only. See 99-post-load.sql for the reset.
      POSTGRES_INITDB_ARGS: "--data-checksums"
    command:
      - postgres
      - -c
      - max_connections=200
      - -c
      - shared_buffers=1GB
      - -c
      - effective_cache_size=3GB
      - -c
      - work_mem=16MB
      - -c
      - maintenance_work_mem=512MB
      - -c
      - random_page_cost=1.1
      - -c
      - track_io_timing=on
      - -c
      - shared_preload_libraries=pg_stat_statements
      - -c
      - pg_stat_statements.max=10000
      - -c
      - pg_stat_statements.track=all
      - -c
      - log_min_duration_statement=500
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./seed:/seed:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orderflow -d orderflow"]
      interval: 5s
      timeout: 3s
      retries: 40
    deploy:
      resources:
        limits:
          cpus: "4.0"
          memory: 4g

  # ---------------------------------------------------------------------------
  # Redis. Product catalogue cache (Topic 51's L2 / Spring Cache).
  # ---------------------------------------------------------------------------
  redis:
    image: redis:7-alpine
    container_name: orderflow-redis
    command:
      - redis-server
      - --maxmemory
      - 512mb
      - --maxmemory-policy
      - allkeys-lru
      - --save
      - ""                      # no RDB snapshots; this is a cache, not a database
      - --appendonly
      - "no"
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 768m

  # ---------------------------------------------------------------------------
  # Kafka, single-node KRaft. Stubbed until Topic 113 -- it is here so the app's
  # auto-configuration has a real broker to connect to, and so the resource
  # footprint of the stack is the one you will keep.
  # ---------------------------------------------------------------------------
  kafka:
    image: apache/kafka:3.8.0
    container_name: orderflow-kafka
    environment:
      KAFKA_NODE_ID: "1"
      KAFKA_PROCESS_ROLES: "broker,controller"
      KAFKA_LISTENERS: "PLAINTEXT://:9092,CONTROLLER://:9093"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka:9092"
      KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT"
      KAFKA_INTER_BROKER_LISTENER_NAME: "PLAINTEXT"
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka:9093"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: "1"
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: "1"
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: "1"
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: "0"
      KAFKA_NUM_PARTITIONS: "3"
    healthcheck:
      test: ["CMD-SHELL", "/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092 >/dev/null 2>&1"]
      interval: 10s
      timeout: 10s
      retries: 30
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1g

  # ---------------------------------------------------------------------------
  # The service under test.
  # ---------------------------------------------------------------------------
  app:
    build:
      context: ..
      dockerfile: load/Dockerfile
    image: orderflow:load
    container_name: orderflow-app
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: "docker,load"
      SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres:5432/orderflow"
      SPRING_DATASOURCE_USERNAME: "orderflow"
      SPRING_DATASOURCE_PASSWORD: "orderflow"
      SPRING_DATA_REDIS_HOST: "redis"
      SPRING_KAFKA_BOOTSTRAP_SERVERS: "kafka:9092"
      # ---- Every flag here is part of the recorded baseline (Deliverable 6). ----
      JAVA_TOOL_OPTIONS: >-
        -XX:MaxRAMPercentage=70.0
        -XX:InitialRAMPercentage=70.0
        -XX:+UseG1GC
        -XX:MaxGCPauseMillis=200
        -XX:+AlwaysPreTouch
        -XX:+ExitOnOutOfMemoryError
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:HeapDumpPath=/logs
        -Xlog:gc*,gc+heap=info,safepoint:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=32M
        -XX:StartFlightRecording=name=baseline,settings=profile,filename=/logs/baseline.jfr,dumponexit=true
        -Djava.security.egd=file:/dev/./urandom
    ports:
      - "8080:8080"
    volumes:
      - ./logs:/logs
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O - http://localhost:8080/actuator/health/readiness | grep -q UP"]
      interval: 5s
      timeout: 5s
      retries: 60
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2g

  # ---------------------------------------------------------------------------
  # Prometheus. Optional but strongly recommended: it is how you aggregate
  # histogram buckets correctly instead of averaging percentiles (Trap 5).
  # ---------------------------------------------------------------------------
  prometheus:
    image: prom/prometheus:v2.54.1
    container_name: orderflow-prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=7d
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - promdata:/prometheus
    ports:
      - "9090:9090"
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1g

volumes:
  pgdata:
  promdata:
```

`load/prometheus.yml`:

```yaml
global:
  scrape_interval: 5s
  evaluation_interval: 15s

scrape_configs:
  - job_name: orderflow
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["app:8080"]
```

**Three things in that compose file that are load-bearing, and one that is not:**

1. **`cpus: "2.0"` on `app`.** Inside the container, `Runtime.availableProcessors()`
   will report 2. That number decides G1's parallel GC thread count, the ForkJoinPool
   common pool size (Topic 25), and several Spring/Tomcat defaults. If you leave the
   app unlimited it inherits your whole laptop and your baseline describes your laptop
   rather than a deployment.
2. **`MaxRAMPercentage=70.0` with `memory: 2g`.** The heap is sized from the *container
   limit*, not the host. `AlwaysPreTouch` makes the JVM touch every heap page at
   startup so page faults do not appear as latency during the run — it makes startup
   slower and the measurement cleaner, which is the trade you want here.
3. **The `-Xlog:gc*` file, bind-mounted to `./logs`.** This is captured during the
   *same window* as the load run. Without it, a tail spike is unattributable.

Not load-bearing: `StartFlightRecording`. It is here because Topic 78 will want it and
it costs little, but a JFR recording is itself a small perturbation. If you enable it
for the baseline, **you must enable it for every comparison run too**, or you have
changed the system between measurements.

> **Kafka image caveat (R8).** The environment-variable set the `apache/kafka` image
> accepts, and whether it auto-formats storage, has changed across releases. **Verify
> before assuming:**
> ```bash
> docker compose -f load/docker-compose.yml up -d kafka
> docker compose -f load/docker-compose.yml logs kafka | tail -40
> docker compose -f load/docker-compose.yml exec kafka \
>   /opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
> ```
> If the broker does not come up, read the image's own documentation for your tag
> rather than trusting the block above. Kafka is stubbed until Topic 113; if it will
> not start today, you may proceed with it commented out, provided you record that
> fact in `environment.md` — because it changes the memory available to everything
> else.

---

### Part 2 — schema first, seed second

**The schema must come from Flyway, not from Hibernate.** Same rule as Topic 61,
same reason: if the load test runs against a Hibernate-invented schema, its indexes
are not your indexes and every plan you measure is fiction.

```bash
cd load

# 1. Bring up ONLY the datastores.
docker compose up -d postgres redis
docker compose ps

# 2. Let the app run its migrations, then stop it. (Or use the Flyway CLI.)
docker compose up -d app
docker compose logs -f app | grep -i flyway
docker compose stop app

# 3. Confirm the schema is the real one.
docker compose exec postgres psql -U orderflow -d orderflow \
  -c 'select installed_rank, version, description, success from flyway_schema_history order by installed_rank'
```

| What you see | What it means |
|---|---|
| Your migration list, all `success = t` | Correct. Proceed to seeding. |
| `relation "flyway_schema_history" does not exist` | Flyway did not run. Check `spring.flyway.enabled` and `spring.jpa.hibernate.ddl-auto` in the `load` profile. |
| Fewer rows than you have migration files | `spring.flyway.locations` is wrong, or a file is being filtered. |

Then, in `application-load.yml` (Topic 43's `load` profile), the settings that make
the run measurable:

```yaml
spring:
  config:
    activate:
      on-profile: load
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        generate_statistics: false      # ON is a measurable overhead. Off for the baseline.
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
  datasource:
    hikari:
      maximum-pool-size: 20             # RECORD this. Topic 109 will change it.
      minimum-idle: 20
      connection-timeout: 3000
      pool-name: orderflow-pool

logging:
  level:
    org.hibernate.SQL: WARN             # SQL logging under load is itself a bottleneck
    org.springframework.web: WARN

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
  metrics:
    distribution:
      # THIS is what makes correct cross-instance percentile aggregation possible.
      # It publishes cumulative histogram BUCKETS, which are aggregatable.
      percentiles-histogram:
        http.server.requests: true
      minimum-expected-value:
        http.server.requests: 1ms
      maximum-expected-value:
        http.server.requests: 10s
```

> **Property-name check (R8).** `management.metrics.distribution.percentiles-histogram.*`
> is the Boot 3.x path and I expect it to be unchanged on Boot 4.1, but **do not take
> my word for it.** The check is one command and it is definitive:
> ```bash
> curl -s localhost:8080/actuator/prometheus | grep -c 'http_server_requests_seconds_bucket'
> ```
> A count greater than zero means buckets are being published and the property took
> effect. Zero means it did not, whatever the property is called on your version.

---

### Part 3 — the seed, and why `generate_series` rather than 5 million inserts

**The wrong way**, which people genuinely do: a Java `CommandLineRunner` that
`persist`s five million `OrderLine` entities. Even with batching, that is five million
rows crossing a JDBC boundary, through Hibernate's action queue, with a growing
persistence context. It takes hours, it exercises nothing you care about, and it makes
re-seeding so expensive that you stop doing it — which quietly kills reproducibility.

**The right way** is to make Postgres generate the rows itself. `generate_series` is a
set-returning function; `INSERT ... SELECT ... FROM generate_series(1, 5000000)` is a
single statement that never leaves the database. The rows are constructed, written and
committed inside one process, with no network, no ORM, and no client.

`load/seed/00-pre-load.sql`:

```sql
-- Run as: docker compose exec -T postgres psql -U orderflow -d orderflow -v ON_ERROR_STOP=1 -f /seed/00-pre-load.sql
\timing on
\echo '== pre-load: relaxing durability and dropping secondary indexes =='

-- Durability off for the load only. 99-post-load.sql puts it back.
-- NEVER do this to a database whose contents you care about.
SET synchronous_commit = off;
SET maintenance_work_mem = '512MB';

-- Unlogged tables skip the write-ahead log entirely during bulk load.
ALTER TABLE product    SET UNLOGGED;
ALTER TABLE inventory  SET UNLOGGED;
ALTER TABLE wallet     SET UNLOGGED;
ALTER TABLE orders     SET UNLOGGED;
ALTER TABLE order_line SET UNLOGGED;

-- Secondary indexes are rebuilt in 99-post-load.sql. Maintaining them during a
-- 5-million-row insert costs far more than building them once at the end.
DROP INDEX IF EXISTS idx_order_line_order_id;
DROP INDEX IF EXISTS idx_order_line_product_id;
DROP INDEX IF EXISTS idx_orders_customer_id;
DROP INDEX IF EXISTS idx_orders_placed_at;
DROP INDEX IF EXISTS idx_product_sku;
```

`load/seed/01-products.sql` — 100,000 products:

```sql
\timing on
\echo '== products: 100,000 =='

INSERT INTO product (id, sku, name, price_minor, active, created_at)
SELECT
    g,
    'SKU-' || lpad(g::text, 8, '0'),
    'Product ' || g || ' ' ||
        (ARRAY['Keyboard','Monitor','Cable','Dock','Mouse','Headset','Webcam','Stand'])[1 + (g % 8)],
    -- Prices spread over a realistic range, in MINOR units (Topic 01).
    (499 + (g * 37) % 49501)::bigint,
    (g % 50) <> 0,                      -- 2% of products inactive
    now() - ((g % 1095) || ' days')::interval
FROM generate_series(1, 100000) AS g;
```

`load/seed/02-inventory.sql`:

```sql
\timing on
\echo '== inventory: one row per product, hot products stocked deep =='

INSERT INTO inventory (id, product_id, quantity_available, quantity_reserved, version)
SELECT
    g,
    g,
    CASE
        -- The 2,000 "hot" products are the ones the load test will order from.
        -- They need enough stock that a long run never exhausts them, or your
        -- error rate becomes a story about stock rather than about performance.
        WHEN g <= 2000 THEN 100000000
        ELSE 500 + (g * 13) % 4500
    END,
    0,
    0
FROM generate_series(1, 100000) AS g;
```

`load/seed/03-wallets.sql`:

```sql
\timing on
\echo '== wallets: 50,000 customers, funded deep =='

INSERT INTO wallet (id, customer_id, balance_minor, version)
SELECT g, g, 100000000000, 0
FROM generate_series(1, 50000) AS g;
```

`load/seed/04-orders.sql` — 1,000,000 orders:

```sql
\timing on
\echo '== orders: 1,000,000 =='

INSERT INTO orders (id, idempotency_key, customer_id, status, placed_at, total_minor)
SELECT
    g,
    'seed-' || g,
    1 + (g % 50000),                              -- 50,000 customers
    -- Deterministic status mix: ~5% cancelled, ~14% pending, the rest paid.
    -- Derived from g rather than random() so the dataset is byte-identical
    -- on every rebuild. Reproducibility beats realism here.
    CASE WHEN g % 20 = 0 THEN 'CANCELLED'
         WHEN g % 7  = 0 THEN 'PENDING'
         ELSE 'PAID' END,
    now() - ((g % 525600) || ' minutes')::interval,   -- spread over ~1 year
    0                                                  -- recomputed in 99-post-load
FROM generate_series(1, 1000000) AS g;
```

`load/seed/05-order-lines.sql` — 5,395,000 order lines with a long tail and real skew:

```sql
\timing on
\echo '== order lines: ~5.4M, skewed product distribution, long tail on lines-per-order =='

INSERT INTO order_line (id, order_id, product_id, quantity, unit_price_minor)
SELECT
    row_number() OVER ()                          AS id,
    src.order_id,
    -- SKEW. 60% of all lines reference one of 2,000 "hot" products;
    -- the other 40% are spread across all 100,000. The multipliers are
    -- coprime with the modulus so the spread is even inside each branch.
    CASE WHEN (src.order_id * 31 + src.line_no) % 100 < 60
         THEN 1 + ((src.order_id * 7919  + src.line_no * 13) % 2000)
         ELSE 1 + ((src.order_id * 104729 + src.line_no * 97) % 100000)
    END                                           AS product_id,
    1 + ((src.order_id + src.line_no) % 3)        AS quantity,
    499 + ((src.order_id * 17 + src.line_no) % 49501) AS unit_price_minor
FROM (
    SELECT
        o AS order_id,
        -- LONG TAIL on lines per order:
        --   0.1% of orders have 40 lines
        --   0.9% have 15
        --   9%   have 8
        --   90%  have 5
        -- Total: 1000*40 + 9000*15 + 90000*8 + 900000*5 = 5,395,000 lines.
        generate_series(1,
            CASE WHEN o % 1000 = 0 THEN 40
                 WHEN o % 100  = 0 THEN 15
                 WHEN o % 10   = 0 THEN 8
                 ELSE 5 END) AS line_no
    FROM generate_series(1, 1000000) AS o
) AS src;
```

`load/seed/99-post-load.sql` — **the part people skip, and the part that decides
whether your baseline means anything**:

```sql
\timing on
\echo '== post-load: indexes, constraints, sequences, statistics =='

-- 1. Rebuild the secondary indexes. Building once over a full table is far
--    cheaper than maintaining them through 5.4M inserts.
CREATE INDEX idx_order_line_order_id   ON order_line (order_id);
CREATE INDEX idx_order_line_product_id ON order_line (product_id);
CREATE INDEX idx_orders_customer_id    ON orders (customer_id);
CREATE INDEX idx_orders_placed_at      ON orders (placed_at DESC);
CREATE UNIQUE INDEX idx_product_sku    ON product (sku);

-- 2. Restore durability. The load test must measure a normally-configured
--    database, not one with the write-ahead log switched off.
ALTER TABLE product    SET LOGGED;
ALTER TABLE inventory  SET LOGGED;
ALTER TABLE wallet     SET LOGGED;
ALTER TABLE orders     SET LOGGED;
ALTER TABLE order_line SET LOGGED;

-- 3. Recompute order totals from the lines, so the data is internally consistent.
UPDATE orders o
   SET total_minor = agg.total
  FROM (SELECT order_id, sum(quantity * unit_price_minor) AS total
          FROM order_line GROUP BY order_id) agg
 WHERE o.id = agg.order_id;

-- 4. ADVANCE THE SEQUENCES. This is not optional.
--    Hibernate's SEQUENCE generator (allocationSize = 50, Topic 53) will hand out
--    ids starting from wherever the sequence currently sits. Seeded rows occupy
--    1..1,000,000. Without this, the very first POST /orders under load fails with
--    a duplicate-key violation on the primary key -- and you will spend an hour
--    believing you have found a concurrency bug.
SELECT setval('product_seq',    (SELECT max(id) FROM product)    + 1000);
SELECT setval('inventory_seq',  (SELECT max(id) FROM inventory)  + 1000);
SELECT setval('wallet_seq',     (SELECT max(id) FROM wallet)     + 1000);
SELECT setval('order_seq',      (SELECT max(id) FROM orders)     + 1000);
SELECT setval('order_line_seq', (SELECT max(id) FROM order_line) + 1000);

-- 5. STATISTICS. Also not optional, and the most commonly forgotten step.
--    Without ANALYZE the planner has no idea how many rows are in these tables
--    or how values are distributed, so it picks plans for a table it thinks is
--    empty. Your baseline would then measure the wrong query plans entirely.
VACUUM (ANALYZE) product;
VACUUM (ANALYZE) inventory;
VACUUM (ANALYZE) wallet;
VACUUM (ANALYZE) orders;
VACUUM (ANALYZE) order_line;

-- 6. Report what you actually built. These numbers go in the baseline record.
\echo '== row counts =='
SELECT 'product'    AS table_name, count(*) FROM product
UNION ALL SELECT 'inventory',  count(*) FROM inventory
UNION ALL SELECT 'wallet',     count(*) FROM wallet
UNION ALL SELECT 'orders',     count(*) FROM orders
UNION ALL SELECT 'order_line', count(*) FROM order_line;

\echo '== on-disk sizes =='
SELECT relname,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
  FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
 WHERE n.nspname = 'public' AND c.relkind = 'r'
 ORDER BY pg_total_relation_size(c.oid) DESC;

\echo '== skew check: the hot products should dominate =='
SELECT product_id, count(*) AS line_count
  FROM order_line
 GROUP BY product_id
 ORDER BY line_count DESC
 LIMIT 10;
```

Run it:

```bash
cd load
for f in 00-pre-load 01-products 02-inventory 03-wallets 04-orders 05-order-lines 99-post-load; do
  echo "=== $f ==="
  docker compose exec -T postgres psql -U orderflow -d orderflow \
    -v ON_ERROR_STOP=1 -f "/seed/$f.sql"
done
```

Record your own timings — `\timing on` prints each statement's duration:

| Seed step | Rows | Your wall-clock time |
|---|---|---|
| `01-products` | 100,000 | |
| `02-inventory` | 100,000 | |
| `03-wallets` | 50,000 | |
| `04-orders` | 1,000,000 | |
| `05-order-lines` | ~5,395,000 | |
| `99-post-load` (indexes + ANALYZE) | — | |
| **Total** | | |

> **If the total is long enough to discourage re-seeding, snapshot the volume.**
> A reproducible baseline needs a cheap reset:
> ```bash
> docker compose stop postgres
> docker run --rm -v orderflow-load_pgdata:/data -v "$PWD:/backup" alpine \
>   tar czf /backup/pgdata-seeded.tgz -C /data .
> # restore:
> docker compose down -v
> docker volume create orderflow-load_pgdata
> docker run --rm -v orderflow-load_pgdata:/data -v "$PWD:/backup" alpine \
>   tar xzf /backup/pgdata-seeded.tgz -C /data
> ```
> Restoring a tarball is a fixed, known cost. Re-running the seed is not, because
> it depends on your machine's state. For the ±10% gate check you want the fixed one.

---

### Part 4 — `load/k6/orderflow.js`

```js
// ---------------------------------------------------------------------------
// orderflow baseline load test.
//
// OPEN MODEL ONLY. Every scenario uses constant-arrival-rate, because a
// fixed-VU closed loop under-reports tail latency (coordinated omission).
//
// Mix: 70% catalogue read, 20% order read, 10% order placement.
//
// There are NO latency numbers in this file. Latency thresholds are read from
// environment variables, which you supply from YOUR OWN first measured run.
// ---------------------------------------------------------------------------

import http from 'k6/http';
import { check } from 'k6';
import { SharedArray } from 'k6/data';
import exec from 'k6/execution';

// ---- Configuration, all overridable from the command line with -e KEY=value ----
const BASE_URL   = __ENV.BASE_URL   || 'http://localhost:8080';
const RATE       = Number(__ENV.RATE       || 200);   // TOTAL requests/sec across the mix
const WARMUP_S   = Number(__ENV.WARMUP_S   || 180);   // discarded warm-up window, seconds
const DURATION_S = Number(__ENV.DURATION_S || 600);   // measured window, seconds
const PRE_VUS    = Number(__ENV.PRE_VUS    || 300);
const MAX_VUS    = Number(__ENV.MAX_VUS    || 3000);

// Dataset shape. These MUST match what the seed scripts built.
const HOT_PRODUCT_MAX = 2000;
const PRODUCT_MAX     = 100000;
const ORDER_MAX       = 1000000;
const CUSTOMER_MAX    = 50000;

// Mix: 70 / 20 / 10.
const RATE_CATALOGUE = Math.max(1, Math.round(RATE * 0.70));
const RATE_ORDER_GET = Math.max(1, Math.round(RATE * 0.20));
const RATE_ORDER_POST= Math.max(1, Math.round(RATE * 0.10));

// A stable list of hot product ids, parsed once and shared by all VUs.
// SharedArray keeps ONE copy in memory instead of one per VU -- this is a
// generator-memory optimisation, and generator memory is a real constraint
// when maxVUs is in the thousands.
const HOT_PRODUCTS = new SharedArray('hot products', () => {
  const ids = [];
  for (let i = 1; i <= HOT_PRODUCT_MAX; i++) ids.push(i);
  return ids;
});

// ---------------------------------------------------------------------------
// Test plan.
// ---------------------------------------------------------------------------
export const options = {

  // p(99.9) is NOT in k6's default summary. The gate requires it.
  summaryTrendStats: ['avg', 'min', 'med', 'p(50)', 'p(90)', 'p(95)', 'p(99)', 'p(99.9)', 'max', 'count'],

  // Reduces generator CPU and memory: k6 does not keep response bodies it
  // is not asked to inspect. Turn OFF if a check needs the body.
  discardResponseBodies: false,

  scenarios: {

    // ---- WARM-UP. Runs first. Its metrics are tagged phase:warmup and are
    // ---- EXCLUDED from every recorded number by tag.
    warmup: {
      executor: 'constant-arrival-rate',
      exec: 'warmupMix',
      rate: RATE,
      timeUnit: '1s',
      duration: `${WARMUP_S}s`,
      startTime: '0s',
      preAllocatedVUs: PRE_VUS,
      maxVUs: MAX_VUS,
      gracefulStop: '10s',
      tags: { phase: 'warmup', endpoint: 'warmup' },
    },

    // ---- MEASURED WINDOW. All three start after the warm-up has finished.
    catalogue_read: {
      executor: 'constant-arrival-rate',
      exec: 'catalogueRead',
      rate: RATE_CATALOGUE,
      timeUnit: '1s',
      duration: `${DURATION_S}s`,
      startTime: `${WARMUP_S}s`,
      preAllocatedVUs: PRE_VUS,
      maxVUs: MAX_VUS,
      gracefulStop: '30s',
      tags: { phase: 'measure', endpoint: 'catalogue' },
    },

    order_read: {
      executor: 'constant-arrival-rate',
      exec: 'orderRead',
      rate: RATE_ORDER_GET,
      timeUnit: '1s',
      duration: `${DURATION_S}s`,
      startTime: `${WARMUP_S}s`,
      preAllocatedVUs: Math.max(50, Math.round(PRE_VUS / 3)),
      maxVUs: MAX_VUS,
      gracefulStop: '30s',
      tags: { phase: 'measure', endpoint: 'order_read' },
    },

    order_place: {
      executor: 'constant-arrival-rate',
      exec: 'orderPlace',
      rate: RATE_ORDER_POST,
      timeUnit: '1s',
      duration: `${DURATION_S}s`,
      startTime: `${WARMUP_S}s`,
      preAllocatedVUs: Math.max(50, Math.round(PRE_VUS / 5)),
      maxVUs: MAX_VUS,
      gracefulStop: '30s',
      tags: { phase: 'measure', endpoint: 'order_place' },
    },
  },

  thresholds: {
    // -- Structural guards. These are POLICY, not performance claims. --

    // Any dropped iteration means k6 could not start work it had scheduled,
    // which re-introduces coordinated omission. The run is then INVALID.
    'dropped_iterations': ['count==0'],

    // An error budget you are choosing, not a measurement.
    'http_req_failed{phase:measure}': ['rate<0.01'],

    // -- Sub-metric creation. --
    // A threshold on a tagged metric makes k6 track that slice separately,
    // which is what lets handleSummary() report per-endpoint percentiles.
    // The conditions below are trivially true on purpose.
    'http_req_duration{endpoint:catalogue}':   ['p(99)>=0'],
    'http_req_duration{endpoint:order_read}':  ['p(99)>=0'],
    'http_req_duration{endpoint:order_place}': ['p(99)>=0'],
    'http_reqs{endpoint:catalogue}':           ['count>=0'],
    'http_reqs{endpoint:order_read}':          ['count>=0'],
    'http_reqs{endpoint:order_place}':         ['count>=0'],
    'http_req_failed{endpoint:catalogue}':     ['rate>=0'],
    'http_req_failed{endpoint:order_read}':    ['rate>=0'],
    'http_req_failed{endpoint:order_place}':   ['rate>=0'],

    // Generator-health sub-metrics. If time is going into blocked/connecting,
    // the bottleneck is the generator or the network, NOT the service.
    'http_req_blocked{phase:measure}':    ['p(99)>=0'],
    'http_req_connecting{phase:measure}': ['p(99)>=0'],
    'http_req_waiting{phase:measure}':    ['p(99)>=0'],

    // -- YOUR SLO thresholds, supplied from YOUR measured baseline. --
    // Absent on run 1. On run 2 onwards, pass them so the run fails loudly
    // if it regresses:
    //   k6 run -e P99_CATALOGUE=<your number> ...
    ...(__ENV.P99_CATALOGUE
        ? { 'http_req_duration{endpoint:catalogue}': ['p(99)>=0', `p(99)<${__ENV.P99_CATALOGUE}`] }
        : {}),
    ...(__ENV.P99_ORDER_READ
        ? { 'http_req_duration{endpoint:order_read}': ['p(99)>=0', `p(99)<${__ENV.P99_ORDER_READ}`] }
        : {}),
    ...(__ENV.P99_ORDER_PLACE
        ? { 'http_req_duration{endpoint:order_place}': ['p(99)>=0', `p(99)<${__ENV.P99_ORDER_PLACE}`] }
        : {}),
  },
};

// ---------------------------------------------------------------------------
// setup() runs ONCE, before any scenario. Its return value is handed to every
// VU function as the first argument.
// ---------------------------------------------------------------------------
export function setup() {
  const res = http.post(`${BASE_URL}/auth/token`,
    JSON.stringify({ username: 'loadtest', password: __ENV.LOAD_PASSWORD || 'loadtest' }),
    { headers: { 'Content-Type': 'application/json' } });

  if (res.status !== 200) {
    throw new Error(`setup: could not obtain a token. status=${res.status} body=${res.body}`);
  }
  return { token: res.json('accessToken') };
}

function authHeaders(data, extra) {
  return Object.assign({
    'Authorization': `Bearer ${data.token}`,
    'Content-Type': 'application/json',
  }, extra || {});
}

// 80% of catalogue traffic goes to the 2,000 hot products. This is what makes
// the Redis/L2 cache hit ratio realistic -- a uniform draw over 100,000
// products would give a cache hit rate near zero and measure a system nobody
// operates.
function skewedProductId() {
  if (Math.random() < 0.8) {
    return HOT_PRODUCTS[Math.floor(Math.random() * HOT_PRODUCTS.length)];
  }
  return 1 + Math.floor(Math.random() * PRODUCT_MAX);
}

// ---------------------------------------------------------------------------
// Scenario functions.
//
// Note the `name` tag on every request. Without it k6 tags metrics by the full
// URL, so 100,000 distinct product URLs become 100,000 distinct metric series
// and the generator spends its CPU on bookkeeping. This is a real and common
// way to accidentally saturate your own load generator.
// ---------------------------------------------------------------------------

export function warmupMix(data) {
  // Exercise every code path the measured window will use, so that by the time
  // measurement starts: classes are loaded, C2 has compiled the hot methods,
  // the Hikari pool is full, Hibernate's query plan cache is populated, and
  // Postgres has the working set in shared_buffers.
  const r = Math.random();
  if (r < 0.70)      catalogueRead(data);
  else if (r < 0.90) orderRead(data);
  else               orderPlace(data);
}

export function catalogueRead(data) {
  const id = skewedProductId();
  const res = http.get(`${BASE_URL}/products/${id}`, {
    headers: authHeaders(data),
    tags: { name: 'GET /products/:id' },
  });
  check(res, { 'catalogue 200': (r) => r.status === 200 });
}

export function orderRead(data) {
  // The Topic 50 endpoint: a page of orders with lines, products and payment
  // status. This is the query whose statement count you asserted on.
  const page = Math.floor(Math.random() * 200);
  const res = http.get(`${BASE_URL}/orders?page=${page}&size=50`, {
    headers: authHeaders(data),
    tags: { name: 'GET /orders?page&size' },
  });
  check(res, { 'order read 200': (r) => r.status === 200 });
}

export function orderPlace(data) {
  // Two to three lines, drawn from hot products, so inventory contention is
  // real (Topic 52) rather than theoretical.
  const lineCount = 2 + Math.floor(Math.random() * 2);
  const lines = [];
  for (let i = 0; i < lineCount; i++) {
    lines.push({
      productId: HOT_PRODUCTS[Math.floor(Math.random() * HOT_PRODUCTS.length)],
      quantity: 1,
    });
  }

  // A unique idempotency key per attempt. Uniqueness must not depend on the
  // clock alone -- at high arrival rates two VUs share a millisecond.
  const key = `k6-${exec.scenario.iterationInTest}-${exec.vu.idInTest}-${Date.now()}`;

  const res = http.post(`${BASE_URL}/orders`,
    JSON.stringify({
      customerId: 1 + Math.floor(Math.random() * CUSTOMER_MAX),
      lines: lines,
    }),
    {
      headers: authHeaders(data, { 'Idempotency-Key': key }),
      tags: { name: 'POST /orders' },
    });

  check(res, {
    'order placed 201': (r) => r.status === 201,
    // 409 means insufficient stock. It should be ~0 because hot products are
    // stocked deep. If it is not, your error rate is a story about the SEED,
    // not about performance.
    'not out of stock': (r) => r.status !== 409,
  });
}

// ---------------------------------------------------------------------------
// handleSummary() runs once at the end with every aggregated metric.
// It writes the numbers YOU measured straight into a markdown table that goes
// into the committed baseline.
//
// NOTE: defining handleSummary REPLACES k6's default stdout summary entirely.
// That is why a compact one is written below.
// ---------------------------------------------------------------------------
const ENDPOINTS = ['catalogue', 'order_read', 'order_place'];

function stat(metric, key) {
  if (!metric || !metric.values || metric.values[key] === undefined) return 'n/a';
  return metric.values[key].toFixed(2);
}

export function handleSummary(data) {
  const lines = [];
  lines.push('| endpoint | requests | achieved rps | p50 (ms) | p95 (ms) | p99 (ms) | p99.9 (ms) | max (ms) | error rate |');
  lines.push('|---|---|---|---|---|---|---|---|---|');

  for (const e of ENDPOINTS) {
    const d = data.metrics[`http_req_duration{endpoint:${e}}`];
    const r = data.metrics[`http_reqs{endpoint:${e}}`];
    const f = data.metrics[`http_req_failed{endpoint:${e}}`];
    lines.push(
      `| ${e} ` +
      `| ${r ? r.values.count : 'n/a'} ` +
      `| ${r ? r.values.rate.toFixed(2) : 'n/a'} ` +
      `| ${stat(d, 'p(50)')} | ${stat(d, 'p(95)')} | ${stat(d, 'p(99)')} ` +
      `| ${stat(d, 'p(99.9)')} | ${stat(d, 'max')} ` +
      `| ${f ? (f.values.rate * 100).toFixed(3) + '%' : 'n/a'} |`
    );
  }

  const dropped = data.metrics['dropped_iterations'];
  const droppedCount = dropped ? dropped.values.count : 0;

  const generatorHealth = [
    '',
    '### Generator health (check this BEFORE believing anything above)',
    '',
    '| signal | value | verdict |',
    '|---|---|---|',
    `| dropped_iterations | ${droppedCount} | ${droppedCount === 0 ? 'OK' : 'RUN IS INVALID -- raise maxVUs'} |`,
    `| http_req_blocked p99 | ${stat(data.metrics['http_req_blocked{phase:measure}'], 'p(99)')} ms | should be near zero |`,
    `| http_req_connecting p99 | ${stat(data.metrics['http_req_connecting{phase:measure}'], 'p(99)')} ms | should be near zero |`,
    `| http_req_waiting p99 | ${stat(data.metrics['http_req_waiting{phase:measure}'], 'p(99)')} ms | this is the SERVER's time |`,
    '',
  ];

  const md = ['## k6 run summary', '', ...lines, ...generatorHealth].join('\n');

  return {
    'stdout': '\n' + md + '\n',
    'summary.md': md + '\n',
    'summary.json': JSON.stringify(data, null, 2),
  };
}
```

Run it:

```bash
mkdir -p out && cd out

k6 run \
  --out json=results.json \
  -e BASE_URL=http://localhost:8080 \
  -e RATE=200 \
  -e WARMUP_S=180 \
  -e DURATION_S=600 \
  -e PRE_VUS=300 \
  -e MAX_VUS=3000 \
  ../load/k6/orderflow.js
```

---

### Part 5 — the run protocol

The protocol is the deliverable as much as the script is. Follow it identically every
time, or the ±10% check is measuring your procedure rather than your service.

```bash
# ---- 0. Clean slate, seeded data restored from the snapshot. ----
cd load
docker compose down -v
docker volume create orderflow-load_pgdata
docker run --rm -v orderflow-load_pgdata:/data -v "$PWD:/backup" alpine \
  tar xzf /backup/pgdata-seeded.tgz -C /data

# ---- 1. Start everything and wait for real readiness. ----
rm -f logs/gc.log* logs/baseline.jfr
docker compose up -d --wait
docker compose ps

# ---- 2. Record the environment BEFORE the run. ----
docker compose exec app jcmd 1 VM.flags          > ../out/vm-flags.txt
docker compose exec app jcmd 1 VM.command_line   > ../out/vm-command-line.txt
docker compose exec app jcmd 1 VM.system_properties | grep -i version > ../out/jvm-version.txt
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}' > ../out/stats-idle.txt
docker compose images                            > ../out/images.txt

# ---- 3. Confirm the dataset is the one you think it is. ----
docker compose exec -T postgres psql -U orderflow -d orderflow -c "
  select 'product' t, count(*) from product
  union all select 'orders', count(*) from orders
  union all select 'order_line', count(*) from order_line;"

# ---- 4. Reset Postgres statement statistics so the run's queries are isolated.
docker compose exec -T postgres psql -U orderflow -d orderflow \
  -c 'select pg_stat_statements_reset();' \
  -c 'select pg_stat_reset();'

# ---- 5. Run. k6 runs OUTSIDE the compose stack. ----
k6 run --out json=out/results.json \
  -e RATE=200 -e WARMUP_S=180 -e DURATION_S=600 \
  load/k6/orderflow.js | tee out/k6-stdout.txt

# ---- 6. Capture the server's own view, immediately after. ----
curl -s localhost:8080/actuator/metrics/http.server.requests > out/actuator-http.json
curl -s localhost:8080/actuator/prometheus                   > out/prometheus-scrape.txt
docker compose exec app jcmd 1 GC.heap_info                  > out/heap-info.txt
docker compose exec -T postgres psql -U orderflow -d orderflow -c "
  select calls, round(mean_exec_time::numeric,2) mean_ms,
         round(max_exec_time::numeric,2) max_ms, left(query, 90) q
    from pg_stat_statements order by total_exec_time desc limit 20;" > out/top-queries.txt
cp load/logs/gc.log out/gc.log
```

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the load generator is the bottleneck (check this FIRST, always)

**Wrong:** run k6 on the same laptop as the compose stack, at a high rate, and report
the latency it prints.

**Exact symptom — and this is the shape that fools people:** latency climbs steeply
with arrival rate, and it *looks* exactly like a saturated service. But:

- `docker stats` shows the `app` container's CPU well below its limit.
- `http_req_blocked` and `http_req_connecting` percentiles are large — meaning time is
  going into *getting a connection out the door*, not into waiting for a response.
- `http_req_waiting` (the server's actual think time) is small while
  `http_req_duration` is large. `duration = blocked + connecting + tls + sending +
  waiting + receiving`, so when `duration` grows and `waiting` does not, the extra
  time is on your side of the wire.
- `dropped_iterations` is non-zero.
- The k6 process itself sits at or near 100% of one or more cores.

**Root cause:** one of four, and they need different fixes:

1. **k6 CPU-saturated.** JSON serialisation, response-body handling, per-URL metric
   tagging, and TLS all cost generator CPU.
2. **VU pool exhausted.** `maxVUs` too low. k6 schedules an iteration, has no VU, and
   drops it. This is coordinated omission through the back door.
3. **Ephemeral port or file-descriptor exhaustion** on the generator host. Sockets in
   `TIME_WAIT` accumulate; new connections queue.
4. **The generator competes with the service for CPU**, because they are on the same
   machine.

**Fix — and the diagnosis order matters:**

```bash
# 1. The control experiment. Run the SAME rate against a trivial endpoint.
#    /actuator/health does almost no work, so any tail here is NOT the service.
k6 run --summary-trend-stats 'p(50),p(95),p(99),p(99.9),max' \
       -e RATE=200 load/k6/smoke.js

# 2. Watch the generator's own CPU while the real run is going.
#    macOS:
top -l 2 -pid "$(pgrep -x k6)" | tail -20
#    Linux:
pidstat -p "$(pgrep -x k6)" 1 10

# 3. Watch the service's CPU at the same time.
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'

# 4. Sockets and descriptors on the generator host.
ulimit -n
netstat -an | grep -c TIME_WAIT

# 5. The halving test. Run at half the rate from two generator processes.
#    If per-request latency improves, the generator was the limit.
```

| What you see | What it means |
|---|---|
| Control run against `/actuator/health` shows a fat tail at the same rate | The generator or the network is the bottleneck. Nothing you measure about the service is valid yet. |
| Control run is clean; the real run is not | The service is genuinely the bottleneck. Proceed. |
| `dropped_iterations > 0` | Invalid run. Raise `maxVUs` and `preAllocatedVUs`, re-run. |
| k6 process at 100% CPU | Set `discardResponseBodies: true`, check that every request has a low-cardinality `name` tag, and move k6 off this machine. |
| `TIME_WAIT` count in the tens of thousands | Port exhaustion. Ensure connection reuse is on (k6's default), reduce the rate, or widen the ephemeral range. |
| App container CPU at its `cpus:` limit | Legitimate saturation. This is a finding, not a defect in the test. |

**The rule: you may not report a latency number until the control run is clean.**
Doing this check first, every time, is the single habit that separates people whose
load-test numbers are trusted from people whose are not.

---

### Trap 2 — measuring during JIT warm-up

**Wrong:**

```bash
k6 run --duration 60s -e RATE=200 load/k6/orderflow.js   # no warm-up window
```

**Exact symptom:** three observations that all point the same way:

- Bucket the run by 10-second windows (script below). The p99 in the first buckets is
  dramatically higher than the p99 in the last buckets, and it **decays** rather than
  fluctuating.
- `curl -s localhost:8080/actuator/metrics/jvm.compilation.time` sampled 30 seconds
  apart is still climbing steeply during the early part of the run and flattens later.
- Two runs of the same duration disagree by far more than 10%, and the shorter one is
  always worse.

**Root cause:** the JVM is executing a different program at the start of the run than
at the end. Interpreted bytecode, then C1, then C2, with deoptimisation and
recompilation along the way (Topic 74). On top of that: lazy class loading (Topic 67),
an empty connection pool, an empty Hibernate query-plan cache, and a cold Postgres
buffer cache.

**Fix:** an explicit warm-up window, whose length you **measure** rather than guess.

Step 1 — measure the crossover. Run once with no warm-up scenario, export JSON, and
bucket it. `load/k6/bucket-percentiles.py`:

```python
#!/usr/bin/env python3
"""Bucket k6 JSON output by time and print p50/p95/p99 per bucket.

Usage: python3 bucket-percentiles.py results.json [bucket_seconds] [endpoint_tag]

Use it to find where the tail stops decaying. That point is the end of warm-up.
"""
import collections
import json
import sys
from datetime import datetime

path = sys.argv[1]
bucket_s = int(sys.argv[2]) if len(sys.argv) > 2 else 10
want_tag = sys.argv[3] if len(sys.argv) > 3 else None

buckets = collections.defaultdict(list)
t0 = None

with open(path) as fh:
    for line in fh:
        try:
            o = json.loads(line)
        except ValueError:
            continue
        if o.get("type") != "Point" or o.get("metric") != "http_req_duration":
            continue
        d = o["data"]
        if want_tag and d.get("tags", {}).get("endpoint") != want_tag:
            continue
        ts = datetime.fromisoformat(d["time"].replace("Z", "+00:00")).timestamp()
        if t0 is None:
            t0 = ts
        buckets[int((ts - t0) // bucket_s)].append(d["value"])

print(f"{'t_start_s':>10} {'n':>8} {'p50_ms':>10} {'p95_ms':>10} {'p99_ms':>10} {'max_ms':>10}")
for k in sorted(buckets):
    vals = sorted(buckets[k])
    n = len(vals)
    if n == 0:
        continue
    def q(p):
        return vals[min(n - 1, int(p * n))]
    print(f"{k * bucket_s:>10} {n:>8} {q(0.50):>10.1f} {q(0.95):>10.1f} {q(0.99):>10.1f} {vals[-1]:>10.1f}")
```

```bash
python3 load/k6/bucket-percentiles.py out/results.json 10 catalogue
```

Fill in your own crossover table:

| Window (s from start) | n | p50 (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|---|
| 0–10 | | | | |
| 10–20 | | | | |
| 20–30 | | | | |
| 30–60 | | | | |
| 60–120 | | | | |
| 120–180 | | | | |
| 180–240 | | | | |
| 240–300 | | | | |

**How to read it:** find the first window after which p99 stops trending downward and
starts merely fluctuating. **Set `WARMUP_S` to at least double that**, and record the
choice in `environment.md`. Doubling is not superstition — it buys margin for the
deoptimisation-and-recompile cycles that arrive later than the first compilations.

Step 2 — corroborate with the JVM's own view:

```bash
# Sample twice, 30 s apart, during the run.
curl -s localhost:8080/actuator/metrics/jvm.compilation.time
```

| What you see | What it means |
|---|---|
| `jvm.compilation.time` total still rising noticeably between samples | Compilation is still happening. Not warm. |
| Two samples nearly identical | Compilation has largely settled. Warm. |
| A late jump after a long flat period | A deoptimisation and recompile. This is normal and is why you double the window. |

---

### Trap 3 — a warm cache makes run 2 look better, and you call it an improvement

**Wrong:** run the baseline. Change one JVM flag. Run again immediately, on the same
stack, without resetting anything. Observe that everything is faster. Conclude the
flag helped.

**Exact symptom:** run 2 is uniformly better than run 1 — p50, p95 and p99 all
improve together, by a similar proportion. A uniform improvement across every
percentile is the tell, because a real optimisation almost always helps some part of
the distribution more than another.

Confirm it in four places, none of which are your code:

```bash
# Postgres buffer cache hit ratio -- reset before each run and compare.
docker compose exec -T postgres psql -U orderflow -d orderflow -c "
  select blks_read, blks_hit,
         round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) as hit_pct
    from pg_stat_database where datname = 'orderflow';"

# Redis cache effectiveness.
docker compose exec redis redis-cli info stats | grep -E 'keyspace_hits|keyspace_misses'

# The OS page cache under Postgres (Linux).
docker compose exec postgres cat /proc/meminfo | grep -i cached

# JIT state.
curl -s localhost:8080/actuator/metrics/jvm.compilation.time
```

**Root cause:** at least five independent caches warm up across a run, and they all
persist into the next one: the JIT's compiled-code cache, Postgres `shared_buffers`,
the OS page cache underneath it, Redis, and Hibernate's query-plan cache. Run 2
inherits all five.

**Fix: make cache state an explicit, recorded part of the protocol.** There is no
"correct" choice; there is only a *stated and repeated* choice. Pick one:

| Protocol | How | When to use it |
|---|---|---|
| **Always-warm** (recommended for a steady-state baseline) | `docker compose restart app` only, then the k6 warm-up scenario runs before the measured window. Postgres and Redis stay warm. | This models a service that has been serving traffic for hours, which is the normal case. |
| **Always-cold** | `docker compose down && up`, restore the volume snapshot, and on Linux drop the page cache (`sync; echo 3 > /proc/sys/vm/drop_caches`). No warm-up beyond class loading. | Models a cold start or a failover. Worth having as a *second* baseline, not as the primary. |

Whichever you pick, record the hit ratios **in every baseline**, so that a future
comparison can be checked for this exact confound:

| Cache signal | Run 1 | Run 2 | Run 3 |
|---|---|---|---|
| Postgres `blks_hit` % | | | |
| Redis `keyspace_hits` / (hits+misses) | | | |
| `jvm.compilation.time` at end of run | | | |

---

### Trap 4 — a 100-row dataset that proves nothing

**Wrong:** point the load test at the Testcontainers fixture — a few hundred rows —
or at a `data.sql` with twenty products, because seeding five million rows felt like a
lot of effort.

**Exact symptom:** the numbers are wonderful and stay wonderful no matter what you
change. Specifically:

- `EXPLAIN (ANALYZE, BUFFERS)` on your order-list query shows `Seq Scan` and it is
  *fast*, because the whole table is 3 pages.
- `shared read` in the BUFFERS output is zero or near zero. Every page is already in
  `shared_buffers` because the entire database is smaller than the buffer pool.
- Changing an index makes no measurable difference.
- Adding a `LIMIT` makes no measurable difference.
- Then production, on the same code, is many times slower, and nobody can reproduce it.

**Root cause:** at small cardinality you are measuring a different system. Four things
are absent:

1. **No I/O.** The database fits in RAM, so you never measure a page read.
2. **Wrong plans.** The planner correctly chooses a sequential scan for a tiny table,
   and correctly chooses something else for a large one. You measured the plan you
   will not run.
3. **No index depth.** A B-tree over 100 rows is one or two levels; over 5 million it
   is more, and every lookup pays for the extra levels.
4. **No skew.** Uniform access over 20 products gives a 100% cache hit rate. Real
   catalogues are Zipf-shaped: a small number of products dominate, and the cache
   behaviour that follows is the thing you are trying to measure.

**Fix:** the seed in Part 3, then *prove* it did what you think:

```bash
docker compose exec -T postgres psql -U orderflow -d orderflow -c '\timing on' -c "
  select relname,
         pg_size_pretty(pg_total_relation_size(c.oid)) as total,
         (select reltuples::bigint from pg_class where oid = c.oid) as est_rows
    from pg_class c join pg_namespace n on n.oid = c.relnamespace
   where n.nspname='public' and c.relkind='r'
   order by pg_total_relation_size(c.oid) desc;"

docker compose exec -T postgres psql -U orderflow -d orderflow -c '\timing on' -c "
  explain (analyze, buffers, verbose)
  select o.id, o.placed_at, o.status
    from orders o
   order by o.placed_at desc
   limit 50;"
```

| What you see in `EXPLAIN` output | What it means |
|---|---|
| `Index Scan` / `Index Only Scan` using `idx_orders_placed_at` | The index exists and the planner is using it. Realistic. |
| `Seq Scan on orders` with a large row count | Either the index is missing, or `ANALYZE` was never run so the planner thinks the table is empty. Run `VACUUM (ANALYZE)`. |
| `Buffers: shared hit=<n>` with `read=0` on the very first execution | The table already fits in `shared_buffers`. Your dataset is too small, or your buffer pool is too big relative to it. |
| `Buffers: shared hit=<n> read=<n>` with a non-zero read | Real I/O is happening. This is a realistic measurement. |
| `rows=<n>` in the plan far from `actual rows=<n>` | Statistics are stale. `ANALYZE` again. |

Fill in the dataset proof table:

| Table | Target rows | Actual rows | Total size on disk |
|---|---|---|---|
| `product` | ≥100,000 | | |
| `inventory` | 100,000 | | |
| `wallet` | 50,000 | | |
| `orders` | ≥1,000,000 | | |
| `order_line` | ≥5,000,000 | | |

And the skew proof:

| Check | Your value |
|---|---|
| Lines referencing the top 2,000 products, as % of all lines | |
| Line count of the single hottest product | |
| Line count of the median product | |
| Max lines on one order | |

---

### Trap 5 — averaging percentiles across instances

**Wrong:** two `orderflow` instances behind a load balancer. Each exports a p99. Your
dashboard shows `avg(p99)`. You report that as the service p99.

**Exact symptom:** three tells, and any one of them should stop you:

- The dashboard's p99 is **lower than the p99 of the worst single instance**, which is
  arithmetically impossible for a true service-wide p99.
- One instance goes sick — long GC pauses, a slow disk — and the graph barely moves,
  because averaging dilutes it.
- k6's client-side p99 and your dashboard's p99 disagree by a large factor, in the
  direction of the dashboard being optimistic.

**Root cause:** a percentile is a **quantile of a distribution**, not a quantity. The
99th percentile of the union of two sets is not a function of the two individual 99th
percentiles. Averaging them is not an approximation; it is a category error, and the
error grows precisely when the instances differ — which is exactly the case you care
about.

The same error appears in one more place that catches people: **averaging a percentile
over time**. `avg_over_time(p99[1h])` is equally invalid.

**Fix:** aggregate **histogram buckets**, then compute the quantile from the merged
histogram.

Micrometer gives you two different things and only one of them is aggregatable:

| Setting | What is published | Aggregatable? |
|---|---|---|
| `management.metrics.distribution.percentiles` | Pre-computed quantiles, calculated **inside each instance** | **No.** Never sum, never average. |
| `management.metrics.distribution.percentiles-histogram` | Cumulative bucket counts (`..._bucket{le="..."}`) | **Yes.** Buckets are counts; counts add. |

So the correct query, across every instance:

```promql
histogram_quantile(
  0.99,
  sum by (le, uri, method, status) (
    rate(http_server_requests_seconds_bucket[5m])
  )
)
```

`sum by (le, ...)` merges the per-instance bucket counts into one histogram, and
`histogram_quantile` then interpolates within the merged buckets. That is a valid
service-wide p99.

**Proof that buckets are actually being published:**

```bash
curl -s localhost:8080/actuator/prometheus | grep 'http_server_requests_seconds_bucket' | head -20
curl -s localhost:8080/actuator/prometheus | grep -c 'http_server_requests_seconds_bucket'
```

| What you see | What it means |
|---|---|
| Many `..._bucket{...,le="..."}` lines | Histograms are on. Correct aggregation is possible. |
| Only `..._count` and `..._sum` | No histogram. You can compute a mean and nothing else. Turn on `percentiles-histogram`. |
| `..._seconds{quantile="0.99"}` lines | Client-side pre-computed quantiles. Useful for one instance; **must not be summed or averaged.** |
| A very large number of bucket lines | Too many buckets, or too many distinct `uri` values. Bound them with `minimum-expected-value` / `maximum-expected-value`, and make sure `uri` is the template (`/products/{id}`) and not the actual path. |

**A related honesty point that belongs in the baseline record.** The actuator endpoint

```bash
curl -s 'localhost:8080/actuator/metrics/http.server.requests?tag=uri:/products/{id}'
```

gives you `COUNT`, `TOTAL_TIME` and `MAX`. That is a mean and a maximum. **It does not
give you percentiles.** Use it to cross-check throughput and error counts against k6.
Do not use it as a percentile source.

And record both viewpoints, because they measure different things:

| Endpoint | k6 p99 (client-side) | Prometheus p99 (server-side) | Difference |
|---|---|---|---|
| catalogue | | | |
| order_read | | | |
| order_place | | | |

The client number includes network time and client-side queueing. The server number
starts when the request reaches the servlet. **The SLO belongs against the client
number**, because that is what a user experiences; the server number is what you
debug with. A large gap between them is itself a finding — usually queueing in front
of the application (accept backlog, Tomcat's queue) rather than inside it.

---

## Hands-on proof

Every command below is one you run. There is no captured output anywhere in this
document. Where a line's *shape* is shown it is labelled as an illustration and uses
`<n>` placeholders.

### Proof 1 — the stack is up and the limits are real

```bash
cd load
docker compose up -d --wait
docker compose ps --format 'table {{.Name}}\t{{.Service}}\t{{.Status}}\t{{.Ports}}'
```

| What you see | What it means |
|---|---|
| Every service `running (healthy)` | `--wait` honoured the healthchecks. Proceed. |
| `app` restarting in a loop | It cannot reach a dependency. `docker compose logs app` — usually Postgres was not ready, or the migrations failed. |
| A service `running` but not `(healthy)` | Its healthcheck is failing or absent. An absent healthcheck makes `depends_on: service_healthy` unusable. |

Now prove the CPU and memory limits are actually applied — do not assume:

```bash
docker stats --no-stream \
  --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}'

# The cgroup limits, read from inside the container. This is ground truth.
docker compose exec app sh -c 'cat /sys/fs/cgroup/memory.max 2>/dev/null || cat /sys/fs/cgroup/memory/memory.limit_in_bytes'
docker compose exec app sh -c 'cat /sys/fs/cgroup/cpu.max 2>/dev/null || cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us'

# What the JVM believes about the machine it is on.
docker compose exec app jcmd 1 VM.flags | tr ' ' '\n' | grep -E 'MaxHeapSize|InitialHeapSize|UseG1GC|UseZGC|ActiveProcessorCount|MaxRAMPercentage'
```

| What you see | What it means |
|---|---|
| `memory.max` equal to your `memory:` limit in bytes | The limit is applied. `MaxRAMPercentage` will be a fraction of this. |
| `memory.max` reads `max` | **No limit is applied.** Your Compose version ignored `deploy.resources`. Switch to top-level `mem_limit:` / `cpus:` and re-check. |
| `cpu.max` reading `<quota> <period>` where quota/period equals your `cpus:` | CPU is capped. |
| `MaxHeapSize` is roughly 70% of `memory.max` | `MaxRAMPercentage` took effect. Record the exact value. |
| `MaxHeapSize` far larger than the container limit | The JVM is sizing from the host. Container awareness is off or the limit is missing. **Fix before measuring anything.** |

Confirm the flags you set are the flags in use:

```bash
docker compose logs app | grep -i 'Picked up JAVA_TOOL_OPTIONS'
docker compose exec app jcmd 1 VM.command_line
```

The log line has this shape — *illustration of the format, not captured output*:

```
Picked up JAVA_TOOL_OPTIONS: -XX:MaxRAMPercentage=70.0 -XX:+UseG1GC ... -Xlog:gc*...
```

If that line is absent, your flags did not apply and every JVM number you are about to
record describes a default configuration you did not choose.

### Proof 2 — the dataset is real and the planner knows it

```bash
docker compose exec -T postgres psql -U orderflow -d orderflow -c '\timing on' -c "
  select 'product' t, count(*) from product
  union all select 'inventory', count(*) from inventory
  union all select 'orders', count(*) from orders
  union all select 'order_line', count(*) from order_line;"
```

`\timing on` makes psql print each statement's duration. That duration is itself
informative: a `count(*)` over 5 million rows that returns instantly means the table
is entirely cached, which is a fact you want recorded.

```bash
docker compose exec -T postgres psql -U orderflow -d orderflow -c '\timing on' -c "
  explain (analyze, buffers)
  select l.order_id, l.product_id, l.quantity, l.unit_price_minor
    from order_line l
   where l.order_id in (select id from orders order by placed_at desc limit 50);"
```

| What you see | What it means |
|---|---|
| An index scan on `idx_order_line_order_id` | Correct. This is the plan Topic 50's fix relies on. |
| A sequential scan on `order_line` | The index is missing or statistics are stale. `VACUUM (ANALYZE) order_line` and re-check. |
| `Buffers: ... read=<n>` non-zero on first run | Real disk reads. Your dataset exceeds the buffer pool, which is what you want. |
| `Planning Time` larger than `Execution Time` | You are measuring the planner, not the query. The dataset is too small, or you are looking at a trivial query. |

### Proof 3 — the load generator is not the bottleneck

Run this **before** the real load run, every time. It takes three minutes.

```bash
# Control: the same arrival rate against an endpoint that does almost nothing.
k6 run --summary-trend-stats 'p(50),p(95),p(99),p(99.9),max,count' \
       -e RATE=200 load/k6/smoke.js
```

| What you see | What it means |
|---|---|
| Tight percentiles, `dropped_iterations` zero, p99 close to p50 | The generator and the network can deliver this rate cleanly. Your subsequent numbers are about the service. |
| A fat tail on `/actuator/health` | The generator or the network is the limit. **Stop.** Fix that first — nothing you measure now is about `orderflow`. |
| `dropped_iterations` non-zero | Raise `maxVUs`/`preAllocatedVUs`. Re-run the control. |

During the real run, in a second terminal:

```bash
# macOS
top -l 3 -pid "$(pgrep -x k6)" | grep -E 'CPU|k6'
# Linux
pidstat -p "$(pgrep -x k6)" 1 15
```

Fill in:

| Generator saturation check | Your observation | Verdict |
|---|---|---|
| k6 process CPU (% of one core) during the run | | |
| Host cores available to k6 | | |
| `dropped_iterations` | | must be 0 |
| `http_req_blocked` p99 | | should be near 0 |
| `http_req_connecting` p99 | | should be near 0 |
| `ulimit -n` on the generator host | | |
| `TIME_WAIT` socket count at end of run | | |
| Control run (`/actuator/health`) p99 | | must be small |

### Proof 4 — find your warm-up length

```bash
# One run with NO warm-up scenario, so you can see the decay.
k6 run --out json=out/warmup-probe.json \
  -e WARMUP_S=0 -e DURATION_S=420 -e RATE=200 \
  load/k6/orderflow.js

python3 load/k6/bucket-percentiles.py out/warmup-probe.json 10 catalogue
```

The output has this shape — *illustration of the format, not captured output*:

```
 t_start_s        n     p50_ms     p95_ms     p99_ms     max_ms
         0      <n>       <n>        <n>        <n>        <n>
        10      <n>       <n>        <n>        <n>        <n>
```

| What you see | What it means |
|---|---|
| p99 decays over several buckets, then flattens | Normal. The flattening point is the end of warm-up. Set `WARMUP_S` to at least double it. |
| p99 flat from bucket zero | Either the service was already warm from a previous run, or your arrival rate is too low to trigger C2 quickly. Restart the app container and re-probe. |
| p99 rises over the run instead of falling | Not warm-up. Something is degrading: memory growth, a growing table, connection leak. Investigate before recording a baseline. |
| A single high bucket in the middle | Look at `gc.log` for a pause at that timestamp. |

### Proof 5 — the run itself, and the cross-check against the server

```bash
k6 run --out json=out/results.json \
  -e RATE=200 -e WARMUP_S=<your measured value> -e DURATION_S=600 \
  load/k6/orderflow.js | tee out/k6-stdout.txt

curl -s 'localhost:8080/actuator/metrics/http.server.requests' | python3 -m json.tool
curl -s 'localhost:8080/actuator/metrics/http.server.requests?tag=outcome:SERVER_ERROR' | python3 -m json.tool
```

| Cross-check | How | Why |
|---|---|---|
| Request count | k6 `http_reqs` count vs actuator `COUNT` | If they differ by more than a rounding amount, requests are being lost before the servlet — a proxy, the accept queue, or a k6 error you did not check. |
| Error count | k6 `http_req_failed` vs actuator `outcome:SERVER_ERROR` count | Confirms which side is classifying failures. |
| Mean latency | k6 `avg` vs actuator `TOTAL_TIME / COUNT` | The gap is client-side time: connection, network, queueing outside the app. |
| Percentiles | k6 p99 vs `histogram_quantile` from Prometheus | Never compare against the actuator endpoint — it has no percentiles. |

### Proof 6 — attribute the tail to GC, or rule it out

```bash
cp load/logs/gc.log out/gc.log
grep -c 'Pause Young' out/gc.log
grep -c 'Pause Full' out/gc.log
grep 'Pause' out/gc.log | tail -30
```

| What you see | What it means |
|---|---|
| Only `Pause Young (Normal)` entries, short and regular | Healthy. GC is unlikely to be your tail. |
| `Pause Young (Concurrent Start)` followed by concurrent-cycle lines | G1 is doing a concurrent marking cycle. Normal. Topic 71. |
| Any `Pause Full` | A full GC. This will be visible in p999. Record it; Topic 71 diagnoses it. |
| `To-space exhausted` | Evacuation failure. A real problem and a Topic 71 headline. |
| Pause timestamps that line up with your bad latency buckets | You have attributed the tail. Say so in the baseline notes. |

Correlate by timestamp against the bucket output from Proof 4. That correlation — "the
p99 spike at t=340s coincides with a pause in gc.log" — is exactly the kind of
statement Phase 8 is going to ask you for, and doing it once here makes the whole of
Phase 8 easier.

### Proof 7 — where the database time went

```bash
docker compose exec -T postgres psql -U orderflow -d orderflow -c "
  select calls,
         round(total_exec_time::numeric, 1)  as total_ms,
         round(mean_exec_time::numeric, 3)   as mean_ms,
         round(max_exec_time::numeric, 1)    as max_ms,
         rows,
         left(regexp_replace(query, '\s+', ' ', 'g'), 100) as q
    from pg_stat_statements
   order by total_exec_time desc
   limit 20;"
```

| What you see | What it means |
|---|---|
| A small number of statements dominating `total_ms` | Normal and useful. These are your optimisation targets. |
| A statement with a huge `calls` count and a tiny `mean_ms` | A per-row query — the N+1 shape from Topic 50, resurrected. Check whether the load path uses the projection you built. |
| `max_ms` far above `mean_ms` for one statement | That statement contributes to your tail. Could be lock waiting (Topic 52) or a plan flip. |
| `pg_stat_statements` does not exist | `shared_preload_libraries` did not take effect; the extension also needs `CREATE EXTENSION pg_stat_statements;`. |

---

## Practice exercises / Gate deliverables

**These are not practice.** Each item below is a mandatory artefact. Phase 8 is not
startable until all seven are complete and committed.

Create the directory now:

```bash
mkdir -p docs/java/baselines
```

---

### Deliverable 1 — the compose stack

**Produce:** `load/docker-compose.yml` with `app`, `postgres`, `redis` and `kafka`,
each with explicit CPU and memory limits and a working healthcheck.

**Acceptance criteria:**
- [ ] `docker compose up -d --wait` returns success with no manual intervention.
- [ ] `docker compose ps` shows every service `(healthy)`.
- [ ] The cgroup limits read from inside `app` match what you declared.
- [ ] Every image tag is pinned. No `:latest` anywhere.
- [ ] Image digests recorded (from `docker compose images` or `docker image inspect`).

**Fill in — stack inventory (blank template):**

| Service | Image tag | Image digest (sha256:...) | CPU limit | Memory limit | Healthcheck passes |
|---|---|---|---|---|---|
| app | | | | | |
| postgres | | | | | |
| redis | | | | | |
| kafka | | | | | |
| prometheus | | | | | |

| Host fact | Value |
|---|---|
| Machine (model / cloud instance type) | |
| Physical / logical cores | |
| Total RAM | |
| OS and version | |
| Docker version | |
| Docker daemon CPU / memory allocation (Desktop or Colima) | |
| Storage type (NVMe SSD / network volume / …) | |

---

### Deliverable 2 — the seed dataset

**Produce:** `load/seed/*.sql`, runnable end to end, plus a volume snapshot so the
dataset can be restored in a fixed, known time.

**Acceptance criteria:**
- [ ] ≥100,000 products.
- [ ] ≥1,000,000 orders.
- [ ] ≥5,000,000 order lines.
- [ ] A long tail on lines-per-order (not every order the same size).
- [ ] Skew: a small set of hot products carries a large share of the lines.
- [ ] Sequences advanced past the seeded ids (`setval`).
- [ ] `VACUUM (ANALYZE)` run on every seeded table.
- [ ] No individual-row insert loops anywhere. `generate_series` or `COPY` only.

**Fill in — dataset (blank template):**

| Table | Target | Actual rows | Total size on disk | Seed wall-clock |
|---|---|---|---|---|
| `product` | ≥100,000 | | | |
| `inventory` | 100,000 | | | |
| `wallet` | 50,000 | | | |
| `orders` | ≥1,000,000 | | | |
| `order_line` | ≥5,000,000 | | | |
| **Total seed time** | | | | |
| **Snapshot restore time** | | | | |

| Skew / shape check | Your value |
|---|---|
| % of lines referencing the top 2,000 products | |
| Lines on the hottest single product | |
| Lines on the median product | |
| Max lines on one order | |
| Median lines on one order | |
| `shared_buffers` setting | |
| Total database size vs `shared_buffers` | |

---

### Deliverable 3 — the load generator, with an open-model arrival rate

**Produce:** `load/k6/orderflow.js` using `constant-arrival-rate` for every scenario.

**Acceptance criteria:**
- [ ] No `constant-vus` or `ramping-vus` anywhere.
- [ ] `dropped_iterations` threshold set to `count==0`.
- [ ] `summaryTrendStats` includes `p(99.9)`.
- [ ] Every HTTP call has a low-cardinality `name` tag.
- [ ] `handleSummary` writes `summary.md` and `summary.json`.
- [ ] Generator saturation checked and recorded **before** the recorded run.

**Fill in — generator configuration and health (blank template):**

| Setting | Value |
|---|---|
| Generator host (same machine as the stack? separate?) | |
| k6 version (`k6 version`) | |
| Total arrival rate (`RATE`) | |
| `preAllocatedVUs` / `maxVUs` | |
| Warm-up duration (`WARMUP_S`) | |
| Measured duration (`DURATION_S`) | |

| Saturation signal | Value | Pass? |
|---|---|---|
| `dropped_iterations` | | must be 0 |
| k6 process peak CPU | | |
| `http_req_blocked` p99 | | |
| `http_req_connecting` p99 | | |
| Control run (`/actuator/health`) p99 at same rate | | |
| `ulimit -n` | | |
| `TIME_WAIT` count at end of run | | |
| **Verdict: was the generator the bottleneck?** | | |

---

### Deliverable 4 — the scenario mix

**Produce:** a 70 / 20 / 10 split of catalogue read, order read, order placement,
implemented as three independent arrival-rate scenarios.

**Acceptance criteria:**
- [ ] The three scenarios run concurrently, each with its own arrival rate.
- [ ] Each is tagged so its metrics are separable.
- [ ] The achieved mix matches the target within a small margin — verify, do not
      assume.
- [ ] Catalogue reads are **skewed** toward hot products, so cache behaviour is real.
- [ ] Order placements target hot products, so inventory contention (Topic 52) is real.
- [ ] Placement failures caused by stock exhaustion are ~zero. If not, the seed is
      wrong, not the service.

**Fill in — achieved mix (blank template):**

| Scenario | Target share | Target rps | Achieved requests | Achieved rps | Achieved share |
|---|---|---|---|---|---|
| catalogue read | 70% | | | | |
| order read | 20% | | | | |
| order placement | 10% | | | | |
| **Total** | 100% | | | | |

---

### Deliverable 5 — THE RECORDED BASELINE

**Produce:** `docs/java/baselines/<YYYY-MM-DD>-run-01/baseline.md`, containing the
table below **filled in with your own numbers**, committed to git.

**This is the artefact the rest of the curriculum consumes.**

**Fill in — baseline results (blank template — every cell is yours to measure):**

```markdown
# orderflow baseline — run 01

- Date/time (with timezone):
- Git commit of orderflow under test:
- Protocol: [ ] always-warm   [ ] always-cold
- Warm-up window discarded (seconds):
- Measured window (seconds):
- Total arrival rate (rps):

## Client-side (k6) — the numbers the SLO is written against

| Endpoint | Target rps | Achieved rps | Requests | p50 (ms) | p95 (ms) | p99 (ms) | p999 (ms) | max (ms) | Error rate |
|---|---|---|---|---|---|---|---|---|---|
| GET /products/:id        |  |  |  |  |  |  |  |  |  |
| GET /orders?page&size    |  |  |  |  |  |  |  |  |  |
| POST /orders             |  |  |  |  |  |  |  |  |  |
| **All endpoints**        |  |  |  |  |  |  |  |  |  |

## Server-side (Prometheus histogram_quantile) — the numbers you debug with

| Endpoint (uri template) | p50 (ms) | p95 (ms) | p99 (ms) | p999 (ms) |
|---|---|---|---|---|
| /products/{id}   |  |  |  |  |
| /orders          |  |  |  |  |
| /orders (POST)   |  |  |  |  |

## Client vs server gap

| Endpoint | k6 p99 | Prometheus p99 | Gap | Explanation |
|---|---|---|---|---|
| catalogue    |  |  |  |  |
| order_read   |  |  |  |  |
| order_place  |  |  |  |  |

## Resource utilisation at steady state

| Container | Peak CPU % (of limit) | Peak memory | Notes |
|---|---|---|---|
| app        |  |  |  |
| postgres   |  |  |  |
| redis      |  |  |  |
| kafka      |  |  |  |

## Connection pool and database

| Signal | Value |
|---|---|
| `hikaricp.connections.active` peak |  |
| `hikaricp.connections.pending` peak |  |
| `hikaricp.connections.timeout` count |  |
| Postgres `blks_hit` % over the run |  |
| Redis hit ratio over the run |  |
| Top statement by `total_exec_time` |  |
| That statement's `mean_ms` / `max_ms` |  |

## GC during the measured window

| Signal | Value |
|---|---|
| Young pauses (count) |  |
| Full GCs (count) |  |
| Longest single pause |  |
| Any `To-space exhausted`? |  |
| Do pause timestamps correlate with p999 spikes? |  |

## Errors

| Status / outcome | Count | Share | Cause |
|---|---|---|---|
| 4xx |  |  |  |
| 5xx |  |  |  |
| k6 request errors (non-HTTP) |  |  |  |
```

---

### Deliverable 6 — the recorded JVM and container configuration

**Produce:** `docs/java/baselines/<date>-run-01/environment.md`.

Without this, a later comparison is not a comparison. Topic 72 asks "did ZGC help?" —
that question is only answerable if you know exactly what you ran G1 with.

**Fill in — JVM and container configuration (blank template):**

```markdown
# Environment — run 01

## JVM

| Setting | Value | How obtained |
|---|---|---|
| `java -version` output |  | `docker compose exec app java -version` |
| JDK vendor / build |  | same |
| Full command line |  | `jcmd 1 VM.command_line` |
| `JAVA_TOOL_OPTIONS` as applied |  | app startup log |
| Collector in use |  | `jcmd 1 VM.flags \| grep -E 'UseG1GC\|UseZGC\|UseParallelGC'` |
| `MaxHeapSize` (bytes and MB) |  | `jcmd 1 VM.flags` |
| `InitialHeapSize` |  | same |
| `MaxRAMPercentage` |  | same |
| `MaxGCPauseMillis` |  | same |
| `ActiveProcessorCount` / `availableProcessors()` |  | `jcmd 1 VM.flags`; log it at startup |
| `AlwaysPreTouch` on? |  | same |
| JFR recording active during the run? |  | it perturbs; must be identical across compared runs |
| G1 region size |  | `jcmd 1 VM.flags \| grep G1HeapRegionSize` |

## Container limits

| Container | `cpus` declared | `cpu.max` observed | `memory` declared | `memory.max` observed |
|---|---|---|---|---|
| app       |  |  |  |  |
| postgres  |  |  |  |  |
| redis     |  |  |  |  |
| kafka     |  |  |  |  |

## Application configuration

| Setting | Value |
|---|---|
| Active profiles |  |
| `spring.datasource.hikari.maximum-pool-size` |  |
| `spring.datasource.hikari.connection-timeout` |  |
| Tomcat max threads |  |
| Virtual threads enabled? |  |
| `hibernate.jdbc.batch_size` |  |
| `hibernate.generate_statistics` |  |
| SQL logging level |  |
| Second-level cache enabled? |  |
| Redis cache TTLs |  |

## Postgres configuration

| Setting | Value |
|---|---|
| Version |  |
| `shared_buffers` |  |
| `effective_cache_size` |  |
| `work_mem` |  |
| `max_connections` |  |
| `random_page_cost` |  |
```

---

### Deliverable 7 — THE GATE CHECK (±10%)

**Produce:** `docs/java/baselines/<date>-run-01/gate-check.md`.

**Procedure — no shortcuts:**

1. Run the full protocol from Part 5. Record as run 1.
2. `docker compose down -v`, restore the volume snapshot, `up -d --wait`. Run again.
   Record as run 2.
3. Repeat once more. Record as run 3.
4. Compute, for every recorded percentile of every endpoint, the spread across the
   three runs relative to the median.

**Fill in — reproducibility (blank template):**

| Endpoint | Metric | Run 1 | Run 2 | Run 3 | Median | Max deviation from median | Within ±10%? |
|---|---|---|---|---|---|---|---|
| catalogue | p50 |  |  |  |  |  |  |
| catalogue | p95 |  |  |  |  |  |  |
| catalogue | p99 |  |  |  |  |  |  |
| catalogue | p999 |  |  |  |  |  |  |
| catalogue | throughput |  |  |  |  |  |  |
| order_read | p50 |  |  |  |  |  |  |
| order_read | p95 |  |  |  |  |  |  |
| order_read | p99 |  |  |  |  |  |  |
| order_read | p999 |  |  |  |  |  |  |
| order_read | throughput |  |  |  |  |  |  |
| order_place | p50 |  |  |  |  |  |  |
| order_place | p95 |  |  |  |  |  |  |
| order_place | p99 |  |  |  |  |  |  |
| order_place | p999 |  |  |  |  |  |  |
| order_place | throughput |  |  |  |  |  |  |

**If any row fails, the gate is not passed.** Work through this list, in order,
before changing anything about the service:

| Suspected cause | How to test it | Fix |
|---|---|---|
| Warm-up too short | Re-run the bucket script; is p99 still decaying at the start of the measured window? | Increase `WARMUP_S`. |
| Cache state differs between runs | Compare `blks_hit` %, Redis hit ratio and `jvm.compilation.time` across runs | Fix the protocol so every run starts from the same state. |
| Dataset differs between runs | Row counts and `pg_total_relation_size` per run — order placement adds rows | Restore from the snapshot before every run, not just the first. |
| Generator variance | Compare `dropped_iterations` and k6 CPU across runs | Move k6 off the machine; raise `maxVUs`. |
| Background load on the host | Anything else running? Browser, IDE, indexing, backup, another container | Quiesce the machine; re-run. |
| Thermal throttling (laptops) | Watch CPU frequency across a long run; is run 3 always the worst? | Lower the rate, shorten the run, or use a machine with stable clocks. |
| Genuine bimodality in the service | Is one run bad for a *reason* visible in `gc.log` or `pg_stat_statements`? | This is a real finding. Record it. It may be the first thing Phase 8 explains. |
| p999 is noisy but p50/p95/p99 are stable | Expected — p999 is a handful of samples | Lengthen the run so p999 has more samples behind it, then re-check. |

That last row deserves emphasis: **the fix for a noisy p999 is more samples, not a
looser tolerance.** A 600-second run at 200 rps has roughly 120,000 samples, so p999 is
built from about 120 of them. Doubling the run doubles that. If p999 still will not
settle, say so honestly in the gate-check document and record the observed spread —
an honest "p999 reproduces to ±18% and here is why" is a better artefact than a
number you have quietly stopped believing.

---

## Interview questions

### Q1 — "We load tested it. It did 2000 requests per second." (the staple)

**Mid-level answer:** "That sounds good. Was that the peak? What was the average
response time?"

**Senior answer:** "Six questions before I can interpret that number.

**One: open or closed model?** If it was a fixed number of VUs each looping, the tail
latency is under-reported — possibly by an order of magnitude — because during any
stall the test stops sending requests, so the requests that would have been slowest
were never sent. That is coordinated omission, and it makes the measurement most wrong
exactly when the system is behaving worst. If it was an arrival-rate model, I also
want to know whether any iterations were dropped, because a dropped iteration
reintroduces the same bias.

**Two: which percentile, and 2000 rps at what latency?** Throughput without a latency
constraint is meaningless — I can hit any throughput if I accept unbounded queueing.
The number I want is 'sustained 2000 rps with p99 under X'.

**Three: what dataset?** If the tables have a few thousand rows, everything is in the
buffer pool, the planner picks sequential scans that happen to be fast, and no index
depth is being exercised. Our production tables have millions of rows and a skewed
access pattern. A small dataset does not measure a faster version of the same system —
it measures a different system.

**Four: what cache state?** JIT compiled or interpreted, Postgres buffer pool warm or
cold, Redis populated or empty. Five independent caches, and they persist between
runs, so run 2 flatters run 1 for free.

**Five: was the generator saturated?** If the load tool was CPU-bound, it was
measuring itself. The check is a control run at the same rate against a trivial
endpoint — if the tail is fat there, nothing about the service has been measured. I
run that check first, not last.

**Six: can they reproduce it?** A number that does not reproduce within a tolerance is
an anecdote. Our internal rule is ±10%, because everything we do afterwards is a
comparison between two runs and we cannot detect an 8% improvement inside 30% noise."

**What separates them:** the mid answer accepts the number and asks for a second
number. The senior answer treats the number as uninterpretable until the *method* is
described, and names the specific bias in each direction. Crucially, the senior answer
knows which check to do **first** — the generator — because it invalidates everything
else.

**Follow-up:** "Which of those six would you check first if you had five minutes?"
The generator saturation control run. It is the cheapest, and a failure there means
every other number is void.

---

### Q2 — "Explain coordinated omission and how your load test avoids it."

**Mid-level answer:** "It's when the load generator doesn't send requests while it's
waiting for slow responses, so it misses some latency. You avoid it by using a
constant arrival rate."

**Senior answer:** "In a closed loop, each virtual user sends a request, waits, then
sends the next. When the server stalls — a GC pause, a lock, a pool wait — every VU is
blocked, so no new requests are issued for the duration of the stall. Real users do
not do that; requests keep arriving. So the test records one slow sample per VU, when
reality would have produced arrival-rate-times-stall-duration slow samples, and the
worst of those would have been slower still because they queued behind the stall.

Two consequences worth being precise about. First, the tail is under-reported, often
by an order of magnitude. Second — and this is the part people miss — **the error is
proportional to how badly the system is misbehaving.** The measurement is most wrong
exactly when it matters most, which is a property that makes it dangerous rather than
merely imprecise.

The fix is an open model: schedule iteration starts at a fixed rate independent of
completions. In k6 that is `constant-arrival-rate`. Every request that would have
arrived does arrive, and each records its full latency including queueing.

The subtlety is that this can silently regress. Arrival-rate executors draw from a VU
pool; if the pool is exhausted, k6 drops the scheduled iteration rather than running
it late — which is coordinated omission wearing a different hat. So I set a threshold
of `dropped_iterations: count==0` and treat any run that trips it as invalid rather
than as a slow result.

If I had to use a closed-loop tool, the mitigation is latency correction — HdrHistogram
and wrk2 do this, back-filling the samples that would have arrived during a stall. It
is a repair, not a substitute for measuring correctly."

**What separates them:** explaining the *mechanism*, quantifying the direction and
magnitude of the bias, knowing that the error is worst when the system is worst, and
knowing the specific way the open model can silently fail.

**Follow-up:** "You have `dropped_iterations` at 4,000 on a ten-minute run. What do you
do?" Discard the run. Raise `maxVUs` using Little's Law against the *degraded* latency,
not the healthy one. Re-run.

---

### Q3 — "Your p99 is high for the first minute and much lower afterwards. Why, and what do you do?"

**Mid-level answer:** "The JVM needs to warm up — the JIT compiler optimises the code
after it's been run a few times. We should add a warm-up period."

**Senior answer:** "At least five things are cold at the start of a run, and it is
worth separating them because they have different durations and different fixes.

**JIT.** The JVM interprets first, then C1-compiles, then C2-compiles hot methods, with
speculative optimisations that can deoptimise and recompile later. This is the largest
effect and the longest-lived. **Class loading** is lazy, so the first request through
each path pays to load and link. **The connection pool** starts below its minimum idle
unless it is pre-filled. **Hibernate's query plan cache** is empty. **Postgres's buffer
pool and the OS page cache** are cold, so early queries do physical reads.

What I do: measure the warm-up length rather than guess it. I run once with no warm-up
window, export the raw samples, bucket them by ten seconds, and find where p99 stops
trending down and starts merely fluctuating. Then I set the discarded window to at
least double that, because deoptimisation cycles arrive after the first compilations.
I corroborate with `jvm.compilation.time` from the actuator — when its rate of increase
flattens, compilation has largely settled.

Then the warm-up is a separate scenario in the load script, tagged so its metrics are
excluded from the recorded numbers by tag rather than by me remembering to ignore
them.

One thing I would not do is dismiss it as an artefact. If our pods are restarted
frequently — a rolling deploy, autoscaling, spot instances — then cold-start latency is
a real user experience and deserves its own baseline. I would keep a cold-start
baseline alongside the steady-state one, and I would look at AppCDS or a checkpoint
technology if it mattered enough."

**What separates them:** enumerating the distinct cold caches rather than saying "JIT",
measuring the window instead of picking a round number, and recognising that warm-up
is sometimes a real production concern rather than only a measurement artefact.

**Follow-up:** "How would you know the difference between warm-up and a memory leak?"
Warm-up decays and flattens; a leak trends upward. Look at the direction, and at heap
occupancy after each young collection in `gc.log`.

---

### Q4 — "You run six pods. Your dashboard averages their p99s. What's wrong with that?"

**Mid-level answer:** "Averaging isn't quite accurate — you should probably take the
max, or use a proper aggregation."

**Senior answer:** "It is not inaccurate; it is invalid. A percentile is a quantile of
a distribution, not a quantity. The 99th percentile of the union of six sets is not any
function of the six individual 99th percentiles, so averaging them produces a number
that does not correspond to anything.

The failure is directional and it is the wrong direction: averaging dilutes an
outlier, so when one pod is sick with long GC pauses or a slow disk, the dashboard
moves less than the user experience does. Taking the max is not right either — it
over-reports, because it reports a single pod's tail as the service's tail.

The correct approach is to aggregate the underlying histograms and compute the quantile
from the merged histogram. In Micrometer terms that means the distinction between
`percentiles`, which publishes pre-computed client-side quantiles that cannot be
combined, and `percentiles-histogram`, which publishes cumulative bucket counts that
can. Buckets are counts, and counts add. Then:
`histogram_quantile(0.99, sum by (le, uri) (rate(http_server_requests_seconds_bucket[5m])))`.

The same error appears over the time axis: `avg_over_time` of a p99 is equally
invalid, and it is a very common dashboard bug.

The accuracy cost is that a histogram quantile is interpolated within a bucket, so its
precision is bounded by bucket width. That is a bounded, known error, and I would
rather have that than a number that is wrong in an unbounded way. If I need higher
precision, I widen the bucket set around the range I care about, or use a sketch that
supports merging."

**What separates them:** "invalid, not inaccurate", knowing the direction of the bias,
naming the exact Micrometer distinction and the exact PromQL, extending it to the time
axis, and being honest about what the correct method costs.

**Follow-up:** "So what does the `/actuator/metrics/http.server.requests` endpoint give
you?" A count, a total time and a max — so a mean and a maximum, and no percentiles at
all. Useful for cross-checking throughput and errors, useless as a percentile source.

---

### Q5 — "Your baseline re-run comes back 18% off. Ship it anyway?"

**Mid-level answer:** "18% isn't that far off. I'd probably run it a couple more times
and take the average."

**Senior answer:** "No, and taking an average of an unstable measurement makes it
worse, because it hides the instability behind a single number.

18% run-to-run variance means the instrument cannot resolve anything smaller than
about 20%. Everything I plan to do next is a comparison: does this GC flag help, does
this pool size help, does moving to virtual threads help. Real wins of five or ten
percent are common and are exactly what I would now be unable to see — and worse, I
would occasionally see a 15% fluctuation and report it as a win.

So I would find the variance before doing anything else, working from most likely to
least. Warm-up too short — re-bucket the samples and check whether p99 is still
decaying when the measured window opens. Cache state differing between runs — compare
Postgres buffer hit ratio, Redis hit ratio and cumulative compilation time across the
runs. Dataset drifting — the placement scenario writes rows, so run 3 has more data
than run 1 unless I restore the snapshot every time. Generator variance — dropped
iterations and k6's own CPU. Host noise — anything else running on the machine, and on
a laptop, thermal throttling, which has the signature of the third run always being
the worst.

If after all that the variance is real and inherent to the service, that is a finding,
not a blocker to be waved through. Bimodal latency usually means something specific —
a plan flip, a lock convoy, a cache that sometimes misses — and understanding it is
probably worth more than the tuning I was about to do.

The one exception is p999 specifically. At 120,000 samples, p999 is built from about
120 observations, so it is genuinely noisy for statistical reasons. The fix there is a
longer run, not a looser tolerance. And if it still will not settle, I would record the
observed spread honestly rather than quietly widen the gate — an artefact that says
'p999 reproduces to ±18%, here is why' is more useful than one nobody believes."

**What separates them:** refusing to average away instability, having an ordered
diagnostic list, distinguishing statistical noise in p999 from systematic variance,
and treating an unexplained variance as a finding rather than an inconvenience.

**Follow-up:** "How long should the run be?" Long enough that the percentile you care
about most has enough samples behind it, and long enough to include several GC cycles.
Derived from arrival rate and target percentile, not chosen as a round number.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Your closed-loop test and your open-model test report the same p50 but wildly
   different p99. Explain, mechanically, why p50 is unaffected. Then describe a
   service for which even p50 would differ.

2. You increase the arrival rate from 200 to 400 rps and latency roughly doubles.
   Give three structurally different explanations, and for each, the one measurement
   that would confirm or eliminate it.

3. Your app container is limited to 2 CPUs. Name three JVM behaviours that change as a
   direct result of `availableProcessors()` returning 2 instead of your host's core
   count. (Topics 25, 70 and 71 are all relevant.)

4. `AlwaysPreTouch` makes startup slower and the measured window cleaner. Explain the
   mechanism. Then argue the other side: name a situation where enabling it for a load
   test makes your baseline *less* representative of production.

5. The seed uses deterministic arithmetic (`(g * 7919) % 2000`) rather than `random()`
   with a fixed seed. Give the argument for that choice. Then give the strongest
   argument against it — what does a deterministic dataset fail to represent?

6. Your gate check passes at ±4% on p50 and ±22% on p999. You have a deadline. Make
   the case for proceeding to Phase 8 anyway, then make the case against, then say
   what you would actually do and what you would write down.

7. Your k6 p99 is substantially higher than the Prometheus `histogram_quantile` p99 for
   the same endpoint over the same window. List everything that lives in the gap
   between those two measurement points, and say which of them a user would experience.

---

## Quick reference card

### The gate, in one line

> Re-run the baseline. Land within ±10% on every recorded percentile. Otherwise Phase 8
> is not startable.

### The six deliverables

1. `docker compose` stack: app + Postgres + Redis + Kafka, with CPU/memory limits.
2. Seed: ≥100k products, ≥1M orders, ≥5M order lines, with skew and hot products.
3. k6 (or Gatling) with an **open-model arrival rate**.
4. Mix: 70% catalogue read / 20% order read / 10% order placement.
5. Recorded p50/p95/p99/p999 + throughput + error rate per endpoint, in
   `/docs/java/baselines/`.
6. Recorded JVM flags: heap size, collector, container limits.

### The order you do things in

```
1. Control run against /actuator/health  ->  is the GENERATOR the bottleneck?
2. Verify container limits are applied   ->  cgroup files, jcmd VM.flags
3. Verify the dataset                    ->  row counts, sizes, EXPLAIN, ANALYZE
4. Measure the warm-up length            ->  bucket the samples, find the flattening
5. Run the baseline                      ->  warm-up scenario + measured window
6. Cross-check client vs server          ->  k6 vs actuator count, vs Prometheus p99
7. Attribute the tail                    ->  gc.log timestamps, pg_stat_statements
8. Record everything                     ->  baseline.md + environment.md
9. Re-run twice                          ->  gate check, +/-10%
```

### Commands

```bash
# Stack
docker compose up -d --wait
docker compose ps
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'
docker compose images

# Container limits -- ground truth, read from inside
docker compose exec app sh -c 'cat /sys/fs/cgroup/memory.max'
docker compose exec app sh -c 'cat /sys/fs/cgroup/cpu.max'

# JVM configuration -- these go in environment.md
docker compose exec app jcmd 1 VM.flags
docker compose exec app jcmd 1 VM.command_line
docker compose exec app jcmd 1 GC.heap_info
docker compose logs app | grep 'Picked up JAVA_TOOL_OPTIONS'

# Database
docker compose exec -T postgres psql -U orderflow -d orderflow -c '\timing on' -c 'select count(*) from order_line;'
docker compose exec -T postgres psql -U orderflow -d orderflow -c 'explain (analyze, buffers) <query>'
docker compose exec -T postgres psql -U orderflow -d orderflow -c 'select pg_stat_statements_reset();'

# Load
k6 version
k6 run --out json=results.json -e RATE=200 -e WARMUP_S=180 -e DURATION_S=600 load/k6/orderflow.js
python3 load/k6/bucket-percentiles.py results.json 10 catalogue

# Server-side view
curl -s localhost:8080/actuator/metrics/http.server.requests | python3 -m json.tool
curl -s localhost:8080/actuator/prometheus | grep -c http_server_requests_seconds_bucket

# Generator health
pgrep -x k6 && pidstat -p "$(pgrep -x k6)" 1 10      # Linux
top -l 3 -pid "$(pgrep -x k6)"                        # macOS
ulimit -n
netstat -an | grep -c TIME_WAIT
```

### PromQL you will use constantly

```promql
# Correct cross-instance p99. NEVER average per-instance p99s.
histogram_quantile(0.99, sum by (le, uri) (rate(http_server_requests_seconds_bucket[5m])))

# Throughput per endpoint
sum by (uri) (rate(http_server_requests_seconds_count[1m]))

# Error rate
sum(rate(http_server_requests_seconds_count{outcome="SERVER_ERROR"}[1m]))
  / sum(rate(http_server_requests_seconds_count[1m]))

# Connection pool pressure (Topic 109)
hikaricp_connections_pending
hikaricp_connections_active / hikaricp_connections_max
```

### k6 executors

| Executor | Model | Use for the gate? |
|---|---|---|
| `constant-vus` | closed | **No** — coordinated omission |
| `ramping-vus` | closed | **No** |
| `constant-arrival-rate` | **open** | **Yes** — the recorded baseline |
| `ramping-arrival-rate` | open | Capacity search only, never the baseline |

### Gotchas checklist

- [ ] Check the **generator** first, every single time. Control run against a trivial
      endpoint at the same rate.
- [ ] `dropped_iterations > 0` invalidates the run. No exceptions.
- [ ] `summaryTrendStats` must include `p(99.9)` or you will not have p999.
- [ ] Discard the warm-up window, and **measure** its length rather than guessing.
- [ ] Never `constant-vus` for a recorded baseline.
- [ ] Never `:latest` for any image. Record digests.
- [ ] Container CPU and memory limits must be set **and verified** from the cgroup.
- [ ] `VACUUM (ANALYZE)` after seeding, or you measure the wrong query plans.
- [ ] `setval` the sequences past the seeded ids, or the first `POST /orders` fails.
- [ ] Never average percentiles across instances or over time. Aggregate buckets.
- [ ] `/actuator/metrics/...` gives a mean and a max, not percentiles.
- [ ] Record the cache-state protocol (always-warm or always-cold) and repeat it
      exactly.
- [ ] Restore the dataset snapshot before every run — the placement scenario writes.
- [ ] Keep JFR either on for all compared runs or off for all of them.
- [ ] Hot products need deep stock, or your error rate is a story about the seed.
- [ ] Low-cardinality `name` tags on every request, or k6 spends its CPU on
      bookkeeping.

---

## When would I use this at work?

**1. Before a capacity conversation with anyone who controls budget.**
"We need four more pods" is an opinion. "At 2 CPU and 2 GB per pod, one pod sustains
X rps with p99 under our 200 ms SLO, the mix is 70/20/10, the dataset is production-
shaped, and here is the reproducible test" is a proposal. The second one gets funded
and the first one gets debated. The artefact you built here is exactly that second
thing, and Topic 129 turns it into a capacity model.

**2. Before and after any performance change, forever.**
This is the habit, not the project. Every GC flag, pool size, index, cache TTL and
framework upgrade becomes a two-measurement question. Teams without a baseline argue
about performance from anecdote and revert changes that helped. Teams with one settle
it in twenty minutes. The ±10% discipline is what makes the twenty minutes possible.

**3. When an incident review asks "why didn't we catch this?"**
Most performance incidents are catchable in advance, and the reason they are not caught
is almost always one of the five traps above: the test ran against a small dataset,
the test used a closed loop, the generator was saturated, the caches were warm, or the
dashboard was averaging percentiles. Being the person who can name which one applies —
and then fix the test rather than only the bug — is a materially different position in
that room.

---

## Connected topics

### Backwards — what this exercises, and what it will expose

- **43 — Configuration and profiles:** the `load` profile is what this runs against.
  If it is not clean and reproducible, neither is the baseline.
- **50 — N+1:** `GET /orders` is 20% of the mix here. The statement count you asserted
  on is the number that decides whether this endpoint survives the arrival rate. An
  unfixed N+1 does not show up as "slower" under load; it shows up as pool exhaustion.
- **52 — Locking and the oversell scenario:** `POST /orders` targets hot products
  deliberately, so inventory contention is real. Expect to see retry behaviour here
  that a two-thread unit test never produced.
- **53 — Batching:** the sequence generator with `allocationSize = 50` is why the
  `setval` step in the seed is mandatory.
- **55 — Transactions and the connection pool:** **expect this one to bite.** A
  transaction that holds a connection longer than you think is invisible at low
  concurrency and catastrophic at the arrival rate you are about to apply — every
  endpoint fails, including ones that touch no database. Watch
  `hikaricp.connections.pending`.
- **58–60 — The testing stack:** JUnit, Mockito, slices and context caching. This topic
  is where you stop asking "is it correct?" and start asking "is it correct at 200
  requests per second?".
- **61 — Testcontainers:** pinned images, Flyway-owned schema and real readiness checks
  all carry forward. What does *not* carry forward is Testcontainers itself — a load
  stack is long-lived, seeded and resource-limited, which is the opposite of what
  Testcontainers is for.

### Forwards — everything from here profiles this artefact

**State this plainly to yourself: from Topic 66 to Topic 135, the thing being profiled,
tuned, broken and re-measured is the artefact you just built. There is no other
system.**

- **66 — JVM architecture:** the first time you look inside the process you have been
  measuring.
- **71–72 — G1 and the low-pause collectors:** the `gc.log` you captured is the input.
  Topic 71 finds the pause cause behind your p999 spikes; Topic 72 re-runs **this exact
  baseline** under ZGC and compares. That comparison is only meaningful because of the
  ±10% gate.
- **74 — JIT:** the mechanism behind the warm-up curve you measured. You will come back
  to your bucket table and explain its shape.
- **77 — JMH:** when profiling points at one method, JMH is how you measure that method
  properly. This topic measures the system; JMH measures the method. Do not confuse
  them.
- **78 — Profiling:** where CPU actually goes under this exact load. The JFR recording
  in the compose file exists for this.
- **79 — Heap dumps:** what is retained at steady state under this load.
- **101 — Virtual threads:** re-run this baseline with `spring.threads.virtual.enabled`
  and compare. One of the most instructive comparisons in the whole curriculum, and it
  requires this artefact to exist.
- **109 — HikariCP:** the pool size you recorded in `environment.md` becomes the
  independent variable. Pool pressure under this load is where Topic 55's theory
  becomes an observation.
- **118 — Metrics and RED:** the histogram configuration you set up here becomes the
  production observability story. The `histogram_quantile` query is the same query.
- **129 — Capacity modelling (Phase 12):** this baseline is the empirical input. A
  capacity model without a measured baseline is arithmetic about a guess.

---

*Java 21 language baseline, running on JDK 25. Spring Boot 4.1 / Framework 7.0,
Jakarta EE 11, Postgres, Redis, Kafka. Every version-sensitive claim in this document
has a check command next to it — the Boot metrics property path, the `apache/kafka`
environment variables, and whether your Compose version honours `deploy.resources`.
Verify those on your own stack rather than trusting the text.*

***There is not a single measured performance number in this document. Every table of
results is blank and is yours to fill in. That is the point of the gate: the numbers
have to be yours, or they are worth nothing.***
