# 51 — Observability in Kubernetes
## Section: Production Patterns

## ELI5 — The Simple Analogy

Imagine you're the manager of a huge airport. Planes (requests) are landing and taking off constantly. You can't stand on the runway and watch every plane — there are thousands. So you build three things:

1. **A big dashboard of gauges** — "flights per hour", "average delay", "fuel remaining". These are **numbers over time**. That's **metrics**.
2. **A written logbook** — "Flight 447 was diverted at 14:32 because of engine warning". These are **detailed event records**. That's **logs**.
3. **A tracking map that follows one specific passenger** through check-in → security → gate → plane → baggage, timing each step. That's **traces**.

Metrics tell you *something is wrong* ("delays are up 300%"). Logs tell you *what* the error was ("engine warning"). Traces tell you *where in the journey* it went slow ("security took 40 minutes").

**Observability** is having all three so that when a plane is late — or your `orders-api` is slow — you can answer *what, where, and why* without walking the runway.

## The Linux kernel feature underneath

Observability data ultimately comes from places the **kernel** already exposes; the tooling just collects and ships it.

- **Logs** are just **stdout/stderr**. Remember from **Topic 17** that a container's PID 1 writes to file descriptors 1 and 2, and the container runtime redirects those to a file on the node:
  ```
  /var/log/pods/<namespace>_<pod>_<uid>/<container>/0.log
  /var/log/containers/<pod>_<ns>_<container>-<id>.log   ← symlink to the above
  ```
  A log agent (Fluent Bit) is a **DaemonSet** (Topic 47) that `tail`s these files on every node. No app change needed to *collect* logs — the kernel/runtime already wrote them to disk.

- **Metrics** like CPU/memory come from the kernel **cgroup** counters (`/sys/fs/cgroup/.../cpu.stat`, `memory.current` — Topic 44/45), read by cAdvisor in the kubelet. Application metrics (requests/sec) come from your process exposing an HTTP `/metrics` endpoint that Prometheus scrapes.

- **Traces** need in-process instrumentation (the app must propagate a trace-id header), but the *transport* rides on ordinary TCP sockets (Topic 40) to a collector.

So the raw materials — file descriptors, cgroup files, sockets — are all kernel primitives you already know. Observability is the discipline of **collecting, storing, and querying** them at cluster scale.

## What is this?

Observability is the ability to understand a system's internal state from the outside, using three "pillars": **metrics** (numeric time-series — Prometheus), **logs** (discrete event records — Loki/EFK), and **traces** (the path of one request across services — Jaeger/OpenTelemetry). In Kubernetes, these are provided by add-ons because a cluster has too many ephemeral Pods across too many nodes to inspect by hand.

## Why does it matter for a backend developer?

In a single-server world you could `ssh` in and `tail -f app.log`. In Kubernetes that habit **breaks completely**:

- Pods are **ephemeral** — a Pod that logged the error is gone (crashed, rescheduled), and `kubectl logs` can't show a dead Pod's history.
- Your app runs as **many replicas across many nodes** — the error might be on any of 15 Pods on any of 5 nodes.
- Services call services — a slow `/checkout` might actually be a slow Redis call two hops away, and no single log file shows the whole chain.

Without observability, debugging a production incident becomes guesswork. With it, you go: **alert fires (metric) → find the failing endpoint (metric) → read the exact errors (logs) → see which downstream call is slow (trace)** — in minutes, not hours. This is the difference between a 5-minute incident and a 5-hour one.

## The physical reality

A typical observability stack, as objects in your cluster:

```
Namespace: monitoring
├── prometheus            (StatefulSet)  ← scrapes /metrics, stores time-series on a PVC
│   └── PVC: 50Gi         ← metric samples on disk (TSDB blocks)
├── grafana               (Deployment)   ← dashboards; queries Prometheus + Loki
├── loki                  (StatefulSet)  ← stores log streams on a PVC / object storage
├── promtail / fluent-bit (DaemonSet)    ← ONE per node, tails /var/log/containers/*.log
└── alertmanager          (Deployment)   ← routes firing alerts to Slack/PagerDuty

Namespace: tracing
├── otel-collector        (Deployment/DaemonSet) ← receives spans over OTLP
└── jaeger / tempo        (StatefulSet)  ← stores and indexes traces
```

On each **node**, the log agent physically opens the files under `/var/log/containers/`. Prometheus physically holds a **TSDB** (time-series database) on a PersistentVolume (Topic 39) — that's why you size a PVC for it; lose it and you lose metric history.

