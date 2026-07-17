# 89 — Serverless Architecture — Functions as a Service
## Category: HLD Components

---

## What is this?

Serverless means you upload a **single function** — not a server, not a container, just a function — and a cloud platform runs it for you whenever something triggers it. There are absolutely still servers underneath; you just never provision them, patch them, size them, or scale them. Someone else does, and they only charge you for the milliseconds your code was actually executing.

The analogy: **owning a car vs. taking a taxi.** A car (a traditional server, or a container that's always up) costs you money whether you drive it or not — insurance, parking, depreciation. A taxi costs you nothing when you're sitting at home, and you pay strictly per trip. If you commute two hours a day, the car is cheaper. If you take three short trips a month, the taxi is dramatically cheaper. Serverless is the taxi. The entire rest of this document is really just working out *when you're the commuter and when you're the occasional rider.*

---

## Why does it matter?

**What breaks if you don't understand this:** engineers reach for Lambda because it's trendy, ship a user-facing API on it, and then get paged because p99 latency is 2.4 seconds (cold starts) and Postgres is refusing connections (1,000 function instances each opened their own). Both failures are completely predictable from first principles — and both are *the* things interviewers probe.

**The real-work angle:** serverless is genuinely the correct, boring answer for a huge slice of production work — cron jobs, webhook receivers, "resize this image when it lands in S3," glue between two SaaS products, internal tools with 40 users. Choosing a $0.20/month Lambda over a $30/month always-on container for a job that runs 200 times a day is real money and real operational relief. You stop patching an OS.

**The interview angle:** "When would you *not* use serverless?" is one of the highest-signal questions in HLD interviews, because the naive answer ("it scales infinitely, so always!") instantly marks you as someone who has never run it. The strong answer names cold starts, the connection-pool collision, the 15-minute limit, and — crucially — **does the cost arithmetic to find the crossover point** where serverless becomes more expensive than servers. Almost nobody does that arithmetic. Do it, and you separate yourself.

---

## The core idea — explained simply

### The Vending Machine vs. The Restaurant Analogy

Imagine you sell coffee.

**Option A — you open a coffee shop (a traditional server / container).** You sign a lease. You hire a barista who stands there from 6am to 10pm. At 3am the shop is closed but you're still paying rent. At 8am there's a rush and one barista can't cope, so you hire a second — but now that second barista is also idle at 2pm. You pay for *capacity*, and capacity is a guess you make in advance.

**Option B — you install a magic vending machine (serverless).** Someone walks up and presses a button. The machine wakes up, grinds beans, pours coffee, and goes back to sleep. Ten people arrive at once? Ten identical machines materialize instantly, side by side. Nobody arrives for six hours? Zero machines exist and you pay zero. You are billed *per cup*, by the millisecond of grinding.

The catch — and it's the whole story — is that **a machine that was asleep takes a few seconds to warm up.** The first customer of the morning waits. That's a **cold start**. A machine that just served someone is already warm, and the next customer is served instantly — that's a **warm invocation**. Your job as an engineer is to make the warm-up as short as possible and to arrange things so most customers hit a warm machine.

And there's a second catch nobody sees coming. Each vending machine needs a phone line to the bean supplier to check inventory. The supplier's switchboard has **100 lines**. When your traffic spikes and 1,000 machines materialize, all 1,000 machines dial the supplier at once. The switchboard melts. You have taken down your own supplier — and *your supplier is your database*.

**Mapping the analogy back:**

| Analogy | Technical concept |
|---|---|
| The coffee shop with a leased barista | An always-on EC2 instance / container / Kubernetes pod |
| Paying rent at 3am with no customers | Paying for idle compute — the thing serverless eliminates |
| A machine materializing on a button press | A Lambda invocation triggered by an event |
| Ten machines appearing at once | Automatic horizontal scaling; each concurrent request gets its **own** instance |
| The machine warming up for the first customer | **Cold start** — container allocation + runtime + your code + init |
| A machine that just served someone | **Warm instance** — the platform keeps it around for a few minutes |
| Billing per cup, by the millisecond | Pay-per-invocation + GB-second pricing; **zero cost at zero traffic** |
| The supplier's 100-line switchboard | Postgres `max_connections = 100` |
| 1,000 machines dialing at once | The serverless connection-exhaustion problem |
| Paying to keep 5 machines pre-warmed | **Provisioned concurrency** — which reintroduces the idle cost you were escaping |

That last row is the honest punchline. Serverless's superpower is *not paying for idle*. Every mitigation for cold starts chips away at that superpower. Understanding that tension **is** understanding serverless.

---

## Key concepts inside this topic

### 1. The defining properties — what actually makes something "serverless"

Four properties, all of which must hold:

1. **You deploy a function, not a server.** No `Dockerfile`, no port to listen on, no process to keep alive. You export a handler; the platform calls it.
2. **The platform runs it on demand, event-driven.** Nothing runs until something *happens*.
3. **It scales from zero to thousands automatically.** Not "you configure an autoscaler." There is no autoscaler to configure. Concurrency = number of in-flight requests.
4. **You pay per millisecond of execution, and nothing when idle.** This is the load-bearing property. Take it away and it's just a container.

The handler itself is stunningly plain — that's the point:

```js
// handler.js — a complete AWS Lambda function. That's the whole deployable unit.
export async function handler(event) {
  const { userId } = JSON.parse(event.body);
  const user = await fetchUser(userId);

  return {
    statusCode: 200,
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify(user),
  };
}
```

No `app.listen(3000)`. No Express. No health check endpoint. No graceful-shutdown handler. The platform owns the process lifecycle; you own one function body.

### 2. The event-driven trigger model

A function is dead code until an **event source** invokes it. This is why serverless slots so naturally into event-driven architecture (recall [68 — Event-Driven Architecture]: producers emit events, consumers react, and nobody blocks on anybody). Serverless is arguably the purest consumer runtime ever built — a consumer that costs nothing when there are no events.

| Trigger | Concrete example | Shape of the `event` object |
|---|---|---|
| **HTTP** (via API Gateway) | `POST /orders` from your React app | `{ httpMethod, path, headers, body }` |
| **Object storage** | A user uploads `photo.jpg` to S3 | `{ Records: [{ s3: { bucket, object: { key } } }] }` |
| **Queue** | A message lands in SQS / Kafka | `{ Records: [{ body, messageId }] }` — often **batched** |
| **Cron / schedule** | "Every night at 2am, expire stale carts" | `{ time, resources }` |
| **DB change stream** | A row was written to DynamoDB | `{ Records: [{ eventName: 'INSERT', dynamodb: { NewImage } }] }` |
| **Another function** | Step Functions orchestration | Whatever the previous step returned |

The same code, different door:

```js
// Cron-triggered: nightly cleanup. Runs 1×/day. Costs literally cents per month.
export async function expireStaleCarts() {
  const cutoff = Date.now() - 7 * 24 * 60 * 60 * 1000; // 7 days
  const stale = await db.carts.findOlderThan(cutoff);
  await Promise.all(stale.map((c) => db.carts.delete(c.id)));
  console.log(`expired ${stale.length} carts`);
}

// S3-triggered: fires once per uploaded object. Ten uploads at once = ten
// parallel instances, no queue, no worker pool, no code from you.
export async function onImageUpload(event) {
  for (const record of event.Records) {
    const key = decodeURIComponent(record.s3.object.key);
    const original = await s3.getObject({ Bucket: BUCKET, Key: key });
    const thumb = await sharp(original.Body).resize(200, 200).toBuffer();
    await s3.putObject({ Bucket: BUCKET, Key: `thumbs/${key}`, Body: thumb });
  }
}
```

Notice what's missing from the S3 example: no polling loop, no worker pool, no concurrency limit, no queue to provision. The parallelism you'd normally hand-build is the platform's default behaviour.

### 3. Cold starts — the defining trade-off

This is the thing. Give it the space it deserves, in the doc and in the interview.

**Anatomy of a cold start.** When a request arrives and no warm instance exists, the platform must:

```
1. Allocate a micro-VM / container         ~50–200 ms   (platform's job)
2. Download & unpack your deployment bundle ~10–200 ms   (scales with bundle SIZE)
3. Start the language runtime (Node, JVM…)  ~30–100 ms   (Node fast; JVM slow)
4. Load & evaluate your module (top-level)  ~10–500 ms   (scales with imports)
5. Run your init code (DB clients, config)  ~0–2000 ms   (YOUR code — the big lever)
6. ── FINALLY invoke your handler ──
```

Steps 1–5 are the cold start. On a **warm** invocation the platform skips straight to step 6 — steps 1–5 already happened and their side effects are still in memory.

**Rough, honest numbers:**

| Function shape | Typical cold start |
|---|---|
| Slim Node.js, few MB bundle, no VPC | ~100–300 ms |
| Node.js dragging in the full AWS SDK + ORM + 40 MB of deps | ~500 ms – 1 s |
| Java / .NET (JVM startup + classloading) | ~2–8 s |
| Any runtime attached to a **VPC** (historically ENI setup) | add seconds; much improved now, but still the classic trap |
| Cloudflare Workers (V8 isolates, not containers) | **~0–5 ms** — a different mechanism entirely |

**How often does it happen?** Steady traffic → most invocations are warm and cold starts are a rounding error. Bursty traffic → *every burst* summons new instances, so a spike from 5 → 200 concurrent means ~195 cold starts, right when you're busiest. That's the cruel part: cold starts cluster exactly at your traffic peaks and land squarely in your p99.

**Mitigation 1 — move initialization OUTSIDE the handler.** This is the highest-leverage trick in serverless and the single most likely thing an interviewer wants to hear. Module-level code runs *once per instance*, not once per request, and the platform keeps warm instances alive for minutes. So anything expensive belongs at module scope.

```js
// ❌ BAD — every single invocation pays the setup cost, warm or cold.
export async function handler(event) {
  const { S3Client } = await import('@aws-sdk/client-s3'); // re-imported every call
  const s3 = new S3Client({ region: 'us-east-1' });        // rebuilt every call
  const secrets = await fetchSecretsFromSecretsManager();  // network hop EVERY call!
  const db = await connectToDatabase(secrets.dbUrl);       // new TCP+TLS EVERY call
  return await doWork(event, s3, db);
}
```

```js
// ✅ GOOD — module scope. Runs ONCE per container, then every warm invocation
// on that container reuses the already-built clients and the open connection.
import { S3Client } from '@aws-sdk/client-s3';

const s3 = new S3Client({ region: 'us-east-1' });

// A promise, not an awaited value: we kick the work off at module load and every
// invocation awaits the SAME settled promise. No top-level await needed, and
// concurrent first-invocations don't each start their own connect.
const dbReady = (async () => {
  const secrets = await fetchSecretsFromSecretsManager();
  return connectToDatabase(secrets.dbUrl);
})();

export async function handler(event) {
  const db = await dbReady; // cold: waits once. warm: resolves instantly.
  return await doWork(event, s3, db);
}
```

The difference in production is not subtle. If secrets-fetch + DB connect is 250 ms, the bad version adds 250 ms to **every request forever**; the good version adds it to roughly **one request per container lifetime** (often 1 in several hundred).

**Mitigation 2 — keep the bundle small and lazy-load the heavy stuff.**

```js
// Import only the client you use — not the whole SDK barrel.
import { PublishCommand, SNSClient } from '@aws-sdk/client-sns';

const sns = new SNSClient({});

export async function handler(event) {
  if (event.type === 'REPORT') {
    // 30 MB PDF library. Only ~2% of invocations need it, so don't pay for
    // parsing/evaluating it on the other 98% of cold starts.
    const { renderPdf } = await import('./heavy-pdf-renderer.js');
    return renderPdf(event);
  }
  await sns.send(new PublishCommand({ TopicArn: TOPIC, Message: 'ok' }));
}
```

Bundle with esbuild, tree-shake, exclude the SDK if the runtime provides it, and ship 2 MB instead of 60 MB. Bundle size is a direct multiplier on step 2 and step 4 above.

**Mitigation 3 — provisioned concurrency.** You tell AWS "keep 10 instances permanently warm." Cold starts for the first 10 concurrent requests: gone. And you now **pay for those 10 instances 24/7 whether or not anyone calls them.** Read that again — you have re-created the always-on server, with worse ergonomics and a per-request charge on top. It's a legitimate tool (use it for a known-shape morning spike), but be intellectually honest: it partly defeats the point.

**Mitigation 4 — pick a fast runtime.** Node.js and Python start in ~100 ms; Go and Rust in tens of ms; the JVM is measured in seconds. If you must run serverless on a latency path, do not do it in Java.

**The honest verdict:** for a **user-facing, p99-sensitive API** — a checkout button, a search box, an autocomplete — cold starts can be genuinely **disqualifying**. A p50 of 40 ms with a p99 of 1.8 s is a bad API, and no amount of clever bundling fully removes the tail; it only shrinks it. Serverless shines where nobody is staring at a spinner: async jobs, webhooks, cron, event processing, and low-traffic APIs where 300 ms occasionally is simply fine.

### 4. The database connection problem — the classic serverless failure

This is the best "do you actually understand this?" question in the whole topic.

Recall from [65 — Database Connection Pooling] why pooling works: a **long-lived** server process opens, say, 20 TCP connections to Postgres at boot and shares them across thousands of requests. The pool works precisely *because the process outlives the request*. Each Postgres connection is a real forked backend process on the DB server, costing ~5–10 MB of RAM — which is why `max_connections` defaults to 100 and why raising it to 5,000 is not a fix, it's a slower outage.

Now run the same code on Lambda. Traffic spikes; the platform helpfully creates **1,000 concurrent instances**. Each one is its own isolated Node process. Each one runs your module-scope init. Each one opens its own connection — or worse, its own *pool* of 10.

```
Traditional:   10 servers  × pool of 20  =    200 connections   ✅ under the 100–500 limit
Serverless:  1,000 lambdas × pool of 10  = 10,000 connections   💥 max_connections = 100
```

Postgres starts rejecting with `FATAL: sorry, too many clients already`. Every function retries. The retries create *more* instances. **You have DDoSed your own database with your own success**, and the autoscaling that was supposed to save you is the thing killing you.

Worse: a per-instance pool is actively harmful here. One Lambda instance handles **one request at a time** — the platform guarantees it. A pool of 10 connections in a container that can only ever use 1 concurrently is 9 wasted connections, multiplied by your concurrency.

```js
// ❌ BAD on serverless — a pool inside a single-concurrency instance.
import { Pool } from 'pg';
const pool = new Pool({ max: 10 }); // 10 conns × 1,000 instances = 10,000. Dead DB.

// ⚠️ LESS BAD — one connection per instance. Still 1,000 connections at peak.
const pool = new Pool({ max: 1, idleTimeoutMillis: 30_000 });
```

**The three real fixes:**

| Fix | How it works | Cost / caveat |
|---|---|---|
| **External connection pooler** | RDS Proxy / PgBouncer sits between functions and DB. 1,000 functions open cheap connections *to the proxy*; the proxy multiplexes them onto ~20 real DB connections. | An extra hop (~1–3 ms) and an extra bill. This is the standard answer. |
| **HTTP data API** | Talk to the DB over stateless HTTPS (Aurora Data API, Neon/PlanetScale serverless drivers). No TCP connection to hold at all. | Higher per-query latency; feature gaps; not every DB has one. |
| **A store with no connection concept** | DynamoDB is an HTTP API with IAM auth. There is nothing to pool. 10,000 concurrent Lambdas is a non-event. | You've now bought into a key-value data model and its access patterns. |

**And this is exactly why serverless pushes you toward DynamoDB.** It isn't fashion or vendor marketing. A stateless, infinitely-scaling compute layer needs a stateless, infinitely-scaling data layer, and a connection-oriented relational database is architecturally the *opposite* of that. If your answer to "Lambda + Postgres" is "RDS Proxy," you're right. If your answer is "or just use DynamoDB, because the connection model matches the compute model," you sound like you've shipped it.

### 5. The other hard constraints

These kill projects quietly, so know them before you commit:

- **Execution time limit.** AWS Lambda caps at **15 minutes**. Your 40-minute nightly ETL cannot run here. Either chunk it (Step Functions / fan-out) or use a container. There is no flag to raise this.
- **Ephemeral filesystem.** You get a writable `/tmp` (512 MB by default, expandable to 10 GB). It vanishes with the container, and it is *shared across invocations on the same warm container* — which is a cache opportunity and a data-leak footgun at the same time.
- **No state between invocations.** Never keep a counter, a session, or a rate-limit bucket in a module-level variable and expect it to be correct. Instance #1 has no idea instance #847 exists. Anything durable goes in Redis / DynamoDB / S3. (Module-level state is a *cache*, never a source of truth.)
- **Payload limits.** ~6 MB synchronous request/response on Lambda. Big files never travel through the function — you hand out a **pre-signed S3 URL** and let the client upload directly.
- **You cannot hold a WebSocket.** A WebSocket is a long-lived connection; a function is a short-lived execution. They are fundamentally incompatible. You need a managed WebSocket layer (API Gateway WebSocket API) that holds the socket, stores `connectionId`s, and *invokes your function per message* — and your function pushes back out through the gateway's API. The socket lives outside your code.
- **Concurrency limits are real.** Accounts have a default cap (e.g. 1,000 concurrent executions per region). "Infinite scale" is a marketing simplification.

### 6. Cost — do the arithmetic, because this is where it's actually decided

Take the well-known Lambda price shape: **~$0.20 per 1M requests** plus **~$0.0000166667 per GB-second** of execution.

**Case A — the spiky, low-traffic API.** 3,000 requests/day, 128 MB memory, 200 ms average duration.

```
Requests/month  = 3,000 × 30                     = 90,000
Request charge  = 90,000 / 1,000,000 × $0.20     = $0.018

GB-seconds      = 90,000 × 0.2 s × (128/1024) GB = 2,250 GB-s
Compute charge  = 2,250 × $0.0000166667          = $0.0375

TOTAL                                            ≈ $0.06 / month
```

Six cents. And the free tier likely makes it literally $0. The always-on alternative — one small container (0.25 vCPU / 0.5 GB on Fargate, or a t4g.small) running 24/7 — is roughly **$10–$30/month**, and you'd want two for redundancy, so call it **$20–$60**. Plus patching. **Serverless wins by ~500×.** This is not a close call.

**Case B — the crossover. Sustained 500 req/s, 24/7.** Same 128 MB / 200 ms function.

```
Requests/month  = 500 × 86,400 × 30              = 1,296,000,000  (1.296 B)
Request charge  = 1,296 × $0.20                  = $259.20

GB-seconds      = 1.296e9 × 0.2 s × 0.125 GB     = 32,400,000 GB-s
Compute charge  = 32,400,000 × $0.0000166667     = $540.00

TOTAL                                            ≈ $799 / month
```

Now price the servers. At 500 req/s and 200 ms per request, Little's Law says you need `500 × 0.2 = 100` concurrent workers. A single modern box running Node handles that comfortably — say **3 medium instances** (2 for capacity + 1 for headroom/redundancy) at ~$30–$60/month each on reserved pricing:

```
3 × ~$40 (reserved)  ≈ $120 / month  + a load balancer (~$20)  ≈ $140 / month
```

**$799 vs $140 — serverless is ~5–6× more expensive, forever, and the gap widens linearly with traffic.** At 5,000 req/s it's ~$8,000/month vs a few hundred. Notice *why*: at steady load the container is ~100% utilized, so you're paying for almost no idle — and idle is the only thing serverless was saving you from. When there's no idle to eliminate, all you're left with is the per-invocation premium.

**Where's the break-even?** Roughly where a container's fixed monthly cost equals your invocation bill — for typical small functions, somewhere in the **low millions of invocations per month** (order of ~5–50 req/s sustained, depending heavily on duration and memory). Below that, serverless. Above it, containers.

> **The rule, and say it exactly like this in an interview:** *serverless wins on spiky, low-duty-cycle, unpredictable workloads and loses on steady high throughput.* You are buying elasticity and paying a premium per unit of compute. If your utilization is already high, you have nothing to buy. (This is a core lever in [136 — Cost Optimization].)

### 7. The operational taxes: lock-in, testing, debugging

- **Vendor lock-in.** The handler body is portable JavaScript. Everything *around* it — the `event` shape, IAM roles, API Gateway config, the SDK calls, the CloudFormation/CDK/Serverless-Framework stack — is not. Migrating a 60-function Lambda system to GCP is a rewrite of the plumbing, not a redeploy. Mitigate by keeping business logic in plain functions and treating the handler as a thin adapter:

```js
// core/createOrder.js — pure, portable, unit-testable. No cloud anywhere.
export async function createOrder({ userId, items }, { orders, payments }) {
  if (!items?.length) throw new ValidationError('empty order');
  const total = items.reduce((s, i) => s + i.price * i.qty, 0);
  await payments.charge(userId, total);
  return orders.insert({ userId, items, total, status: 'PAID' });
}

// handlers/createOrder.js — the ONLY file that knows what "Lambda" is.
import { createOrder } from '../core/createOrder.js';
import { orders, payments } from '../adapters/index.js';

export async function handler(event) {
  try {
    const order = await createOrder(JSON.parse(event.body), { orders, payments });
    return { statusCode: 201, body: JSON.stringify(order) };
  } catch (err) {
    const code = err instanceof ValidationError ? 400 : 500;
    return { statusCode: code, body: JSON.stringify({ error: err.message }) };
  }
}
```

Now your tests call `createOrder()` directly with fakes — no cloud emulator, no `event` fixtures for your business rules — and porting to Cloud Run means rewriting one thin file.

- **Harder local testing.** `node index.js` doesn't work; there's no server. You use SAM/LocalStack/serverless-offline, all of which emulate the cloud imperfectly. The gaps that bite are IAM permissions and event shapes — things that only fail *in* the cloud.
- **Harder debugging.** A single user action may traverse API Gateway → Lambda A → SQS → Lambda B → DynamoDB Stream → Lambda C. There is no stack trace across that. You need **distributed tracing** (X-Ray / OpenTelemetry) with a correlation ID threaded through every event, or you are debugging by grepping twelve separate log groups at 3am. Budget for observability *up front*; it is not optional here the way it almost is in a monolith. (Compare [71 — Microservices vs Monolith] — serverless is microservices with the granularity dial turned to maximum, and it inherits every distributed-debugging pain, amplified.)

### 8. The middle ground that ate serverless's territory

This is the most current-sounding thing you can say, and it's true.

**Scale-to-zero container platforms — Google Cloud Run, AWS Fargate/App Runner, Fly.io.** You ship a normal Docker container running a normal Express app. The platform scales it to zero when idle and up on demand, and bills roughly per request-second. You get: no servers to patch, no idle cost, automatic scaling — **most of the operational win** — while keeping: a normal HTTP server, real connection pooling (one container handles many concurrent requests, so *one pool of 20 is correct again*), no 15-minute limit, trivial local testing (`docker run`), and near-zero lock-in.

That combination quietly removed the reason for a lot of FaaS adoption. If someone asks "why not just Cloud Run?", the honest answer is often **"you're right, use Cloud Run."** FaaS still wins for true event-glue (S3 triggers, DB streams, cron) and for the absolute floor of cost and effort.

**Edge functions — Cloudflare Workers, Deno Deploy, Vercel Edge.** A genuinely different trade-off, not a variation. Instead of a container per instance, they run your JS in a **V8 isolate** — the same lightweight sandbox Chrome uses per browser tab. Thousands of isolates share one process, and spinning one up costs **microseconds, not hundreds of milliseconds**. Cold start effectively disappears, and the code runs in a datacenter physically near the user.

The price of that: it is **not Node.js**. No `fs`, no raw TCP sockets, restricted npm compatibility, tight CPU-time budgets (single-digit to tens of ms), and small memory. You get Web-standard APIs (`fetch`, `Request`, `Response`, Web Crypto). It is superb for auth checks, redirects, A/B routing, header rewriting, personalization, and cache logic at the edge — and unsuitable for a heavy ORM-driven backend.

```js
// A Cloudflare Worker. Note: no `handler(event)`, no Node globals — Web APIs.
// Cold start here is ~0 ms, which is why edge auth checks are viable.
export default {
  async fetch(request, env) {
    const token = request.headers.get('authorization')?.replace('Bearer ', '');
    if (!token || !(await isValid(token, env.JWT_SECRET))) {
      return new Response('Unauthorized', { status: 401 });
    }
    // Verified at the edge, ~10 ms from the user. Only then pay for the origin hop.
    return fetch(request);
  },
};
```

---

## Visual / Diagram description

**Diagram 1 — request flow, cold path vs warm path.**

```
   EVENT SOURCES
 ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
 │  Browser /   │  │  S3 upload   │  │  SQS message │  │ Cron (2am)   │
 │  Mobile app  │  │  (object)    │  │  (queue)     │  │ (schedule)   │
 └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
        │ HTTPS           │                 │                 │
        ▼                 │                 │                 │
 ┌──────────────┐         │                 │                 │
 │ API GATEWAY  │         │                 │                 │
 │ authn, throt-│         │                 │                 │
 │ tle, routing │         │                 │                 │
 └──────┬───────┘         │                 │                 │
        │                 │                 │                 │
        └────────┬────────┴────────┬────────┴────────┬────────┘
                 ▼                 ▼                 ▼
        ┌────────────────────────────────────────────────────┐
        │            LAMBDA INVOCATION DISPATCHER            │
        │   "is there a WARM instance free for this fn?"     │
        └───────┬─────────────────────────────┬──────────────┘
           NO   │                             │  YES
      ┌─────────▼───────────┐        ┌────────▼─────────────┐
      │   ❄ COLD PATH       │        │   ☀ WARM PATH        │
      │ 1 allocate micro-VM │        │  instance already    │
      │ 2 unpack bundle     │        │  exists; module      │
      │ 3 start Node runtime│        │  scope already ran;  │
      │ 4 eval module scope │        │  DB conn still open  │
      │ 5 run init (DB conn)│        │                      │
      │   ≈ 150 ms – 2 s    │        │       ≈ 0 ms         │
      └─────────┬───────────┘        └────────┬─────────────┘
                └──────────────┬──────────────┘
                               ▼
                    ┌─────────────────────┐
                    │  YOUR handler(event)│   ← the only part you wrote
                    │   ~20 ms of work    │
                    └──────────┬──────────┘
                               │  N concurrent instances = N connections (!)
                               ▼
                    ┌─────────────────────┐
                    │     RDS PROXY       │  ← multiplexes 1,000 → ~20
                    │  (or PgBouncer)     │     THE fix for the conn problem
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │  POSTGRES           │
                    │  max_connections=100│
                    └─────────────────────┘

   Alternative that sidesteps the whole problem:
        handler ──HTTPS──▶ DynamoDB   (no connections to exhaust at all)
```

**What the diagram shows.** Read it top to bottom. Any of four unrelated event sources can invoke the *same* function — HTTP traffic is just one door, and API Gateway (authn, throttling, routing) sits only on that door. The dispatcher is where serverless's whole personality lives: it asks one question — *do I have a warm instance?* — and the answer determines whether your user waits 20 ms or 1.5 seconds. The cold path's five numbered steps are exactly the anatomy from section 3, and steps 4–5 are **the part you control**; that's why "hoist init out of the handler" matters so much, because on the warm path those steps are already done and paid for.

The bottom half is the connection problem drawn to scale. The arrow out of `handler` fans out as wide as your concurrency, and the proxy is the funnel that stops 1,000 fan-out arrows from hitting a 100-connection database. The dashed alternative at the bottom is the structural fix: talk to a store that has no connections.

**Diagram 2 — the cost crossover, which is the decision.**

```
  $/month
    │
900 ┤                                            ╭──── Serverless
    │                                       ╭────╯      (linear in traffic,
700 ┤                                  ╭────╯            zero at zero)
    │                             ╭────╯
500 ┤                        ╭────╯
    │                   ╭────╯
300 ┤              ╭────╯
    │         ╭────╯          ┌─────────────────────────────────
140 ┤    ╭────╯          ┌────┘   Containers (step function:
    │╭───╯          ┌────┘        you add a box at a time; never
 40 ┼╯       ┌──────┘             free, even at zero traffic)
    │  ┌─────┘
  0 └──┴─────────────┴─────────────────────────────────▶ req/s
     0      ~10-50   (CROSSOVER)                    500

  LEFT of crossover : serverless is 10–500× cheaper. Take the taxi.
  RIGHT of crossover: containers are 5×+ cheaper, and the gap only grows.
```

The container line starts *above zero* (you pay rent at 3am) and climbs in steps. The serverless line starts *at exactly zero* and climbs smoothly forever. They cross. **Your job is to know which side of that crossing your workload lives on** — and it's a question you answer with arithmetic, not vibes.

---

## Real world examples

### Netflix — media pipeline and operational glue

Netflix has publicly described using AWS Lambda for event-driven pieces of its encoding pipeline and infrastructure automation rather than for its core streaming path. The shape is instructive: when a publisher uploads a source file to S3, that upload event triggers functions that validate, split, and enqueue encoding work. The workload is **bursty** (a new title lands, then nothing for hours), **stateless**, and **not latency-critical for a human** — which is precisely the serverless sweet spot. The actual video streaming, which is sustained, high-throughput, and latency-critical, does not run on Lambda. That split — *glue and reaction on serverless, steady hot path on servers/CDN* — is the pattern to copy.

### The classic image-processing pipeline (the canonical use case)

Countless products implement this identically, and it's worth internalizing as the reference architecture: a client requests a **pre-signed S3 URL** and uploads a photo **directly to S3** (never through your function — remember the ~6 MB payload limit). The `s3:ObjectCreated` event fires a Lambda, which reads the object, generates thumbnails with `sharp`, writes them back to a `thumbs/` prefix, and updates a DynamoDB record. Ten uploads at once → ten concurrent functions, automatically. Zero uploads → zero cost. There is no queue to size, no worker fleet to scale, and no server to patch. Compare the always-on equivalent: a worker fleet sized for peak, idle most of the day.

### Cloudflare Workers as an architectural counter-example

Cloudflare's Workers platform is the clearest illustration that "serverless" names a billing/ops model, not one implementation. By running V8 isolates instead of containers, they make cold starts effectively vanish — reversing the single biggest objection to FaaS — but pay for it with a constrained, non-Node runtime and strict CPU limits. It's the trade-off dial moved to a completely different setting, and naming it in an interview shows you understand that the cold-start tax is an *implementation* consequence, not a law of nature.

---

## Trade-offs

| Advantage | What it actually buys you |
|---|---|
| **Zero cost at zero traffic** | A prototype, an internal tool, or a webhook costs cents. The defining win. |
| **No servers to operate** | No patching, no OS upgrades, no capacity planning, no autoscaler tuning. |
| **Automatic scaling to thousands** | A traffic spike needs *no* action from you. Scaling is the platform's problem. |
| **Fast to ship** | A working endpoint in an afternoon; no infra to stand up first. |
| **Fine-grained cost attribution** | You can see exactly what each feature costs to run. |

| Cost | What it actually takes away |
|---|---|
| **Cold starts** | A p99 tail measured in hundreds of ms to seconds. Can disqualify user-facing APIs. |
| **Connection exhaustion** | Relational DBs need a proxy, or you must move to DynamoDB. Non-negotiable. |
| **Hard limits** | 15 min max, ~6 MB payloads, ephemeral `/tmp`, no held WebSockets. |
| **Expensive at scale** | Above the crossover, 5×+ the cost of containers, growing linearly. |
| **Vendor lock-in** | Handler is portable; the event shapes, IAM, and stack are not. |
| **Debugging & local dev** | No stack trace across 12 functions. Tracing becomes mandatory, not optional. |

**The sweet spot:** reach for serverless when the workload is **event-driven, bursty or low-duty-cycle, stateless, short (well under 15 min), and tolerant of an occasional few-hundred-ms tail.** Cron jobs, webhook handlers, S3-triggered processing, queue consumers, internal APIs, prototypes.

**Rule of thumb:** if your traffic is *spiky or small* → serverless, and you'll be delighted. If it's *steady, high-throughput, or latency-critical* → containers, and you'll save both money and your p99. And if you want most of the operational win without any of the FaaS constraints, reach for **scale-to-zero containers (Cloud Run / Fargate)** — for a large fraction of "should we go serverless?" conversations, that's the actual right answer.

---

## Common interview questions on this topic

### Q1: "What is a cold start, and how would you reduce it?"

**Hint:** Define it as the setup the platform must do when no warm instance exists — allocate container, load runtime, load your code, run init — *before* your handler runs. Then give the mitigations in order of leverage: (1) **move initialization out of the handler into module scope** so warm invocations reuse DB clients and SDK objects — say this first, it's the highest-signal answer; (2) shrink the bundle and lazy-`import()` heavy deps; (3) use a fast runtime (Node/Go, never JVM on a latency path); (4) provisioned concurrency — and immediately note that it costs money for idle capacity, which partly defeats the point of serverless. Close with the honest caveat: for a p99-sensitive user-facing API, cold starts may be disqualifying regardless.

### Q2: "You put 1,000 concurrent Lambdas in front of Postgres and it fell over. Why, and what's the fix?"

**Hint:** Pooling (topic 65) works because a long-lived process amortizes N connections over many requests. Serverless has no long-lived process — each of the 1,000 instances opens its own connection (or its own *pool*, which is even worse, and pointless since one instance serves one request at a time). Postgres `max_connections` is ~100 and each connection is a real backend process costing MBs of RAM, so you can't just raise it. Three fixes: an **external pooler** (RDS Proxy / PgBouncer) that multiplexes 1,000 → 20; an **HTTP data API** with no persistent connection; or a store with **no connection concept at all** — DynamoDB over HTTPS. Then land the insight: this is *why* serverless architectures gravitate to DynamoDB — a stateless compute layer wants a stateless data layer.

### Q3: "We're at 500 req/s sustained, 24/7. Should we move from ECS to Lambda to save money?"

**Hint:** No — and prove it with arithmetic rather than opinion. 500 × 86,400 × 30 ≈ 1.3B invocations/month ≈ $259 in request charges, plus ~32M GB-seconds ≈ $540 → roughly **$800/month**. The container equivalent (Little's Law: 500 × 0.2 s = 100 concurrent workers → ~3 instances + an LB) is ~**$140/month** reserved. Serverless is ~5–6× more expensive here and the gap grows linearly. State the principle: serverless sells you *elasticity*, and at high steady utilization there is no idle left to eliminate, so you're paying a premium for nothing. Invert it to show range: at 3,000 req/day serverless is ~$0.06 vs $30+ for the container.

### Q4: "Can you build a real-time chat app on Lambda?"

**Hint:** Not with functions alone — a WebSocket is a long-lived connection and a function is a short-lived execution; they're structurally incompatible. You need a managed WebSocket layer (API Gateway WebSocket API) that *holds* the sockets, stores each `connectionId` in DynamoDB, and invokes your function **per message**; your function then pushes replies out through the gateway's post-to-connection API. Note the extra costs: fan-out to a room means N gateway calls, and you're paying per connection-minute. Then be pragmatic: for a serious chat system, a stateful WebSocket server (or Cloud Run holding sockets) is usually simpler and cheaper.

### Q5: "When would you choose a container over a serverless function — and is there something in between?"

**Hint:** Container when: sustained high throughput (above the cost crossover), p99-critical user paths, jobs longer than 15 minutes, persistent connections (WebSockets, long-lived DB or Kafka clients), heavy runtimes, or anything needing state and local disk. Then earn the extra point by naming the middle ground: **scale-to-zero container platforms (Cloud Run, Fargate, App Runner)** give you no-servers-to-patch and no idle cost while keeping a normal HTTP server, real pooling, no time limit, easy local testing, and low lock-in — and they've absorbed a lot of what FaaS used to be for. And **edge functions (Cloudflare Workers)** use V8 isolates instead of containers to get near-zero cold start, at the price of a restricted, non-Node runtime.

---

## Practice exercise

### "Serverless or Not?" — three workloads, one decision memo

You're the tech lead. For **each** of the three workloads below, produce a short written decision (roughly 150 words each) that must contain: **the verdict** (Lambda / scale-to-zero container / always-on container), **the arithmetic**, and **the one constraint that dominated your call**.

1. **Nightly report generator.** Runs once a day at 3am. Pulls 2M rows from Postgres, builds a 40 MB PDF, emails it. Takes ~22 minutes today on a laptop.
2. **Public product-search API.** 40 req/s average, spikes to 600 req/s during a 20-minute flash sale twice a week. The PM's stated requirement is **p99 under 200 ms**.
3. **Stripe webhook receiver.** ~800 events/day, arriving unpredictably. Each event does one DynamoDB write and one SQS publish. Must never lose an event.

**What to produce:**

- A one-page memo with your three verdicts.
- For workload 2, the **full cost comparison at both 40 req/s and 600 req/s** (show every multiplication, as in section 6), plus an explicit sentence on whether cold starts break the p99 requirement — and if you'd use provisioned concurrency, how many instances and what that costs.
- For workload 3, a **10–20 line Node.js handler** that correctly hoists its initialization out of the handler function (module scope), and one sentence on why that matters here given the traffic pattern.
- Redraw **Diagram 1** from memory on paper, including the cold path, the warm path, and the RDS Proxy. Then answer: which single box would you delete if the database were DynamoDB, and why?

*(Answer sketch for #1: 22 minutes > 15-minute Lambda limit — that constraint alone decides it. #2: the p99 requirement is the whole ballgame, and the 600 req/s spike is the interesting cost case. #3: a textbook serverless win — pennies, spiky, stateless, tolerant of a 300 ms cold start because Stripe simply retries.)*

---

## Quick reference cheat sheet

- **Serverless ≠ no servers.** It means *you* don't provision, patch, or scale them. Taxi vs. owning a car.
- **The four defining properties:** deploy a function; event-triggered; scales 0 → thousands automatically; **pay per millisecond, nothing when idle.** Lose the last one and it's just a container.
- **Triggers:** HTTP (API Gateway), S3 object events, queue messages, cron, DB change streams. A natural consumer runtime for event-driven architecture.
- **Cold start anatomy:** allocate container → load runtime → load your code → run init → *then* your handler. Slim Node: ~100–300 ms. JVM or VPC-attached: seconds.
- **Highest-leverage optimization: hoist initialization OUT of the handler.** Module scope runs once per container; warm invocations reuse the DB connection and SDK clients for free.
- **Provisioned concurrency kills cold starts by paying for idle** — which is the exact thing serverless was supposed to eliminate. Use it knowingly, not reflexively.
- **The connection problem:** 1,000 concurrent instances × 1 connection each vs. Postgres `max_connections = 100`. You DDoS your own database at peak.
- **Fixes:** external pooler (RDS Proxy / PgBouncer), an HTTP data API, or a store with no connections at all — which is **why serverless pushes you toward DynamoDB**.
- **Never put a `pg` Pool of 10 in a Lambda.** One instance = one request at a time; a pool there is 9 wasted connections × your concurrency.
- **Hard limits:** 15-minute max execution, ephemeral `/tmp`, no state between invocations, ~6 MB payloads, **cannot hold a WebSocket**.
- **Cost, spiky:** 3,000 req/day ≈ **$0.06/month** vs $30+ for an always-on container. Serverless wins by ~500×.
- **Cost, steady:** 500 req/s 24/7 ≈ **$800/month** vs ~$140 for 3 reserved containers. Containers win by 5×+, and the gap grows linearly.
- **The rule:** serverless wins on **spiky / low-duty-cycle / unpredictable**; loses on **steady high throughput**. You're buying elasticity — at high utilization there's no idle left to buy.
- **The middle ground:** scale-to-zero containers (Cloud Run, Fargate) give most of the operational win without the FaaS constraints. **Edge functions** (Cloudflare Workers) use V8 isolates for near-zero cold start, at the price of a non-Node runtime.
- **Also true:** vendor lock-in in the plumbing, awkward local testing, and no stack trace across 12 functions — make distributed tracing non-optional.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [./88-containers-and-orchestration.md](./88-containers-and-orchestration.md) — the alternative deployment model; serverless is what you get when you push containers all the way to per-request granularity, and scale-to-zero containers are the middle ground between them |
| **Next** | [./136-cost-optimization.md](./136-cost-optimization.md) — the crossover arithmetic in this doc is one instance of the general skill of costing an architecture before you build it |
| **Related** | [./65-database-connection-pooling.md](./65-database-connection-pooling.md) — read this first; the serverless connection-exhaustion failure is only comprehensible once you know *why* pooling depends on a long-lived process |
| **Related** | [./68-event-driven-architecture.md](./68-event-driven-architecture.md) — serverless functions are the most natural event consumers ever built: they cost nothing when there are no events |
| **Related** | [./71-microservices-vs-monolith.md](./71-microservices-vs-monolith.md) — serverless is microservices with the granularity dial at maximum, and it inherits every distributed-debugging pain in amplified form |
