# 44 — Resource Requests and Limits
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Imagine you're booking tables at a restaurant for your team.

- A **request** is the table you **reserve**. You tell the host "we need seats for 4." The host writes it in the book and *guarantees* those 4 seats are yours — even if you show up late and only 2 people come, those seats are held. The host uses reservations to decide **which restaurant even has room for your party** — they won't seat you where there isn't enough reserved space left.

- A **limit** is the **maximum number of chairs you're allowed to drag to your table**. Reserve 4, but maybe on a quiet night you can grab 2 extra empty chairs. The limit says "you may use up to 6 chairs, never more." If you try to grab a 7th chair for food (memory), the bouncer **throws your whole party out** (OOMKilled). If you try to grab a 7th chair for elbow room (CPU), they don't throw you out — they just **make you wait / slow you down** (throttling).

The two crucial kid-level truths:

1. **Requests are about getting a seat at all** (scheduling). **Limits are about how much you may grab once seated** (enforcement).
2. **Running out of food-chairs (memory) gets you kicked out. Running out of elbow-chairs (CPU) just slows you down.** Memory and CPU are punished completely differently, and that difference is the whole chapter.

---

## The Linux kernel feature underneath

This is all **cgroups** (control groups) — the same kernel feature from **Topic 01** and **Topic 25**. Kubernetes doesn't invent resource control; it just translates your YAML into cgroup files that the kubelet writes on the node. There is nothing "Kubernetes-magic" about limits — they are Linux cgroup v2 knobs.

**Memory limit → `memory.max`.** When you write `limits.memory: 512Mi`, the kubelet writes `536870912` into the container's cgroup file:

```
/sys/fs/cgroup/kubepods.slice/.../memory.max  →  536870912
```

The kernel tracks every page the container's processes touch in `memory.current`. The instant `memory.current` would exceed `memory.max` and no pages can be reclaimed, the kernel's **OOM killer** fires and kills a process in that cgroup — that's `OOMKilled`. It is a hard, synchronous kernel action. **Memory is incompressible**: you can't "use a little less" once you've allocated it, so the only enforcement is death.

**CPU limit → `cpu.max`.** When you write `limits.cpu: 500m` (half a core), the kubelet writes a quota/period pair:

```
/sys/fs/cgroup/kubepods.slice/.../cpu.max  →  "50000 100000"
                                                │     └─ period: 100ms (100000µs)
                                                └─ quota: 50ms of CPU time per 100ms period
```

Meaning: in every 100ms window, this cgroup may run on CPU for at most 50ms. If it uses its 50ms early, the **CFS bandwidth controller** (Completely Fair Scheduler) **freezes the process until the next 100ms window**. That freeze is **throttling**. CPU is **compressible**: the kernel can just make you wait, so no one gets killed — they get slow.

**CPU request → `cpu.weight`.** A request (`requests.cpu: 250m`) becomes a *proportional share* (`cpu.weight`), deciding who wins when the CPU is contended. It's a relative priority, not a hard cap.

So the entire topic is: your YAML numbers become four cgroup files (`memory.max`, `memory.low`/accounting, `cpu.max`, `cpu.weight`), and the kernel enforces them with two totally different personalities — **kill for memory, throttle for CPU.**

---

## What is this?

**Requests** are the amount of CPU/memory a container is *guaranteed* — the scheduler uses them to decide which node has room. **Limits** are the *maximum* a container may use — the kernel enforces them via cgroups. Requests are a scheduling promise; limits are a runtime ceiling.

From these two numbers Kubernetes also derives a **Quality of Service (QoS) class** (Guaranteed, Burstable, BestEffort) that decides who gets killed first when a node runs out of memory.

---

## Why does it matter for a backend developer?

Without correct requests and limits, three specific disasters hit your `orders-api`:

1. **Silent OOMKills.** You set no memory limit (or too low). Under a traffic spike, `orders-api` grows to 800Mi, hits the cap, and the kernel kills it mid-request. Customers see dropped connections; you see mysterious `RESTARTS` climbing with `Reason: OOMKilled` and no app log explaining it (because the kernel killed it, the app never got to log).

