# 129 — Capacity, Cost, and Latency Budgets

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a first-principles capacity and cost-per-request model for `orderflow`, calibrated against your own Topic 65 baseline, ending in the break-even point for one proposed optimisation

---

> **A note on this document before you start.**
>
> There are no worked numbers in this topic. Not in the examples, not in the
> tables, not in the interview answers. Every table is blank and every formula is
> symbolic, and you fill them in from **your own** Topic 65 baseline, **your own**
> GC logs, and **your own** cloud provider's pricing page.
>
> This is deliberate and it is not a limitation. A capacity model containing
> numbers you did not measure is worse than no model at all, because the numbers
> get quoted. Somebody screenshots your table, it appears in a planning document,
> and six months later a decision is made on a figure that was an illustration.
> I have watched that happen. So: symbols and empty cells, and the work of filling
> them is the artefact.

---

## Mechanical statement

> **Fleet size is derived, not chosen.**
>
> **Measured allocation rate and live set give you heap. Measured latency and
> target throughput give you concurrency via Little's Law. The binding constraint
> is whichever resource saturates first — and it is usually not CPU.**

Three claims, and each one is a formula rather than an opinion.

**Derived, not chosen.** "We run six instances" is an answer to a question nobody
asked. The question is: what is the capacity of one instance, what is the peak
arrival rate, what headroom does the latency target require, and what redundancy
does the availability target require. Four inputs, one output. If you cannot
produce the four inputs, you do not have a capacity model; you have a number
somebody set during an incident and nobody has revisited.

**The binding constraint.** A JVM service has at least six candidate ceilings —
CPU, the connection pool, thread capacity, GC headroom, memory, and whatever
downstream shares its capacity with you. Per-instance capacity is the **minimum**
of those, not the average and not the one you happen to have a dashboard for. And
the whole practical value of the model is that it tells you *which one is the
minimum*, because that is the only one worth spending engineering time on. Making
a non-binding constraint faster produces exactly zero change in cost and exactly
zero change in throughput, and the engineering time is gone.

**Usually not CPU.** This is the part that surprises people coming from Node, and
it surprises them because in Node it is more often true that CPU is the ceiling.
In a JVM service fronting a relational database, the ceiling is very often the
connection pool (Topic 109) or the database behind it, and the second most common
is GC headroom driven by live set rather than by allocation (Topic 70). CPU is
the ceiling people plan against because it is the metric the autoscaler reads.

---

## The bridge from what you know

### What transfers — completely, and I will not re-teach it

- **Little's Law.** `L = λW`. You know it. You know what it means for in-flight
  requests. I will use it and not derive it.
- **Queueing intuition.** You know that latency rises non-linearly as utilisation
  approaches one, and that planning to run a resource near saturation is planning
  for a tail-latency problem.
- **Unit economics.** Cost per request, cost per customer, cost per order. You
  have built these models. The arithmetic transfers whole.
- **Reading a cloud pricing page** and knowing that the sticker price is not the
  bill: reserved versus on-demand, egress, the managed-service premium, the
  observability bill. Same skill.
- **Percentiles over averages, and why the tail is what the user experiences.**
  Topic 65 already made you rigorous about this.
- **Arguing for headroom against someone who wants to cut it.** Same negotiation,
  and Topic 130 gave you the error-budget framing for it.

None of that belongs in your artefact as explanation. It belongs in it as *use*.

### What does not transfer — the JVM's cost profile

This is the whole topic. A JVM service and a Node service have genuinely
different cost shapes, and if you carry your Node intuitions across you will size
the fleet wrong in a specific, predictable direction.

| You know (Node) | JVM reality | Verdict |
|---|---|---|
| Memory footprint is roughly what your objects take, plus V8 overhead. `--max-old-space-size` is most of the story | **Heap is one line of the footprint.** Add metaspace, the code cache, thread stacks, direct/mapped buffers, GC data structures, and JVM native. Setting `-Xmx` to the container limit is how you get OOM-killed | **NO ANALOGUE** — and it is the single most common container-sizing error. |
| Memory is roughly proportional to concurrent work | **Heap is driven by the LIVE SET, not by allocation rate.** You can allocate enormously with a tiny live set and barely pay; you can allocate very little with a large live set and pay on every cycle | **NO ANALOGUE.** This inverts the instinct. Topic 70. |
| Scale by process count: one process per core, N processes per box | **One JVM, many threads, one shared heap.** Vertical scaling is native; the cost per instance rises with heap and the *number* of instances multiplies fixed overheads (metaspace, code cache, one live set each) | **NO ANALOGUE** — the shape of the fleet-size-versus-instance-size trade is different. |
| A new process is at full speed immediately | **A new JVM is not at capacity for some warm-up period.** The interpreter runs first, C1 then C2 compile the hot paths as profiles accumulate, and until then the same code is materially slower (Topic 74) | **NO ANALOGUE — and it breaks naive autoscaling.** |
| More memory is monotonically better | **Crossing roughly 32 GB of heap disables compressed oops**, so references widen and a 33 GB heap can hold *fewer* objects than a 31 GB one (Topic 69) | **NO ANALOGUE** — a capacity cliff that runs backwards. |
| CPU is usually your ceiling | **The connection pool is usually your ceiling**, and Postgres does a process per connection so the pool cannot simply be enlarged (Topic 109) | **PARTIAL** — you know about connection limits; the cost per connection is much higher here. |
| GC is opaque and untunable | **Allocation rate and live set are measurable from GC logs**, and they are direct inputs to heap sizing and collector choice | **NO ANALOGUE** — you gain the measurement and the obligation to use it. |

### The three JVM cost properties, stated plainly

**1. Memory-heavy.** A JVM instance carries fixed costs no Node process has: the
live set (which does not shrink with load), metaspace holding class metadata for
every loaded class in your framework stack, a code cache holding JIT-compiled
methods, and a stack per thread. Running two instances instead of one does not
halve the per-instance memory; the fixed part is paid twice. That pushes the
economics toward **fewer, larger instances** than your Node instincts suggest —
right up until you hit the compressed-oops boundary or a GC pause budget, at
which point it pushes back the other way. Where the optimum sits is a question
your model answers, not a rule.

**2. CPU-elastic.** JVM CPU usage is genuinely elastic under load in a way that
makes CPU a poor sole scaling signal: the JIT compiles more aggressively when a
path is hot, GC threads consume CPU in proportion to allocation and live set, and
a service that looks CPU-comfortable at 40% may be *entirely* blocked on a
connection pool. High CPU on a JVM can mean "doing useful work", "compiling", or
"garbage collecting", and the autoscaler cannot tell which.

**3. Warm-up-sensitive.** This is the one that costs money in practice. A JVM that
has just started serves the same requests more slowly than one that has been
running, because the hot paths have not been compiled and the profile has not
accumulated. The operational consequences:

- **Autoscaling reacts too late by construction.** By the time CPU crosses the
  threshold, you need capacity now, and the new instance will not be at capacity
  for a while yet.
- **A readiness probe that returns 200 as soon as the context loads is lying.**
  The instance is *live*; it is not at capacity (Topic 121).
- **Rolling deploys create a latency spike** proportional to the fraction of the
  fleet that is cold. With a small fleet, that fraction is large.
- **Scale-to-zero is not available** on the JVM in the way it is for a small Node
  process, unless you have paid for it with native image or an AOT cache (Topic
  83) — which is itself a capacity and cost lever with its own trade.

Every one of those is a line in your model. None of them exists in the Node
version of the same model.

---

## What is this?

A **capacity model** is a set of formulas plus measured inputs that answers three
questions:

1. What is the capacity of one instance, and **which resource sets it**?
2. How many instances does the peak arrival rate require, given a latency target
   and an availability target?
3. What does that cost, per hour and per request, all lines included?

A **cost-per-request model** is the third question with the denominator changed,
and it exists to make an engineering decision comparable to a money decision.

A **latency budget** is the decomposition of your target percentile into the
components that spend it, so that an optimisation can be judged by how much of
the budget it recovers rather than by how much faster something got.

A **break-even analysis** puts an engineering investment and a recurring saving in
the same units and returns a number of months.

### The five inputs, and where each comes from

Everything in this topic derives from five measurements. If you do not have all
five, get them before writing anything; four of the five you already have.

| # | Input | Symbol | Where it comes from |
|---|---|---|---|
| 1 | Arrival rate at peak, per endpoint | λ | Topic 65 baseline + Topic 118 RED metrics in production |
| 2 | Service time distribution per endpoint (p50/p95/p99) | S, S₉₅, S₉₉ | Topic 65 baseline |
| 3 | Allocation rate and live set | A, L | GC logs (Topics 68, 70) |
| 4 | Resource holding time per request — connection, thread | C, T | HikariCP metrics (Topic 118 USE), profiler (Topic 78) |
| 5 | CPU-seconds consumed per request | U | Profiler, or CPU utilisation ÷ throughput |

Note the asymmetry: four of these are load-test outputs you already own from
Topic 65, and one — CPU-seconds per request — you may need to derive. That is a
short piece of work and it unlocks the whole CPU ceiling.

### The formulas, all of them, in one place

Nothing here is exotic. The value is in applying them in the right order and
taking the minimum at the end.

**Heap and memory**

```
L      = live set, MB          — heap occupancy immediately after a full GC at steady load
A      = allocation rate, MB/s — from GC log young-collection deltas over time
P      = promotion rate, MB/s  — from GC log old-gen growth over time

Heap size:            H  ≥  L × k        (k = headroom multiplier, determined by experiment)
Young collections/s:  f  =  A / Eden
Old-gen fill time:    t  =  (H − L) / P   → sets concurrent-cycle / full-GC frequency
```

