# 111 — Monitoring and alerting

## What is this?

Monitoring is the practice of continuously collecting numbers about your running application — how many requests per second, how long they take, how much memory is used, how many errors happened — and storing that history so you can look back at it. Alerting is the second half: automatically watching those numbers and paging a human the moment something crosses a dangerous line. Think of it like the dashboard and warning lights in a car — the speedometer and fuel gauge are monitoring (constant readings), and the "check engine" light turning on is alerting (something needs attention now, before you find out the hard way on the highway).

## Why does it matter for backend development?

A backend service that "seems to work" in manual testing can still be silently degrading in production — memory creeping up, response times doubling, a downstream database timing out for 2% of requests. Without monitoring, the first person to notice is an angry customer, hours after it started. Every real production Node.js service exposes metrics (usually via **Prometheus** and the `prom-client` library), visualizes them in **Grafana** dashboards, and has **alerting rules** that page an on-call engineer automatically. Backend developers are expected to instrument their own code — add counters for business events, histograms for latency, gauges for resource usage — because nobody understands what "healthy" looks like for an endpoint better than the person who wrote it.

---

## Syntax / API

```js
// npm install prom-client
const client = require('prom-client');   // the Prometheus client library for Node.js

// Collect default Node.js metrics (CPU, memory, event loop lag, GC) automatically
client.collectDefaultMetrics({ prefix: 'myapp_' });   // prefix avoids name collisions

// ── Counter — a number that only ever goes UP (e.g. total requests) ────────
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',              // metric name Prometheus will scrape
  help: 'Total number of HTTP requests',    // human-readable description (required)
  labelNames: ['method', 'route', 'status'] // dimensions you can filter/group by
});
httpRequestsTotal.inc({ method: 'GET', route: '/users', status: '200' }); // +1

// ── Gauge — a number that goes UP AND DOWN (e.g. active connections) ───────
const activeConnections = new client.Gauge({
  name: 'active_connections',               // current snapshot value
  help: 'Number of currently open connections'
});
activeConnections.inc();   // connection opened, +1
activeConnections.dec();   // connection closed, -1

// ── Histogram — measures distribution of values (e.g. request duration) ────
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency in seconds',
  labelNames: ['route'],
  buckets: [0.05, 0.1, 0.3, 0.5, 1, 2, 5]   // bucket boundaries in seconds
});
const endTimer = httpRequestDuration.startTimer({ route: '/users' }); // start clock
endTimer();   // stop clock, records the elapsed time into the right bucket

// ── Exposing metrics for Prometheus to scrape ───────────────────────────────
const express = require('express');
const app = express();

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType); // Prometheus text format
  res.end(await client.register.metrics());              // dump all metrics as text
});
```

---

## How it works — line by line

Prometheus works on a **pull model**: it does not want you to push data to it — instead, your Node.js app exposes a plain-text `/metrics` endpoint, and a Prometheus server periodically visits that URL (called "scraping") every few seconds, reading and storing every number it finds.

The `prom-client` library gives you three core metric types to describe those numbers:

- A **Counter** only increases — perfect for "total requests served" or "total errors thrown". You never decrease it; if you need a resettable count, you want a Gauge instead.
- A **Gauge** can go up or down — perfect for "requests currently in flight" or "memory used right now".
- A **Histogram** buckets values into ranges so you can later compute percentiles (like "95% of requests finish under 300ms") — perfect for latency and payload sizes.

`client.register.metrics()` collects every metric you have defined, formats it into the plain-text exposition format Prometheus understands, and your `/metrics` route just serves it as a response body — no special protocol, just HTTP and text.

Grafana is a separate tool that connects to Prometheus as a data source and turns those stored numbers into graphs, tables, and dashboards a human can actually read. Alerting rules are conditions written in Prometheus's query language (PromQL) — for example "error rate above 5% for 5 minutes" — that, when true, fire an alert which a tool like Alertmanager routes to Slack, email, or a pager. An APM (Application Performance Monitoring) tool like New Relic or Datadog does a similar job but goes further out of the box — it auto-instruments your code (no manual Counter/Histogram setup needed), traces a single request across multiple services, and gives you flame graphs of exactly which function call was slow, at the cost of being a paid, hosted product instead of self-run open source.

---

## Example 1 — basic

```js
// File: metrics-demo.js
// Minimal working example: expose Node's own health metrics on /metrics

const express = require('express');            // web framework to serve the endpoint
const client  = require('prom-client');         // Prometheus client for Node.js

const app = express();

// Automatically track CPU usage, memory (RSS/heap), event loop lag, GC pauses
client.collectDefaultMetrics({ prefix: 'demo_' });   // one line, zero manual wiring

// A custom counter to track how many times this endpoint itself is hit
const metricsScrapeCounter = new client.Counter({
  name: 'demo_metrics_scrapes_total',           // metric name
  help: 'Number of times /metrics was scraped'  // description shown in Prometheus UI
});

app.get('/metrics', async (req, res) => {
  metricsScrapeCounter.inc();                          // count this scrape
  res.set('Content-Type', client.register.contentType); // tell client it's Prometheus text
  res.end(await client.register.metrics());             // send all registered metrics
});

app.listen(3000, () => {
  console.log('Metrics available at http://localhost:3000/metrics'); // where to check
});

// Run it, then visit http://localhost:3000/metrics — you will see lines like:
// demo_process_cpu_user_seconds_total 0.12
// demo_metrics_scrapes_total 1
```