Your `orders-api` Pod contributes by:
- Writing JSON logs to **stdout** (collected automatically).
- Exposing **`GET /metrics`** on a port (scraped by Prometheus).
- Sending **spans** to the OTel collector (via the OpenTelemetry SDK).

## How it works — step by step

**Metrics path (Prometheus, pull model):**

1. Your Node app uses `prom-client` to expose `GET /metrics` returning text like `http_requests_total{route="/orders",status="200"} 48213`.
2. You add a **ServiceMonitor** (or a `prometheus.io/scrape: "true"` annotation) so Prometheus discovers the Pod via the Kubernetes API (Topic 38 labels).
3. Prometheus **scrapes** `http://<pod-ip>:3000/metrics` every 15–30s and appends samples to its TSDB on disk with a timestamp.
4. **Grafana** queries Prometheus with **PromQL** to draw graphs; **Alertmanager** evaluates alert rules (e.g. "error rate > 5% for 5m") and pages you.

**Logs path (Loki + Fluent Bit, push model):**

1. `orders-api` writes structured JSON to stdout: `{"level":"error","msg":"db timeout","trace_id":"abc123","order_id":42}`.
2. The runtime writes it to `/var/log/containers/orders-api-*.log` on the node.
3. **Fluent Bit** (DaemonSet) tails that file, attaches Kubernetes metadata (namespace, pod, labels) from the API, and **pushes** the line to **Loki**.
4. Loki indexes by **labels** (not full text — that's what makes it cheap) and stores the log body.
5. In Grafana you run a **LogQL** query: `{app="orders-api", namespace="orders"} |= "db timeout"`.

**Traces path (OpenTelemetry):**

1. A request hits `orders-api`. The OTel SDK starts a **span** and generates a **trace-id**.
2. When `orders-api` calls Redis and Postgres, it **propagates** the trace-id in headers (`traceparent`) so downstream spans join the same trace.
3. Each service exports spans to the **OTel Collector** over OTLP.
4. The collector forwards to **Jaeger/Tempo**, which reconstructs the full timeline: `HTTP 120ms → [auth 5ms][redis 2ms][postgres 95ms]` — instantly showing Postgres is the slow hop.

**How the three connect** — the golden thread is the **trace-id**: it's in the log line *and* the trace. So from a slow trace you jump to the exact logs, and from an error log you jump to the full request path. That correlation is the whole point.

## Exact syntax breakdown

**Node.js `/metrics` endpoint with `prom-client`:**

```js
const client = require('prom-client');
const express = require('express');
const app = express();

// A counter: monotonically increasing count of HTTP requests
const httpRequests = new client.Counter({
  name: 'http_requests_total',          // metric name (Prometheus convention: _total for counters)
  help: 'Total HTTP requests',          // description shown in tooling
  labelNames: ['method', 'route', 'status'],  // dimensions to slice by
});

// A histogram: distribution of request durations (for latency percentiles)
const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Request duration',
  labelNames: ['route'],
  buckets: [0.05, 0.1, 0.3, 0.5, 1, 3],  // latency buckets in seconds
});

app.use((req, res, next) => {
  const end = httpDuration.startTimer({ route: req.path });
  res.on('finish', () => {
    httpRequests.inc({ method: req.method, route: req.path, status: res.statusCode });
    end();
  });
  next();
});

app.get('/metrics', async (_req, res) => {          // Prometheus scrapes THIS
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

**ServiceMonitor** (tells the Prometheus Operator what to scrape):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: orders-api
  namespace: orders
  labels:
    release: prometheus            # must match the Prometheus's serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: orders-api              # pick the Service whose Pods to scrape (Topic 38)
  endpoints:
    - port: http                   # named port on the Service
      path: /metrics               # scrape path
      interval: 30s                # how often
```

Annotated key lines:

```
    matchLabels:
      app: orders-api      ← selects the Service; Prometheus then scrapes its backing Pods
  endpoints:
    - port: http           ← the NAME of the Service port (not the number), where /metrics lives
      interval: 30s        ← scrape frequency; shorter = higher resolution + more storage
```

**A PromQL alert rule** (error rate):

```
sum(rate(http_requests_total{status=~"5..", app="orders-api"}[5m]))
  / sum(rate(http_requests_total{app="orders-api"}[5m])) > 0.05
│         │                                        │        │
│         │                                        │        └── threshold: 5%
│         │                                        └── over a 5-minute sliding window
│         └── rate() = per-second increase of the counter
└── ratio of 5xx responses to all responses
```

## Example 1 — basic

Give `orders-api` structured logs and a metrics endpoint — the two cheapest, highest-value steps.

```js
// logger.js — structured JSON logs to stdout (collected automatically, Topic 17)
const pino = require('pino');
const log = pino();                          // outputs JSON: {"level":30,"time":...,"msg":...}

// somewhere in a handler:
log.info({ order_id: 42, user_id: 7 }, 'order created');
log.error({ err, order_id: 42 }, 'failed to reserve stock');
```

```dockerfile
# No special logging config needed — just DON'T write to a file inside the container.
# Write to stdout/stderr; Kubernetes + Fluent Bit handle the rest.
CMD ["node", "server.js"]
```

Then `kubectl logs deploy/orders-api -n orders` shows JSON immediately, and once Loki is installed those same lines are searchable cluster-wide and survive Pod death.

## Example 2 — production scenario

**Context:** At 2:14pm, Alertmanager pages: *"orders-api p99 latency > 2s for 5 minutes."* You're on call. Here's the observability-driven runbook:

1. **Metric (what/where):** Open Grafana. The latency graph shows p99 jumped from 150ms to 2.3s at 2:09pm, only on the `POST /checkout` route. Throughput is normal, so it's not a traffic spike.
2. **Metric drill-down:** A panel of downstream call durations shows `postgres_query_duration` climbing in lockstep. CPU and memory of the Pods are normal (rules out an app hot loop or OOM).
3. **Logs (why):** In Grafana/Loki: `{app="orders-api"} |= "checkout" | json | duration_ms > 1000`. You see hundreds of `"pg pool exhausted, waiting for connection"` warnings starting 2:09pm.
4. **Trace (confirm):** Click a slow request's `trace_id`. Jaeger shows the trace spent 2.1s in **"acquire db connection"** before Postgres even ran the query — the pool is starved, not the DB.
5. **Root cause:** A deploy at 2:08pm (correlate with the Deployment revision, Topic 46) bumped `maxReplicas` on the HPA, so `orders-api` scaled 3 → 18 Pods, and 18 × its pool size blew past Postgres `max_connections`.
6. **Fix:** Add PgBouncer / lower per-Pod pool size / cap `maxReplicas`. Roll back the change (`kubectl rollout undo`). Latency recovers; you confirm on the same Grafana graph.

**The lesson:** each pillar answered a different question, and the **trace-id in the logs** let you jump straight from "an alert" to "the exact starved request" without SSH-ing anywhere. That's observability earning its keep.

## Common mistakes

**Mistake 1 — Logging to a file inside the container.**

```
# app writes to /app/logs/app.log
$ kubectl logs orders-api-xxx -n orders
(empty)                 # nothing on stdout
```

**Root cause:** Fluent Bit and `kubectl logs` read **stdout/stderr**, not files inside the container's filesystem. A file in the container is invisible to the log pipeline and vanishes when the Pod dies. → **Fix:** log to stdout (12-factor, Topic 15). Let the platform collect and ship it.

**Mistake 2 — Unbounded label cardinality in metrics.**

```
http_requests_total{user_id="12345", route="/orders/98765"}   # user_id and order_id as labels
```

**Root cause:** Every unique label combination is a **separate time-series**. Putting high-cardinality values (user IDs, order IDs, full URLs with IDs) as labels creates millions of series and **OOM-kills Prometheus**. → **Fix:** keep labels low-cardinality (method, route *template* `/orders/:id`, status). Put the specific ID in a **log line or trace**, not a metric label.

**Mistake 3 — No resource requests/limits on the observability stack.**

Prometheus with a growing TSDB and no memory limit slowly eats a node and gets OOMKilled, losing recent data. → **Fix:** set `requests`/`limits` (Topic 44) and a retention policy (`--storage.tsdb.retention.time=15d`) and back it with a right-sized PVC (Topic 39).

**Mistake 4 — Treating `kubectl logs` as the strategy.**

`kubectl logs` only shows the **current** container of a **live** Pod (use `--previous` for the last crash). It cannot search across replicas or show history after a Pod is gone. → **Fix:** it's a debugging convenience, not observability. You need a log **store** (Loki/Elasticsearch) for anything past "is this one Pod OK right now".

**Mistake 5 — Metrics without alerts, or alerts on the wrong thing.**

Beautiful dashboards nobody looks at until a customer complains. Or alerting on CPU (a *cause*) instead of symptoms users feel. → **Fix:** alert on **symptoms** using the **RED method** for request services — **R**ate, **E**rrors, **D**uration (latency). Those map directly to "is the user having a bad time?"

## Hands-on proof

```bash
# 1. Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager) via Helm (Topic 49)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# 2. See what's running
kubectl get pods -n monitoring

# 3. Port-forward Grafana and log in (default admin / prom-operator)
kubectl port-forward -n monitoring svc/kps-grafana 3000:80
#   open http://localhost:3000  → explore built-in Kubernetes dashboards

# 4. PROVE the log source is just files on the node:
NODE_POD=$(kubectl get pod -n orders -l app=orders-api -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $NODE_POD -n orders -o wide            # find its node
# on that node (minikube ssh / docker exec):
ls /var/log/containers/ | grep orders-api             # the raw log file the agent tails

# 5. PROVE metrics are pull-based: curl your own /metrics
kubectl port-forward -n orders deploy/orders-api 3000:3000
curl -s localhost:3000/metrics | grep http_requests_total   # the exact text Prometheus scrapes

# 6. See cluster-level metrics that come from kernel cgroups:
kubectl top pods -n orders
```

## Practice exercises

### Exercise 1 — easy
Add `pino` (or `console.log` JSON) to any Node app and deploy it. Run `kubectl logs -f deploy/<app>` and confirm you see **structured JSON**, one object per line. Then kill the Pod and try `kubectl logs --previous` — observe what you can and cannot retrieve. Write down why a log store would help.

### Exercise 2 — medium
Install `kube-prometheus-stack` with Helm. Add a `/metrics` endpoint to your app using `prom-client` with a request counter and a latency histogram. Create a **ServiceMonitor**. In Grafana's Explore, write a PromQL query for requests-per-second by route: `sum(rate(http_requests_total[1m])) by (route)`. Generate some load and watch it move.

### Exercise 3 — hard (production simulation)
Instrument `orders-api` with the **three pillars**: (a) RED metrics, (b) structured logs that include a `trace_id`, (c) OpenTelemetry traces to a local Jaeger. Introduce an artificial 500ms sleep in the Postgres-calling code path. Then, starting *only* from a latency alert, use metric → log → trace to locate the slow span and prove (via the shared `trace_id`) that the log line and the trace refer to the same request.

## Mental model checkpoint

1. Name the three pillars and the one question each is best at answering.
2. Why must your app log to stdout instead of a file, and what component collects it?
3. Why is Prometheus a *pull* model, and what object tells it which Pods to scrape?
4. What is label cardinality and how can it crash Prometheus?
5. What is the "golden thread" that connects a log line to a distributed trace?
6. What does `kubectl logs` *not* give you that a log store does?
7. What does the RED method stand for, and why alert on it instead of CPU?

## Quick reference card

| Thing | Pillar | Tool(s) | Key detail |
|---|---|---|---|
| Numeric time-series | Metrics | Prometheus + Grafana | Pull model; scrapes `/metrics`; low-cardinality labels |
| Event records | Logs | Loki / EFK + Fluent Bit | Agent = DaemonSet tailing `/var/log/containers` |
| Request path across services | Traces | OpenTelemetry + Jaeger/Tempo | Needs trace-id propagation in headers |
| `/metrics` in Node | Metrics | `prom-client` | Counter/Gauge/Histogram; expose via HTTP |
| `ServiceMonitor` | Metrics | Prometheus Operator | Discovers scrape targets by labels |
| PromQL `rate()[5m]` | Metrics | Prometheus | Per-second rate over a window |
| LogQL `{app="x"} \|= "err"` | Logs | Loki | Filter by labels then text |
| Alertmanager | Alerting | Prometheus stack | Routes firing alerts to Slack/PagerDuty |
| RED method | Strategy | — | Rate, Errors, Duration = user-facing symptoms |
| `kubectl top` / `kubectl logs --previous` | Debug | built-in | Quick checks, not a strategy |

## When would I use this at work?

1. **Production incident response.** A latency alert fires; you go metric → log → trace to find the root cause (a starved DB pool) in minutes instead of SSH-guessing across 15 Pods on 5 nodes.
2. **Capacity & autoscaling signals.** The same metrics that dashboards use feed custom-metric HPAs (Topic 45) — e.g. scale `orders-api` on requests-per-second rather than CPU.
3. **Post-deploy verification / SLOs.** After a rollout (Topic 46), you watch error-rate and p99 latency on Grafana to confirm the new version is healthy, and you define SLOs ("99.9% of checkouts < 500ms") with alerts that page before customers notice.

## Connected topics

- **Study before:** Topic 17 (Logging — stdout & log drivers), Topic 44/45 (cgroup metrics that feed CPU/mem), Topic 47 (DaemonSets — how log/metric agents run per node), Topic 39 (PVs — where Prometheus/Loki store data).
- **Study after:** Topic 52 (CI/CD — wire alerts and dashboards into release verification), Topic 54 (Multi-Environment — separate observability per env), and revisit Topic 43 (probes are a form of local health signal complementary to observability).