2. **Invisible CPU throttling.** You set `limits.cpu: 250m` "to be safe." Now under load, `orders-api` gets frozen for tens of milliseconds every 100ms. Your p99 latency balloons from 50ms to 800ms. Nothing crashes, nothing logs an error — the app is just mysteriously slow, and it's *because* of your limit.

3. **Noisy-neighbor evictions and bad packing.** With no requests, the scheduler thinks `orders-api` needs nothing, packs 30 pods onto one node, and when memory pressure hits, the kernel and kubelet start **evicting** pods — possibly your critical `orders-api` instead of a batch job — because it had no requests to protect it.

Requests and limits are how you tell Kubernetes "this is what my app needs and this is how much it's allowed to grab." Get them wrong and you get outages that look like bugs but are really capacity misconfiguration. This is the second most common self-inflicted K8s incident after probes (Topic 43) — and the two interact: a CPU-throttled pod fails its liveness probe and gets restarted, turning slowness into a crashloop.

---

## The physical reality

When `orders-api` runs with requests and limits, here's what physically exists.

**1. The numbers are in the pod spec (etcd):**

```bash
kubectl -n orders get pod orders-api-xxx -o jsonpath='{.spec.containers[0].resources}'
# {"requests":{"cpu":"250m","memory":"256Mi"},"limits":{"cpu":"500m","memory":"512Mi"}}
```

**2. They become real cgroup files on the node.** Exec into the pod and read them (cgroup v2):

```bash
kubectl -n orders exec orders-api-xxx -- cat /sys/fs/cgroup/memory.max
# 536870912                 ← 512Mi limit, in bytes
kubectl -n orders exec orders-api-xxx -- cat /sys/fs/cgroup/cpu.max
# 50000 100000              ← 500m limit = 50ms quota per 100ms period
kubectl -n orders exec orders-api-xxx -- cat /sys/fs/cgroup/memory.current
# 148905984                 ← currently using ~142Mi right now
```

**3. Live usage is tracked by the kernel** in `memory.current` and CPU accounting in `cpu.stat`:

```bash
kubectl -n orders exec orders-api-xxx -- cat /sys/fs/cgroup/cpu.stat
# nr_throttled 4123         ← how many times this pod was FROZEN (throttled)
# throttled_usec 91230000   ← total microseconds spent frozen — this is your latency killer
```

**4. The node's allocatable budget** is tracked by the scheduler. `requests` are subtracted from it:

```bash
kubectl describe node node-1 | grep -A6 "Allocated resources"
#   Resource   Requests      Limits
#   cpu        1250m (62%)   2 (100%)     ← sum of all pods' requests vs node capacity
#   memory     2Gi (51%)     3Gi (76%)
```

**5. An OOMKill leaves a fingerprint** in the pod's last state:

```bash
kubectl -n orders get pod orders-api-xxx -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# OOMKilled                 ← the kernel OOM killer, exit code 137 (128 + SIGKILL 9)
```

So resource management is: numbers in etcd → cgroup files on disk → kernel counters in memory → a scheduler ledger → an exit reason when it goes wrong.

---

## How it works — step by step

Full trace from `kubectl apply` to enforcement.

1. **You declare resources in the pod spec** and apply it. The API server stores requests+limits in etcd.

2. **The scheduler reads `requests` only.** For each node it checks: `node allocatable − sum(requests of pods already there) ≥ this pod's requests`? Nodes that fail are filtered out. **Limits are ignored by the scheduler** — only requests decide placement. This is why requests are your "reservation."

3. **Scheduler binds the pod to a node** with enough *requestable* room. Note: because it only sums requests, a node can be **overcommitted** — the sum of *limits* can exceed node capacity. That's allowed and normal (that's what "burstable" means).

4. **The kubelet on that node creates the cgroup** for the pod and its containers and writes the enforcement files: `memory.max` (from memory limit), `cpu.max` (from CPU limit), `cpu.weight` (from CPU request). If a resource has a request but no limit, there's no `.max` cap — it can use whatever's free.

5. **The container runs; the kernel accounts every page and CPU slice.** Memory pages touched → counted in `memory.current`. CPU time used → debited against the `cpu.max` quota each 100ms period.

6. **CPU limit enforcement (throttling).** If the container uses its full quota (e.g. 50ms) before the 100ms period ends, the CFS bandwidth controller **dequeues the tasks** — they simply don't get scheduled onto a CPU until the next period refills the quota. `nr_throttled` increments. The app is alive but frozen for the remainder of the window. **No kill.**

