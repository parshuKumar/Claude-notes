# 80 — Monitoring and Observability — Logs, Metrics, Traces
## Category: HLD Components

---

## What is this?

**Monitoring** is watching your system for problems you already know how to look for — "is CPU above 90%?", "did the server stop responding?". You decided the questions in advance, and you built dashboards and alerts to answer them.

**Observability** is a *property* of your system: can you answer questions about it that you **never anticipated**, using only the data it already emits — without shipping new code? "Why are Android users in Brazil on the checkout page seeing 3-second latency, but only since Tuesday?" Nobody built a dashboard for that. If you can still answer it from your existing logs, metrics, and traces, your system is observable.

Think of a car. **Monitoring** is the dashboard: speedometer, fuel gauge, check-engine light — a fixed set of gauges someone chose for you. **Observability** is being able to plug a diagnostic tool into the car and ask *any* question — "what was the fuel-injector timing on cylinder 3 at 8:04am on the highway?" — and actually get an answer.

---

## Why does it matter?

In a monolith, debugging is easy: SSH into the box, `tail -f app.log`, read the stack trace. One process, one log file, one machine.

In a distributed system, that world is gone. A single "Place Order" click touches API Gateway → Auth Service → Cart Service → Inventory Service → Pricing Service → Payment Service → Notification Service. Seven services, each on 20 machines, each writing its own logs. The user says "checkout is slow." Which of the 140 processes is slow? You have no idea. Without observability you are debugging blind, and **your mean time to recovery (MTTR) is measured in hours instead of minutes.**

This is the price of microservices (recall from [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md)): you trade a debuggable single process for an undebuggable mesh — *unless* you invest in observability up front.

**The interview angle:** Almost every senior HLD interview ends with "how would you monitor this?" or "an alert fires at 3am saying checkout is slow — walk me through your debugging." Candidates who say "I'd check the logs" sound junior. Candidates who say "I'd look at the RED metrics for each service to find which one's p99 spiked, then pull a trace for a slow request to confirm, then filter logs by that trace ID" sound like they have run production.

**The real-work angle:** Observability is what stands between "we fixed it in 12 minutes" and "we were down for 6 hours and still don't know why." It is also, at scale, a genuinely large line item on your cloud bill — so you need to understand its cost model, not just its benefits.

---

## The core idea — explained simply

### The Hospital Analogy

A patient walks into a hospital. Three different kinds of information exist about that patient, and each answers a different kind of question.

**1. The patient's chart (LOGS).**
A timestamped list of discrete events: "08:14 — admitted, complaining of chest pain." "08:22 — administered 5mg aspirin." "08:40 — patient vomited." Rich, detailed, full of context. Perfect if you want to know exactly what happened to **this one patient**. Useless if you want to know "how many patients did we treat this month?" — you'd have to read ten thousand charts.

**2. The hospital's vital-sign monitors and stats board (METRICS).**
Numbers over time: heart rate 92 bpm, 47 patients currently in the ER, average wait time 38 minutes. Cheap to collect, easy to graph, easy to alarm on ("beep if heart rate > 140"). But a heart-rate number will never tell you **why** the heart rate is high. It just tells you *that* it is.

**3. The patient's journey through the hospital (TRACES).**
Reception (4 min) → triage (11 min) → **waiting room (2 hours 40 min)** → X-ray (18 min) → doctor (12 min) → discharge. Immediately you can see which department is the bottleneck. It's the waiting room. Not the X-ray machine everyone was blaming.

You need all three. The stats board tells you *something is wrong right now*. The journey tells you *where*. The chart tells you *exactly what*.

| Hospital | Software | Answers |
|----------|----------|---------|
| Patient chart entries | **Logs** | "What exactly happened to THIS request?" |
| Vital-sign monitors, stats board | **Metrics** | "Is the system healthy? Is it getting worse?" |
| The patient's route through departments | **Traces** | "Which service in the chain is the slow one?" |
| Patient ID number on everything | **Correlation / Trace ID** | "Show me all three views for the same request" |
| The doctor being paged | **Alerting** | "A human must act, now" |

That last row is the glue. A patient ID on the chart, on the monitor readout, and on the route sheet is what lets you connect them. In software that's the **trace ID** — and it is the single most valuable thing in this entire document.

---

## Key concepts inside this topic

### 1. Pillar One — Logs

A log is a **discrete, timestamped event with rich context**. "At 14:02:11.482, order 8831 failed payment because the card issuer returned DECLINED."

**Good at:** Deep detail about one specific thing. The story of a single request. Stack traces. Audit trails.
**Bad at:** Volume and aggregation. Logs are the most expensive pillar — you pay per gigabyte ingested and stored, and a busy service emits gigabytes per hour. Answering "what's my p99 latency?" from raw logs means scanning every line.

**Structured logging — the single biggest upgrade you can make.**

Here is the way most people log, and it is a trap:

```javascript
// BAD — the "human-readable string" trap
console.log('user ' + userId + ' failed checkout in ' + ms + 'ms');
// → user 8831 failed checkout in 2471ms
```