---

## Example 2 — real world backend use case

```js
// File: src/middleware/metrics.js
// Production pattern: instrument every HTTP request automatically, expose /metrics,
// and track a business-level counter (failed logins) that feeds an alerting rule.

const client = require('prom-client');

client.collectDefaultMetrics({ prefix: 'api_' });   // baseline Node health metrics

// Tracks every request by method, route, and status code
const httpRequestsTotal = new client.Counter({
  name: 'api_http_requests_total',
  help: 'Total HTTP requests handled',
  labelNames: ['method', 'route', 'status']
});

// Tracks how long each route takes to respond — feeds latency dashboards + alerts
const httpRequestDuration = new client.Histogram({
  name: 'api_http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 3]   // covers fast reads to slow writes
});

// Business-level metric: an alerting rule can fire on "too many failed logins"
const failedLoginAttempts = new client.Counter({
  name: 'api_failed_login_attempts_total',
  help: 'Total failed login attempts',
  labelNames: ['reason']   // e.g. 'bad_password', 'unknown_user'
});

// Express middleware — attach this once, near the top of the middleware stack
function metricsMiddleware(req, res, next) {
  const endTimer = httpRequestDuration.startTimer();   // start the latency clock

  res.on('finish', () => {                             // runs after response is sent
    const route = req.route ? req.route.path : req.path; // grouped route, not /users/42
    const labels = { method: req.method, route, status: res.statusCode };

    httpRequestsTotal.inc(labels);   // +1 request counted with its outcome
    endTimer(labels);                // stop clock, records duration into histogram
  });

  next();   // pass control to the next middleware/route handler
}

// Call this from your login controller when a login fails
function recordFailedLogin(reason) {
  failedLoginAttempts.inc({ reason });   // e.g. recordFailedLogin('bad_password')
}

module.exports = { metricsMiddleware, recordFailedLogin, register: client.register };

// ── Wiring it up in server.js ───────────────────────────────────────────────
// const { metricsMiddleware, register } = require('./middleware/metrics');
// app.use(metricsMiddleware);                       // instrument every request
// app.get('/metrics', async (req, res) => {
//   res.set('Content-Type', register.contentType);
//   res.end(await register.metrics());
// });

// ── A matching Prometheus alerting rule (alerts.yml) ────────────────────────
// groups:
//   - name: api-alerts
//     rules:
//       - alert: HighErrorRate
//         expr: |
//           sum(rate(api_http_requests_total{status=~"5.."}[5m]))
//           /
//           sum(rate(api_http_requests_total[5m])) > 0.05
//         for: 5m                       # must stay true for 5 minutes before firing
//         labels:
//           severity: critical
//         annotations:
//           summary: "5xx error rate above 5% for 5 minutes"
```

---

## Common mistakes

### Mistake 1 — Using raw IDs as label values (label cardinality explosion)

```js
// ❌ WRONG — using userId as a label creates a NEW time series per user,
// Prometheus memory usage explodes with real traffic (millions of series)
const requestCounter = new client.Counter({
  name: 'http_requests_total',
  help: 'Total requests',
  labelNames: ['userId', 'route']   // userId has unbounded, ever-growing values
});
requestCounter.inc({ userId: req.user.id, route: req.path });

// ✅ CORRECT — only use labels with a small, known set of possible values
const requestCounter2 = new client.Counter({
  name: 'http_requests_total',
  help: 'Total requests',
  labelNames: ['method', 'route', 'status']   // bounded: a handful of methods/routes/codes
});
requestCounter2.inc({ method: req.method, route: req.route.path, status: 200 });
// If you need per-user analysis, log it (Topic 63) — don't put it in a metric label
```

### Mistake 2 — Creating a new metric instance on every request

```js
// ❌ WRONG — `new client.Counter(...)` is called inside the request handler,
// Prometheus throws "metric already registered" after the first request,
// or (with try/catch swallowing it) silently creates duplicate/broken registration
app.get('/orders', (req, res) => {
  const ordersCounter = new client.Counter({          // recreated every single request!
    name: 'orders_total',
    help: 'Total orders fetched'
  });
  ordersCounter.inc();
  res.json({ orders: [] });
});

// ✅ CORRECT — define metrics ONCE at module load time, reuse the same instance
const ordersCounter = new client.Counter({   // created once, when the file is required
  name: 'orders_total',
  help: 'Total orders fetched'
});

app.get('/orders', (req, res) => {
  ordersCounter.inc();   // reuse the already-registered counter, just increment it
  res.json({ orders: [] });
});
```

### Mistake 3 — Setting alert thresholds with no "for" duration (alert noise / flapping)