7. **Memory limit enforcement (OOMKill).** If the container's allocations push `memory.current` toward `memory.max`, the kernel first tries to reclaim (drop caches). If it can't reclaim enough and an allocation would exceed `memory.max`, the **cgroup OOM killer** picks a process in that cgroup and sends `SIGKILL`. Exit code **137**. Pod status → `OOMKilled`. If it was the main process, the container restarts (subject to backoff).

8. **Node-level memory pressure → eviction.** Separately, if the *whole node* runs low on memory (below the kubelet's `--eviction-hard` threshold, e.g. `memory.available<100Mi`), the **kubelet proactively evicts pods** to save the node. It ranks victims by QoS and by how far each pod exceeds its *requests*: **BestEffort first, then Burstable over-request, Guaranteed last.** Evicted pods are deleted and rescheduled elsewhere.

9. **QoS class was computed at admission** (step 1) from the request/limit shape, and it's what step 8 uses to decide the killing order.

The one-line summary: **requests place you (step 2), limits cap you via cgroups (steps 6–7), QoS decides who dies under node pressure (step 8).**

---

## Exact syntax breakdown

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

```
resources:
  requests:               ← GUARANTEED minimum; used by the SCHEDULER to place the pod
    cpu: "250m"           ← 250 millicores = 0.25 of one CPU core.
    │    │                    1000m = 1 full core. Becomes cpu.weight (proportional share).
    │    └─ scheduler subtracts this from the node's allocatable CPU
    memory: "256Mi"       ← 256 mebibytes (256 × 1024² bytes). Guaranteed & scheduler-reserved.
    │                         Mi = mebibyte (1024²); M = megabyte (1000²). Know the difference!
    │
  limits:                 ← MAXIMUM allowed; enforced by the KERNEL via cgroups
    cpu: "500m"           ← hard cap. Exceed → THROTTLED (frozen till next 100ms period). No kill.
    │                         Becomes cpu.max = "50000 100000".
    memory: "512Mi"       ← hard cap. Exceed → OOMKilled (SIGKILL, exit 137). Becomes memory.max.
    └─ if you set a limit but NO request, K8s sets request = limit automatically.
```

**CPU unit breakdown:**

```
cpu: "1"     = 1 whole core          = 1000m
cpu: "500m"  = 0.5 core = 50ms/100ms period
cpu: "250m"  = 0.25 core
cpu: "2"     = 2 whole cores (needs a node with ≥2 cores free to REQUEST)
              │
              └─ "m" = millicore = 1/1000 of a core. CPU is COMPRESSIBLE → throttled, not killed.
```

**Memory unit breakdown (the classic footgun):**

```
memory: "512Mi"  = 512 × 1024 × 1024  = 536,870,912 bytes   (mebibyte, power of 1024)
memory: "512M"   = 512 × 1000 × 1000  = 512,000,000 bytes   (megabyte, power of 1000)
                   │
                   └─ Mi/Gi/Ki are what you almost always want. Using M/G silently gives you ~5% less.
                      Memory is INCOMPRESSIBLE → exceed the limit = OOMKilled, not slowed.
```

**The QoS classes** (you don't write this field — Kubernetes computes it):

```
Guaranteed  │ EVERY container has limits == requests for BOTH cpu AND memory.
            │   Last to be evicted. Best for critical stateful apps (a database).
Burstable   │ At least one request set, but not fully equal to limits (the common case).
            │   Can burst above requests up to limits; evicted after BestEffort.
BestEffort  │ NO requests and NO limits on any container.
            │   FIRST to be OOM-killed / evicted under node pressure. Only for throwaway jobs.
```

---

## Example 1 — basic

`orders-api` with sensible Burstable resources, every line explained.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: orders-api
  namespace: orders
spec:
  containers:
    - name: orders-api
      image: orders-api:1.5.0
      resources:
        requests:
          cpu: "250m"        # guaranteed 1/4 core → scheduler finds a node with this free
          memory: "256Mi"    # guaranteed 256Mi → reserved on the node for this pod
        limits:
          cpu: "1000m"       # may burst to a full core when the node has spare CPU
          memory: "512Mi"    # hard ceiling — cross it and the kernel OOMKills this container
```

This is **Burstable** QoS: requests are set, limits are higher than requests. Why this shape is good for a web API:

- **CPU: low request, high limit.** `orders-api` is idle most of the time but spikes on traffic bursts. A low request (250m) packs it densely and cheaply; a high limit (1000m) lets it grab spare cores during a spike so latency stays low. CPU is compressible, so a high limit is safe — worst case it's throttled, never killed.
- **Memory: request close to limit.** Node.js heap usage is fairly steady. Set the request near real usage (256Mi) and the limit with headroom (512Mi) for GC spikes. **Never set the memory limit tighter than your app's real peak**, or you get OOMKilled.

The rule of thumb this teaches: **CPU limit can be generous; memory limit must be honest.**

---

## Example 2 — production scenario

**The situation.** `orders-api` latency mysteriously spikes to 900ms p99 under moderate load, but CPU dashboards show the pod using "only 48%." No errors, no OOMKills, no restarts. The team is baffled — "we're not even using the CPU we have."

The manifest:

```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }   # ← CPU limit == request, and it's low
```

**What's really happening.** The "48% CPU" the dashboard shows is an *average over a whole second*. But CPU quota is enforced in **100ms windows**. Under a burst, `orders-api` uses its entire 50ms quota in the first 30ms of a window, then gets **frozen for the remaining 70ms**. Averaged over the second it "looks" like 48%, but in reality it's being repeatedly slammed to a halt. Proof:

```bash
$ kubectl -n orders exec orders-api-xxx -- cat /sys/fs/cgroup/cpu.stat
nr_periods     10000
nr_throttled   6200          ← throttled in 62% of all 100ms windows!
throttled_usec 305000000     ← ~305 seconds spent FROZEN — that IS your latency
```

62% of windows throttled means the app is frozen most of the time it's trying to serve requests. The low CPU **limit** is strangling it even though average utilization looks fine. This is the invisible CPU throttling trap.

**The fix.** Keep the request modest (for good packing) but raise the limit so bursts aren't strangled — or drop the CPU limit entirely (common advice for latency-sensitive services):

```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }   # still guaranteed half a core
  limits:   { memory: "512Mi" }                 # NO cpu limit → can use any spare node CPU
  #                                                memory limit stays (memory MUST be capped)
