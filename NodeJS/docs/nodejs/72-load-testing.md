# 72 — Load testing

## What is this?

Load testing means throwing a large number of fake requests at your server on purpose, before real users do it, to see how it behaves under pressure. Tools like **autocannon**, **artillery**, and **k6** act like a crowd of virtual users hammering your API, while measuring how fast it responds and where it starts to choke. Think of it like stress-testing a bridge with weighted trucks before opening it to real traffic — you want to find the breaking point in a controlled test, not during rush hour with real cars on it.

---

## Why does it matter for backend development?

A backend that works fine for one developer clicking around in Postman can fall over completely once 500 real users hit it at the same time. Load testing is how you answer questions like: "How many requests per second can this endpoint handle?", "What happens to response time as concurrency increases?", "Does that new database query I added slow things down under load?", and "Will this survive a marketing campaign spike?". Every serious backend team runs load tests before a major release, after adding a new heavy endpoint, and as part of capacity planning — it turns "I think it'll be fine" into a measured, provable answer.

---

## Syntax / API

```bash
# ── autocannon — fast, zero-config HTTP benchmarking tool (great for quick checks) ──
npx autocannon -c 100 -d 20 http://localhost:3000/api/products
# -c 100  → 100 concurrent connections hitting the server at once
# -d 20   → run the test for 20 seconds
# target  → the URL being load tested

# ── artillery — YAML-based, scriptable scenarios with multiple steps ──────────
npx artillery run load-test.yml
# reads a YAML config describing phases (ramp-up, sustained load) and requests

# ── k6 — JavaScript-based scripting, rich metrics, thresholds/pass-fail ────────
k6 run load-test.js
# runs a JS file that defines virtual users (VUs) and what each one does
```

```js
// File: load-test.js — a minimal k6 script (JavaScript, but run by the k6 binary, not Node)
import http from 'k6/http';   // k6's built-in HTTP client (not Node's http module)
import { sleep, check } from 'k6'; // sleep() pauses a VU, check() asserts on the response

export const options = {
  vus: 50,          // 50 virtual users running the script concurrently
  duration: '30s',  // keep the test running for 30 seconds
};

export default function () {
  // Each virtual user repeatedly runs this function in a loop for the test duration
  const res = http.get('http://localhost:3000/api/products'); // send a GET request

  // Assert the response looks correct — failed checks show up in the report
  check(res, {
    'status is 200': (r) => r.status === 200,   // did the server respond OK?
    'has body': (r) => r.body.length > 0,        // did it actually return data?
  });

  sleep(1); // wait 1 second before this VU repeats — simulates real user think-time
}
```

---

## How it works — line by line

A load test tool spins up many "virtual users" (VUs) — simulated clients that each send requests independently and in parallel, exactly like real browsers or mobile apps would. While the test runs, the tool records, for every single request: how long it took (latency), whether it succeeded or failed (status code), and how many requests completed per second (throughput). At the end it prints a summary: average/median/p95/p99 latency, total requests, error rate, and requests-per-second.

- **autocannon** is the simplest — one command, one target URL, good for a quick sanity check on a single endpoint. No scripting needed.
- **artillery** lets you describe a multi-step scenario in YAML — log in, then fetch a resource, then post something — and control how load ramps up over time (start at 10 users/sec, ramp to 200/sec over 2 minutes).
- **k6** is the most powerful — you write actual JavaScript logic per virtual user (loops, conditionals, custom checks) and can set pass/fail **thresholds** (e.g. "fail the test if p95 latency exceeds 500ms"), which makes it suitable for CI pipelines.

The key numbers to read afterward are: **latency percentiles** (p50 = typical user, p95/p99 = your slowest users — these matter more than the average), **requests/sec (throughput)**, and **error rate**. If throughput plateaus while latency keeps climbing as you add more virtual users, you have found your server's bottleneck — the point past which it cannot keep up.

---

## Example 1 — basic

```js
// File: server.js
// A tiny Express server we will load test — nothing fancy, just something to hit
const express = require('express');
const app = express();

// Simulate a small amount of work per request (e.g. formatting a response)
app.get('/api/products', (req, res) => {
  const products = [
    { id: 1, name: 'Keyboard', price: 49.99 },
    { id: 2, name: 'Mouse', price: 19.99 },
  ];
  res.json(products); // send JSON back — this is what we will benchmark
});

app.listen(3000, () => console.log('Server ready on port 3000'));
```

