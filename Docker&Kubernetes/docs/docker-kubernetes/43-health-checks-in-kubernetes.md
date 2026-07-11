# 43 — Health Checks in Kubernetes
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Imagine a shop with three different checks the manager does on each new worker.

1. **"Are you even alive?"** — The manager pokes the worker every few seconds. If the worker stops responding (fell asleep, fainted), the manager doesn't wait around — they **fire that worker and hire a fresh one**. That's the **liveness** check.

2. **"Are you ready to serve customers?"** — A worker can be alive but still not ready: they just clocked in, they're tying their apron, warming up the till. The manager does NOT send customers to them yet. Only once they raise their hand "I'm ready" does the manager **let customers walk up to their counter**. That's the **readiness** check.

3. **"Are you done with your slow first-day setup?"** — A brand new worker on their very first day needs extra time to boot up: read the manual, log into ten systems. During this one-time slow start, the manager gives them lots of patience and does NOT poke them with the "are you alive?" stick yet. That's the **startup** check.

The key insight a 10-year-old needs: **firing a worker (liveness) and not sending customers to a worker (readiness) are two completely different actions.** Mixing them up is the #1 way people break their own app. If you fire a worker just because they're briefly busy, you get a shop where the manager keeps firing and re-hiring forever and nobody serves customers.

---

## The Linux/Kubernetes mechanism underneath

There is no "health" primitive in the Linux kernel. Health checking in Kubernetes is done entirely by the **kubelet** — the agent process running on every worker node (remember from **Topic 29 — Kubernetes Architecture**). The kubelet is the thing that actually talks to the container runtime (containerd/CRI) and owns the pods on its node.

Here is what physically happens:

- The kubelet keeps a **probe worker (a goroutine) per container per probe type**. Each worker runs on a timer (`periodSeconds`).
- For an **HTTP probe**, the kubelet itself opens a TCP socket to the pod's IP and sends a raw `GET` request — *from inside the node*, not through a Service. It is a plain Go `net/http` client call.
- For a **TCP probe**, the kubelet does a `connect()` syscall to `podIP:port` and immediately closes it. Success = the 3-way TCP handshake (SYN → SYN/ACK → ACK) completed.
- For an **exec probe**, the kubelet calls the CRI runtime to run a command *inside the container's namespaces* (it enters the container's mount + pid + net namespaces — the same illusion from **Topic 01**) and checks the process **exit code**. `0` = healthy, anything else = failure.

The consequences are also kubelet actions:

- **Liveness fails** → kubelet asks the runtime to **kill the container** (`SIGTERM`, then `SIGKILL` after the grace period) and restart it *in the same pod*. The pod's `RESTARTS` counter goes up.
- **Readiness fails** → kubelet updates the pod's status. The **EndpointSlice controller** (in the control plane) notices the pod is `Ready: false`, and **removes the pod's IP from the Service's EndpointSlice** (Topic 35). kube-proxy then removes the matching iptables/IPVS rule, so no new traffic is routed to it. The container is **NOT killed**.
- **Startup fails (until it passes once)** → kubelet **holds off liveness and readiness probes entirely**, and if the startup probe exhausts its failure budget, kills the container.

So health checks are a control loop living inside the kubelet, plus a routing consequence living in the control plane. No kernel magic — just timers, sockets, exit codes, and etcd updates.

---

## What is this?

Health checks (called **probes**) are small tests Kubernetes runs against your container to decide two independent questions: *should I restart this container?* (liveness) and *should I send it traffic?* (readiness), plus a special helper for slow-starting apps (startup). Each probe can be an HTTP request, a raw TCP connection, or a command run inside the container.

They are the mechanism behind Kubernetes' famous **self-healing**. Without them, Kubernetes only knows if your *process* is alive, not whether your *app* actually works.

---

## Why does it matter for a backend developer?

Your `orders-api` process can be "running" (PID exists) while being completely useless:

- It's deadlocked waiting on a lock that will never release.
- Its event loop is blocked and it can't answer any HTTP request.
- It booted but hasn't finished connecting to Postgres yet, so every request 500s.
- It lost its Redis connection and every cache read throws.

By default, Kubernetes only checks if the **main process is running**. A deadlocked Node process is still "running" — so without a liveness probe, Kubernetes happily leaves your frozen `orders-api` in the load balancer, sending it real customer traffic, forever. Users get timeouts; your dashboards look "green."