That line is readable by a human and *completely opaque to a machine*. You cannot ask your log system "show me all failed checkouts over 2000ms for users in Brazil," because there is no field called `duration_ms`, no field called `country`, no field called anything. There is only a sentence. To query it you'd have to write a regex — and the moment someone changes the wording, your regex breaks.

Now emit JSON instead:

```javascript
// GOOD — structured logging: every value is a queryable field
logger.warn({
  event: 'checkout_failed',
  userId: 8831,
  country: 'BR',
  platform: 'android',
  durationMs: 2471,
  reason: 'PAYMENT_DECLINED',
  traceId: 'a1b2c3d4e5f6',      // ties this line to metrics + traces
  service: 'checkout-service',
  version: '2.14.1'
});
// → {"level":"warn","event":"checkout_failed","userId":8831,"country":"BR", ...}
```

Now the "unknown unknown" question from the top of this doc becomes a query:

```
event="checkout_failed" AND country="BR" AND platform="android" AND durationMs > 2000
```

That is the difference between monitoring and observability, in one code change. In Node, use `pino` (fast, JSON by default) or `winston`. Never build log lines with `+` or template literals.

**Log levels — and when each one is correct:**

| Level | Use it when | Example |
|-------|-------------|---------|
| `error` | Something failed and a user was affected or data is at risk. **Someone should look at this.** | Payment gateway unreachable, DB write failed |
| `warn` | Something is wrong but the system recovered or degraded gracefully | Retry succeeded on attempt 2; cache miss rate high; deprecated API called |
| `info` | Notable business events worth keeping | "Order placed", "user signed up", "service started" |
| `debug` | Detail useful only while diagnosing | Query parameters, intermediate state |
| `trace` | Firehose. Off in production. | Every function entry/exit |

Rule of thumb: **if `error` fires and nobody needs to act, it isn't an error — it's a `warn`.** Teams that log recoverable retries as `error` end up with 40,000 "errors" a day and stop reading them. That's how you miss the real one.

**What to NEVER log — non-negotiable:**

- Passwords, in any form — plaintext, "just the first 3 characters", anything
- Auth tokens, JWTs, API keys, session cookies, refresh tokens
- Full credit card numbers, CVV, or full bank account numbers (**PCI-DSS violation**)
- PII you don't have a lawful reason to store: full names, home addresses, national ID numbers, health data (GDPR/HIPAA exposure)
- Entire request bodies "just in case" — that's how secrets leak by accident

Redact at the logger, not at each call site — one central redaction config, so no engineer can forget:

```javascript
import pino from 'pino';

// Redaction lives in ONE place so an individual dev can never leak by accident.
export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: {
    paths: [
      'req.headers.authorization',
      'req.headers.cookie',
      'password',
      '*.password',
      'card.number',
      'card.cvv'
    ],
    censor: '[REDACTED]'
  },
  base: { service: 'checkout-service', version: process.env.APP_VERSION }
});
```

Log an ID (`userId: 8831`, `cardLast4: '4242'`), never the secret itself.

**Sampling.** At high traffic you cannot afford to keep every `info` line. Keep 100% of `error` and `warn`, and sample the happy path — e.g. keep 1 in 100 successful requests. You lose nothing statistically, and your bill drops by ~99% for the noisy tier.

---

### 2. Pillar Two — Metrics

A metric is a **numeric time series**: a name, a value, a timestamp, and a set of labels (`http_requests_total{service="checkout", status="500"}`).

The magic property: metrics are **pre-aggregated**. The service doesn't ship one data point per request — it keeps a counter in memory and reports the number every 15 seconds. That means **a metric costs the same whether you serve 1 request per second or 1 million.** Logs scale linearly with traffic; metrics scale with the number of *distinct label combinations*. This is why metrics are the backbone of dashboards and alerts, and logs are not.

**Good at:** Cheap, fast, aggregatable. "Is the system healthy?" over any time window. Alerting.
**Bad at:** Zero per-request detail. A metric can tell you `error_rate = 4.2%`. It can **never** tell you *why*, or *which* users. For that you need logs and traces. A metric is a smoke alarm, not an investigation.

**The four metric types:**

| Type | What it is | Example | Node usage |
|------|-----------|---------|-----------|
| **Counter** | Only goes up (resets on restart) | `orders_placed_total`, `http_errors_total` | `counter.inc()` |
| **Gauge** | Goes up and down; a snapshot of "right now" | `active_connections`, `queue_depth`, `memory_bytes` | `gauge.set(42)` |
| **Histogram** | Buckets observations into ranges, so percentiles can be computed | request duration | `histogram.observe(0.34)` |
| **Summary** | Computes percentiles *client-side* on each instance | request duration | `summary.observe(0.34)` |

**Why you MUST use a histogram to get p99.** This is the concept interviewers probe, and most candidates get it wrong.