```bash
# Terminal 1 — start the server
node server.js

# Terminal 2 — run autocannon against it
npx autocannon -c 50 -d 10 http://localhost:3000/api/products
# -c 50  → 50 concurrent virtual connections
# -d 10  → run for 10 seconds

# Example output (abbreviated):
#   Latency:   avg 4.12ms   p97.5 9.01ms   p99 12.44ms
#   Req/Sec:   avg 11,850   min 10,200     max 12,900
#   200 codes: 118,500      Errors: 0
# Reading it: median latency under 5ms and zero errors — this endpoint handles
# 50 concurrent users comfortably at this load level.
```

---

## Example 2 — real world backend use case

```yaml
# File: load-test.yml — artillery scenario simulating a realistic checkout flow
config:
  target: "http://localhost:3000"     # base URL of the API under test
  phases:
    - duration: 30                    # phase 1: warm-up — lasts 30 seconds
      arrivalRate: 5                  # start with 5 new virtual users per second
    - duration: 60                    # phase 2: sustained load — lasts 60 seconds
      arrivalRate: 5
      rampTo: 100                     # ramp from 5/sec up to 100/sec over this phase
  defaults:
    headers:
      authorization: "Bearer {{ authToken }}"   # reused auth header for every request

scenarios:
  - name: "Login and checkout"
    flow:
      # Step 1 — log in and capture the returned token for later requests
      - post:
          url: "/api/login"
          json:
            email: "loadtest_user@example.com"
            password: "TestPass123!"
          capture:
            json: "$.token"
            as: "authToken"           # save token from response into a variable

      # Step 2 — browse the product catalog (read-heavy, usually fast)
      - get:
          url: "/api/products"

      # Step 3 — add an item to the cart (write operation, hits the DB)
      - post:
          url: "/api/cart/items"
          json:
            productId: 101
            quantity: 2

      # Step 4 — checkout (the heaviest step — payment + inventory + email)
      - post:
          url: "/api/checkout"
          json:
            paymentMethod: "card"
```

```bash
# Run the scenario and save a JSON report for later analysis
npx artillery run load-test.yml --output report.json

# Turn the raw JSON into a readable HTML report with charts
npx artillery report report.json
# Open report.html — look at p95/p99 latency per endpoint and the error rate.
# If /api/checkout shows much higher p99 latency than the other steps, that
# step (likely the DB write or payment call) is your bottleneck to investigate.
```

```js
// File: k6-checkout.js — the same idea expressed as a k6 script with a hard threshold
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // ramp from 0 to 20 VUs over 30s
    { duration: '60s', target: 20 },   // hold steady at 20 VUs for 60s
    { duration: '10s', target: 0 },    // ramp down to 0 (cool-down)
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // FAIL the test run if p95 latency > 500ms
    http_req_failed: ['rate<0.01'],    // FAIL if more than 1% of requests error out
  },
};

export default function () {
  const loginRes = http.post('http://localhost:3000/api/login', JSON.stringify({
    email: 'loadtest_user@example.com',
    password: 'TestPass123!',
  }), { headers: { 'Content-Type': 'application/json' } });

  const authToken = loginRes.json('token'); // pull the token out of the JSON body

  const checkoutRes = http.post(
    'http://localhost:3000/api/checkout',
    JSON.stringify({ productId: 101, quantity: 2 }),
    { headers: { authorization: `Bearer ${authToken}`, 'Content-Type': 'application/json' } }
  );

  check(checkoutRes, { 'checkout succeeded': (r) => r.status === 200 });
  sleep(1); // pause before the next iteration for this virtual user
}
```

---

## Common mistakes

### Mistake 1 — Load testing on your own dev machine and trusting the numbers

```bash
# ❌ WRONG — running the API server AND the load test tool on the same laptop
# The tool competes with the server for CPU, so both the "load" and the
# "results" are distorted — numbers look artificially slow or capped
node server.js &
npx autocannon -c 200 -d 30 http://localhost:3000/api/products
```

```bash
# ✅ CORRECT — run the server on a dedicated machine/VM/container and the
# load generator from a separate machine, closer to production conditions
# Machine A (server):
node server.js
# Machine B (load generator):
npx autocannon -c 200 -d 30 http://192.168.1.50:3000/api/products
```

### Mistake 2 — Testing an endpoint with no realistic data or auth, so it always hits a fast path

```js
// ❌ WRONG — hitting an endpoint with an empty DB, no real query, no real payload size
// so the test measures an unrealistic best case that hides real bottlenecks
app.get('/api/orders', (req, res) => {
  res.json([]); // DB has zero orders in the test environment — always instant
});
```