```yaml
# ❌ WRONG — fires an alert for even a single one-second CPU spike,
# pages the on-call engineer at 3 AM for a blip that self-resolved instantly
- alert: HighCpuUsage
  expr: rate(process_cpu_user_seconds_total[1m]) > 0.8
  # no "for:" — evaluates true once and fires immediately, causing alert fatigue

# ✅ CORRECT — require the condition to hold for a sustained window before paging
- alert: HighCpuUsage
  expr: rate(process_cpu_user_seconds_total[1m]) > 0.8
  for: 10m                     # must stay true for 10 continuous minutes
  labels:
    severity: warning
  annotations:
    summary: "CPU usage above 80% for 10 minutes — investigate before it worsens"
```

---

## Practice exercises

### Exercise 1 — easy

Create an Express server with a single route `GET /ping` that returns `{ status: 'ok' }`. Add `prom-client`, call `collectDefaultMetrics()`, and expose the results on `GET /metrics`. Add a custom `Counter` named `ping_requests_total` that increments every time `/ping` is hit. Start the server, hit `/ping` three times, then check `/metrics` and confirm `ping_requests_total` shows `3`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `requestTimer` Express middleware that uses a Prometheus `Histogram` named `route_duration_seconds` (labels: `route`, `status`) to record how long every request takes. Apply it globally with `app.use()`. Create two routes: `GET /fast` that responds immediately, and `GET /slow` that waits 800ms (using a `Promise` + `setTimeout`) before responding. Expose `/metrics`, hit both routes a few times, and confirm the histogram buckets for `/slow` land in a higher bucket than `/fast`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small monitored order-processing service with these pieces:
1. `POST /orders` — accepts `{ userId, amount }` in `requestBody`, and randomly (30% chance) throws a simulated "payment provider timeout" error.
2. A `Counter` called `orders_total` labeled by `status` (`'success'` or `'failed'`).
3. A `Histogram` called `order_processing_duration_seconds` measuring how long the handler takes.
4. A `Gauge` called `orders_in_flight` that increments when an order starts processing and decrements when it finishes (success or failure) — use a `try/finally` so it always decrements.
5. `GET /metrics` exposing all of the above.
6. Write (as a comment block at the bottom of the file, not runnable code) one PromQL alerting rule expression that would fire if the failed-order rate exceeds 20% over a 5-minute window, with an appropriate `for:` duration.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE CONCEPTS
  Monitoring  → continuously collecting numeric metrics over time
  Alerting    → automated rules that page a human when metrics cross a threshold
  Prometheus  → time-series database that PULLS metrics from your app's /metrics endpoint
  Grafana     → dashboard tool that queries Prometheus (or others) and draws graphs
  APM         → New Relic / Datadog style tool — auto-instruments code + distributed tracing

METRIC TYPES (prom-client)
  Counter    → only goes up            → .inc()                → total requests, total errors
  Gauge      → goes up and down        → .inc() / .dec() / .set(value) → active connections, queue size
  Histogram  → distribution of values  → .observe(value) / .startTimer() → request latency, payload size
  Summary    → like Histogram, calculates quantiles client-side (rarely preferred over Histogram)

SETUP PATTERN
  const client = require('prom-client');
  client.collectDefaultMetrics({ prefix: 'app_' });   // CPU, memory, GC, event loop lag — free
  const counter = new client.Counter({ name, help, labelNames });   // define ONCE, module scope
  app.get('/metrics', async (req, res) => {
    res.set('Content-Type', client.register.contentType);
    res.end(await client.register.metrics());
  });

LABELS — RULES OF THUMB
  Good labels : method, route, status, region   (small, bounded set of values)
  Bad labels  : userId, requestId, email, IP     (unbounded → cardinality explosion)

ALERTING RULE SHAPE (PromQL + Alertmanager)
  - alert: <Name>
    expr: <PromQL expression that evaluates to boolean>
    for: 5m                # must stay true this long before firing — avoids flapping
    labels: { severity: critical }
    annotations: { summary: "human readable message" }

COMMON GOTCHAS
  Creating metrics inside a request handler   → "already registered" errors, memory bloat
  High-cardinality labels (userId, requestId) → Prometheus storage/memory explosion
  No "for:" duration on alert rules           → alert fatigue from single-sample blips
  Forgetting to decrement a Gauge on error    → use try/finally around inc()/dec() pairs
  Scraping too frequently for expensive apps  → tune Prometheus scrape_interval (default 15s)

APM VS SELF-HOSTED PROMETHEUS/GRAFANA
  Prometheus+Grafana : free, self-run, you write the instrumentation, full control
  New Relic / Datadog: paid SaaS, auto-instruments, built-in distributed tracing, faster setup
```

---

## Connected topics

- **63 — Logging in Node** — structured logs (winston/pino) complement metrics: metrics tell you *something* is wrong, logs tell you *why*.
- **65 — Health checks and readiness probes** — `/health` and `/ready` endpoints are simple binary signals; `/metrics` is the detailed numeric picture behind them.
- **110 — Process management in production** — PM2/systemd restart crashed processes, but monitoring is what tells you *why* they crashed and alerts you before the next crash.