```

Now `orders-api` bursts freely into idle node CPU during spikes; `nr_throttled` drops to near zero and p99 falls back to 50ms. Note we **kept the memory limit** — you always cap memory (incompressible, protects the node), but a CPU limit on a latency-sensitive service often does more harm than good. If you must keep a CPU limit for multi-tenant fairness, set it well above the p99 burst, not at the average.

**The deeper lesson:** memory and CPU limits have opposite risk profiles. A too-low **memory** limit kills you loudly (OOMKilled, visible). A too-low **CPU** limit strangles you silently (throttling, invisible). Always monitor `cpu.stat`'s `nr_throttled`.

---

## Common mistakes

**Mistake 1 — No memory limit at all (or BestEffort pods).**

```yaml
# resources: {}   ← nothing set
```

Under a leak or spike, `orders-api` grows unbounded, the *node* runs out of memory, and the kubelet starts evicting pods — often innocent bystanders. And because it's BestEffort QoS, `orders-api` itself is first in line to be killed.

```
$ kubectl -n orders get pod orders-api-xxx -o jsonpath='{.status.reason}'
Evicted
$ kubectl -n orders describe pod orders-api-xxx | grep Message
The node was low on resource: memory. Container orders-api was using 1900Mi, exceeds request of 0.
```

Root cause: no request = no protection, no limit = no ceiling. Right way: always set memory requests+limits.

**Mistake 2 — Confusing `Mi` and `M` (or `Gi` and `G`).**

```yaml
limits: { memory: "512M" }   # meant 512 mebibytes, got 512 megabytes = ~488Mi
```

Root cause: `M` is 1000², `Mi` is 1024². Your app that peaks at 500Mi now OOMKills because the real cap is only ~488Mi. Right way: use `Mi`/`Gi` unless you truly mean SI units.

**Mistake 3 — Setting a CPU limit equal to the request "to be safe."**

```yaml
requests: { cpu: "500m" }
limits:   { cpu: "500m" }   # kills all burst headroom
```

```
$ kubectl exec orders-api-xxx -- grep throttled /sys/fs/cgroup/cpu.stat
nr_throttled 6200           ← silent latency destroyer
```

Root cause: the app can never use idle node CPU, so bursts get throttled and latency spikes with zero visible errors. Right way: request = steady-state, limit = generous headroom (or no CPU limit for latency-sensitive services).

**Mistake 4 — Reading exit code 137 as "app crashed."**

```
State: Terminated
Reason: OOMKilled
Exit Code: 137
```

Root cause: `137 = 128 + 9` (SIGKILL). The kernel killed it for exceeding `memory.max`; your app code did nothing wrong and never got a chance to log. Right way: recognize 137/OOMKilled as a memory-limit problem — raise the limit or fix the leak; don't hunt for a bug in application logs that won't be there.

**Mistake 5 — Requests way higher than real usage (wasting the cluster).**

```yaml
requests: { cpu: "2", memory: "2Gi" }   # app really uses 200m / 300Mi
```

Root cause: the scheduler *reserves* the full request even though the app never uses it, so a node that could hold 10 pods holds 1. You pay for idle capacity. Right way: set requests near observed p50–p90 usage (use `kubectl top pod` / metrics), not a guess.

---

## Hands-on proof

Run on any cluster to see OOMKill and throttling happen for real.

```bash
kubectl create namespace orders 2>/dev/null