And the flip side is worse if you get it wrong: a *misconfigured* liveness probe will **kill a perfectly healthy pod** that was just slow for a second, causing a **CrashLoopBackOff** where Kubernetes restarts your app over and over. A misconfigured readiness probe can either send traffic to a not-yet-ready pod (users see errors during every deploy) or never mark any pod ready (100% outage while everything looks "up").

This is the single most common way backend engineers cause their own production incidents in Kubernetes. Getting probes right is not optional polish — it is the difference between self-healing and self-destructing.

---

## The physical reality

When `orders-api` is running with probes configured, here is what actually exists.

**1. The probe config lives in etcd** as part of the pod spec. You can read it back:

```bash
kubectl -n orders get pod orders-api-7d4f-abcde -o jsonpath='{.spec.containers[0].livenessProbe}'
```

**2. The kubelet holds live probe state in memory** on the node. It tracks, per container, consecutive successes/failures and the current result (`Success`/`Failure`/`Unknown`).

**3. The results surface in the pod status** (also in etcd), which you see as:

```
$ kubectl -n orders get pod orders-api-7d4f-abcde
NAME                    READY   STATUS    RESTARTS   AGE
orders-api-7d4f-abcde   1/1     Running   4          22m
                        │                 │
                        │                 └─ RESTARTS=4 → liveness probe killed it 4 times
                        └─ READY 1/1 → readiness probe currently passing (in the Service)
```

**4. Readiness changes the EndpointSlice object** in the control plane:

```bash
kubectl -n orders get endpointslice -l kubernetes.io/service-name=orders-api -o yaml
# Ready pods appear under `endpoints[].conditions.ready: true`
# A not-ready pod's IP is either absent or marked ready: false → kube-proxy won't route to it
```

**5. Probe events are recorded** on the pod, visible via `kubectl describe`:

```
Warning  Unhealthy  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 500
Warning  Unhealthy  kubelet  Readiness probe failed: Get "http://10.244.1.7:3000/ready": context deadline exceeded
```

So the "health" of your pod is: a config in etcd, a set of counters in the kubelet's memory, a status field, and (for readiness) a live edit to the Service's endpoint list.

---

## How it works — step by step

Full trace of a pod starting up with all three probes, `orders-api` style.

1. **Pod scheduled, container created.** The kubelet pulls the image and starts the container. The main process (`node server.js`) becomes PID 1 in the pod (Topic 33).

2. **Startup probe begins first.** If a `startupProbe` is defined, the kubelet runs *only* it. Liveness and readiness are **suspended**. The kubelet hits `GET /health` every `periodSeconds`. Because `orders-api` needs ~40s to run DB migrations and warm caches, the startup probe is configured to allow up to `failureThreshold × periodSeconds` = 30 × 5 = 150s of failures before giving up.

3. **Startup probe succeeds once.** The moment `GET /health` returns 200, the startup probe is marked done and **never runs again**. The kubelet now "unlocks" liveness and readiness.

4. **Readiness probe begins.** The kubelet hits `GET /ready` every `periodSeconds`. `/ready` checks "can I reach Postgres AND Redis right now?" The first time it returns 200, the kubelet marks the container `Ready: true`.

5. **Pod becomes an endpoint.** The EndpointSlice controller sees `Ready: true`, adds `podIP:3000` to the `orders-api` Service's EndpointSlice. kube-proxy programs an iptables rule. **Now, and only now, does customer traffic reach this pod.**

6. **Liveness probe begins (and runs forever).** In parallel, the kubelet hits `GET /health` every `periodSeconds`. As long as it returns 200, nothing happens.

7. **Something breaks.** Say the Node event loop deadlocks. `GET /health` now times out. The kubelet counts consecutive failures.

8. **Liveness failure budget exhausted.** After `failureThreshold` consecutive failures (say 3), the kubelet decides the container is dead. It sends `SIGTERM`, waits `terminationGracePeriodSeconds`, then `SIGKILL`. The container restarts *in the same pod* (same IP, same volumes). `RESTARTS` increments.

9. **Readiness reacts too.** While the container is dead/restarting, its readiness probe also fails, so its IP is pulled from the Service. Traffic stops flowing to it until it passes readiness again — automatically closing the loop.

10. **Repeat, or back off.** If the container keeps failing liveness right after restart, Kubernetes applies exponential backoff (10s, 20s, 40s… capped at 5m) — the `CrashLoopBackOff` state.