You cannot compute a percentile from an average. An average throws away the shape of the distribution — it collapses a million numbers into one, and the information about the tail is destroyed forever. If your average latency is 100ms, the p99 could be 120ms (everyone's fine) or 4,000ms (1% of users are suffering badly). The average cannot distinguish these.

And the classic mistake: **averaging p99s across servers is mathematically meaningless.** If server A reports p99 = 200ms and server B reports p99 = 800ms, the fleet-wide p99 is **not** 500ms. Percentiles are not additive. Server A may have handled 100,000 requests and server B only 12. The true p99 of the combined population could be anywhere.

A histogram fixes this because it ships **bucket counts**, and counts *are* additive:

```
Server A buckets:  ≤50ms: 8000 | ≤100ms: 1500 | ≤500ms: 400 | ≤1s: 90 | ≤5s: 10
Server B buckets:  ≤50ms:    2 | ≤100ms:    3 | ≤500ms:   2 | ≤1s:  3 | ≤5s:  2
                   ------------------------------------------------------------
Fleet (sum):       ≤50ms: 8002 | ≤100ms: 1503 | ≤500ms: 402 | ≤1s: 93 | ≤5s: 12
```

Now you add the buckets, then compute the percentile **once, over the true combined distribution.** That is correct. (A `summary` computes the percentile on each box and cannot be re-aggregated — which is exactly why Prometheus users reach for histograms almost always.)

```javascript
import { Counter, Gauge, Histogram, register } from 'prom-client';

const httpRequests = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status']   // keep cardinality LOW
});

const httpDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Request duration in seconds',
  labelNames: ['method', 'route'],
  // Buckets chosen around your SLO (300ms) so the percentile near it is precise.
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5]
});

const queueDepth = new Gauge({ name: 'jobs_queue_depth', help: 'Pending jobs' });

export function metricsMiddleware(req, res, next) {
  const done = httpDuration.startTimer({ method: req.method, route: req.route?.path ?? 'unknown' });
  res.on('finish', () => {
    done();
    httpRequests.inc({ method: req.method, route: req.route?.path ?? 'unknown', status: res.statusCode });
  });
  next();
}

// Prometheus SCRAPES this endpoint every ~15s — you don't push.
app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

**The cardinality trap.** Never put a unique value in a metric label — `userId`, `orderId`, `email`, raw URL with IDs in it. Every distinct label combination creates a *new time series*. Add `userId` as a label with 10 million users and you have just created 10 million time series and killed your metrics backend. High-cardinality data belongs in **logs and traces**, never in metrics. This is the #1 way real teams blow up their observability stack.

---

### 3. Pillar Three — Traces

A **trace** is the end-to-end journey of ONE request across every service it touches.

- **Trace ID** — a unique ID for the whole request (e.g. `a1b2c3d4e5f6`). Every service that participates stamps it on everything.
- **Span** — one unit of work inside the trace: "checkout-service handled POST /orders" (120ms), or "postgres SELECT users" (8ms). A span has a start time, duration, name, and attributes.
- **Parent span ID** — how spans form a tree. The payment span's parent is the checkout span, which is the parent of the gateway span. From the tree you reconstruct the whole call graph.
- **Context propagation** — the trace ID and current span ID travel between services in an HTTP header. The W3C standard is `traceparent`:
  `traceparent: 00-a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4-00f067aa0ba902b7-01`
  If a service forgets to forward that header, the trace **breaks in half** and you lose the chain. Use an instrumentation library (OpenTelemetry) rather than doing this by hand.

**Sampling — you cannot trace 100% of traffic at scale.** A trace is far more expensive than a metric and often more expensive than a log line. Two strategies:

| Strategy | How it works | Pro | Con |
|----------|--------------|-----|-----|
| **Head-based** | Decide at the very first service, before the request runs: "keep 1% of traces." Decision travels in the header. | Cheap, simple, no buffering | You will randomly throw away the interesting slow/failed traces |
| **Tail-based** | Buffer all spans of a trace; decide AFTER it completes: "keep it if it errored or took >1s" | Keeps exactly the traces you actually want | Needs a collector holding spans in memory; more infra, more cost |

The pragmatic answer most teams land on: head-sample the boring happy path at ~1%, and keep **100% of traces that error or exceed the latency SLO** via tail sampling. That is the shape of the answer an interviewer wants.

---

### 4. What to actually measure — the frameworks

Don't guess. There are three well-worn frameworks, and knowing them by name is worth real points.

**The Four Golden Signals (Google SRE).** If you can only have four graphs per service:

1. **Latency** — how long requests take. Split successful vs failed! A fast 500 will otherwise drag your latency graph *down* and hide the pain.
2. **Traffic** — demand on the system (requests/sec, messages/sec).
3. **Errors** — rate of failed requests (explicit 5xx, and implicit — a 200 with the wrong body).
4. **Saturation** — how full the system is. The resource closest to its limit (memory, disk, thread pool, connection pool). Saturation is your *leading* indicator: it predicts the outage before it happens.

**RED — for services (things that handle requests):**
- **R**ate — requests per second
- **E**rrors — failed requests per second
- **D**uration — distribution of latency (histogram, not average!)

**USE — for resources (CPU, disk, memory, connection pools):**
- **U**tilization — % of time the resource was busy
- **S**aturation — how much work is *queued* waiting for it
- **E**rrors — error events (disk write failures, dropped packets)

Simple mapping: **RED for every service you own, USE for every machine and pool underneath them.**

**Measure percentiles, not averages.** Recall from [06 — Latency and Throughput](./06-latency-and-throughput.md) that the average hides the pain. Concretely: 1,000,000 requests/day, average 100ms, sounds great. But if p99 = 5s, then **10,000 requests a day take 5 seconds.** Those aren't 10,000 unlucky random users either — they're disproportionately your heaviest users, the ones with the most data, the biggest carts, the most followers. The average tells you the system is fine. The p99 tells you your best customers hate you.

Always graph **p50, p90, p99 and p99.9** side by side. p50 is the typical experience; p99 is the experience you get judged on.

---

### 5. Alerting done right — the most practically valuable section

**Alert on SYMPTOMS, not causes.**

A symptom is something a *user feels*: error rate is 5%, checkout p99 is 4 seconds, orders-per-minute dropped 40% versus the same hour last week. A cause is an internal condition: CPU is at 92%, a pod restarted, disk is 80% full.

If CPU is at 92% and every user is happy and fast — **who cares?** That's a machine doing its job. Paging a human at 3am for it is pure damage. Conversely, if error rate is 5%, you must wake someone *even if every internal metric looks green*, because users are actually being hurt.

| Page a human (symptom) | Do NOT page — dashboard/ticket only (cause) |
|---|---|
| Checkout error rate > 2% for 5 min | CPU > 90% |
| p99 latency > 2s for 10 min | Memory at 85% |
| Orders/min dropped 40% vs last week | A pod restarted once |
| Payment success rate < 95% | Cache hit rate fell from 95% → 91% |
| Queue depth growing for 30 min (never draining) | A single slow query |

Causes still belong on dashboards — that's *how you debug* once a symptom alert fires. They just aren't worth someone's sleep.

**Every alert must be actionable and urgent.** Before creating an alert, ask three questions:
1. Is a real user being harmed right now (or imminently)?
2. Does a human need to do something *now*, that automation can't do?
3. Is there a **runbook** — a short document saying what to check and what to do?

If the answer to any is "no," it is not a page. Make it a ticket, a dashboard, or delete it.

**Alert fatigue is a real cause of outages.** This is not a soft concern. If your on-call gets 30 alerts a night and 29 are noise, they will start reflexively acknowledging and going back to sleep — and one night the 30th one is real, and it gets acknowledged too. **If your engineers routinely ignore alerts, your alerting is broken, and it is now an availability problem.** Ruthlessly delete alerts that never led to action.

**SLOs and error budgets — the disciplined way to decide when to page.**

Recall from [07 — Availability and Reliability](./07-availability-and-reliability.md) the nines. An **SLO** (Service Level Objective) is a target you commit to, e.g. *"99.9% of checkout requests succeed in under 300ms, measured over 30 days."*

The **error budget** is what's left: 100% − 99.9% = **0.1%**. Over 30 days with 100M requests, that's **100,000 requests you are allowed to fail.** That is a budget, and budgets are meant to be *spent* — a service at 100% reliability is a service you over-invested in and under-shipped features for.

Now alerting has a principled rule: **page on burn rate**, not on raw thresholds.

```
Burn rate = how fast you're consuming the 30-day error budget.

Burn rate 1x  → you'll exactly exhaust the budget at day 30. Fine.
Burn rate 14x → you'll exhaust a 30-day budget in ~2 days. PAGE SOMEONE NOW.
Burn rate 2x  → you'll exhaust it in 15 days. Ticket, not a page. Look at it Monday.
```

That single idea kills most alert noise: a 30-second blip that burns 0.001% of the budget shouldn't wake anyone. A sustained error rate that will blow the whole quarter's budget by Thursday absolutely should.

And when the budget is **exhausted**, the team stops shipping features and spends the sprint on reliability. That turns "should we prioritize reliability?" from an argument into arithmetic.

**Runbooks.** Every alert links to a page that says: what this alert means, what the user impact is, the first three things to check, and the known mitigations ("roll back the last deploy", "scale the read replicas", "flip the feature flag off"). A 3am page with no runbook forces an engineer to rediscover the system from scratch while it's on fire.

---

### 6. The thread that ties all three pillars together — trace IDs in Node

Logs, metrics, and traces are only worth their combined price if you can **jump between them**. Alert fires (metric) → open a slow trace (trace) → click into the logs for that exact request (logs). The connector is a single ID.

The naive approach is to thread `traceId` through every function signature as an argument. That poisons your entire codebase — every function now takes a parameter it doesn't care about, just to pass it down.

Node has a built-in answer that most people don't know exists: **`AsyncLocalStorage`**. It gives you a per-request "ambient context" that survives across `await`s, callbacks, and timers — like a thread-local variable, but for async. Set it once at the edge of the request, read it anywhere, including inside the logger itself.

```javascript
// context.js
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

export const requestContext = new AsyncLocalStorage();

// Runs ONCE per request. Everything downstream — every await, every callback,
// every DB call in this request — can now read the same context object.
export function contextMiddleware(req, res, next) {
  const store = {
    // Reuse the caller's trace ID if there is one, so the trace spans services.
    traceId: req.header('x-trace-id') ?? randomUUID(),
    userId: req.user?.id ?? null,
    route: req.path
  };
  res.setHeader('x-trace-id', store.traceId);   // give it back so clients can quote it
  requestContext.run(store, () => next());
}
```

```javascript
// logger.js — the logger reads the context itself; call sites never pass traceId.
import pino from 'pino';
import { requestContext } from './context.js';

const base = pino({ redact: ['req.headers.authorization', 'password'] });

// mixin() is called on EVERY log line and merges its return value in.
export const logger = pino({
  mixin() {
    const store = requestContext.getStore();
    return store ? { traceId: store.traceId, userId: store.userId } : {};
  },
  redact: { paths: ['req.headers.authorization', 'password', '*.password'], censor: '[REDACTED]' }
});
```

```javascript
// Anywhere, however deep, with zero plumbing:
async function chargeCard(order) {
  logger.info({ event: 'charge_started', orderId: order.id });   // traceId auto-attached
  try {
    return await paymentGateway.charge(order.total);
  } catch (err) {
    // Still auto-attached — even after an await, even inside a catch.
    logger.error({ event: 'charge_failed', orderId: order.id, reason: err.code });
    throw err;
  }
}
```

And crucially — **propagate it outward** so the next service joins the same trace:

```javascript
import { requestContext } from './context.js';

export async function callService(url, body) {
  const { traceId } = requestContext.getStore() ?? {};
  return fetch(url, {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'x-trace-id': traceId          // ← the entire chain hangs off this one header
    },
    body: JSON.stringify(body)
  });
}
```

Now a user emails you "my order failed, the page said reference a1b2c3d4." You paste `a1b2c3d4` into your log search and get **every log line from all seven services** for that one request, in order. That is observability, and it cost you about forty lines of code.

(In production, prefer **OpenTelemetry's** auto-instrumentation, which does exactly this using the W3C `traceparent` header and also builds the span tree for you. The code above is the mental model of what it's doing under the hood.)

---

### 7. The tooling landscape, and what it costs

| Pillar | Common tools | Note |
|--------|--------------|------|
| **Metrics** | **Prometheus** (scrape + store) + **Grafana** (dashboards); Datadog, CloudWatch | Prometheus *pulls* from your `/metrics` endpoint |
| **Logs** | **ELK** (Elasticsearch + Logstash + Kibana), **Grafana Loki**, Splunk, Datadog | Loki indexes only labels, not full text — much cheaper |
| **Traces** | **Jaeger**, **Zipkin**, Tempo, Datadog APM, AWS X-Ray | |
| **All three** | **OpenTelemetry (OTel)** | The vendor-neutral CNCF standard: one set of SDKs and one wire format for logs, metrics, and traces |

**Instrument with OpenTelemetry, not with a vendor SDK.** OTel is the industry's answer to lock-in: you instrument your code once, and you can swap Jaeger for Datadog for Honeycomb by changing an exporter config — not by rewriting every service. That is now the default correct answer in an interview.

**The cost reality — this is not a footnote.** Observability data routinely **exceeds your production data volume**. A service handling 5,000 req/s that emits 2 KB of logs per request produces:

```
5,000 req/s × 2 KB          = 10 MB/s
10 MB/s × 86,400 s          = 864 GB/day
864 GB/day × 30             = ~26 TB/month  (from ONE service)
```

At roughly $0.50–$2.00 per GB ingested on a managed platform, that is **$13,000–$52,000 per month for one service's logs.** Teams genuinely discover their observability bill has passed their compute bill.

You control it with three levers:
1. **Sampling** — keep 100% of errors, 1% of the happy path.
2. **Retention tiers** — hot/searchable for 7 days, warm for 30, cold object storage (S3/Glacier) for a year of compliance data. Cold storage is ~50x cheaper.
3. **Move data to the right pillar** — if you're grepping logs to count something, that thing should have been a metric. If you're logging every function entry, that should have been a span.

---

## Visual / Diagram description

### Diagram 1: The observability pipeline

```
   ┌───────────────────────────────────────────────────────────────┐
   │                    YOUR SERVICES (Node.js)                     │
   │                                                                │
   │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │
   │  │ Gateway  │──▶│ Checkout │──▶│ Payment  │──▶│Inventory │   │
   │  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘   │
   │       │              │              │              │          │
   │       └──── each emits logs + metrics + spans ─────┘          │
   │          all stamped with the SAME traceId: a1b2c3d4          │
   └───────────────────────────┬───────────────────────────────────┘
                               │  OpenTelemetry SDK
                               ▼
                  ┌────────────────────────────┐
                  │   OTel Collector           │
                  │  (batch, sample, redact,   │
                  │   enrich, then fan out)    │
                  └───┬─────────┬──────────┬───┘
                      │         │          │
          ┌───────────▼──┐  ┌───▼──────┐  ┌▼─────────────┐
          │  Prometheus  │  │  Loki /  │  │   Jaeger     │
          │  (METRICS)   │  │   ELK    │  │  (TRACES)    │
          │              │  │  (LOGS)  │  │              │
          └───────┬──────┘  └────┬─────┘  └──────┬───────┘
                  │              │               │
                  └──────────────┼───────────────┘
                                 ▼
                        ┌─────────────────┐          ┌──────────────┐
                        │     Grafana     │─────────▶│ Alertmanager │
                        │  (dashboards,   │  rules   │  → PagerDuty │
                        │   correlation)  │  fire    │  → Slack     │
                        └─────────────────┘          └──────────────┘