# --- PROOF 1: memory limit → OOMKilled ---
cat <<'EOF' | kubectl -n orders apply -f -
apiVersion: v1
kind: Pod
metadata: { name: oom-demo }
spec:
  containers:
    - name: hog
      image: polinux/stress
      resources:
        limits: { memory: "64Mi" }        # tiny cap
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]  # try to grab 150M
  restartPolicy: Never
EOF

sleep 8
# Proof: the kernel OOM-killed it for exceeding 64Mi.
kubectl -n orders get pod oom-demo
# NAME       READY   STATUS      RESTARTS   AGE
# oom-demo   0/1     OOMKilled   0          8s
kubectl -n orders get pod oom-demo -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{"\n"}'
# OOMKilled
kubectl -n orders get pod oom-demo -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}{"\n"}'
# 137     ← 128 + SIGKILL(9)

# --- PROOF 2: CPU limit → throttling (no kill) ---
cat <<'EOF' | kubectl -n orders apply -f -
apiVersion: v1
kind: Pod
metadata: { name: cpu-demo }
spec:
  containers:
    - name: burn
      image: polinux/stress
      resources:
        limits: { cpu: "200m" }           # 20ms per 100ms window
      command: ["stress"]
      args: ["--cpu", "4"]                # try to burn 4 cores → will be throttled hard
EOF

sleep 15
# Proof: it's STILL RUNNING (not killed) but heavily throttled.
kubectl -n orders get pod cpu-demo
# cpu-demo   1/1   Running   0   15s        ← alive!
kubectl -n orders exec cpu-demo -- cat /sys/fs/cgroup/cpu.stat
# nr_throttled <big number>                 ← proof it was frozen repeatedly
# throttled_usec <big number>               ← total time spent frozen

# --- Show QoS classes Kubernetes assigned ---
kubectl -n orders get pod oom-demo cpu-demo -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.qosClass}{"\n"}{end}'
# oom-demo   Burstable    (has a limit, no request → Burstable... actually BestEffort-ish)
# cpu-demo   Burstable