`k` is not a constant you can look up. It is the multiplier at which your GC
overhead and pause distribution meet your latency budget, and you find it by
running your Topic 65 load at several heap sizes and recording pause distribution
and throughput. That experiment is a required part of the artefact.

**Container footprint (Topic 82)**

```
RSS  ≈  Heap
      + Metaspace + compressed class space
      + Code cache
      + (thread count × thread stack size)
      + Direct / mapped byte buffers
      + GC internal structures
      + JVM native (JIT, symbols, class loading)
      + JNI / native library allocations

Container memory limit  M  >  RSS × safety     (or the kernel OOM-kills you)
MaxRAMPercentage        =  H / M × 100
```

Measure every line with `jcmd <pid> VM.native_memory summary` (Native Memory
Tracking enabled) rather than estimating. This is Topic 80's machinery.

**Concurrency (Little's Law)**

```
In-flight requests:        N_inflight  =  λ × W          (W = residence time)
Threads for I/O-bound:     N_threads   ≈  λ × S
Connection pool needed:    pool        ≈  λ × C          (C = connection hold time)
Utilisation of a resource: ρ           =  λ / μ          (μ = service rate)
```

Two warnings about ρ. First, tail latency degrades sharply as ρ approaches 1, so
your planning target is a ρ well below 1 and the specific value is a decision you
make against your latency budget, not a constant. Second, the textbook M/M/1
waiting-time relation `W = S / (1 − ρ)` is a *model* with assumptions your service
does not satisfy. Use it to reason about the shape of the curve — that halving
the headroom more than doubles the queueing delay — never to produce a number you
will quote.

**Per-instance capacity — the six candidate ceilings**

```
CPU:            C_cpu   =  cores_available / U            (U = CPU-seconds per request)
Connection pool:C_pool  =  pool_size / C                  (C = connection hold time, s)
Thread pool:    C_thr   =  threads / T                    (T = thread hold time, s)
GC headroom:    C_gc    =  A_max / a                      (a = MB allocated per request;
                                                           A_max = allocation rate at which
                                                           GC overhead breaches your budget)
Memory:         C_mem   =  (H − L_base) / m               (m = per-request retained bytes,
                                                           for requests that retain)
Downstream:     C_down  =  your allocated share of a shared dependency

Per-instance capacity:   C_inst  =  min(C_cpu, C_pool, C_thr, C_gc, C_mem, C_down)
```

**Write down which term is the minimum.** That single fact is the most valuable
output of the entire model, and it is the answer to "should we optimise X".

**Fleet size**

```
From load:          N_load   =  ceil( λ_peak / (C_inst × u) )    (u = target utilisation, < 1)
From availability:  N_avail  =  the minimum count that survives losing
                                one instance / one zone, at λ_peak
From warm-up:       N_warm   =  extra capacity to absorb the cold fraction
                                during a deploy or a scale-up

Fleet size:         N  =  max(N_load, N_avail) + N_warm
```

**The shared ceiling — and why scaling out can stop working**

```
Fleet-wide DB connections:  conns  =  N × pool_size
Constraint:                 conns  ≤  DB connection budget
                            (a practical ceiling well below max_connections, because
                             Postgres runs a process per connection)

⇒  N_max_by_db  =  DB_connection_budget / pool_size
⇒  Fleet throughput ceiling  =  min( N × C_inst , C_database )
```

That last line is the whole "we need more instances" conversation in one
expression. Once `N × C_inst` exceeds `C_database`, additional instances buy
nothing — and because each one adds `pool_size` connections to a
process-per-connection database, they can actively make things worse.

**Cost**

```
Cost_hour  =  N × price_instance_hour
            + database (instance / storage / IOPS / replicas)
            + cache
            + message broker
            + load balancer
            + network egress
            + observability (metrics series, log volume, trace volume)
            + non-production environments amortised

Requests_hour    =  λ × 3600
Cost per request =  Cost_hour / Requests_hour
```

**Break-even for an optimisation**

```
E      = engineering cost           = engineer-days × your organisation's loaded day rate
ΔC     = recurring monthly saving   = (Cost_month_before − Cost_month_after)
M      = added monthly maintenance cost of the optimisation (ongoing)
Net    = ΔC − M

Break-even (months)  =  E / Net          (undefined, i.e. never, if Net ≤ 0)
```

And the gate that comes **before** the arithmetic:

```
If the optimisation does not relieve the binding constraint, ΔC = 0.
If the optimisation reduces N but N was set by N_avail rather than N_load, ΔC = 0.
```

Both of those produce a saving of exactly zero while producing an impressive
percentage improvement in a benchmark. Check them first; they are free to check
and they end perhaps a third of these conversations.

---

## Why does it matter?

**1. Because "we need more instances" is the most expensive sentence in
engineering, and it is almost never accompanied by a model.** It is cheap to say,
it is easy to approve, and it is recurring cost forever. A Principal engineer's
job in that moment is to convert it into "we are pool-bound at X per instance,
the database ceiling is Y, so instance seven onward buys nothing and here is what
does" — in the room, from data you already have.

**2. Because optimisation without a binding-constraint analysis is engineering
time set on fire.** The failure is invisible: the optimisation *works*, the
benchmark improves, everyone is pleased, and the bill does not move. Nobody
reports it, because the work succeeded on its own terms. It is one of the largest
sources of wasted senior engineering effort I know of, and the check that prevents
it takes about an hour.

**3. Because the JVM's cost profile is genuinely different and the difference
costs money in both directions.** Under-sizing heap gives you GC overhead and
tail latency; over-sizing gives you a bill and, past ~32 GB, a capacity
regression. Sizing the container from `-Xmx` gives you OOM-kills. Autoscaling on
CPU without accounting for warm-up gives you thrash and a latency spike on every
scale event. Each of those is a specific, avoidable, JVM-shaped mistake and none
of them is intuitive from a Node background.

**4. Because a cost-per-request number changes what conversations are possible.**
Once it exists, "should we cache this" and "is this feature worth building" and
"can we afford this SLO" become the same kind of question, with the same units.
Topic 130's negotiation with product — "the extra nine costs this much" — needs a
number, and this is where the number comes from.

---

## The decision, framed

The model exists to answer one question well: **which resource is binding, and
what is the cheapest way to relieve it?** Everything else falls out.

### Step 1 — Establish the unit of capacity

One instance, at your production shape: the container limits from Topic 82, the
JVM flags you actually deploy, the dataset size from Topic 65, and the scenario
mix from Topic 65 (`orderflow`'s is 70% catalogue read, 20% order read, 10% order
placement). Capacity measured at a different mix is capacity for a different
service.

### Step 2 — Measure the five inputs

Not estimate. Measure, per endpoint where the formula is per-endpoint. Record
where each number came from, because in three months you will need to know whether
to trust it.

### Step 3 — Compute all six ceilings and take the minimum

Fill the table. The output you care about is the *ordering*, not the absolute
values: which is smallest, and how far the second-smallest is behind it. If the
top two are close, relieving the first just moves you to the second, and your
break-even analysis must model the second ceiling rather than assume unlimited
headroom after the fix.

### Step 4 — Derive fleet size three ways and take the maximum

Load, availability, warm-up. It is common for availability to dominate at low
traffic — you need three instances to survive losing a zone regardless of whether
the load needs one — and that fact ends a great many optimisation proposals,
because reducing `N_load` below `N_avail` saves nothing.

### Step 5 — Price it, including the lines that are not compute

The lines people omit, roughly in order of how often they are omitted:
observability (metric series count from Topic 118, log volume, trace sampling),
non-production environments, network egress, the managed-database premium, and
backup/storage growth. A cost model that prices only instances typically
understates the bill by a large factor, and the first person to compare it with
the actual invoice will discard the whole model.

### Step 6 — Take one proposed optimisation to break-even

Gate first: does it relieve the binding constraint? Does it reduce `N` given that
`N = max(N_load, N_avail) + N_warm`? If either answer is no, the recurring saving
is zero and the analysis is over — which is a useful, publishable result.

If both are yes: engineering cost, monthly saving, ongoing maintenance, break-even
in months. Then the judgment: compare the break-even against the expected life of
the system and against **the alternative use of that engineering time**. An
optimisation that pays back in a period comparable to the system's remaining life
is not a good investment even though the arithmetic is positive.

---

## Example 1 — a minimal illustration

Deliberately symbolic. No numbers, for the reason stated at the top of this
document. Work it with your own numbers on paper as you read.

### The situation

One endpoint. It reads a row from Postgres, does some in-memory work, and returns
JSON. It runs on instances with `cores` CPUs and a HikariCP pool of `pool_size`.
Someone asks: how many instances do we need for `λ_peak`?

### The answer most people give

`N = λ_peak / (throughput we measured per instance)`, rounded up, plus one for
safety.

That is not wrong so much as **uninterpretable**. It gives you a number and tells
you nothing about what would change it, which means the next time load doubles you
have to re-run the load test rather than reason. And it hides the thing you most
need to know: which resource was saturated when you measured that throughput.

### The derivation

Measure four things at steady load:

| Symbol | Meaning | How you get it |
|---|---|---|
| `U` | CPU-seconds consumed per request | CPU utilisation ÷ throughput, or a profiler (Topic 78) |
| `C` | Seconds a request holds a pool connection | HikariCP `usage` timer (Topic 118) |
| `a` | Bytes allocated per request | Allocation profiler, or `A` ÷ throughput |
| `S` | Service time, p50 and p99 | Topic 65 baseline |

Then:

```
C_cpu  = cores / U
C_pool = pool_size / C
C_gc   = A_max / a
```

where `A_max` is the allocation rate at which GC overhead breaches your latency
budget, which you find by running the load at increasing rates and watching the
pause distribution.

`C_inst = min(C_cpu, C_pool, C_gc)`.

### The point of the exercise

Whichever of those three is smallest is **the only thing worth optimising**, and
you now know which it is instead of guessing.

Here is the pattern that recurs constantly in JVM services: `C` — connection hold
time — is often much larger than the query time, because the connection is held
for the whole transaction (Topic 55), and the transaction often includes work
that is not database work. In that case `C_pool` is small, the pool is binding,
and the fix is not more instances and not a bigger pool. It is **shortening the
transaction**: move the non-database work outside it, and `C` drops, and
`C_pool = pool_size / C` rises with no additional resources at all.

That is a capacity increase obtained by reading a formula. It costs nothing, it is
invisible without the model, and it is the single most common finding when someone
builds one of these for the first time.

### What you write down

Four rows and one sentence:

| Ceiling | Formula | Your value | Binding? |
|---|---|---|---|
| CPU | `cores / U` | | |
| Pool | `pool_size / C` | | |
| GC | `A_max / a` | | |
| **Instance capacity** | `min(…)` | | |

> "We are ______-bound at ______ requests per second per instance. The next
> ceiling is ______ at ______, so relieving the first buys us at most ______."

That last sentence is the whole model. Everything in Example 2 is that sentence
with more rows.

---

## Example 2 — the real decision on the project spine

### The setting

`orderflow` is production-shaped (Topic 124). You have:

- a Topic 65 baseline: p50/p95/p99/p999 and throughput per endpoint, at a known
  scenario mix, against a known dataset, with recorded JVM flags;
- GC logs from Phase 8, so allocation rate and live set are measurable;
- RED metrics per endpoint and USE metrics on every pool (Topic 118);
- container limits and JVM flags recorded (Topic 82);
- a known pool-vs-thread-pool interaction and a tuned pool size (Topic 109).

The trigger: traffic is forecast to grow, and someone has proposed an
optimisation. You are asked how many instances will be needed and whether the
optimisation is worth doing.

**Everything below is a form to fill in.** Fill it in from your own measurements.

### Sheet 1 — Measured inputs

Record the conditions first, because a capacity number without its conditions is
not a number:

| Condition | Value |
|---|---|
| Scenario mix | 70% catalogue read / 20% order read / 10% order placement |
| Dataset | ≥100k products, ≥1M orders, ≥5M order lines |
| Instance shape (vCPU / memory limit) | |
| JVM flags (heap, collector, `MaxRAMPercentage`, `ActiveProcessorCount`) | |
| Virtual threads enabled? | |
| Pool size (HikariCP `maximumPoolSize`) | |
| Date of measurement | |

Per-endpoint measurements:

| Endpoint | λ at baseline (rps) | p50 (ms) | p95 (ms) | p99 (ms) | CPU-s/req `U` | Connection hold `C` (ms) | Alloc/req `a` (KB) |
|---|---|---|---|---|---|---|---|
| `GET /products` (catalogue read) | | | | | | | |
| `GET /orders` (order read) | | | | | | | |
| `POST /orders` (order placement) | | | | | | | |
| **Weighted average at mix** | | | | | | | |

JVM-level measurements:

| Quantity | Symbol | Value | How measured |
|---|---|---|---|
| Live set after full GC at steady load | `L` | | `-Xlog:gc*`, occupancy after a full collection |
| Allocation rate | `A` | | Young-collection deltas over elapsed time |
| Promotion rate | `P` | | Old-gen growth over elapsed time |
| GC overhead (% of wall clock) | | | GC log pause sum ÷ elapsed |
| p99 GC pause | | | GC log |

### Sheet 2 — Heap and memory

The heap experiment. Run the Topic 65 load at several heap sizes and record:

| `H` (heap) | `H / L` = `k` | GC overhead % | p99 pause | p99 request latency | Throughput |
|---|---|---|---|---|---|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |

Pick the smallest `H` that meets your latency budget. Two things to notice while
you do it: below some `H` the overhead curve turns sharply upward, and above some
`H` you are paying for memory that buys nothing measurable. The interesting region
is narrow and it is specific to your live set.

Then the footprint, which is what the container limit must cover:

| Component | Value | How measured |
|---|---|---|
| Heap (`H`) | | your chosen `-Xmx` |
| Metaspace + compressed class space | | `jcmd VM.native_memory summary` |
| Code cache | | `jcmd VM.native_memory summary` |
| Thread stacks (count × stack size) | | thread count from JFR/metrics × `-Xss` |
| Direct / mapped buffers | | `jcmd VM.native_memory summary` (Topic 80) |
| GC internal structures | | `jcmd VM.native_memory summary` |
| JVM native / other | | `jcmd VM.native_memory summary` |
| **Total RSS** | | `docker stats` / cgroup `memory.current` to cross-check |
| **Container limit** `M` | | must exceed RSS with margin |
| **Resulting `MaxRAMPercentage`** | `H / M × 100` | |

If your total RSS is materially larger than your heap — it will be — that gap is
the reason `-Xmx = container limit` produces OOM-kills, and it is worth stating in
one line in the artefact because somebody will propose it.

### Sheet 3 — The six ceilings

| Ceiling | Formula | Inputs used | Value (rps/instance) | Binding? |
|---|---|---|---|---|
| CPU | `cores / U` | | | |
| Connection pool | `pool_size / C` | | | |
| Thread capacity | `threads / T` | | | |
| GC headroom | `A_max / a` | | | |
| Memory (retained/req) | `(H − L) / m` | | | |
| Downstream (payment gateway, cache, broker) | your allotted share | | | |
| **`C_inst`** | `min(...)` | | | |

Then, and this is the sentence the model exists to produce:

> **`orderflow` is ______-bound at ______ rps per instance. The second ceiling is
> ______ at ______ rps. Relieving the first therefore buys at most ______ rps per
> instance, not unlimited headroom.**

### Sheet 4 — The shared ceiling

Per-instance capacity is not the whole story, because instances share a database.

| Quantity | Value | How determined |
|---|---|---|
| `pool_size` per instance | | config |
| Practical DB connection budget | | **not** `max_connections` — the count beyond which p99 degrades, found by experiment |
| `N_max_by_db` = budget / `pool_size` | | |
| Database throughput ceiling `C_database` | | load test against the DB directly, or observed saturation |
| Fleet ceiling = `min(N × C_inst, C_database)` | | |
| **N at which extra instances stop helping** = `C_database / C_inst` | | |

That final row is the number that ends the "we need more instances" conversation,
and it is worth putting in bold in your artefact. Note the second-order effect
too: because Postgres runs a process per connection, instances added past that
point do not merely fail to help — each one adds `pool_size` connections and can
push the database further into contention.

### Sheet 5 — Fleet size

| Derivation | Formula | Value |
|---|---|---|
| From load | `ceil(λ_peak / (C_inst × u))`, `u` = target utilisation | |
| Target utilisation `u`, and why | (from your latency budget and the ρ curve) | |
| From availability | minimum to survive losing one instance / one zone at `λ_peak` | |
| From warm-up | extra capacity to cover the cold fraction during deploy or scale-up | |
| **`N` = max(load, availability) + warm-up** | | |
| Which term dominates? | | |

The "which term dominates" row matters more than the total. If availability
dominates, then every load-reducing optimisation has zero cost impact until it
reduces `N_load` below `N_avail`, and you should know that before anyone spends a
sprint.

The warm-up term deserves its own small table, because it is the JVM-specific one:

| Quantity | Value | How measured |
|---|---|---|
| Time from container start to context loaded | | startup logs |
| Time from context loaded to **steady-state** throughput | | load a fresh instance and watch throughput/latency converge (Topic 74) |
| Capacity of a cold instance as a fraction of warm | | same experiment |
| Rolling-deploy cold fraction | (batch size ÷ `N`) | |
| Predicted p99 impact during a rolling deploy | | measured, not assumed |
| Mitigations in use | readiness gated on warm-up (Topic 121), LB slow-start, AppCDS / AOT cache (Topic 83) | |

### Sheet 6 — Cost

From **your** cloud provider's current pricing page. Not from memory, not from
me, and note the pricing date because it moves.

| Line | Monthly cost | Basis / source |
|---|---|---|
| Compute: `N` × instance price | | pricing page, instance type, region, purchase model |
| Database: instance | | |
| Database: storage + IOPS | | |
| Database: read replicas | | |
| Cache (Redis) | | |
| Message broker (Kafka) | | |
| Load balancer + fixed networking | | |
| Network egress | | often omitted; check the bill |
| Observability: metric series | | series count × price (Topic 118 — cardinality is a cost line, not just a stability risk) |
| Observability: log volume | | |
| Observability: trace volume / sampling | | |
| Non-production environments | | amortised |
| Backups / snapshots | | |
| **Total monthly** | | |
| **Total hourly** | | |

| Quantity | Value |
|---|---|
| Requests per hour at `λ` | `λ × 3600` |
| **Cost per request** | `Cost_hour / Requests_hour` |
| **Cost per order placed** (the business unit) | | |

That last row is usually the one a non-engineer will remember, and it is the one
that makes the model useful outside engineering. Requests are an engineering unit;
orders are a business unit.

### Sheet 7 — The break-even for one proposed optimisation

State the optimisation in one sentence. Then the gate, before any arithmetic:

| Gate question | Answer | If "no" |
|---|---|---|
| Does it relieve the **binding** constraint from Sheet 3? | | ΔC = 0. Stop here and publish that. |
| After relieving it, what is the next ceiling, and how much headroom does that leave? | | The saving is capped by the second ceiling, not unlimited |
| Does it reduce `N`, given `N = max(N_load, N_avail) + N_warm`? | | ΔC = 0 if availability dominates |
| Does it change any non-compute line (DB size, egress, series count)? | | |

Then the arithmetic:

| Quantity | Value | Basis |
|---|---|---|
| `E` — engineering cost (engineer-days × loaded day rate) | | your organisation's rate; state the estimation method (Topic 126) |
| `ΔC` — monthly infrastructure saving | | from Sheet 6, recomputed with the new `N` |
| `M` — added monthly maintenance cost | | a cache to operate, a new failure mode, a new dashboard |
| `Net = ΔC − M` | | |
| **Break-even = `E / Net` months** | | |
| Expected remaining life of this system | | |
| Alternative use of the same engineering time | | **the real comparison** |
| **Decision** | | |

Two candidate optimisations for `orderflow` that make good subjects, because they
sit on different ceilings and therefore have different answers:

- **Shorten the order-placement transaction** so the connection hold time `C`
  falls (Topic 55's lesson applied as capacity work). If you are pool-bound this
  raises `C_pool` with zero additional resources, and its `E` is small. This is
  usually the highest-return item in a JVM service's capacity model and it is
  almost never on anyone's list, because it does not look like an optimisation.
- **Add a read replica** for the catalogue read path. This moves the *shared*
  ceiling `C_database` rather than the per-instance one, and it has a real
  recurring cost `M` — so it is a genuine break-even calculation rather than a
  free win, and it also has a correctness consequence (replica lag) that belongs
  in the analysis.

A third that is worth modelling and usually surprises people: **reducing
allocation per request `a`**. It raises `C_gc`, which is frequently *not* the
binding constraint, in which case the answer is a clean zero — and publishing that
zero saves a sprint.

---

## Wrong approach → exact symptom → root cause → fix

Organisational and operational symptoms. What you would have *seen*.

---

### Wrong approach 1 — scaling out past the point where the database is the ceiling

**Wrong:** latency is bad under load, so instances are added. It helps a little.
More are added.

**Exact symptom — what you would have SEEN:**

- The fleet grows and throughput at the load balancer is **flat**. The graph of
  instance count against throughput has visibly bent over and gone horizontal.
- p99 is *worse* after the last scale-up than before it, on endpoints that were
  not the problem.
- Database CPU sits near saturation and connection count sits at or near the
  configured maximum. Each new instance adds `pool_size` connections to a database
  that runs a process per connection, so scaling out is adding contention.
- HikariCP's pending-connection metric is non-zero across the fleet, which is the
  USE saturation signal from Topic 118 doing exactly its job while nobody reads it.
- Application CPU utilisation is comfortable — perhaps low — on every instance.
  Everything looks fine at the instance level, which is why the fleet keeps
  growing.
- The monthly bill has a step change and someone in finance asks a question that
  nobody in engineering can answer with a number.
- Somebody proposes raising `maximumPoolSize` "so we stop queueing", which makes
  it worse, for the reason Topic 109 spent a whole topic on.

**Root cause:** the fleet was sized against the resource that had a dashboard
(CPU) rather than against the binding constraint. Nobody computed
`C_database / C_inst` — the instance count past which additional instances
contribute nothing — so there was no signal that the scaling had stopped working
other than the absence of improvement, which is easy to attribute to "we need even
more".

**Fix:**
1. Compute all six ceilings (Sheet 3) and name the minimum. Do this *first*, every
   time, before any scaling decision. It is an hour of work against data you
   already have.
2. Compute `N_max_by_db` and the fleet throughput ceiling (Sheet 4). Put the number
   in the runbook so the next person does not rediscover it during an incident.
3. Alert on the **saturation** signal (pending connections, DB connection count as
   a fraction of budget), not on CPU. Utilisation without saturation is a
   misleading pair; USE exists precisely for this.
4. Relieve the actual ceiling. In order of cost: shorten the transaction so
   connection hold time falls; cache the read path so the query never happens
   (Topic 110); add a read replica; shard. The first is usually free and is
   usually skipped.

---

### Wrong approach 2 — sizing heap from a rule of thumb, or from the container limit

**Wrong:** heap is set to "half the container", or to the container limit, or to
whatever the previous service used.

**Exact symptom — what you would have SEEN:**

**Under-sized version:** GC overhead is a visible fraction of wall clock in the GC
log. p99 request latency tracks GC pause frequency. Full collections appear under
load. Throughput plateaus below what CPU headroom suggests it should reach. The
team concludes the service "can't handle more" and adds instances — each one
carrying its own copy of the same too-small heap and its own copy of the same live
set, so cost rises linearly and the per-instance problem is untouched.

**Over-sized version:** heap far exceeds anything the live set requires. You are
paying for memory that does nothing. Pause times are longer than they need to be
because a larger heap means more to traverse. And if the heap crosses roughly 32
GB, compressed oops turn off and every reference widens — so the instance holds
*fewer* objects than it did with a smaller heap, which is a capacity regression
caused by adding memory (Topic 69).

**Container version:** `-Xmx` is set equal to the container's memory limit. The
container is OOM-killed under load, with no `OutOfMemoryError` in the application
log and no heap dump, because the kernel killed the process rather than the JVM
failing an allocation. The event looks like a mysterious restart. Someone raises
the container limit, `-Xmx` is raised with it, and the same thing happens at
higher load.

**Root cause:** heap was chosen rather than derived, and the footprint was assumed
to equal the heap. The inputs — live set and allocation rate — were available in
the GC logs and were not used.

**Fix:**
1. Measure `L` (live set after a full GC at steady load) and `A` (allocation rate)
   from the GC log. Two numbers, both already in a file you have.
2. Run the heap experiment (Sheet 2). Pick the smallest heap that meets the latency
   budget. The multiplier that results is *your* multiplier, not a rule.
3. Size the container from **measured RSS** via Native Memory Tracking, not from
   heap. Set `MaxRAMPercentage` so the non-heap components fit with margin.
4. Watch for the compressed-oops boundary as a hard planning fact when considering
   large heaps, and prefer more instances to crossing it — or verify with your own
   measurement that crossing it is neutral for your object mix.
5. Note the direction the arithmetic points: because metaspace, code cache and the
   live set are paid *per instance*, running twice as many instances at half the
   size costs more memory in total, not the same. That trade belongs in the model.

---

### Wrong approach 3 — autoscaling a JVM on CPU without accounting for warm-up

**Wrong:** a CPU-threshold autoscaler, a readiness probe that returns 200 as soon
as the Spring context loads, and a load balancer that sends full traffic to a new
instance immediately.

**Exact symptom — what you would have SEEN:**

- A scale-up event, and then a **latency spike**. p99 gets worse immediately after
  capacity is added, which is the opposite of the intent and reliably confusing.
- Requests routed to the new instance are much slower than the fleet average for
  a period after it joins, because the hot paths are still interpreted or
  C1-compiled (Topic 74).
- The new instance's CPU is *high* while it is slow — it is compiling, and it is
  also doing more work per request — so the autoscaler reads high CPU and scales
  again. Now two cold instances. This oscillates.
- The same shape appears on every rolling deploy, at a magnitude proportional to
  the fraction of the fleet replaced at once. On a small fleet the fraction is
  large and the spike is severe.
- The error budget (Topic 130) is consumed by deploys rather than by incidents,
  which is a genuinely absurd place for it to go and is invisible unless you
  attribute burn to events.
- Someone "fixes" it by raising the scale-up threshold, which makes the fleet
  react later, which makes the spike worse.

**Root cause:** the model assumed capacity is available the moment an instance is
running. For a JVM it is not, and the gap is large enough to matter. The readiness
probe (Topic 121) reported liveness rather than capacity, so the load balancer had
no way to know.

**Fix:**
1. **Measure the warm-up curve.** Start a fresh instance, apply the Topic 65 load,
   and record throughput and p99 until they converge. That curve is a required
   input to the model and almost nobody has it.
2. Gate readiness on warm-up, not on context load: a short synthetic warm-up of
   the hot paths before reporting ready, or a readiness check that requires the
   instance to have served some traffic successfully.
3. Use load-balancer slow start / connection weighting so a new instance ramps.
4. Scale on a **leading** signal — arrival rate or queue depth — rather than on
   CPU, and scale earlier by the warm-up duration.
5. Carry the cold-capacity term (`N_warm`) in the fleet model rather than hoping.
6. If startup cost genuinely dominates your economics — spiky traffic,
   scale-to-zero ambitions — that is the honest argument for an AOT cache or a
   native image (Topic 83), and it should be evaluated as a cost decision with
   Topic 126's four buckets, not as a performance preference.

---

### Wrong approach 4 — optimising the constraint that was not binding

**Wrong:** a profiling exercise finds a hot method, an engineer spends two weeks
making it much faster, and the change ships.

**Exact symptom — what you would have SEEN:**

- The microbenchmark improves substantially. The flame graph is visibly better.
  The pull request is genuinely good work and gets deserved praise.
- End-to-end p99 at the Topic 65 load: **unchanged**, within noise.
- Instance count: unchanged. Bill: unchanged.
- The endpoint's latency profile still shows the same shape, because the time was
  never in that method from the request's point of view — it was spent waiting for
  a pool connection, which does not appear in a CPU flame graph at all (Topic 78's
  wall-clock-versus-CPU distinction, arriving as a bill).
- Nobody writes this up, because from the inside it looks like a success. So the
  same pattern repeats next quarter on a different method.
- Variant of the same failure: the optimisation *does* increase `C_inst`, but `N`
  was set by availability (three zones) rather than by load, so `N` does not move
  and the saving is exactly zero.

**Root cause:** the optimisation target was chosen from a profile rather than from
a capacity model. A profile tells you where CPU time goes inside a request; it does
not tell you whether CPU is the resource you are short of. Those are different
questions and only the second one predicts a cost change.

**Fix:**
1. Before any optimisation is funded, run the Sheet 3 gate: which ceiling is
   binding, and does this change relieve it? An hour of work, and it will end a
   meaningful fraction of proposals with a clean, defensible "no".
2. If it does relieve the binding constraint, model the **second** ceiling too.
   Relieving the first only helps as far as the second, so the achievable gain is
   bounded and should be stated as a range.
3. Check `N = max(N_load, N_avail) + N_warm` before claiming a cost saving. If
   availability dominates, the saving is zero regardless of how much faster the
   code got.
4. Publish the negative results. "We modelled this and the saving is zero" is a
   valuable, reusable artefact, and normalising it is what stops the pattern
   repeating. A team that only publishes successful optimisations will keep
   choosing them the same way.

---

### Wrong approach 5 — a cost model that prices compute and nothing else

**Wrong:** cost per request is computed as instance cost ÷ requests. The number is
small and reassuring and gets quoted in a planning document.

**Exact symptom — what you would have SEEN:**

- The actual cloud bill is a large multiple of the model. Nobody can reconcile
  them, so the model is quietly abandoned and the organisation goes back to
  arguing about cost without numbers.
- The missing lines, in roughly the order they turn up: the observability bill
  (metric series count is a cost, and Topic 118's cardinality drill was about
  stability but the same tag is also a line item; plus log volume, plus trace
  volume); non-production environments, which for many teams rival production;
  network egress; the managed-database premium and its storage growth; backups.
- A decision is made on the model — "this feature is cheap to run" — and is wrong
  by the multiple.
- The cost-per-request number is a fleet average, so an expensive endpoint and a
  cheap one are indistinguishable, and the one product decision the model could
  have informed (should we build this endpoint) cannot be answered.
- Nobody recorded the pricing date or the purchase model, so when prices or
  commitments change the model silently becomes wrong and no one notices.

**Root cause:** the model priced the resource that is easiest to attribute.
Everything shared, everything indirect, and everything non-production was left out
because attributing it requires a decision about allocation — and making that
decision felt arbitrary, so it was not made at all.

**Fix:**
1. Build the cost table from **the actual bill**, top down, not from the resources
   you can think of, bottom up. Start with the invoice total for the service's
   footprint and account for it. Anything you cannot attribute goes in an
   explicit "unattributed" row rather than being dropped.
2. Include non-production, amortised, with the allocation rule stated. An
   arbitrary but stated rule is fine; an omission is not.
3. Compute cost per request **per endpoint**, weighted by the Topic 65 mix, not as
   one fleet average. Also compute cost per business unit — cost per order placed
   — because that is the number that will be quoted outside engineering.
4. Record the pricing date, the region, the instance type and the purchase model
   next to every figure, and set a review trigger: re-check when the commitment
   renews or when traffic changes by some threshold.
5. Treat observability as a first-class cost line with an owner. It grows silently
   with cardinality, and it is one of the few lines an engineer can accidentally
   multiply with a single tag.

---

## Artefact — what you must produce

### Specification

**Title:** `orderflow — capacity and cost model`

**Length:** 1,200–2,000 words of prose, plus the sheets. The sheets are the
substance; the prose exists to state the conclusions and the uncertainties. If the
prose is long and the sheets are empty, you have written an essay about capacity
planning.

**Format:** the seven sheets from Example 2, filled in, with a **Method** or
**How measured** entry for every single number. A number without a method is not
admissible.

**Required content — the specific things I will look for:**

**1. Conditions, recorded.** Instance shape, JVM flags, pool size, scenario mix,
dataset size, date, and whether virtual threads are on. A capacity number without
its conditions is not transferable to anything.

**2. All six ceilings computed, with the binding one named.** And the sentence:
*"`orderflow` is ______-bound at ______ rps per instance; the second ceiling is
______ at ______."*

**3. The heap experiment.** At least three heap sizes, with GC overhead, pause
distribution, request p99 and throughput at each. Your chosen `H` and why.

**4. The footprint decomposition from Native Memory Tracking**, not from an
estimate, and the resulting container limit and `MaxRAMPercentage`.

**5. The warm-up curve.** Time to steady state, cold-instance capacity as a
fraction of warm, and what that implies for autoscaling and rolling deploys. This
is the JVM-specific section and it is the one most likely to be missing.

**6. The shared ceiling.** `N_max_by_db`, `C_database`, and the instance count at
which scaling out stops helping. In bold.

**7. Fleet size derived three ways** — load, availability, warm-up — with the
dominant term named.

**8. The full cost table**, from your provider's current pricing page, with the
pricing date, including observability, non-production and egress. Cost per request
per endpoint and cost per order.

**9. One optimisation taken to break-even.** The gate first (does it relieve the
binding constraint; does it reduce `N`), then `E`, `ΔC`, `M`, break-even in
months, and the decision — including the comparison against the alternative use of
the engineering time.

**10. What I am least sure about.** One paragraph. In a capacity model this is
usually one of: the CPU-seconds-per-request measurement, the database connection
budget, the loaded engineer day rate, or the peak-traffic forecast. Name yours.

### Constraints

- **No number without a method.** Every cell has a measurement, a source with a
  date, or a stated estimation method. This is the constraint; the others are
  details.
- **No number I gave you.** There are none in this document to take. Everything
  comes from your baseline, your GC logs, your NMT output, and your provider's
  pricing page.
- **Conditions recorded next to every capacity figure.** Capacity at a different
  mix or a different dataset size is a different number.
- **Predictions labelled as predictions**, with what would falsify them.
- **The break-even gate before the break-even arithmetic.** A break-even
  computed for an optimisation that does not relieve the binding constraint is a
  false answer with decimal places, which is worse than no answer.
- **Cost lines reconciled against an actual bill**, with an explicit unattributed
  row if needed.

---

## How I will review it

### The three questions that usually break a document of this kind

**Question 1 — "Which resource is binding, and how do you know?"**

This is the question the whole model exists to answer, and the most common failure
is that the document computes fleet size without ever identifying the constraint.

*How the document fails:*
- Fleet size is derived from measured throughput per instance, with no ceiling
  analysis. That is a measurement, not a model: it tells you today's number and
  nothing about what would change it.
- CPU is assumed to be binding because it is the metric on the dashboard and the
  autoscaler's input.
- The ceilings are listed but the *minimum* is not named, or the second-smallest is
  not mentioned — so the reader cannot tell how much headroom relieving the first
  would actually deliver.
- Connection hold time `C` is assumed to be the query time. It is not; the
  connection is held for the transaction, which may include work that is not
  database work (Topic 55). If `C` was not measured from the HikariCP usage timer,
  the pool ceiling is wrong and it is the one most likely to be binding.

*What a strong answer looks like:* six ceilings computed from measured inputs, the
minimum named, the runner-up named with its value, and — best of all — the
observation that the binding constraint can be relieved without buying anything,
by shortening the transaction.

**Question 2 — "Where does this number come from?" asked of any cell, at random.**

I will pick cells, including boring ones. The model's credibility is uniform: one
number that turns out to be a guess makes every other number a guess until proven
otherwise, because the reader has no way to tell them apart.

*How the document fails:*
- A Method column that is empty for some rows, or says "estimated".
- Live set taken from a heap dump at an idle moment rather than after a full GC at
  steady load. Those are very different numbers.
- Allocation rate estimated rather than computed from young-collection deltas.
- Instance price from memory, or from a different region, or without stating
  on-demand versus committed.
- The loaded engineer day rate invented. If your organisation has one, use it; if
  not, state the assumption and its basis explicitly, because the break-even is
  linear in it.
- A capacity figure with no recorded conditions, which makes it unusable the next
  time anything changes.

*What a strong answer looks like:* every cell traceable. And where a number is
genuinely an estimate, it is labelled as one with a method and a confidence, so
that I can attack it proportionally instead of attacking everything equally.

**Question 3 — "Your optimisation saves this much. Does the fleet actually get
smaller?"**

This is where break-even analyses die, and they die quietly because the arithmetic
is correct.

*How the document fails:*
- The optimisation improves a resource that is not binding. Saving: zero. The
  arithmetic in the document is impeccable and the answer is wrong.
- The optimisation improves the binding constraint, and the fleet does not shrink
  because `N` was set by availability. Saving: zero.
- The optimisation shrinks `N_load` from a value to a smaller one, but `N` only
  changes at integer boundaries, so a large percentage improvement can produce no
  instance change at all — and a small one, at the right point, can remove an
  instance. The document treats the saving as continuous when it is a step
  function.
- The ongoing maintenance cost `M` is omitted. A cache to operate, a replica to
  monitor, a new failure mode to be on call for — these have real recurring cost
  and they are always left out, which makes every optimisation look better than it
  is.
- The break-even is compared against nothing. A payback period is meaningless
  without the expected system life and the alternative use of the time.

*What a strong answer looks like:* the gate answered first and honestly, the
integer-step nature of `N` respected, `M` included, the break-even compared
against both the system's remaining life and the alternative investment — and, if
the answer is "not worth it", saying so. A model that has never produced a "no" is
a model that is not being used to decide anything.

### The other attacks, in order

- **"What is your RSS, and what is your `-Xmx`?"** If they are the same number,
  you will be OOM-killed and the model is wrong about memory.
- **"What is a cold instance's capacity as a fraction of a warm one?"** If this is
  not measured, the fleet model has no warm-up term and the autoscaling section is
  guesswork.
- **"How many connections does the whole fleet open, and what can the database
  take?"** The single most common omission that leads directly to Wrong Approach 1.
- **"Your cost per request is a fleet average. Which endpoint is expensive?"**
  The average cannot inform any product decision.
- **"Did you include observability?"** Almost nobody does. It is a real line and it
  is one an engineer can multiply accidentally with one tag.
- **"What is the utilisation target `u`, and where did it come from?"** If it is a
  round number with no derivation from the latency budget, it is a preference
  wearing a formula's clothes.
- **"What would make this model wrong?"** A capacity model has a shelf life. Name
  the review trigger: a traffic threshold, a schema change, a dependency added, a
  pricing change.

### What I will not attack

The absolute values, the choice of utilisation target, or the conclusion. I have
no view on how many instances you need — I cannot, because I do not have your
measurements, and any number I offered would be exactly the fabrication this
document opened by refusing. I am attacking the derivation, the provenance of the
inputs, and whether the binding constraint was identified before anything was
optimised.

---

## Interview questions (Senior → Principal)

> The rubric line from the master plan:
> *"we need more instances" → "at 800 rps per instance we're pool-bound, not
> CPU-bound, and Postgres is the real ceiling at 4000 rps. Scaling out past six
> instances buys nothing and costs $X/month. The read replica costs $Y and moves
> the ceiling to 12000. Here's the break-even."*
>
> The rubric's numbers are illustrative of the *shape* of the answer. Yours come
> from your own measurements.

---

### Q1 — "Traffic is going to triple. How many instances do we need?"

**A senior answer sounds like:** "I'd look at our current throughput per instance
from the load tests and scale linearly, then add headroom — so roughly three times
the current fleet plus a margin. I'd want to load-test at the higher rate to
confirm it holds, and check that the database can take it."

That is a good answer. It is empirical, it plans to verify, and it remembers the
database. Most people stop at "three times".

**A principal answer sounds like:** "Linear scaling is the assumption I'd want to
break before I answered, because if it holds, this is easy, and if it doesn't, the
answer is a different shape entirely.

So first: what is the binding constraint per instance today? I'd compute all six
and take the minimum — CPU from cores over CPU-seconds per request, the pool from
pool size over connection hold time, threads, GC headroom from the allocation rate
at which overhead breaches our budget over allocation per request, memory, and our
share of any shared downstream. In my experience with this shape of service the
minimum is the connection pool, not CPU, and that changes everything downstream
because the pool ceiling is per-instance but the *database* ceiling is shared.

Which is the second question: what is the fleet ceiling? Total connections is
instance count times pool size, against a database that runs a process per
connection, so there is a connection budget well below `max_connections` beyond
which p99 degrades for everyone. That gives a maximum useful instance count. And
separately there is a database throughput ceiling. If tripled traffic exceeds that,
adding instances does not merely stop helping — each one adds connections and makes
it worse. So the answer might not be 'more instances' at all; it might be a read
replica, or a cache on the catalogue path, or shortening the order-placement
transaction so the connection hold time falls, which raises the pool ceiling for
free.

Third: what actually sets the fleet size? It is the max of three derivations —
load, availability, and warm-up — not just load. If we need three instances to
survive losing a zone regardless, then below that point the load calculation is
irrelevant. And the warm-up term is real on the JVM: a fresh instance is not at
capacity for a period we should have measured, so an autoscaler reacting to load
is reacting too late by construction.

And I'd want the cost of each option next to the capacity of each option, because
the real question behind 'how many instances' is usually 'what will this cost' —
and once I have cost per order rather than cost per request, that conversation can
happen with the people who care about it."

**What separates them:** four things.

1. The senior answer extrapolates a measurement. The principal answer **finds the
   constraint first**, because extrapolation is only valid while the same resource
   remains binding — and tripling traffic is exactly the event that changes which
   one is binding.
2. The senior answer treats the database as a check. The principal answer treats
   it as a **shared ceiling with its own arithmetic**, and knows that instances
   past that point are actively harmful because of process-per-connection.
3. The senior answer sizes from load. The principal answer sizes from
   `max(load, availability) + warm-up` and knows which term dominates.
4. The principal answer's option set includes options that are not instances, and
   the cheapest one — shortening the transaction — costs nothing and is invisible
   without the model.

**Adversarial follow-up:** *"You've spent the meeting telling me what you'd
measure. I need a number today. Give me one."*

The honest answer gives one, with its assumptions attached and its failure
condition stated. Something like: "Assuming the pool stays binding and the
database has headroom — both of which I can check by the end of tomorrow — it
scales close to linearly and the number is N. The specific thing that would break
that is the database connection budget, and if we cross it the answer changes from
'more instances' to 'a replica', which has a lead time. So the number is N, and
the check that matters takes a day." Refusing to give a number is not rigour, it
is unhelpfulness with good manners. Giving a number without its failure condition
is the thing this whole topic exists to prevent. Do both.

---

### Q2 — "An engineer wants two weeks to optimise a hot path. It'll cut CPU per request by a third. Do you approve it?"

**A senior answer sounds like:** "I'd want to see the profiling data that
identified it, and confirm it's actually on the critical path. If it's a genuine
hot spot and the change is well-tested, a third less CPU is significant — that
could reduce our instance count. I'd approve it if the benchmark holds up
end-to-end and not just in a microbenchmark."

Good. The insistence on end-to-end rather than microbenchmark verification is the
right instinct and catches most bad optimisations.

**A principal answer sounds like:** "There is a gate before the benchmark, and it
takes about an hour with data we already have.

Is CPU the binding constraint? If we are pool-bound — which for this service I
would expect — then cutting CPU per request by a third changes our capacity by
exactly nothing. The instance count does not move, the bill does not move, and
we have spent two engineer-weeks on a change that will benchmark beautifully. That
failure is invisible from the inside, because the work succeeds on its own terms
and nobody writes it up.

If CPU *is* binding, then two more questions. What is the second ceiling, and how
much headroom does relieving CPU actually buy before we hit it? The gain is
bounded by the runner-up, not unlimited. And does the fleet actually get smaller?
Instance count is an integer and it is the max of load, availability and warm-up —
so a third less CPU might remove an instance, or might remove none, or might
remove none because availability sets the floor. The saving is a step function, not
a percentage.

If all of that clears, then the arithmetic: two engineer-weeks at our loaded rate
against the monthly saving, minus whatever ongoing maintenance the change adds,
gives a break-even in months. Then I compare that against the system's expected
life and against what else those two weeks could do — because a positive payback
is not the same as the best use of the time.

And here is what I would probably say instead: from the same model, shortening the
order-placement transaction so it stops holding a pool connection through work that
isn't database work would raise the pool ceiling directly, and it is probably a
couple of days rather than two weeks. If the pool is what's binding us, that is the
same engineer, a fifth of the time, on the constraint that actually matters.

What I would not do is turn this into a no. The engineer found something real. The
question is whether it is the most valuable real thing, and the model answers that
in an hour."

**What separates them:** the senior answer verifies that the optimisation *works*.
The principal answer asks whether it **matters**, which is a different question
with a different method — a capacity model rather than a profile. It also knows
that fleet size is a step function so percentage gains do not translate linearly
into savings, includes ongoing maintenance cost, compares against the alternative
use of the time, and — the move that makes it constructive rather than obstructive
— redirects the engineer to the binding constraint rather than declining the work.

**Adversarial follow-up:** *"You told them no and they're demotivated. Was
identifying a real inefficiency the wrong thing to do?"*

No, and the framing of the answer matters more than its content. The finding was
good; what was missing was the gate, and the gate is a team capability rather than
a personal failing. The correction is to make the binding-constraint check part of
how optimisation work gets picked — publish the model, put the current binding
constraint on the dashboard, and make "which ceiling does this relieve" a normal
question rather than one senior person's veto. Then the same engineer's next
finding lands on the constraint that matters and nobody has to be told no. And I
would publish this specific negative result, because a team that only sees
successful optimisations will keep choosing them the same way, and a documented
zero is a genuinely valuable artefact.

---

### Q3 — "How do you size a JVM's heap?"

**A senior answer sounds like:** "From the live set — the memory still in use
after a full GC — with headroom on top, usually a couple of times the live set.
I'd verify by running the load test and checking GC overhead and pause times. And
the container limit needs to be bigger than the heap, because there's metaspace,
thread stacks and off-heap memory too."

That is a genuinely strong answer. It names the right input, the right
verification, and the container distinction that most people miss.

**A principal answer sounds like:** "Live set is the input, and the multiplier is
an experiment rather than a constant.

Concretely: measure the live set as heap occupancy immediately after a full
collection at steady load — not at idle, and not from a heap dump taken at a
convenient moment, because those are different numbers. Measure the allocation
rate from the young-collection deltas in the GC log. Then run the Topic 65 load at
several heap sizes and record GC overhead, pause distribution, request p99 and
throughput at each. Pick the smallest heap that meets the latency budget. The
multiplier that falls out is ours, for this workload; someone else's is not
transferable because it encodes their live set and their allocation profile.

Then the footprint, which is a separate calculation and where the operational
failures actually come from. RSS is heap plus metaspace and compressed class
space, plus the code cache, plus a stack per thread, plus direct and mapped
buffers, plus GC internal structures, plus JVM native. I'd measure each with
Native Memory Tracking rather than estimate, because the gap between heap and RSS
is where OOM-kills live: if `-Xmx` equals the container limit, the kernel kills the
process with no `OutOfMemoryError` and no heap dump, and it presents as a
mysterious restart.

Two more things that are planning facts rather than tuning facts. Crossing roughly
32 GB of heap turns off compressed oops, so references widen and a bigger heap can
hold fewer objects — a capacity regression caused by adding memory, and a hard
ceiling on the vertical-scaling direction. And metaspace, the code cache and the
live set are paid *per instance*, so two instances at half the heap cost more total
memory than one at full — which is the real trade between scaling up and scaling
out on the JVM, and it is the opposite of the Node instinct where processes are
cheap.

And the thing I'd say to whoever asked: heap size is not a tuning knob you turn
until it feels right. It is derived from two measured numbers that are already in
our GC logs."

**What separates them:** the senior answer knows the inputs and the method. The
principal answer treats the multiplier as an **experiment with a recorded result**
rather than a rule, distinguishes the measurement conditions (after a full GC, at
steady load) that make the live-set number valid, decomposes the footprint from
NMT rather than estimating, and adds the two facts that are capacity-planning
facts rather than tuning facts: the compressed-oops cliff and the per-instance
fixed cost that shapes the scale-up-versus-scale-out decision.

**Adversarial follow-up:** *"Our live set grows over the day and resets on the
nightly deploy. What do you size against?"*

The honest answer starts by naming what that pattern usually is: a live set that
grows monotonically and resets on restart is the signature of a retention problem
rather than a sizing problem, and it is Topic 79's territory — an unbounded cache,
an unremoved listener, a `ThreadLocal` on a pooled thread. So the first move is a
heap dump near the peak and a look at the dominator tree, because sizing the heap
to accommodate a leak is buying memory to postpone an incident.

If it is genuinely legitimate growth — a cache that fills through the day and is
*bounded*, a working set that follows traffic — then you size against the peak
live set plus headroom, and you note that the nightly restart is load-bearing,
which makes it a dependency rather than a coincidence: if someone stops the daily
deploy, the service changes behaviour. That belongs in the runbook and probably in
the risk register. The better answer is to bound the cache so the live set has a
ceiling you chose rather than one the traffic chose, at which point you are sizing
against a number you control.

---

### Q4 — "What's your cost per request, and why should I care?"

**A senior answer sounds like:** "We'd take the monthly infrastructure cost and
divide by the number of requests. It matters because it tells us whether we're
spending efficiently and lets us project what growth will cost."

Correct, and the projection use is a real one.

**A principal answer sounds like:** "Two things about the number itself and then
what it is for.

The number: a fleet average is nearly useless, because the endpoints have very
different costs. A catalogue read that hits a cache and a transactional order
placement that holds a connection through several writes are not the same unit of
work, and averaging them hides exactly the information a product decision needs.
So I compute it per endpoint, weighted by the actual traffic mix, and I also
compute cost per *order placed*, because that is the business unit and it is the
one that will get quoted outside engineering.

And the denominator of the cost side has to be the whole bill, not compute. The
lines that get left out are consistent: observability — metric series count, log
volume, trace volume, and series count is something an engineer can multiply
accidentally with a single tag; non-production environments, which are frequently
comparable to production; egress; the managed-database premium and its storage
growth. If the model prices only instances it will be wrong by a large multiple,
somebody will compare it to the invoice, and then nobody trusts any of it. I'd
build it top-down from the actual bill with an explicit unattributed row rather
than bottom-up from resources I can think of.

What it is for is the part I care about. Once cost per order exists, three
conversations that were previously about opinion become arithmetic. Whether an SLO
target is affordable — the extra nine costs this much, and here is the revenue at
risk. Whether an optimisation is worth two engineer-weeks — break-even in months,
compared against the alternative use of the time. And whether a proposed feature is
economically sensible at the volume product is forecasting. Without the number
those are all seniority contests. With it they are decisions, and the person with
the number is usually the person the room defers to — which is a considerable
amount of influence for a spreadsheet."

**What separates them:** the senior answer computes the metric. The principal
answer knows the metric's **failure modes** — the average hiding the variance, the
bill hiding in the lines nobody attributes — chooses a business-meaningful
denominator, builds it top-down so it reconciles, and is explicit about the
conversations it makes possible. The last part is the real difference: a cost model
is not a reporting artefact, it is a decision-making instrument, and it is one of
the cheapest sources of influence available to an engineer.

**Adversarial follow-up:** *"Finance already has a cost-per-transaction number and
it doesn't match yours. Now what?"*

Reconcile, publicly, and expect to be the one who is wrong about something. The
usual causes are boundary and allocation: they may be including support,
licensing, payment-processing fees or amortised headcount; you may be excluding
non-production or attributing shared infrastructure differently. Neither number is
wrong — they answer different questions — but two numbers in circulation with the
same name is worse than one, so the output should be a single agreed definition
with the scope written next to it, and if two numbers are genuinely needed they get
two different names. The thing not to do is defend yours; the value is in the
reconciliation, and doing it with finance rather than at them is also how you get
them to bring you the next cost question early instead of late.

---

### Q5 — "We're at 40% CPU across the fleet. Can we cut instances?"

**A senior answer sounds like:** "Not necessarily — CPU isn't the only constraint,
and we need headroom for spikes and for losing an instance. I'd check memory,
connection pool usage and latency at the current level before cutting, and I'd do
it gradually while watching p99."

Good instincts, right caution, right verification method.

**A principal answer sounds like:** "40% CPU tells me CPU is not binding. It tells
me nothing about whether we can cut, and on a JVM it is one of the least
informative numbers on the dashboard, because that 40% is a mix of request work,
JIT compilation and GC threads and the three are not distinguishable from the
outside.

What I'd look at instead is saturation, not utilisation. Is the connection pool's
pending-acquire count ever non-zero? Is queue depth ever non-zero? Those are the
USE saturation signals, and a resource can be at low utilisation and still be the
one queueing — which is exactly the pool-bound case, where CPU sits comfortable
while requests wait.

Then the three things that actually set the floor. Availability: what is the
minimum count that survives losing an instance or a zone at peak? If that is the
binding term, CPU is irrelevant to the decision. Warm-up: during a rolling deploy
some fraction of the fleet is cold and below capacity, so a fleet cut also cuts the
absorbing capacity for every deploy, and the p99 spike gets worse. And peak, not
average: 40% is presumably a mean over a window, and the number that matters is the
worst minute, not the average hour.

If after all that there is genuine slack, then yes, cut — but I'd want to know what
we are buying, because instance cost may be a small fraction of a bill dominated by
the database and observability, in which case removing two instances is a rounding
error and the effort is better spent elsewhere. And I'd cut one at a time with the
p99 and the saturation signals watched, because the model is a prediction and this
is the experiment that tests it."

**What separates them:** the senior answer knows CPU is not the only constraint.
The principal answer knows **why CPU specifically is a poor signal on a JVM**,
substitutes saturation for utilisation as the diagnostic, names the three terms
that set the floor including the JVM-specific warm-up one, and then asks the
question nobody asks — whether the saving is material against the whole bill.
Treating the cut as an experiment that tests the model, rather than as an action,
is the last piece.

**Adversarial follow-up:** *"We cut two instances, nothing broke, and the p99 is
unchanged. So the model was wrong and we were over-provisioned. Cut two more?"*

The honest answer accepts the evidence and refuses the extrapolation. Nothing
breaking is real information and it should update the model — write down the new
observed capacity and note that the previous headroom estimate was conservative.
But "nothing broke at steady state" is not the test that matters. The tests that
matter are: what happens at the actual peak minute, not the average; what happens
during a rolling deploy when a fraction of the smaller fleet is cold; and what
happens when one instance is lost, which is now a larger fraction of the fleet than
it was. Each of those is deliberately exercisable — run the peak, do a deploy under
load, kill an instance under load — and each is cheap. Cut two more *after* those,
not instead of them. And there is a floor that no amount of evidence moves: the
availability minimum. Below it you are not over-provisioned, you are
under-redundant, and the failure mode is not latency, it is an outage.

---

## Mental model checkpoint

Reason these out in writing. Use your own numbers.

1. Your service is at low CPU utilisation and requests are queueing. Name every
   resource that could be binding, and for each one name the *specific metric*
   that would confirm it. Which of those metrics do you currently have?

2. Heap is derived from the live set, not from the allocation rate. Construct a
   workload with a very high allocation rate and a tiny live set, and one with the
   reverse. Which needs more heap, and which needs a different collector? Now say
   which one `orderflow`'s order-placement path resembles.

3. A JVM instance is not at capacity for some period after start. Work through
   every operational consequence you can find — autoscaling, rolling deploys,
   readiness probes, canaries, load-balancer weighting, error-budget attribution —
   and say which of them your current setup handles.

4. Two instances at half the heap versus one at full: work through the total
   memory cost, including the per-instance fixed components. Now add the
   compressed-oops boundary and the availability floor. Where does the optimum sit
   for `orderflow`, and what would move it?

5. An optimisation reduces CPU per request by a third. Enumerate every reason the
   monthly bill might not change at all. There are at least four.

6. Fleet size is `max(N_load, N_avail) + N_warm`. Construct a plausible situation
   for `orderflow` where `N_avail` dominates, and say what that implies about which
   optimisations are worth funding at that traffic level.

7. You have to give a capacity number in the next ten minutes with no time to
   measure. What do you say, and what exactly do you attach to it so that it is
   safe for someone else to quote? Now: what would make it unsafe to give a number
   at all?

---

## Quick reference card

### The formulas

```
HEAP
  H ≥ L × k                      L = live set after full GC at steady load
  f_young = A / Eden             A = allocation rate (MB/s)
  old-gen fill = (H − L) / P     P = promotion rate (MB/s)

FOOTPRINT
  RSS ≈ H + metaspace + code cache + (threads × stack)
        + direct buffers + GC structures + JVM native
  M > RSS × safety               MaxRAMPercentage = H/M × 100

CONCURRENCY (Little's Law)
  N_inflight = λ × W
  threads   ≈ λ × S
  pool      ≈ λ × C              C = connection HOLD time (not query time)
  ρ = λ/μ                        plan well below 1

CEILINGS  (take the minimum, and name it)
  C_cpu  = cores / U             C_pool = pool_size / C
  C_thr  = threads / T           C_gc   = A_max / a
  C_mem  = (H − L) / m           C_down = your share of a shared dependency
  C_inst = min(all of the above)

SHARED CEILING
  fleet connections = N × pool_size ≤ DB connection budget
  N_max_by_db       = budget / pool_size
  scaling stops helping at N = C_database / C_inst

FLEET
  N = max( ceil(λ_peak / (C_inst × u)), N_avail ) + N_warm

COST
  Cost_hour = compute + DB + cache + broker + LB + egress
            + observability + non-prod amortised
  CPR = Cost_hour / (λ × 3600)

BREAK-EVEN
  Net = ΔC − M
  months = E / Net       — and ΔC = 0 if the constraint relieved was not binding,
                           or if N was set by availability
```

### The gate, before any optimisation is funded

- [ ] Which ceiling is binding? (compute all six)
- [ ] Does this change relieve **that** one?
- [ ] What is the second ceiling, and how much headroom does relief actually buy?
- [ ] Does `N` change, given `N = max(N_load, N_avail) + N_warm` and `N` is an integer?
- [ ] Is `M` (ongoing maintenance) counted?
- [ ] Break-even versus system life **and** versus the alternative use of the time?

### JVM cost properties your Node instincts will miss

1. **Memory-heavy** — heap is one line of RSS; per-instance fixed costs are paid
   per instance, which favours fewer, larger instances.
2. **Live set, not allocation rate**, drives heap and GC cost.
3. **CPU-elastic and ambiguous** — 40% CPU may be work, JIT, or GC, and low CPU is
   fully compatible with being saturated on the pool.
4. **Warm-up-sensitive** — a fresh instance is not at capacity; this breaks CPU
   autoscaling and puts a spike on every rolling deploy.
5. **~32 GB compressed-oops cliff** — more memory can mean less capacity.
6. **Pool before CPU** — and Postgres runs a process per connection, so the pool
   cannot simply be enlarged.

### Measurement sources

| Input | Source |
|---|---|
| λ, S, p95, p99 | Topic 65 baseline; Topic 118 RED metrics |
| `L`, `A`, `P` | `-Xlog:gc*` (Topics 68, 70) |
| RSS decomposition | `jcmd <pid> VM.native_memory summary` (Topic 80) |
| `C` connection hold time | HikariCP usage timer (Topics 109, 118) |
| `U` CPU-seconds/request | Profiler (Topic 78), or CPU% ÷ throughput |
| `a` allocation/request | Allocation profiler, or `A` ÷ throughput |
| Warm-up curve | Fresh instance under Topic 65 load (Topic 74) |
| Prices | **Your** provider's pricing page, with the date |

### The sentence the whole model produces

> "`orderflow` is ______-bound at ______ rps per instance. The second ceiling is
> ______ at ______. The fleet ceiling is ______, so instance number ______ onward
> buys nothing. Relieving the binding constraint costs ______ and pays back in
> ______ months."

---

## When would I use this at work?

**1. The next time someone says "we need more instances".**
This is the highest-frequency use and it is nearly free once the model exists.
The conversation currently ends with a decision made from a CPU graph and a
feeling. With the model it ends with "we are pool-bound; instance seven onward
does nothing because the database ceiling is here; the cheapest relief is
shortening the transaction and it is two days". That reframing, done once in
public, changes how your organisation makes every subsequent scaling decision —
and the model is mostly built from data you already collected in Topic 65.

**2. When an optimisation is proposed.**
Run the gate before the work is funded, not after it ships. One hour, against data
you have. It will sometimes return a clean zero, and publishing that zero is
valuable in itself: it is the only thing that stops the team choosing the next
optimisation the same way. Over a year, the compounding effect of directing
optimisation effort at the binding constraint rather than at whatever the profiler
made salient is very large, and it is invisible unless someone is doing this.

**3. Whenever a cost or an SLO conversation happens with people outside
engineering.**
Cost per order is a unit that a product manager, a finance partner and a director
can all reason about. Once it exists, "can we afford 99.99%" (Topic 130), "is this
feature economically sensible", and "should we spend two weeks on this" are all
the same kind of question with the same units. The person who has that number is
usually the person whose recommendation is adopted — not because they argued
better, but because everyone else is arguing without one. That is a considerable
amount of influence for a spreadsheet, and it is one of the cheapest to acquire.

---

## Connected topics

**Prerequisites — the measurements this model consumes:**

- **Topic 65 — the load baseline.** λ, service times, percentiles, the scenario
  mix and the dataset. This model is unbuildable without it, which is why it was a
  hard gate.
- **Topic 68 — TLABs, eden, promotion.** Where allocation rate and promotion rate
  come from, and why allocation is cheap and retention is not.
- **Topic 69 — object layout and compressed oops.** The ~32 GB cliff as a capacity
  planning fact, and per-object overhead as an input to live-set forecasting.
- **Topic 70 — GC fundamentals.** Live set and allocation rate as *the* two inputs
  to heap sizing. This is the topic this model rests on most heavily.
- **Topics 71–72 — G1, ZGC.** Collector choice follows the latency budget, and the
  heap experiment in Sheet 2 is where you find out which one you need.
- **Topic 74 — JIT and warm-up.** The warm-up curve, and why a fresh instance is
  not capacity.
- **Topic 78 — profiling, wall-clock versus CPU.** `U` comes from here, and the
  wall-clock/CPU distinction is why a CPU flame graph cannot see pool waiting.
- **Topic 80 — native memory tracking.** The RSS decomposition in Sheet 2.
- **Topic 82 — container awareness.** `MaxRAMPercentage`, `ActiveProcessorCount`,
  and the OOM-kill that follows from `-Xmx` = container limit.
- **Topic 83 — native image / AOT.** The startup-and-footprint lever, and the
  honest answer when warm-up dominates your economics.
- **Topic 90 — pool sizing from Little's Law.** The same law, applied to thread
  pools; this topic applies it to fleets.
- **Topic 109 — HikariCP and the real ceiling.** The single most important
  prerequisite for the "usually not CPU" claim.
- **Topic 118 — metrics, RED and USE.** Saturation versus utilisation, and metric
  cardinality as a cost line rather than only a stability risk.
- **Topic 121 — liveness versus readiness.** Why the warm-up term needs a probe
  that reports capacity rather than existence.
- **Topic 124 — the readiness review.** Capacity headroom at baseline is one of its
  required sections; this is how you compute it properly.

**This unlocks and connects:**

- **Topic 126 — build vs buy vs adopt.** The operational lines in its recurring-cost
  table — extra memory in the live set, extra threads, extra pools, startup time —
  are computed here rather than guessed.
- **Topic 127 — migration planning.** The JDK step changes the default collector,
  which changes the heap experiment's answer. Every "verified by baseline
  comparison" cell in that plan is checked against this model.
- **Topic 128 — monolith to services.** A split doubles the pools against one
  database and doubles the warm-ups per deploy. Its predicted-effects section is
  this model applied to a proposed architecture.
- **Topic 130 — SLOs and error budgets.** The utilisation target `u` comes from the
  latency budget, and "the extra nine costs this much" is a number this model
  produces.
- **Topic 131 — design docs.** Any design doc with a performance or cost claim
  should cite this model. "We think this will be faster" versus "this relieves the
  binding constraint, which is X, and the ceiling after it is Y."
- **Topic 135 — the capstone.** "How many instances do you need and why" is a
  standard principal-loop prompt, and the follow-up is always some version of "the
  fleet is at 40% CPU — cut it".

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0. The
mechanics in this topic — live set driving heap, Little's Law driving concurrency,
the minimum of the ceilings driving per-instance capacity, process-per-connection
driving the fleet ceiling, and JIT warm-up driving the autoscaling term — are
properties of the platform and are stable. Every number is yours. There are none of
mine in this document, deliberately: prices, instance capacities and latency
figures move, and a figure you did not measure will eventually be quoted as if you
had.*