The crucial takeaway: **startup gates the other two, readiness gates traffic, liveness gates the process. Three timers, three different consequences.**

---

## Exact syntax breakdown

A container with all three probes:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3
  successThreshold: 1
```

```
livenessProbe:              ← "should I restart this container?"
  httpGet:                  ← probe TYPE: HTTP GET (others: tcpSocket, exec, grpc)
    path: /health           ← URL path the kubelet requests
    port: 3000              ← container port (NOT the Service port)
  │ │
  │ └─ 2xx or 3xx status = success; 4xx/5xx or timeout = failure
  │
  initialDelaySeconds: 0    ← wait this long AFTER container start before the FIRST probe
  │                            (use startupProbe instead of a big value here — see below)
  periodSeconds: 10         ← run the probe every 10 seconds
  timeoutSeconds: 1         ← if no response within 1s, count as a FAILURE
  │                            (default 1s — often too tight; a slow GC pause can trip it)
  failureThreshold: 3       ← 3 consecutive failures in a row → take action (restart)
  │                            so worst-case detection = periodSeconds × failureThreshold = 30s
  successThreshold: 1       ← consecutive successes to be considered healthy again
                               (MUST be 1 for liveness & startup; only readiness may be >1)
```

Every timing field, precisely:

```
initialDelaySeconds  │ Seconds to wait after the container starts before the first probe. Default 0.
periodSeconds        │ How often to run the probe. Default 10. Minimum 1.
timeoutSeconds       │ How long to wait for a response before calling it a failure. Default 1.
failureThreshold     │ Consecutive failures before the probe "gives up" and acts. Default 3.
successThreshold     │ Consecutive successes to flip back to healthy. Default 1. Must be 1
                     │   for liveness/startup. Only readiness can set it higher.
terminationGracePeriodSeconds │ (probe-level, since v1.20) grace period used specifically when
                     │   THIS probe triggers a kill. Overrides the pod-level value for the probe.
```

The three probe **types** (any probe can use exactly one):

```yaml
# 1) HTTP — kubelet sends GET, checks status code (2xx/3xx = ok)
httpGet:
  path: /health
  port: 3000
  httpHeaders:              # optional custom headers
    - name: X-Probe
      value: kubelet
  scheme: HTTP              # or HTTPS (default HTTP)

# 2) TCP — kubelet opens a TCP connection; success = handshake completes
tcpSocket:
  port: 5432               # e.g. is Postgres accepting connections?

# 3) exec — kubelet runs a command INSIDE the container; exit code 0 = success
exec:
  command:
    - sh
    - -c
    - "pg_isready -U orders || exit 1"
```

```
httpGet   │ Best for HTTP apps. App can return a MEANINGFUL health answer (check DB, deps).
          │   Beware: a 3xx redirect counts as success; auth walls can make it 401 = failure.
tcpSocket │ Best for non-HTTP (Postgres, Redis). ONLY proves the port accepts connections —
          │   NOT that the app inside is actually working. Weakest signal.
exec      │ Most flexible (any script), but MOST EXPENSIVE: forks a process every period,
          │   entering the container's namespaces. Overusing exec probes burns node CPU.
```

---

## Example 1 — basic

A minimal `orders-api` container with a liveness and readiness probe. Every line commented.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: orders-api
  namespace: orders                # our running-example namespace
spec:
  containers:
    - name: orders-api
      image: orders-api:1.4.0
      ports:
        - containerPort: 3000      # the port the Node/Express app listens on
      readinessProbe:              # "is it safe to send traffic?"
        httpGet:
          path: /ready             # this endpoint checks Postgres + Redis are reachable
          port: 3000
        periodSeconds: 5           # check every 5s
        failureThreshold: 3        # 3 fails → pull from Service (stop traffic)
      livenessProbe:               # "is it deadlocked / should I restart?"
        httpGet:
          path: /health            # this endpoint just proves the event loop responds
          port: 3000
        periodSeconds: 10          # check every 10s
        failureThreshold: 3        # 3 fails → restart the container
```

The matching Express handlers make the difference concrete:

```js
// /health — LIVENESS. Cheap. Only proves the event loop is not frozen.
// Do NOT check the database here! If Postgres blips, you don't want K8s
// killing every replica of orders-api at once.
app.get('/health', (req, res) => res.status(200).send('ok'));

// /ready — READINESS. Checks real dependencies. If the DB is down, this pod
// should stop receiving traffic — but should NOT be killed.
app.get('/ready', async (req, res) => {
  try {
    await pg.query('SELECT 1');   // can we reach Postgres?
    await redis.ping();           // can we reach Redis?
    res.status(200).send('ready');
  } catch (e) {
    res.status(503).send('not ready');  // 503 → kubelet marks not-ready → traffic stops
  }
});
```

The golden rule this example teaches: **liveness = cheap self-check, readiness = dependency check.** Never check external dependencies in liveness, or a shared dependency outage will make Kubernetes restart-storm your whole fleet.

---

## Example 2 — production scenario

**The situation.** Your team ships `orders-api` v1.5.0. On boot it runs database migrations and warms a large in-memory product cache from Redis — this takes about 35 seconds. The team, wanting "fast detection," sets:

```yaml
livenessProbe:
  httpGet: { path: /health, port: 3000 }
  initialDelaySeconds: 10   # ← the mistake
  periodSeconds: 5
  failureThreshold: 3
```

**What happens.** The container starts. At 10s the liveness probe begins hitting `/health`, but the app is still running migrations and can't answer — the HTTP server hasn't even bound the port yet. Probe fails at 10s, 15s, 20s → 3 consecutive failures → the kubelet **kills the container at ~20s**, long before the 35s warmup finishes. It restarts. Same thing happens again. And again.

```
$ kubectl -n orders get pods
NAME                    READY   STATUS             RESTARTS   AGE
orders-api-5c9d-2xk4    0/1     CrashLoopBackOff   6          4m
                                │                  │
                                │                  └─ killed & restarted 6 times, never got to serve
                                └─ never became ready; users see the deploy "stuck"

$ kubectl -n orders describe pod orders-api-5c9d-2xk4
  Warning  Unhealthy  Liveness probe failed: Get "http://10.244.2.9:3000/health": connection refused
  Normal   Killing    Container orders-api failed liveness probe, will be restarted
```

The app is **perfectly healthy** — it just needs 35 seconds. The liveness probe is *killing it before it can finish being born*. This is the classic self-inflicted CrashLoopBackOff.

**The fix — a startup probe.** The startup probe exists precisely for slow starts. It gives the app a big, generous budget to boot, and **suspends liveness/readiness until it passes**:

```yaml
startupProbe:
  httpGet: { path: /health, port: 3000 }
  periodSeconds: 5
  failureThreshold: 30        # 30 × 5s = 150s of runway to finish warming up
livenessProbe:
  httpGet: { path: /health, port: 3000 }
  periodSeconds: 10           # only STARTS after startupProbe passes once
  failureThreshold: 3         # now it can be aggressive & fast safely
  timeoutSeconds: 2
readinessProbe:
  httpGet: { path: /ready, port: 3000 }
  periodSeconds: 5
  failureThreshold: 3
```

Now the flow is correct: up to 150s of patient startup checks, then — only after the app says "I'm up" once — fast liveness (detect a real freeze within 30s) and readiness (gate traffic on Postgres+Redis). No more restart storm, and you *still* get aggressive freeze detection once the app is live. **The startup probe lets you have both patience at birth and strictness in life.**

---

## Common mistakes

**Mistake 1 — Checking the database in the liveness probe.**

```yaml
livenessProbe:
  httpGet: { path: /ready, port: 3000 }   # /ready pings Postgres — WRONG for liveness
```

```
# Postgres has a 30-second blip during a failover.
$ kubectl -n orders get pods
orders-api-a   0/1   CrashLoopBackOff   3
orders-api-b   0/1   CrashLoopBackOff   3
orders-api-c   0/1   CrashLoopBackOff   3   ← ALL replicas restart at once
```