```

Read it left to right: your services emit all three signal types through one SDK; a **collector** sits in the middle doing the batching, sampling and redaction (so your app doesn't have to and so you can change backends without redeploying app code); each signal lands in the store best suited to it; Grafana unifies them for a human; alert rules evaluate the *metrics* and wake someone via PagerDuty.

### Diagram 2: A trace waterfall — this visual IS the payoff

A user reports "checkout took 3 seconds." One trace tells you everything:

```
TRACE a1b2c3d4  —  POST /api/checkout  —  total 3,010 ms
Time ──▶ 0ms        500ms      1000ms     1500ms     2000ms     2500ms    3000ms
         │           │          │          │          │          │         │
gateway  ├───────────────────────────────────────────────────────────────┤  3010ms
  └ auth ├─┤ 22ms
  └ checkout-service
         ├─────────────────────────────────────────────────────────────┤   2960ms
      └ cart-svc  GET /cart
           ├──┤ 45ms
      └ inventory-svc  POST /reserve
              ├────┤ 88ms
      └ pricing-svc  POST /quote
                   ├──────────────────────────────────────────────┤        2680ms  ◀── ★
             └ redis GET prices:*
                     ├─┤ 3ms   (MISS)
             └ postgres  SELECT ... FROM price_rules  (no index on region_id)
                        ├───────────────────────────────────────┤          2610ms  ◀── ★★
      └ payment-svc  POST /charge
                                                                  ├────┤     115ms
      └ notify-svc  (async, fire-and-forget)
                                                                       ├┤    9ms