```js
// ✅ CORRECT — seed the test database with realistic volume (thousands of rows)
// before running the load test, so queries, indexes, and joins are actually exercised
// seed-test-data.js
const dbConnection = require('./db');

async function seedOrders() {
  const orders = Array.from({ length: 50000 }, (_, i) => ({
    userId: `user_${i % 1000}`,
    total: (Math.random() * 200).toFixed(2),
  }));
  await dbConnection('orders').insert(orders); // realistic row count for the test
}

seedOrders();
```

### Mistake 3 — Only looking at the average latency and declaring victory

```
# ❌ WRONG — reading only this line and calling the API "fast"
Avg latency: 45ms      ← looks great!

# But the full picture was hidden:
Avg latency: 45ms
p95 latency: 890ms     ← 1 in 20 users waits almost a full second
p99 latency: 4200ms    ← 1 in 100 users waits over 4 seconds — real users notice this
```

```
# ✅ CORRECT — always check p95 and p99, not just the average
# Averages hide tail latency; percentiles reveal your worst-case user experience.
# In k6, enforce this automatically with a threshold:
thresholds: {
  http_req_duration: ['p(95)<500', 'p(99)<1000'],  // fails the run if tail latency is bad
}
```

---

## Practice exercises

### Exercise 1 — easy

Build a tiny Express server with a single `GET /api/health` route that returns `{ status: 'ok' }`. Install and run `autocannon` against it with 20 concurrent connections for 10 seconds. Note down the average latency, p99 latency, and total requests/sec from the output, then write them as a comment above your server code.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express server with two routes:
- `GET /api/fast` — returns a JSON object immediately
- `GET /api/slow` — artificially delays for 200ms (using `setTimeout` wrapped in a Promise) before responding, to simulate a slow database query

Write an **artillery** YAML config (or a k6 script) that sends load to both routes over a 30-second test, with load ramping from 10 to 100 requests/sec. Run it and compare the p95 latency of `/api/fast` vs `/api/slow` — explain in a comment which one becomes a bottleneck first as load increases.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a realistic mini-API with three routes: `POST /api/login` (returns a fake token), `GET /api/orders` (requires the `authorization` header, returns a hardcoded list of 100 fake orders), and `POST /api/orders` (accepts a body and simulates a 100ms write delay). Then write a k6 script that: logs in once per virtual user, captures the token, uses it to call `GET /api/orders` three times, then calls `POST /api/orders` once — with k6 `thresholds` that fail the test if p95 latency exceeds 300ms or the error rate exceeds 2%. Run the test with a staged ramp (0 → 50 → 0 virtual users) and report whether your thresholds passed or failed.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
TOOLS AND WHEN TO USE THEM
  autocannon → fastest to set up, single endpoint, quick sanity check, no config file
  artillery  → multi-step YAML scenarios, phased ramp-up, good HTML reports
  k6         → full JS scripting, pass/fail thresholds, best for CI pipelines

KEY METRICS TO READ (never trust averages alone)
  p50 (median)  → typical user experience
  p95           → 1 in 20 users — usually the number that matters most
  p99           → 1 in 100 users — your worst-case tail latency
  req/sec       → throughput — how many requests the server completes per second
  error rate    → % of requests that failed (non-2xx, timeouts, connection errors)

BASIC COMMANDS
  npx autocannon -c 100 -d 20 <url>         → 100 connections, 20 seconds
  npx artillery run scenario.yml            → run a YAML scenario
  npx artillery report report.json          → turn JSON output into an HTML report
  k6 run script.js                          → run a k6 JS script

READING RESULTS — WHAT TO LOOK FOR
  Throughput plateaus + latency keeps climbing  → you found the bottleneck point
  Error rate rises sharply at some concurrency  → server is saturated past that point
  p95/p99 far higher than average               → tail latency problem, investigate slow paths

FINDING BOTTLENECKS
  1. Run load test → note where latency/errors spike
  2. Check CPU/memory on the server during the test (top, htop, or APM tool)
  3. Check DB query time separately — is the DB or the app code the slow part?
  4. Add one change at a time (index, cache, connection pool size) → re-run test → compare

NEVER DO
  Run the load generator on the same machine as the server under test
  Test against an empty/unrealistic dataset
  Only report the average latency and ignore p95/p99
  Load test directly against production without warning your team
```

---

## Connected topics

- **67 — Profiling Node.js** — once a load test reveals a bottleneck, profiling (`--inspect`, clinic.js) is how you find the exact slow function causing it
- **70 — Rate limiting** — load testing is how you decide what limits are safe to set — you need real throughput numbers before choosing a request cap
- **73 — Horizontal scaling with PM2** — load test results tell you whether one process instance is enough or whether you need cluster mode across multiple cores