Root cause: liveness answers "should I restart the container?" If it depends on a shared external system, then when that system blips, **every replica fails liveness simultaneously** and Kubernetes kills them all — turning a 30s DB hiccup into a full outage plus a thundering herd of restarts hammering the recovering database. Right way: liveness = cheap, self-contained. Dependency checks belong in **readiness** (which only removes traffic, doesn't restart).

**Mistake 2 — Using `initialDelaySeconds` instead of a startup probe for slow boots.**

```yaml
livenessProbe:
  initialDelaySeconds: 60   # "just wait a minute before checking"
```

Root cause: `initialDelaySeconds` is a *fixed* wait. If boot takes 65s you still crashloop; if it takes 5s you've made failure detection 55s slower forever. A `startupProbe` polls and unlocks the *instant* the app is ready — adaptive, not a guess. Right way: use `startupProbe` for slow starts, keep `initialDelaySeconds` near 0.

**Mistake 3 — `timeoutSeconds: 1` (the default) on a busy app.**

```
Warning  Unhealthy  Liveness probe failed: Get "http://10.244.1.5:3000/health": context deadline exceeded
```

Root cause: the default timeout is **1 second**. Under a GC pause, a CPU-throttled pod (Topic 44), or a burst of load, even a healthy `/health` can take >1s to answer. Three such blips and the kubelet kills a healthy pod. Right way: set `timeoutSeconds: 2`–`5` and keep `/health` handlers trivially cheap (no I/O, no locks).

**Mistake 4 — No readiness probe, so deploys drop traffic.**

Without readiness, a pod is added to the Service the instant its process starts — before it can serve. During every rolling update (Topic 46), new pods receive traffic while still booting:

```
502 Bad Gateway   ← users hit new pods that aren't listening yet
```

Root cause: Kubernetes assumes "process started = ready" unless you tell it otherwise. Right way: always define a readiness probe that returns 200 only when the app can actually serve.

**Mistake 5 — Readiness that never turns off traffic during graceful shutdown.**

When a pod is being terminated, it gets `SIGTERM`, but it may still be in the Service's endpoints for a moment (endpoint removal is asynchronous). If the app closes its server immediately on `SIGTERM`, in-flight requests get dropped.

```
Error: connect ECONNREFUSED 10.244.1.7:3000   ← client hit a pod that already stopped listening
```

Root cause: readiness removal and pod deletion race. Right way: on `SIGTERM`, first make `/ready` return 503 (so K8s pulls you from the Service), sleep a few seconds to drain, *then* stop the server. Combine with `terminationGracePeriodSeconds`.

---

## Hands-on proof

Run these on a cluster (minikube/kind/any) to watch probes act. We'll fake `orders-api` with a tiny image but real probe behavior.

```bash
# 0. Namespace
kubectl create namespace orders 2>/dev/null

# 1. A pod whose liveness probe WILL fail on purpose (path that doesn't exist),
#    so we can watch Kubernetes restart it.
cat <<'EOF' | kubectl -n orders apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
    - name: web
      image: nginx:alpine
      livenessProbe:
        httpGet:
          path: /this-path-404s     # nginx returns 404 → probe FAILS
          port: 80
        periodSeconds: 3
        failureThreshold: 2         # 2 × 3s = ~6s to first restart
EOF

# 2. Watch RESTARTS climb — proof liveness kills & restarts the container.
kubectl -n orders get pod probe-demo -w
# ... after ~10s you'll see RESTARTS go 0 → 1 → 2 ...

# 3. See the exact probe failure events.
kubectl -n orders describe pod probe-demo | grep -A2 Unhealthy
# Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404

# 4. Now demonstrate READINESS gating traffic. Deploy an app + Service where the
#    readiness path is wrong, so the pod is Running but NEVER Ready.
cat <<'EOF' | kubectl -n orders apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ready-demo
  labels: { app: ready-demo }
spec:
  containers:
    - name: web
      image: nginx:alpine
      readinessProbe:
        httpGet: { path: /nope, port: 80 }   # 404 → never ready
        periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: ready-demo
spec:
  selector: { app: ready-demo }
  ports: [{ port: 80, targetPort: 80 }]
EOF

# 5. Pod is RUNNING but 0/1 READY:
kubectl -n orders get pod ready-demo
# ready-demo   0/1   Running   0   20s

# 6. PROOF the readiness failure removed it from the Service — empty endpoints:
kubectl -n orders get endpoints ready-demo
# NAME         ENDPOINTS   AGE
# ready-demo   <none>      25s     ← no IP here → kube-proxy routes NOTHING to it

# 7. Clean up
kubectl -n orders delete pod probe-demo ready-demo svc/ready-demo
```

What you verified: liveness **restarts** (step 2, RESTARTS climbs), readiness **removes from Service** but does NOT restart (steps 5–6, Running yet empty endpoints).

---

## Practice exercises

### Exercise 1 — easy
Create a pod running `nginx:alpine` with a liveness `httpGet` on path `/` port `80` and `periodSeconds: 5`. Confirm `RESTARTS` stays at `0` for a minute (it's healthy). Then `kubectl edit` the pod and change the path to `/broken`. Watch `RESTARTS` start climbing. Explain in one sentence which kubelet action caused the restart.

### Exercise 2 — medium
Write an `orders-api`-style pod with BOTH a readiness probe (path `/nope`, always 404) and a liveness probe (path `/`, always 200). Predict before applying: will `RESTARTS` climb? Will the pod be `READY 1/1`? Apply it, check `kubectl get pod` and `kubectl get endpoints`, and explain why readiness failing did not restart the container.

### Exercise 3 — hard (production simulation)
Simulate the Example 2 slow-boot crashloop. Build a tiny Node image whose server sleeps 40 seconds before it starts listening. Give it ONLY a liveness probe with `initialDelaySeconds: 10, periodSeconds: 5, failureThreshold: 3`. Observe the `CrashLoopBackOff`. Now fix it by adding a `startupProbe` with `failureThreshold: 30, periodSeconds: 5` and observe it boot cleanly. Write down: (a) at roughly what second the broken version got killed, and (b) why the startup probe let it survive.

---

## Mental model checkpoint

Answer from memory:

1. Liveness fails → Kubernetes does ______. Readiness fails → Kubernetes does ______. These are different because ______.
2. Which control-plane object does a readiness failure edit, and what does kube-proxy do as a result?
3. Why should you NEVER check Postgres in a liveness probe? What outage pattern does it cause?
4. What problem does a `startupProbe` solve that `initialDelaySeconds` cannot?
5. Worst-case detection time for `periodSeconds: 5, failureThreshold: 3` is ______ seconds.
6. Name the three probe types and one situation where each is the right choice.
7. Which probe types may set `successThreshold > 1`, and which must keep it at `1`?

---

## Quick reference card

| Field / concept | What it does | Key detail |
|---|---|---|
| `livenessProbe` | Restart the container if it fails | Keep it cheap & self-contained; never check external deps |
| `readinessProbe` | Add/remove pod from the Service | Does NOT restart; use for Postgres/Redis dependency checks |
| `startupProbe` | Suspends liveness/readiness until app is up | Best tool for slow boots; adaptive, not a fixed wait |
| `httpGet` | HTTP GET, 2xx/3xx = success | App can give a meaningful answer; watch auth walls (401) |
| `tcpSocket` | TCP handshake = success | Only proves port is open, not that the app works |
| `exec` | Run a command, exit 0 = success | Most flexible, most CPU-expensive (forks each period) |
| `periodSeconds` | How often to probe | Default 10 |
| `timeoutSeconds` | Wait before calling it a failure | Default 1s — often too tight, raise to 2–5 |
| `failureThreshold` | Consecutive fails before acting | Default 3; detection = period × threshold |
| `initialDelaySeconds` | Delay before first probe | Prefer `startupProbe` over a large value here |
| `CrashLoopBackOff` | Repeated restart with backoff | Usually a too-aggressive/misconfigured liveness probe |

---

## When would I use this at work?

1. **Zero-downtime deploys.** You add a readiness probe checking Postgres+Redis so that during a rolling update (Topic 46), new `orders-api` pods only receive traffic once they can actually serve — killing the "502 during every deploy" complaints.

2. **Catching silent deadlocks.** Your `orders-api` occasionally freezes on a bad Redis connection but the process stays alive. A liveness probe on `/health` auto-restarts the frozen pod within 30s instead of a human getting paged at 3am.

3. **Fixing a mysterious CrashLoopBackOff.** A teammate's new service with a 60s warmup keeps restarting. You recognize the liveness-kills-slow-boot pattern instantly, add a `startupProbe`, and it goes green — a 10-minute fix that would otherwise burn a day.

---

## Connected topics

- **Study before:**
  - **Topic 29 — Kubernetes Architecture**: the kubelet is the process that runs every probe.
  - **Topic 33 — Pods in Depth**: probes are defined per container in the pod spec.
  - **Topic 35 — Services**: readiness controls which pods land in the Service's EndpointSlice.
  - **Topic 26 — Health Checks (Docker HEALTHCHECK)**: the Docker-level cousin; K8s probes supersede it in a cluster.
- **Study after:**
  - **Topic 44 — Resource Requests and Limits**: CPU throttling can make probes time out; the two interact constantly.
  - **Topic 46 — Rolling Updates and Rollbacks**: readiness gating is what makes rolling updates zero-downtime.