```

**How to read it.** Each bar is a **span**; indentation shows the parent/child tree; the horizontal position is *when* it happened and the length is *how long* it took. Your eye goes straight to the longest bar that isn't just containing other bars: `pricing-svc` ate 2,680 of the 3,010 ms. Zoom in one more level and the real culprit appears — a single Postgres query taking 2,610 ms, because a cache miss fell through to a table scan on `price_rules`.

Without the trace, you'd have four teams each insisting their service is fine (and all four would be right — they *are* fine; they're just waiting on pricing). With the trace, you have the answer in eight seconds and it's a missing index. **This is what you buy with tracing.**

---

## Real world examples

### 1. Google — the Four Golden Signals and the error budget

Google's SRE practice, published in the *Site Reliability Engineering* book, gave the industry two of the ideas in this doc. The **Four Golden Signals** (latency, traffic, errors, saturation) is their prescribed minimum instrumentation for any user-facing system. The **error budget** is their organizational mechanism: a service has an SLO, the gap to 100% is a budget, and when a team burns through it, feature launches stop until reliability is restored. It turns a political argument between product and SRE into a number both sides agreed to in advance. Google's Dapper paper (2010) is also the direct ancestor of Jaeger, Zipkin, and OpenTelemetry tracing.

### 2. Twitter/X — Zipkin, and why distributed tracing exists at all

Zipkin was built at Twitter and open-sourced in 2012, directly inspired by Dapper. The motivating problem was exactly the one in this doc: a single tweet-timeline request fanned out across dozens of internal services, and when the timeline got slow, no team could tell whose service was responsible. Zipkin's trace-ID-plus-span model — propagate an ID in headers, report timing spans to a central collector, render a waterfall — is now the standard shape of every tracing system in the industry.

### 3. Netflix — symptom-based alerting on business metrics

Netflix has publicly described monitoring **SPS (starts per second)** — the rate at which members successfully begin playing something — as a primary health signal. It's a business metric, not a machine metric, and that's the whole point: it is the purest possible *symptom*. If SPS drops below its expected curve for this hour of this day of the week, something is broken for real users, no matter how green every CPU graph looks. Conversely, a server can be pinned at 95% CPU and if SPS is normal, nobody gets woken up. That is "alert on symptoms, not causes" put into practice at scale.

---

## Trade-offs

| Pillar | Strength | Weakness | Cost model |
|--------|----------|----------|-----------|
| **Logs** | Maximum detail, arbitrary context, per-request truth | Expensive, noisy, slow to aggregate | Scales **linearly with traffic** — the expensive one |
| **Metrics** | Cheap, fast, aggregatable, ideal for alerts | No per-request detail; can never tell you *why* | Scales with **cardinality**, not traffic |
| **Traces** | Shows the bottleneck across services instantly | Requires propagation everywhere; must be sampled | Between the two; sampling is mandatory |

| Decision | Benefit | Cost |
|----------|---------|------|
| Log everything at `debug` in prod | Nothing is ever missing during an incident | Massive bill; the real signal drowns in noise |
| Sample logs at 1% | ~99% cheaper on the happy path | The one request you desperately want may be gone |
| Trace 100% of requests | Perfect visibility | Prohibitively expensive; often slows the app |
| Head-based sampling | Simple, cheap, no buffering | Randomly discards the slow and failed traces you needed |
| Tail-based sampling | Keeps exactly the interesting traces | Collector must buffer spans; more infra to run |
| Alert on every anomaly | Nothing slips through | Alert fatigue → alerts get ignored → real outage missed |
| Alert only on SLO burn | Every page is real and worth waking for | Slow burns can go unnoticed for a while (use a 2-window rule) |

**The sweet spot:** Metrics on **everything** (they're cheap and they're what you alert on). Traces sampled — 1% of the happy path plus **100% of errors and slow requests**. Logs structured as JSON, `error`/`warn` kept in full, `info` sampled, `debug` off in production. Alert on **user-facing symptoms tied to an SLO**, never on CPU. And put a trace ID on every single line of all three.

**Rule of thumb:** If you find yourself grepping logs to *count* something, that should have been a metric. If you're reading logs from four services to figure out *where* time went, that should have been a trace.

---

## Common interview questions on this topic

### Q1: "What's the difference between monitoring and observability?"
**Hint:** Monitoring answers questions you knew to ask in advance — known unknowns ("is CPU > 90%?"). Observability is the property of being able to answer questions you *never anticipated*, from data you already emit — unknown unknowns ("why are Android users in Brazil slow on checkout, only since Tuesday?"). Monitoring is a subset of what an observable system gives you. Microservices force you into the second world because you can no longer predict where the failure will be.

### Q2: "Your p99 latency alert fires at 3am. Walk me through your debugging."
**Hint:** Show the pillar hand-off. (1) **Metrics** first: which service's RED duration spiked? Did traffic or error rate spike with it? Is saturation up (connection pool exhausted)? Did anything deploy? (2) **Traces**: pull traces from the affected window that exceeded the SLO, look at the waterfall, find the longest non-container span. (3) **Logs**: filter by the trace ID of one slow request to see the exact error/query. (4) Check the runbook, mitigate first (roll back / flip the flag / scale out), root-cause after. Naming the metrics → traces → logs sequence is the whole answer.

### Q3: "Why can't I just average my p99 latency across servers?"
**Hint:** Percentiles are not additive — you can't average them and get a meaningful number, because each server handled a different number of requests and the underlying distributions differ. If A's p99 is 200ms and B's is 800ms, the fleet p99 is not 500ms. The fix is **histograms**: each server ships *bucket counts*, which ARE additive; you sum the buckets across the fleet and compute the percentile once over the true combined distribution. This is exactly why Prometheus histograms beat summaries.

### Q4: "How would you decide what to alert on?"
**Hint:** Alert on **symptoms users feel** (error rate, latency, "orders/min dropped 40%"), never on **causes** (CPU high, pod restarted) — those go on dashboards. Every alert must be (a) user-impacting, (b) actionable right now, (c) attached to a runbook. Formalize it with an **SLO and error budget**, and page on **burn rate**: 14x burn = wake someone; 2x burn = a ticket for Monday. And say the words *alert fatigue* — noisy alerting causes outages, because ignored alerts include the real one.

### Q5: "How do you correlate a log line in service A with a log line in service D?"
**Hint:** A **trace ID** generated at the edge (API gateway), propagated to every downstream service in a header (`traceparent` / `x-trace-id`), and stamped onto every log line, span, and error. In Node, `AsyncLocalStorage` gives you per-request ambient context so you don't thread the ID through every function signature — hook it into the logger's `mixin` and every line gets it for free. Bonus: return the trace ID to the client so a user's bug report includes it.

### Q6: "Observability is costing us more than our production servers. What do you do?"
**Hint:** Three levers. (1) **Sampling** — 100% of errors and slow requests, ~1% of the happy path. (2) **Retention tiers** — 7 days hot/searchable, 30 days warm, then cold object storage for compliance (~50x cheaper). (3) **Move data to the cheapest pillar that answers the question** — anything you're counting in logs should be a metric; anything you're logging to see where time went should be a span. Also kill high-cardinality metric labels, which explode the time-series count for no benefit.

---

## Practice exercise

### Instrument a two-service checkout flow end to end

Build (or sketch, if you're short on time) a tiny Express system with **two services**: `order-service` (port 3000) and `payment-service` (port 3001). `POST /orders` on the first calls `POST /charge` on the second. Make `payment-service` deliberately slow: `await sleep(1200)` on ~10% of requests, and return a 500 on ~5%.

**Produce all of the following:**

1. **Structured logging.** Set up `pino` in both services with a central `redact` config. Log `order_received`, `charge_started`, `charge_failed`. Every line must be JSON with queryable fields — no string concatenation anywhere.

2. **Correlation.** Use `AsyncLocalStorage` in both services. `order-service` generates a `traceId` if the incoming request has none, forwards it as `x-trace-id`, and `payment-service` picks it up. Verify by curl-ing one order and confirming the **same** `traceId` appears in both services' logs.

3. **Metrics.** Add `prom-client` to both. Expose `/metrics`. Emit: a **counter** `http_requests_total{route,status}`, a **histogram** `http_request_duration_seconds` with buckets `[0.05, 0.1, 0.3, 0.5, 1, 2, 5]`, and a **gauge** for in-flight requests. Hit the system 200 times with a loop and compute the p99 from the histogram buckets **by hand** — literally add up the bucket counts until you cross 99% of total. Do it once manually and you will never forget why the buckets matter.

4. **Alert design (on paper, no code).** Write **three** alert rules for this system. For each one state: the exact condition, whether it's a **symptom or a cause**, whether it **pages** or files a ticket, and a three-line runbook. At least one must be based on an **SLO burn rate** — pick an SLO first ("99.5% of orders succeed in under 500ms over 30 days") and derive the budget.

5. **The write-up.** In five sentences, answer: *your p99 just doubled — which pillar do you look at first, second, and third, and what does each one tell you that the others can't?*

**Deliverable:** two running services, one `/metrics` endpoint each, log output showing a shared trace ID across both, your hand-computed p99, and the three alert definitions.

---

## Quick reference cheat sheet

- **Monitoring** answers questions you knew to ask (known unknowns). **Observability** answers questions you didn't (unknown unknowns).
- **The three pillars:** **Logs** = what happened to THIS request. **Metrics** = is the system healthy? **Traces** = WHICH service is slow?
- **Structured logging:** emit JSON with fields, never concatenated strings. A string log line is unqueryable and therefore nearly worthless.
- **Never log:** passwords, tokens, JWTs, API keys, full card numbers, CVV, PII. Redact centrally in the logger config, not at each call site.
- **Metrics are pre-aggregated** — they cost the same at 1 req/s and 1M req/s. Logs scale linearly with traffic. That's why you alert on metrics.
- **Metric types:** counter (up only), gauge (up/down), histogram (buckets → percentiles), summary (client-side percentiles, can't be re-aggregated).
- **You need histograms for p99.** You cannot compute a percentile from an average, and **averaging p99s across servers is meaningless** — percentiles aren't additive; bucket counts are.
- **Never put user IDs / order IDs in metric labels.** High cardinality kills the metrics backend. That data belongs in logs and traces.
- **Trace anatomy:** trace ID (whole journey) → spans (units of work) → parent span ID (the tree) → propagated via the `traceparent` header.
- **Sampling:** head-based (decide up front, cheap, may drop the good stuff) vs tail-based (decide after, keeps errors and slow requests, needs a buffering collector).
- **Four Golden Signals:** Latency, Traffic, Errors, Saturation. **RED** (Rate, Errors, Duration) for services. **USE** (Utilization, Saturation, Errors) for resources.
- **Alert on symptoms, not causes.** Error rate and latency page a human; CPU at 90% does not. Every alert needs a runbook and must be actionable.
- **Alert fatigue causes outages** — if engineers routinely ignore alerts, your alerting is broken and it's now an availability problem.
- **SLO → error budget → burn rate.** 99.9% over 30 days = 0.1% budget. 14x burn = page now; 2x burn = a Monday ticket.
- **Node superpower:** `AsyncLocalStorage` + a pino `mixin` puts the trace ID on every log line automatically, with zero plumbing through function arguments.
- **Tooling:** Prometheus + Grafana (metrics), ELK/Loki (logs), Jaeger/Zipkin (traces), **OpenTelemetry** as the vendor-neutral standard for all three.
- **Cost:** observability data often exceeds production data volume. Sampling and retention tiers are not optional at scale.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [73 — Circuit Breaker](./73-circuit-breaker.md) — you can only break the circuit on a failing dependency if you're measuring its error rate in the first place |
| **Next** | [72 — Service Discovery](./72-service-discovery.md) — the other piece of microservice plumbing; discovery tells services where to find each other, observability tells you what happened when they did |
| **Related** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) — microservices are what make observability mandatory rather than nice-to-have |
| **Related** | [06 — Latency and Throughput](./06-latency-and-throughput.md) — why you measure p99, not the average; the average hides the users who are suffering |
| **Related** | [07 — Availability and Reliability](./07-availability-and-reliability.md) — the nines, SLOs, and error budgets that turn alerting from guesswork into arithmetic |