# Clean up
kubectl -n orders delete pod oom-demo cpu-demo --force 2>/dev/null
```

What you verified: a memory limit **kills** (Proof 1: OOMKilled, exit 137); a CPU limit **only slows** (Proof 2: Running but high `nr_throttled`). Same idea, opposite consequence.

---

## Practice exercises

### Exercise 1 — easy
Deploy an `nginx:alpine` pod with `requests: {cpu: 100m, memory: 64Mi}` and `limits: {cpu: 200m, memory: 128Mi}`. Run `kubectl get pod <name> -o jsonpath='{.status.qosClass}'`. What QoS class did it get, and why isn't it Guaranteed?

### Exercise 2 — medium
Create a pod with `limits.memory: 100Mi` running `polinux/stress` trying to allocate 200M. Confirm it's `OOMKilled` with exit code 137. Then raise the limit to `300Mi` and confirm it now stays Running. Read `/sys/fs/cgroup/memory.max` inside the surviving pod and verify it matches your YAML (in bytes).

### Exercise 3 — hard (production simulation)
Reproduce the Example 2 throttling incident. Run a CPU-bound workload (`stress --cpu 2`) with `limits.cpu: 300m`. Every 5 seconds, read `nr_throttled` and `throttled_usec` from `/sys/fs/cgroup/cpu.stat` and watch them climb while the pod stays `Running`. Then remove the CPU limit, restart the pod, and confirm `nr_throttled` stops climbing. Write down: why did CPU pressure never produce an OOMKill, and why did memory pressure in Exercise 2 always produce one?

---

## Mental model checkpoint

Answer from memory:

1. The scheduler uses ______ (requests / limits) to place a pod and completely ignores the other. Why?
2. Exceeding a memory limit causes ______; exceeding a CPU limit causes ______. Why the difference at the kernel level?
3. What exit code and status does an OOMKill produce, and why won't your app logs explain it?
4. Name the three QoS classes and the order in which they're evicted under node memory pressure.
5. Which cgroup file holds the memory ceiling, and which holds the CPU quota/period pair?
6. Why is a too-low CPU limit "invisible" while a too-low memory limit is "loud"?
7. `512M` vs `512Mi` — which is bigger and by roughly how much?

---

## Quick reference card

| Field / concept | What it does | Key detail |
|---|---|---|
| `requests.cpu/memory` | Guaranteed minimum; scheduler placement | Only thing the scheduler reads; becomes `cpu.weight` |
| `limits.cpu` | Hard CPU cap | Exceed → **throttled** (frozen), never killed. `cpu.max` |
| `limits.memory` | Hard memory cap | Exceed → **OOMKilled**, exit 137. `memory.max` |
| CPU unit `m` | Millicore, 1000m = 1 core | CPU is compressible → throttling |
| Memory `Mi` vs `M` | 1024² vs 1000² bytes | Use `Mi`/`Gi`; `M`/`G` gives ~5% less |
| QoS **Guaranteed** | limits == requests, cpu+mem, all containers | Evicted LAST — use for critical pods |
| QoS **Burstable** | some requests, limits higher | The common web-app case |
| QoS **BestEffort** | no requests, no limits | Evicted FIRST — avoid for real services |
| `OOMKilled` / 137 | Kernel killed for memory | Raise limit or fix leak, not an app bug |
| `cpu.stat` `nr_throttled` | Times the pod was frozen | Your hidden latency source |
| Eviction | Kubelet removes pods under node pressure | Ranked by QoS + over-request amount |

---

## When would I use this at work?

1. **Right-sizing a new service.** Before shipping `orders-api`, you run it under load, read `kubectl top pod` and `cpu.stat`, and set requests near p90 usage and a memory limit above true peak — so it schedules densely but never OOMKills.

2. **Debugging "it's slow but nothing's wrong."** Latency spikes with green CPU dashboards → you check `cpu.stat nr_throttled`, discover throttling, and remove/raise the CPU limit. A 5-minute fix that looks like a deep performance mystery.

3. **Protecting a critical pod from noisy neighbors.** You make the payments database Guaranteed QoS (limits == requests) so that when a batch job floods the node with memory, the kubelet evicts the batch job (BestEffort/Burstable) first and never touches payments.

---

## Connected topics

- **Study before:**
  - **Topic 01 / 02 — Containers & Linux Foundations**: cgroups are the kernel feature doing all the enforcement here.
  - **Topic 25 — Resource Limits (Docker)**: the single-container Docker version of `--memory`/`--cpus`; K8s does the same via YAML.
  - **Topic 29 — Kubernetes Architecture**: the scheduler and kubelet are the actors in this topic.
- **Study after:**
  - **Topic 43 — Health Checks**: CPU throttling makes liveness probes time out — the two topics interact directly.
  - **Topic 45 — Horizontal Pod Autoscaler**: HPA scales on CPU/memory *utilization relative to requests* — so your requests directly drive autoscaling.
