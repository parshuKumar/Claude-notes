# 45 — Horizontal Pod Autoscaler
## Section: Kubernetes in Practice

## ELI5 — The Simple Analogy

Imagine a pizza shop. On a quiet Tuesday afternoon, **one** chef in the kitchen is plenty. But on Friday night, orders flood in and one chef can't keep up — pizzas come out slow, customers get angry.

A smart manager watches the kitchen. When the chefs are all working flat-out (busy), she calls in **more chefs**. When it's quiet again and chefs are standing around, she sends some home so she isn't paying for idle hands.

The **Horizontal Pod Autoscaler (HPA)** is that manager. It watches how hard your Pods are working (CPU, memory, or custom numbers like "requests per second"). When they're maxed out, it **adds more Pods** (scales *out*). When they're idle, it **removes Pods** (scales *in*) — but never below a floor you set, and never above a ceiling.

"Horizontal" = more copies of the same thing (more chefs). This is different from "vertical" = making one chef stronger (more CPU/RAM per Pod, which is a different tool called VPA).

## The Linux kernel feature underneath

HPA itself is a *control-plane* object — it doesn't touch the kernel directly. But the **numbers it reacts to come straight from the Linux kernel**, so you can't understand HPA without knowing where "CPU usage" actually comes from.

Remember from **Topic 02** and **Topic 44** that every container runs inside a **cgroup** (control group). The kernel keeps live accounting for each cgroup in a virtual filesystem:

```
/sys/fs/cgroup/                          ← cgroup v2 unified hierarchy
└── kubepods.slice/.../<container>/
    ├── cpu.stat        ← usage_usec: total CPU microseconds this cgroup has used
    ├── memory.current  ← current bytes of memory in use
    └── cpu.max         ← the limit (quota + period) — see Topic 44
```

The chain of who reads what:

1. The **kernel** increments `cpu.stat`'s `usage_usec` every time the container's processes run on a core.
2. **cAdvisor** (built into the **kubelet** on every node — Topic 29) reads those cgroup files ~every 10–15s.
3. **metrics-server** scrapes each kubelet's `/metrics/resource` endpoint and keeps the latest value in memory.
4. The **HPA controller** (inside the controller-manager — Topic 30) asks the `metrics.k8s.io` API for "current CPU of these Pods", does math, and updates the Deployment's replica count.

So "the Pod is at 80% CPU" is ultimately a number the **kernel wrote to `cpu.stat`**. HPA is just a feedback loop wrapped around kernel accounting.

## What is this?

The Horizontal Pod Autoscaler is a Kubernetes controller that automatically changes the **number of replicas** of a Deployment (or ReplicaSet/StatefulSet) based on observed metrics. You give it a target — e.g. "keep average CPU at 70%" — plus a `minReplicas` and `maxReplicas`, and it continuously adjusts the replica count to hit that target.

It is a standard API object (`kind: HorizontalPodAutoscaler`) and it *drives* the Deployment; it does not replace it.

## Why does it matter for a backend developer?

Traffic to `orders-api` is not constant. It spikes at lunch, on payday, during a marketing push. Without autoscaling you have two bad choices:

- **Provision for peak** — run 20 replicas 24/7. You pay for 20 even at 3am when 2 would do. Money burned.
- **Provision for average** — run 3 replicas. The lunch spike overwhelms them, response times blow up, requests time out, customers see errors.

HPA gives you the third option: **run 3 when quiet, 20 when slammed, automatically.** You pay for what you use and you survive spikes without a human getting paged at midnight to `kubectl scale`.

Critically, HPA depends on things you learned earlier: it **only works if your Pods declare CPU/memory `requests`** (Topic 44), because "70% CPU" means "70% of the *request*", not 70% of the node. Miss that and HPA silently does nothing.

## The physical reality

There is no file on a node called "hpa". What physically exists:

1. **An HPA object in etcd** — desired policy: target metric, min, max, current status.
2. **metrics-server** — one Deployment (usually in `kube-system`) holding the latest CPU/memory reading for every Pod **in memory only** (it is not durable; it's a live cache).
3. **The kernel cgroup counters** on each node (the source of truth for usage).
4. **The Deployment's `.spec.replicas` field in etcd** — the thing HPA actually writes to.

You can see the live state:

```bash
kubectl get hpa orders-api -n orders
# NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS
# orders-api   Deployment/orders-api   43%/70% (cpu)   3         20        5
#                                      ▲ current/target        ▲ right now HPA chose 5
```

`43%/70%` literally means: right now the average Pod is using 43% of its CPU *request*, and the goal is 70%. Since 43 < 70, there's headroom, so HPA won't add Pods (and may remove some).

## How it works — step by step

The HPA controller runs a loop every **15 seconds** by default (`--horizontal-pod-autoscaler-sync-period`). One cycle:

1. **Read desired state.** Fetch the HPA object from the API server: target = 70% CPU, min=3, max=20, scaleTargetRef = Deployment/orders-api.
2. **Find the Pods.** Follow the Deployment → its ReplicaSet → its Pods via label selectors (Topic 38). Say 5 Pods are Running.
3. **Get current metric.** Query `metrics.k8s.io` for each Pod's CPU. metrics-server returns, e.g., an average of 90% (of request) across the 5 Pods.
4. **Run the algorithm.** The core formula:

   ```
   desiredReplicas = ceil( currentReplicas × ( currentMetric / targetMetric ) )
   desiredReplicas = ceil( 5 × ( 90 / 70 ) )
                   = ceil( 5 × 1.285 )
                   = ceil( 6.43 )
                   = 7
   ```

5. **Apply tolerance.** If the ratio is within ±10% of 1.0 (`currentMetric/targetMetric` between 0.9 and 1.1), HPA does **nothing** — this prevents flapping. Here the ratio is 1.28, well outside, so it acts.
6. **Clamp to bounds.** `7` is between min(3) and max(20), so it's allowed. If the math said 25, it'd be clamped to 20.
7. **Check scaling behavior / stabilization window.** For scale-*down*, HPA looks back over the last **5 minutes** (default `stabilizationWindowSeconds`) and uses the *highest* recommendation in that window, so a brief dip doesn't prematurely kill Pods. Scale-*up* has no stabilization window by default (react fast).
8. **Write the new replica count.** HPA does the equivalent of `kubectl scale deployment orders-api --replicas=7`. It **patches the Deployment's `.spec.replicas`**.
9. **The Deployment controller takes over** (Topic 34) — it updates the ReplicaSet, which creates 2 new Pods. The scheduler places them (Topic 44), kubelet starts them.
10. **Loop.** 15 seconds later, re-measure. Now with 7 Pods the load spreads out, CPU drops toward 70%, and the system settles.

Note the clean separation: HPA only ever touches **one number** — `.spec.replicas`. Everything else (creating Pods, scheduling, starting) is done by controllers you already know. This is the reconciliation loop from **Topic 30** stacked on itself.

## Exact syntax breakdown

A full HPA manifest (the modern `autoscaling/v2` API):

```yaml
apiVersion: autoscaling/v2          # v2 supports memory + custom + multiple metrics (v1 = CPU only)
kind: HorizontalPodAutoscaler
metadata:
  name: orders-api
  namespace: orders
spec:
  scaleTargetRef:                   # WHAT to scale
    apiVersion: apps/v1
    kind: Deployment
    name: orders-api
  minReplicas: 3                    # never go below this (survive baseline traffic)
  maxReplicas: 20                   # never go above this (cost + cluster capacity ceiling)
  metrics:                          # WHAT to measure (can list several; HPA uses the max recommendation)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization         # percentage of the Pod's CPU *request*
          averageUtilization: 70    # goal: 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:                         # OPTIONAL: fine-tune how aggressively it scales
    scaleUp:
      stabilizationWindowSeconds: 0     # scale up instantly
      policies:
        - type: Percent
          value: 100                    # at most double the pods...
          periodSeconds: 15             # ...every 15s
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min of low load before shrinking (avoid flapping)
      policies:
        - type: Pods
          value: 1                      # remove at most 1 pod...
          periodSeconds: 60             # ...per minute (drain gently)
```

Field-by-field on the two lines people get wrong:

```
          averageUtilization: 70
          │                   │
          │                   └── 70% of the container's cpu REQUEST (Topic 44), NOT of the node
          └── "Utilization" = percentage. Use "AverageValue" for absolute (e.g. 500m) instead
```

```
  minReplicas: 3          maxReplicas: 20
  │                       │
  │                       └── hard ceiling — HPA will NEVER exceed this even if CPU is 100%
  └── hard floor — even at 0% CPU you keep 3 (for HA across nodes/zones)
```

The imperative shortcut (great for a quick test, generates a v1 HPA):

```
kubectl autoscale deployment orders-api --cpu-percent=70 --min=3 --max=20 -n orders
│       │         │          │           │               │       │        │
│       │         │          │           │               │       │        └── namespace
│       │         │          │           │               │       └── maxReplicas
│       │         │          │           │               └── minReplicas
│       │         │          │           └── target: 70% of CPU request
│       │         │          └── target Deployment name
│       │         └── the resource kind to scale
│       └── subcommand: create an HPA
└── kubectl
```

## Example 1 — basic

Autoscale `orders-api` on CPU. **Prerequisite: the Deployment must set CPU requests**, or HPA has no denominator.

```yaml
# orders-api Deployment (excerpt) — the requests are what make HPA possible
spec:
  template:
    spec:
      containers:
        - name: orders-api
          image: ghcr.io/acme/orders-api:1.4.2
          resources:
            requests:
              cpu: 250m           # HPA's "100%" == 250 millicores. 70% == 175m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders-api
  namespace: orders
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa orders-api -n orders --watch     # watch TARGETS and REPLICAS change live
```

## Example 2 — production scenario

**Context:** It's Black Friday. Your team runs `orders-api` with `minReplicas: 3, maxReplicas: 20`, target 70% CPU. Marketing sends a push notification at 10:00. Within seconds, 50,000 users open the app.

**What you'd watch happen:**

```
10:00:00  REPLICAS 3   TARGETS 45%/70%   (calm)
10:00:15  REPLICAS 3   TARGETS 140%/70%  (spike hits; CPU pegged, requests queueing)
10:00:16  HPA math: ceil(3 × 140/70) = 6  →  scaleUp policy allows doubling → go to 6
10:00:30  REPLICAS 6   TARGETS 120%/70%  (still hot, new pods warming up)
10:00:45  HPA: ceil(6 × 120/70) = 11  →  go to 11
10:01:15  REPLICAS 11  TARGETS 80%/70%
10:01:45  REPLICAS 13  TARGETS 68%/70%   (settled — inside the ±10% tolerance, stop)
...spike ends at 10:20...
10:20:00  TARGETS 30%/70%  (load gone, but scaleDown stabilizationWindow=300s holds)
10:25:00  HPA finally scales down 1 pod/min back toward 3
```

**The subtle production lessons baked in here:**

- **Cold-start lag.** New Pods take time to become Ready (pull image, boot Node, pass readiness probe — Topic 43). During that ~30–60s the existing Pods stay overloaded. Mitigate with a small `startupProbe`, a warm base image, and `minReplicas` high enough to absorb the *initial* burst.
- **Downstream saturation.** If `orders-api` scales from 3 → 20 Pods but they all hammer the **same Postgres** (Topic 39), you just moved the bottleneck. Autoscaling the stateless tier can DoS your database via connection exhaustion. Fix with a connection pooler (PgBouncer) and cap `maxReplicas` to what the DB can take.
- **Cluster capacity.** HPA adds Pods; it does **not** add nodes. If there's no room, new Pods sit `Pending`. Pair HPA with the **Cluster Autoscaler** (adds nodes) for true elasticity.
- **Slow scale-down on purpose.** The 5-minute stabilization window means you keep paying for extra Pods a few minutes after the spike. That's deliberate — cheaper than flapping up and down every 15s.

## Common mistakes

**Mistake 1 — No resource requests set.**

```
$ kubectl get hpa orders-api -n orders
NAME         REFERENCE               TARGETS              ...
orders-api   Deployment/orders-api   <unknown>/70% (cpu)  ...
```

**Root cause:** `Utilization` is a percentage *of the request*. With no `resources.requests.cpu` on the container, there's no denominator, so metrics-server/HPA reports `<unknown>` and **never scales**. → **Fix:** always set `requests.cpu` (and memory) on the container. This is the #1 reason HPA "doesn't work".

**Mistake 2 — metrics-server not installed.**

```
$ kubectl top pods -n orders
error: Metrics API not available
$ kubectl describe hpa orders-api -n orders
... FailedGetResourceMetric  unable to get metrics for resource cpu: no metrics returned
```

**Root cause:** HPA on CPU/memory *requires* the metrics-server add-on; it is not built in. → **Fix:** `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` (on kind/minikube you may also need `--kubelet-insecure-tls`). Verify with `kubectl top pods`.

**Mistake 3 — minReplicas: 1 in production.**

Setting `minReplicas: 1` means during quiet hours you have a single Pod. When that node reboots or the Pod restarts, `orders-api` is **fully down** for the restart window. → **Fix:** `minReplicas: 2` or `3` so you always survive a single-Pod/node failure. HPA is for scaling, not for availability floors — but the floor is where you set your availability.

**Mistake 4 — Scaling on CPU for an I/O-bound app.**

`orders-api` mostly waits on Postgres/Redis; its CPU stays at 20% even while requests pile up because Pods are *blocked on I/O*, not burning CPU. HPA sees "low CPU" and never scales, while latency climbs. → **Fix:** scale on a metric that reflects the real bottleneck — requests-per-second or queue depth via **custom metrics** (`type: Pods` / `type: External` with Prometheus Adapter). CPU is the wrong signal for I/O-bound services.

**Mistake 5 — HPA and manual `kubectl scale` fighting.**

You `kubectl scale deployment orders-api --replicas=10`; HPA immediately overrides it back to its computed value. → **Fix:** don't hand-scale a Deployment that has an HPA. If you need a manual floor, change `minReplicas`. Also never put `replicas:` in your Git-managed Deployment manifest if an HPA owns it — GitOps sync (Topic 52) and HPA will thrash each other.

## Hands-on proof

Run these on a cluster with metrics-server (minikube: `minikube addons enable metrics-server`).

```bash
# 1. Deploy a CPU-burnable app with requests set
kubectl create deployment php-apache --image=registry.k8s.io/hpa-example
kubectl set resources deployment php-apache --requests=cpu=200m
kubectl expose deployment php-apache --port=80

# 2. Create the HPA
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# 3. Watch it (leave this running in one terminal)
kubectl get hpa php-apache --watch

# 4. In ANOTHER terminal, generate load
kubectl run -it --rm load --image=busybox -- /bin/sh -c \
  "while true; do wget -q -O- http://php-apache; done"

# 5. Watch TARGETS climb past 50% and REPLICAS grow 1 → several.
#    Ctrl-C the load generator and watch REPLICAS shrink back (after ~5 min).

# PROVE the metric source is real kernel accounting:
kubectl top pods                       # live CPU/mem from metrics-server → cAdvisor → cgroups
kubectl describe hpa php-apache        # see events: "New size: 4; reason: cpu resource utilization above target"
```

## Practice exercises

### Exercise 1 — easy
Deploy any Deployment with `resources.requests.cpu: 100m`. Create an HPA (min=2, max=6, cpu 60%). Run `kubectl get hpa -w`. Confirm the `TARGETS` column shows a real percentage (not `<unknown>`). If it shows `<unknown>`, diagnose whether it's the requests or metrics-server.

### Exercise 2 — medium
Using the `php-apache` demo above, generate load and record the replica count every 15s until it stabilizes. Then plot (on paper) replicas-over-time. Explain: why did it *not* jump straight to max? Why did scale-down take much longer than scale-up? Reference the algorithm and the stabilization window.

### Exercise 3 — hard (production simulation)
Write an `autoscaling/v2` HPA for `orders-api` that scales on **both** CPU (70%) and memory (80%), with a `behavior` block that: scales up at most 100%/15s, scales down at most 1 Pod/min with a 300s stabilization window, min=3, max=15. Then deliberately break it by removing the CPU `requests` from the Deployment and describe exactly what `kubectl describe hpa` reports and why. Fix it and confirm.

## Mental model checkpoint

1. What is the exact formula HPA uses to compute desired replicas?
2. "70% CPU utilization" is 70% of *what* — the node, the limit, or the request? Why does that matter?
3. Which two add-ons/settings must exist or HPA silently does nothing?
4. Why does scale-down have a stabilization window but scale-up (by default) doesn't?
5. What single field does HPA actually write, and which controller does the rest?
6. Why is CPU a poor scaling metric for an I/O-bound `orders-api`, and what would you use instead?
7. HPA added 15 Pods but they're all `Pending`. What's missing, and which other autoscaler fixes it?

## Quick reference card

| Command / field | What it does | Key detail |
|---|---|---|
| `kubectl autoscale deploy X --cpu-percent=70 --min=3 --max=20` | Create a v1 HPA imperatively | Quick; CPU-only |
| `kubectl get hpa` | List HPAs with current/target | `<unknown>` = no requests or no metrics-server |
| `kubectl describe hpa X` | Events + scaling decisions | Shows "New size / reason" |
| `apiVersion: autoscaling/v2` | Modern HPA API | Needed for memory/custom/behavior |
| `minReplicas` / `maxReplicas` | Hard floor / ceiling | Floor is your availability baseline |
| `averageUtilization` | % of resource **request** | Requires `resources.requests` set |
| `type: Utilization` vs `AverageValue` | Percentage vs absolute value | Use AverageValue for custom metrics |
| `behavior.scaleDown.stabilizationWindowSeconds` | Anti-flapping delay before shrinking | Default 300s |
| metrics-server | Supplies CPU/mem metrics | Not built in; required |
| `kubectl top pods` | Live usage from cgroups | Quick health check for the metrics pipeline |

## When would I use this at work?

1. **Traffic-driven web/API tier.** `orders-api` sees daily peaks and troughs — HPA on CPU (or better, requests/sec) keeps latency flat during spikes and cost low overnight, with no human intervention.
2. **Queue/worker autoscaling.** A background worker consuming a job queue: scale on **external metric** = queue depth (via KEDA or Prometheus Adapter), so the number of workers tracks backlog, draining spikes fast then scaling to `minReplicas`.
3. **Cost control with a safety ceiling.** Finance wants predictable spend. You set `maxReplicas` to a budget-derived cap and `minReplicas` for HA, giving elasticity within guardrails — and pair it with Cluster Autoscaler so nodes follow Pods.

## Connected topics

- **Study before:** Topic 34 (Deployments — HPA drives one), Topic 44 (Resource Requests & Limits — HPA's denominator), Topic 30 (Reconciliation Loop — HPA is one), Topic 38 (Labels & Selectors — how HPA finds Pods).
- **Study after:** Topic 46 (Rolling Updates — how HPA and rollouts interact), Topic 51 (Observability — where custom scaling metrics come from), Topic 52 (CI/CD/GitOps — don't let Git and HPA fight over `replicas`).
