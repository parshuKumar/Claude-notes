# 18 — Rate Limiting at the HTTP Level

> **Phase 3 — Topic 5 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `14-http-headers-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a popular ice cream shop with one scooper working the counter.

If everyone rushes in at once, the line becomes chaos, the scooper burns out, and nobody gets served. So the shop puts up a rule: **"Maximum 5 scoops per customer per hour."**

- There's a little counter next to your name on the wall (**your remaining allowance**).
- When you've had your 5 scoops, the clerk says **"Come back in 12 minutes"** and points at the clock (**Retry-After**).
- A polite customer reads the sign, waits the 12 minutes, and comes back calmly.
- A rude customer keeps shouting "SCOOP! SCOOP! SCOOP!" every second — which just makes the line *worse* for everyone and gets them nowhere.

HTTP rate limiting is exactly this. The server tells you your limit, how many requests you have left, and when to come back. A **well-behaved client reads those signs and waits**. A badly written client hammers the door and makes everything worse — that's called a *retry storm*.

---

## Where This Lives in the Network Stack

Rate limiting is an **application-layer (Layer 7)** concept. It reads the HTTP request — the URL, the headers, the API key — and decides whether to serve or reject. It is *not* something IP or TCP knows about.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP)                                  │  ← 429, Retry-After,
│      ▲                                                          │     X-RateLimit-* live here
│      │  rate limiter inspects: URL, API key, IP, headers        │
│  Layer 6 — Presentation  (TLS)                                  │
│  Layer 5 — Session                                              │
│  Layer 4 — Transport     (TCP — port 443)                       │  ← a *different* kind of
│  Layer 3 — Network       (IP)                                   │     limiting (SYN floods)
│  Layer 2 — Data Link                                            │     happens down here
│  Layer 1 — Physical                                             │
└─────────────────────────────────────────────────────────────────┘
```

**Where in your infrastructure it gets enforced** (a request may pass through several of these, each with its own limit):

```
Client
  │
  ▼
┌──────────┐   ┌───────────────┐   ┌──────────────┐   ┌──────────────┐   ┌────────┐
│   CDN    │─► │  API Gateway  │─► │ Reverse Proxy│─► │  Your App    │─► │   DB   │
│(Cloudflare)  │ (Kong, AWS    │   │  (nginx)     │   │ (Node.js)    │   │        │
│ edge WAF │   │  API Gateway) │   │ limit_req    │   │ per-user biz │   │ (the   │
│ per-IP   │   │ per-API-key   │   │ per-IP       │   │  rules       │   │ thing  │
│          │   │               │   │              │   │              │   │ you're │
└──────────┘   └───────────────┘   └──────────────┘   └──────────────┘   │protect-│
                                                                          │  ing)  │
   coarse  ──────────────────────────────────────────────────►  fine     └────────┘
   (cheap, high up)                                        (expensive, context-aware)
```

The earlier (further "left") you reject an abusive request, the cheaper it is — you never pay for the DB query. That's why CDNs and gateways do the bulk-blocking, and your app does the smart, per-user stuff.

---

## What Is This?

**Rate limiting** is the practice of restricting how many requests a client can make to a server in a given time window. When a client exceeds its allowance, the server rejects further requests with HTTP status **`429 Too Many Requests`** and tells the client when it can try again.

It rests on a small, mostly-informal contract of HTTP headers:

| Header | Direction | Meaning |
|--------|-----------|---------|
| `429 Too Many Requests` | response status | "You've exceeded your rate limit." |
| `Retry-After` | response | How long to wait before retrying (**seconds** or an **HTTP-date**). |
| `X-RateLimit-Limit` | response | Total requests allowed in the window. |
| `X-RateLimit-Remaining` | response | Requests you have left in the current window. |
| `X-RateLimit-Reset` | response | When the window resets (Unix epoch seconds, or seconds-until). |

`429` and `Retry-After` are **standardized** (RFC 6585 and RFC 9110 respectively). The `X-RateLimit-*` family is **an informal de-facto standard** — widely used (GitHub, Twitter/X, many others) but never formally specified, so the exact names and the meaning of `Reset` (absolute epoch vs. seconds-remaining) **vary between APIs**. Always read the docs of the API you call.

A newer IETF draft (`RateLimit` / `RateLimit-Policy`, the "RateLimit header fields for HTTP" draft) aims to standardize this. You'll see it start to appear, but `X-RateLimit-*` is what's in the wild today.

---

## Why Does It Matter for a Backend Developer?

You are on **both sides** of this contract, constantly.

**As the API provider**, rate limiting is how you:
- **Survive abuse and DoS** — a single buggy or malicious client can't take your service down for everyone.
- **Enforce fair use** — one noisy tenant can't starve the other 999.
- **Control cost** — every request costs CPU, bandwidth, and often money (a downstream paid API, a Lambda invocation, an LLM token bill).
- **Protect fragile downstream resources** — your database has a connection pool of maybe 100. Ten thousand concurrent requests will exhaust it and take down *everyone*, even users who were behaving. Rate limiting is a shield in front of the DB (see `39-database-connections-over-network.md`).

**As the API consumer**, you must understand it because:
- Third-party APIs you depend on (Stripe, GitHub, Twilio, OpenAI) **will** return `429` under load, and a badly written client turns one `429` into a **retry storm** that makes the outage worse and can get your key throttled or banned.
- Getting the client behavior right — honoring `Retry-After`, exponential backoff **with jitter** — is the difference between a self-healing system and a cascading failure.

If you take away one thing: **a `429` is not an error to panic over. It's an instruction. The server is telling you exactly how to behave. Obey it.**

---

## The Packet/Protocol Anatomy

A `429` is just a normal HTTP response — same TCP connection, same TLS tunnel — but with a specific status line and a specific set of headers. Here is a **raw 429 response** on the wire (what you'd see after the TLS layer is decrypted):

```
HTTP/1.1 429 Too Many Requests
Date: Sun, 12 Jul 2026 14:03:21 GMT
Content-Type: application/json
Retry-After: 30                          ← wait 30 seconds (delta-seconds form)
X-RateLimit-Limit: 100                    ← you get 100 requests per window
X-RateLimit-Remaining: 0                  ← you have 0 left right now
X-RateLimit-Reset: 1752328431             ← window resets at this Unix timestamp
Content-Length: 76

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Retry after 30s."
}
```

Compare it against a **successful (2xx) response** from the *same* endpoint — note the limit headers appear on success too, so a well-behaved client can slow down *before* getting blocked:

```
HTTP/1.1 200 OK
Content-Type: application/json
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 63                  ← 63 left, plenty of headroom
X-RateLimit-Reset: 1752328431
Content-Length: 512

{ "data": [ ... ] }
```

**`Retry-After` has two legal forms** (RFC 9110 §10.2.3):

```
Retry-After: 120                                  ← delta-seconds: wait 120 seconds
Retry-After: Sun, 12 Jul 2026 14:05:21 GMT        ← HTTP-date: wait until this moment
```

Your client MUST handle both. The date form is common with `503 Service Unavailable` during maintenance; the seconds form is more common with `429`.

**Where the headers sit in the encapsulated request** (recap from Topic 01 — the rate limiter reads Layer 7 only):

```
┌────────────────────────────────────────────────┐
│ IP  │ src/dst addr  → used for PER-IP limiting  │  Layer 3
│ TCP │ port 443                                  │  Layer 4
│ TLS │ encrypted                                 │  Layer 6
│ HTTP│ GET /v1/charges HTTP/1.1                  │  Layer 7  ← limiter reads:
│     │ Authorization: Bearer sk_live_...  ───────┼─────────►  the API key (per-key limiting)
│     │ Host: api.stripe.com                      │            the path   (per-endpoint limiting)
└────────────────────────────────────────────────┘
```

---

## How It Works — Step by Step

The server keeps a **counter** for each client (keyed by IP, API key, user ID, etc.). On every request it checks the counter against the limit. The interesting part is *how* the counter is defined — the **algorithm**. Five common ones, each with a diagram and trade-off.

### 1. Fixed Window

Divide time into fixed buckets (e.g. every 60 seconds). Count requests per bucket. Reset the count to zero when the next bucket starts.

```
Limit: 5 requests / 60s window

 window A (00:00–01:00)        window B (01:00–02:00)
├────────────────────────────┤├────────────────────────────┤
 ✓ ✓ ✓ ✓ ✓  ✗ ✗ ✗              ✓ ✓ ✓ ✓ ✓
 count: 1 2 3 4 5  (blocked)   count resets to 1 2 3 4 5
```

**Simple and cheap** (one integer per client). But it has the notorious **burst-at-boundary problem**:

```
Limit 5/min. A client sends 5 at 00:59 and 5 more at 01:01.
That's 10 requests in 2 real seconds — but they land in two different windows,
so both are allowed. Effective burst = 2× the intended limit.

00:59 ✓✓✓✓✓  │ boundary │  01:01 ✓✓✓✓✓
              ▲
        window resets here → counter zeroed → floodgates open
```

### 2. Sliding Window Log

Store a **timestamp for every request**. To check the limit, count how many timestamps fall within the last 60 seconds (a moving window).

```
now = 01:00:30, window = last 60s (00:59:30 → 01:00:30)

log: [00:59:20] [00:59:45] [01:00:10] [01:00:25] [01:00:29]
        ✗out       ✓in        ✓in        ✓in        ✓in
     (older than 60s, dropped)   ← 4 requests in window, limit 5 → ALLOW
```

**Perfectly accurate** — no boundary bursting. But **memory-expensive**: you store every timestamp for every client. A million clients at 100 req/window each = 100M timestamps to keep and prune.

### 3. Sliding Window Counter

The pragmatic compromise. Keep the **fixed-window counters** but **weight the previous window** by how far into the current window you are. Approximates the sliding log with just two integers.

```
Formula:
  estimated = current_count + previous_count × (overlap fraction of prev window)

Example: limit 100/min. We are 25% into the current minute.
  previous window count = 80
  current  window count = 30
  estimated = 30 + 80 × (1 − 0.25) = 30 + 60 = 90   → under 100, ALLOW
```

**Cheap AND smooth** — no boundary burst, tiny memory. This is what Cloudflare and many gateways actually use.

### 4. Token Bucket  ← most common in the wild

A bucket holds tokens. Every request **removes one token**. Tokens **refill at a steady rate**. If the bucket is empty, the request is rejected (or queued). The bucket has a max **capacity**, which allows short bursts up to that capacity.

```
Capacity: 10 tokens.  Refill: 1 token / second.

  [●●●●●●●●●●]  full → client can burst 10 requests instantly
       │  10 requests fire ↓
  [          ]  empty → next request BLOCKED (429)
       │  wait 3s, refills +3 ↓
  [●●●       ]  3 tokens → 3 more requests allowed

Allows BURSTS (up to capacity) but enforces a long-run AVERAGE rate (refill rate).
```

**Why it's popular:** real traffic is bursty. Token bucket says "sustained rate X, but I'll tolerate a short spike." Great fit for user-facing APIs. Used by AWS API Gateway, Stripe, and most cloud services.

### 5. Leaky Bucket

Requests pour into a bucket (a queue). They **leak out at a fixed rate** to be processed. If the bucket overflows, new requests are dropped. Unlike token bucket, it **smooths output** to a constant rate — no bursts reach the backend.

```
     incoming (bursty)
        │ │ │ │ │
        ▼ ▼ ▼ ▼ ▼
       ┌─────────┐
       │ █ █ █ █ │  ← queue (fixed capacity; overflow = dropped/429)
       │ █ █     │
       └────┬────┘
            │  leaks at constant 2 req/s
            ▼
        processed (smooth, steady)
```

**Protects a fragile backend** that needs a *steady* input rate (e.g. a legacy DB, a payment processor). The cost: added **latency** (requests wait in the queue) and it doesn't allow the useful bursts token bucket does.

### Comparison Table

| Algorithm | Memory | Allows bursts? | Boundary problem? | Best for |
|-----------|--------|----------------|-------------------|----------|
| Fixed window | 1 int | Yes (accidentally, 2×) | **Yes — the big flaw** | Rough, cheap limiting |
| Sliding window log | O(N) timestamps | No | No | Exact limits, low client count |
| Sliding window counter | 2 ints | No | No | General purpose (CDN/gateway default) |
| **Token bucket** | 2 numbers | **Yes (intentional, capped)** | No | **User-facing APIs (most common)** |
| Leaky bucket | queue | No (smooths) | No | Protecting steady-rate backends |

---

## Exact Syntax Breakdown

### Reading a 429 with curl

```
curl  -sS  -D -   -o /dev/null   https://api.example.com/v1/items
│     │ │   │      │             │
│     │ │   │      │             └── the endpoint
│     │ │   │      └── discard the BODY (we only care about headers here)
│     │ │   └── dump response HEADERS to stdout (-D - means "- = stdout")
│     │ └── -S: show errors even in silent mode
│     └── -s: silent (no progress meter)
└── transfer URL
```

### Anatomy of the headers you're reading

```
< HTTP/1.1 429 Too Many Requests     ← the status: you are blocked
< Retry-After: 30                     ← wait THIS long. Seconds OR an HTTP-date.
< X-RateLimit-Limit: 100              ← ceiling per window
< X-RateLimit-Remaining: 0            ← what's left (0 → you're out)
< X-RateLimit-Reset: 1752328431       ← WHEN it resets. Read the docs:
│                                        GitHub → absolute Unix epoch seconds
│                                        Some APIs → seconds-until-reset
```

### The two `Retry-After` forms — how to parse

```javascript
function retryAfterMs(headerValue) {
  if (!headerValue) return null;
  // Form 1: delta-seconds, e.g. "30"
  if (/^\d+$/.test(headerValue.trim())) {
    return Number(headerValue) * 1000;
  }
  // Form 2: HTTP-date, e.g. "Sun, 12 Jul 2026 14:05:21 GMT"
  const when = Date.parse(headerValue);
  if (!Number.isNaN(when)) {
    return Math.max(0, when - Date.now());
  }
  return null; // malformed → fall back to your own backoff
}
```

---

## Example 1 — Basic

**Goal: trip a rate limit yourself and watch the headers change.** The clearest self-contained demo is a tiny Express server. First the server:

```javascript
// limiter.js — minimal fixed-window limiter, 5 requests / 10 seconds per IP
const express = require("express");
const app = express();

const WINDOW_MS = 10_000;
const LIMIT = 5;
const hits = new Map(); // ip -> { count, resetAt }

app.use((req, res, next) => {
  const ip = req.ip;
  const now = Date.now();
  let rec = hits.get(ip);
  if (!rec || now > rec.resetAt) {
    rec = { count: 0, resetAt: now + WINDOW_MS };
    hits.set(ip, rec);
  }
  rec.count++;

  const remaining = Math.max(0, LIMIT - rec.count);
  res.set("X-RateLimit-Limit", String(LIMIT));
  res.set("X-RateLimit-Remaining", String(remaining));
  res.set("X-RateLimit-Reset", String(Math.ceil(rec.resetAt / 1000)));

  if (rec.count > LIMIT) {
    const retryAfter = Math.ceil((rec.resetAt - now) / 1000);
    res.set("Retry-After", String(retryAfter));
    return res.status(429).json({ error: "rate_limit_exceeded" });
  }
  next();
});

app.get("/", (req, res) => res.json({ ok: true }));
app.listen(3000, () => console.log("listening on :3000"));
```

Now hit it in a loop and watch the counter drain, then the `429` appear:

```bash
# Fire 7 requests quickly; print status + the rate-limit headers each time
for i in $(seq 1 7); do
  curl -sS -o /dev/null -w \
    "req %{http_code}  remaining=%header{x-ratelimit-remaining}  retry=%header{retry-after}\n" \
    http://localhost:3000/
done
```

Output:

```
req 200  remaining=4  retry=
req 200  remaining=3  retry=
req 200  remaining=2  retry=
req 200  remaining=1  retry=
req 200  remaining=0  retry=
req 429  remaining=0  retry=9      ← 6th request: blocked, told to wait 9s
req 429  remaining=0  retry=9      ← 7th: still blocked
```

You can *see* the contract: `remaining` counts down, then `429 + Retry-After` kicks in. Wait 10 seconds and the window resets — `remaining` is back to 4.

---

## Example 2 — Production Scenario

**The situation:** Your Node.js service calls the **GitHub API** to enrich user profiles. During a nightly batch job you fan out 5,000 requests. GitHub's limit is **5,000 requests/hour per token**, and secondary limits kick in on bursts. Suddenly your job is drowning in `429` (and `403` with `Retry-After` for secondary limits). Worse — your first version **retried immediately on every 429**, turning 5,000 requests into 40,000 and getting the token temporarily blocked.

**What GitHub sends back:**

```bash
curl -sS -D - -o /dev/null \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/users/torvalds
```

```
HTTP/2 200
x-ratelimit-limit: 5000
x-ratelimit-remaining: 12          ← almost out — a good client slows down NOW
x-ratelimit-reset: 1752332000      ← absolute Unix epoch when the window refills
x-ratelimit-used: 4988
```

When you go over, GitHub returns `403`/`429` with either `retry-after` (secondary limit) or you compute the wait from `x-ratelimit-reset`.

**The fix — a client that honors the contract: read `Retry-After`, back off exponentially with jitter, and coalesce duplicate work.**

```javascript
// github-client.js — well-behaved retrying client

// (1) Exponential backoff with FULL JITTER (AWS's recommended formula)
function backoffMs(attempt) {
  const base = 500;            // 0.5s
  const cap = 60_000;          // never wait more than 60s
  const exp = Math.min(cap, base * 2 ** attempt);   // 0.5, 1, 2, 4, 8...
  return Math.random() * exp;  // full jitter: pick anywhere in [0, exp)
}

// (2) Parse Retry-After (both forms) OR fall back to X-RateLimit-Reset
function serverToldWaitMs(res) {
  const ra = res.headers.get("retry-after");
  if (ra) {
    if (/^\d+$/.test(ra)) return Number(ra) * 1000;
    const t = Date.parse(ra);
    if (!Number.isNaN(t)) return Math.max(0, t - Date.now());
  }
  const reset = res.headers.get("x-ratelimit-reset");
  const remaining = res.headers.get("x-ratelimit-remaining");
  if (reset && remaining === "0") {
    return Math.max(0, Number(reset) * 1000 - Date.now());
  }
  return null;
}

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function fetchWithBackoff(url, opts = {}, maxAttempts = 6) {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const res = await fetch(url, opts);

    if (res.status !== 429 && res.status !== 403) return res; // success (or a real error)

    // Server's instruction ALWAYS wins over our guess:
    const serverWait = serverToldWaitMs(res);
    const wait = serverWait ?? backoffMs(attempt); // honor Retry-After, else backoff+jitter
    console.warn(`429/403 on ${url} — waiting ${Math.round(wait)}ms (attempt ${attempt + 1})`);
    await sleep(wait);
  }
  throw new Error(`Gave up after ${maxAttempts} attempts: ${url}`);
}

// (3) REQUEST COALESCING — if 50 callers ask for the same user at once,
//     make ONE upstream request and share the promise. Cuts load dramatically.
const inflight = new Map();
function getUser(login) {
  if (inflight.has(login)) return inflight.get(login);   // reuse in-flight request
  const p = fetchWithBackoff(`https://api.github.com/users/${login}`, {
    headers: { Authorization: `Bearer ${process.env.GITHUB_TOKEN}` },
  })
    .then((r) => r.json())
    .finally(() => inflight.delete(login));
  inflight.set(login, p);
  return p;
}
```

**The three fixes that turned the retry storm into a calm, self-healing job:**

1. **Honor `Retry-After` / `X-RateLimit-Reset`** — if the server says "wait 30s", wait 30s. Don't guess less.
2. **Exponential backoff with full jitter** — without jitter, all 5,000 requests retry at the *same* instant (a **thundering herd**) and collide again. Jitter spreads them out.
3. **Request coalescing + a global concurrency cap** (e.g. `p-limit` set to 10) — so you never fan out 5,000 at once in the first place. Fewer requests → fewer `429`s → faster overall completion.

**Flip side — you are the API PROVIDER and a client's retry storm is amplifying load.** The cure is to send a *proper* `429` (not a `500`) with a *generous* `Retry-After`, so well-behaved clients back off instead of hammering:

```javascript
// As the provider: shed load correctly under pressure
if (overLimit) {
  res.set("Retry-After", "60");                 // tell them to actually wait
  res.set("X-RateLimit-Limit", "100");
  res.set("X-RateLimit-Remaining", "0");
  res.set("X-RateLimit-Reset", String(resetEpoch));
  return res.status(429).json({ error: "rate_limit_exceeded" });
  // A 429 is a SIGNAL. A 500 is NOISE — clients retry 500s aggressively,
  // making your overload worse. Return the right status.
}
```

---

## Common Mistakes

### Mistake 1: Retrying immediately on 429 (the retry storm)

```javascript
// ❌ WRONG — this turns one overload into a self-inflicted DDoS
while (true) {
  const res = await fetch(url);
  if (res.status !== 429) break;
  // no wait → instantly retries → server gets MORE load → stays overloaded → 429 forever
}
```

```javascript
// ✅ RIGHT — wait, and back off
const res = await fetchWithBackoff(url); // honors Retry-After, then exponential backoff
```

A `429` means "you're sending too much." Immediately retrying sends *even more*. You and every other client hammering the endpoint form a **retry storm** that keeps the server pinned down long after the original spike passed.

---

### Mistake 2: Ignoring `Retry-After`

```javascript
// ❌ WRONG — server explicitly said "wait 60s", client waits 1s
await sleep(1000);
retry();
```

```javascript
// ✅ RIGHT — the server knows better than you do
const wait = serverToldWaitMs(res) ?? backoffMs(attempt);
await sleep(wait);
```

The server told you *exactly* when to come back. Ignoring it and retrying early wastes both parties' resources and just earns you another `429`. **`Retry-After` always overrides your own backoff guess.**

---

### Mistake 3: Exponential backoff with NO jitter (the thundering herd)

```javascript
// ❌ WRONG — every client computes the SAME delay and retries in lockstep
const wait = base * 2 ** attempt;   // 1000 clients all wait exactly 2000ms...
await sleep(wait);                  // ...then all fire at the exact same millisecond
```

```javascript
// ✅ RIGHT — full jitter spreads the retries across the window
const wait = Math.random() * Math.min(cap, base * 2 ** attempt);
await sleep(wait);
```

Backoff without jitter just **synchronizes** all your clients. They collide, get `429`, back off to the same next value, and collide again. Jitter (randomizing the wait) is what actually breaks the herd apart. This is not optional.

---

### Mistake 4: Rate limiting per-instance instead of globally

```
❌ WRONG: in-memory counter on each of 10 app instances, limit = 100/min each
   Real effective limit = 10 instances × 100 = 1,000/min  (10× your intended limit!)

   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ instance1│  │ instance2│  │ instance3│  ... each counts independently
   │ count 100│  │ count 100│  │ count 100│
   └──────────┘  └──────────┘  └──────────┘
```

```
✅ RIGHT: shared store (Redis) — all instances increment ONE counter

   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ instance1│  │ instance2│  │ instance3│
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        └────────────┬┴─────────────┘
                     ▼
              ┌────────────┐
              │   Redis    │  INCR user:123  → 1 global count = true limit
              │  atomic    │  (use a Lua script or INCR+EXPIRE for atomicity)
              └────────────┘
```

The moment you run more than one app instance (and you will), an in-memory counter multiplies your limit by the instance count. Use a **shared, atomic store** — Redis with `INCR`/`EXPIRE` or a Lua script — so the limit is enforced *globally*. (This is a big reason rate limiting is often pushed up to the API gateway; see `34-api-gateways.md`.)

---

### Mistake 5: Returning 503/500 instead of 429

```
❌ WRONG:  return res.status(500)  // "Internal Server Error"
❌ WRONG:  return res.status(503)  // ambiguous — is the service down?
```
```
✅ RIGHT:  return res.status(429)  // "Too Many Requests" — unambiguous
```

`500` tells the client *"I'm broken"* — and most retry libraries retry `5xx` **aggressively**, amplifying the exact load you're trying to shed. `429` tells the client *"you're going too fast, here's when to return."* Only `429` carries the semantic that makes a client slow down instead of speeding up.

---

### Mistake 6: Not exposing limit headers — clients fly blind

```
❌ WRONG:  return res.status(429).end();
   Client has no idea what the limit is, how much is left, or when to retry.
   It can only guess → guesses badly → retry storm.
```
```
✅ RIGHT:  always send X-RateLimit-Limit / -Remaining / -Reset (on 2xx too!),
           and Retry-After on the 429.
   Now a smart client throttles ITSELF before it ever hits the wall.
```

Sending the headers on **successful** responses is the pro move: clients see `remaining` dropping and pre-emptively slow down, so they never trigger the `429` at all.

---

## Hands-On Proof

```bash
# 1. See a real API's live rate-limit headers (GitHub, unauthenticated = 60/hour)
curl -sS -D - -o /dev/null https://api.github.com/users/torvalds | grep -i ratelimit
#   x-ratelimit-limit: 60
#   x-ratelimit-remaining: 57
#   x-ratelimit-reset: 1752335000
#   x-ratelimit-used: 3

# 2. Actually trip GitHub's unauthenticated limit and catch the 403/429 + reset
for i in $(seq 1 65); do
  curl -sS -o /dev/null -w "%{http_code} " https://api.github.com/users/torvalds
done; echo
#   200 200 200 ... then ... 403 403 403   ← you hit the wall around request 60

# 3. Watch Retry-After appear on a 429 (run against the Example 1 server)
for i in $(seq 1 7); do
  curl -sS -o /dev/null -w "%{http_code} retry=%header{retry-after}\n" http://localhost:3000/
done

# 4. Convert an X-RateLimit-Reset epoch into a human time (how long until reset?)
RESET=$(curl -sS -D - -o /dev/null https://api.github.com/users/torvalds \
        | awk -F': ' 'tolower($1)=="x-ratelimit-reset"{print $2}' | tr -d '\r')
echo "Resets in $(( RESET - $(date +%s) )) seconds"

# 5. Prove Retry-After can be an HTTP-DATE, not just seconds (some 503s use this)
curl -sS -D - -o /dev/null https://httpstat.us/429   # a test endpoint that returns 429
```

---

## Practice Exercises

### Exercise 1 — Easy: Read the contract from a live API

```bash
# Run this and answer the questions:
curl -sS -D - -o /dev/null https://api.github.com/rate_limit

# Also try the JSON body version:
curl -sS https://api.github.com/rate_limit | head -40

# Answer:
# 1. What is your X-RateLimit-Limit unauthenticated?
# 2. What is X-RateLimit-Remaining right now?
# 3. Take X-RateLimit-Reset. Is it a Unix epoch or seconds-until? How can you tell?
#    (Hint: is the number ~1.7 billion, or a small number like 3600?)
# 4. How many seconds until your window resets?
```

### Exercise 2 — Medium: Build a token-bucket limiter and prove the burst

```javascript
// Implement a token bucket: capacity 10, refill 1 token/sec.
// Requirements:
//   - allow() returns true if a token was available (and consumes it), else false
//   - refill based on elapsed wall-clock time, not a setInterval
//
// class TokenBucket {
//   constructor(capacity, refillPerSec) { ... }
//   allow() { ... }   // returns boolean
// }
//
// Then PROVE these two behaviors with a script:
//   A) Fire 15 requests instantly → first 10 succeed (burst), next 5 fail.
//   B) Wait 5 seconds, fire again → exactly 5 succeed (refilled 5 tokens).
//
// Question: how does this differ from a fixed-window limiter for a bursty client?
```

### Exercise 3 — Hard (Production Simulation): Survive a flaky rate-limited API

```javascript
// You must call a mock API that:
//   - allows 3 requests per 10-second window
//   - returns 429 + "Retry-After: <seconds>" when exceeded
//   - occasionally returns 503 (transient) which you SHOULD retry
//   - occasionally returns 400 (your bug) which you should NOT retry
//
// Write a client that fetches 20 items and:
//   1. Honors Retry-After on 429 (parse BOTH seconds and HTTP-date forms).
//   2. Uses exponential backoff WITH full jitter for 503.
//   3. Does NOT retry 4xx other than 429 (a 400 is your fault — retrying won't help).
//   4. Caps total attempts per item at 5, then gives up with a clear error.
//   5. Never runs more than 3 requests concurrently (self-throttle below the limit).
//
// Deliverable: a run log showing NO retry storm — waits should match Retry-After,
// and concurrent 503 retries should be spread out (different jittered delays),
// not all firing at the same millisecond.
//
// Bonus: add request coalescing so duplicate item IDs make only ONE upstream call.
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **A server is overloaded. Why is returning `429` strictly better than returning `500` or `503`?** (Think about what a typical retry library does with each status code.)

2. **Your API runs on 8 identical instances, each with an in-memory limiter set to 200 req/min per user. A user reports they can actually make ~1,600 req/min. Explain the bug and the fix.**

3. **`Retry-After: 45` vs `Retry-After: Sun, 12 Jul 2026 14:05:21 GMT` — what are the two forms, and why must a robust client handle both?**

4. **Fixed-window limit of 100/min. Describe the exact sequence by which a client legitimately gets 200 requests through in about 2 seconds. What algorithm fixes this and how?**

5. **You add exponential backoff to your client but the downstream service still falls over every time it recovers. Nothing is wrong with your backoff math. What single ingredient is missing, and why does its absence cause the problem?**

6. **Token bucket vs leaky bucket: one allows bursts, one smooths them out. Which is which, and give one real backend situation where you'd deliberately choose the smoothing one.**

7. **Your service calls a third-party API 50 times to enrich one page, and 40 of those requests are for the *same* resource. Before touching backoff or limits, what one technique cuts your upstream request count the most, and what is it called?**

---

## Quick Reference Card

| Header / Code | Meaning | Notes |
|---------------|---------|-------|
| `429 Too Many Requests` | You exceeded the limit | RFC 6585. THE status for rate limiting. |
| `Retry-After: 30` | Wait 30 seconds | RFC 9110. delta-seconds form. |
| `Retry-After: <HTTP-date>` | Wait until that moment | Same header, date form. Handle both. |
| `X-RateLimit-Limit` | Ceiling per window | Informal de-facto standard. |
| `X-RateLimit-Remaining` | Requests left now | Sent on 2xx too — throttle before you hit 0. |
| `X-RateLimit-Reset` | When window resets | **Epoch vs. seconds-until varies by API — read docs.** |
| `RateLimit` / `RateLimit-Policy` | IETF-draft standardized versions | Emerging; not yet universal. |

| Algorithm | Bursts? | Memory | Boundary bug? | Use when |
|-----------|---------|--------|---------------|----------|
| Fixed window | accidental 2× | tiny | **yes** | cheapest, rough |
| Sliding log | no | high | no | exact, few clients |
| Sliding counter | no | tiny | no | general (CDN default) |
| Token bucket | **yes, capped** | tiny | no | **user-facing APIs** |
| Leaky bucket | no (smooths) | queue | no | steady-rate backends |

**Client cheat block — how to behave when you get a 429:**
```
1. Read Retry-After.  It wins over any guess you'd make.
   - digits  → wait that many seconds
   - date    → wait until that timestamp
2. No Retry-After? Use X-RateLimit-Reset (if remaining == 0).
3. Still nothing? Exponential backoff: base * 2^attempt, capped.
4. ALWAYS add jitter:  wait = random() * backoff   (breaks the thundering herd)
5. Only retry 429 and 5xx-transient. NEVER retry 400/401/403(auth)/404/422.
6. Cap total attempts. Then fail loudly with a clear error.
7. Coalesce duplicate in-flight requests + cap concurrency so you rarely 429 at all.
```

**Provider cheat block — how to enforce a limit correctly:**
```
1. Return 429 (never 500/503) with a JSON error body.
2. Send Retry-After AND X-RateLimit-* headers (on 2xx too).
3. Use a SHARED store (Redis INCR/EXPIRE, atomic) — never per-instance memory.
4. Pick token bucket for user APIs (tolerate bursts), leaky bucket to protect a DB.
5. Limit on the right dimension: per-IP (anon), per-API-key/user (authed), per-endpoint (expensive routes).
6. Push coarse limits up to the CDN/gateway; keep smart per-user limits in the app.
```

**Key commands:**
```bash
curl -sS -D - -o /dev/null URL | grep -i ratelimit   # inspect live limit headers
curl -sS -o /dev/null -w "%{http_code} %header{retry-after}\n" URL   # status + Retry-After
for i in $(seq 1 65); do curl -s -o /dev/null -w "%{http_code} " URL; done  # trip a limit
```

---

## When Would I Use This at Work?

### Scenario 1: "Our nightly sync job keeps getting throttled by Stripe/GitHub"
You now know the job is fanning out too fast. Add a global concurrency cap (`p-limit`), honor `Retry-After`, back off with jitter, and coalesce duplicate lookups. The job finishes *faster* overall because it stops fighting the limiter. You read `X-RateLimit-Remaining` and pre-emptively slow down before you ever hit `0`.

### Scenario 2: "One customer's script is hammering our API and slowing everyone down"
You add rate limiting at the **API gateway**, keyed on their **API key**, with a token bucket (tolerate their normal bursts, cap the sustained rate). Return `429 + Retry-After`. The noisy tenant self-throttles, and the other customers stop noticing. Because it's enforced in the gateway with Redis, it holds no matter how many app instances you run.

### Scenario 3: "Under a traffic spike our database connection pool gets exhausted and the whole site 500s"
The real fix is a limiter *in front of* the expensive endpoints (a leaky bucket smoothing the input to a rate your DB pool can handle), returning `429` for the overflow. Better that 5% of users see a polite "retry in 10s" than 100% see a hard `500` because the DB fell over. Rate limiting is **load-shedding** — protecting the fragile core (see `39-database-connections-over-network.md`).

### Scenario 4: "Are we vulnerable to credential-stuffing / brute-force on login?"
Yes, unless you rate-limit the auth endpoint specifically — per-IP *and* per-account. A strict `429` on `/login` after N failures (with escalating `Retry-After`) is a first-line defense against password-guessing, independent of the general API limits.

---

## Connected Topics

**Study before this:**
- `14-http-headers-in-depth.md` — `Retry-After`, `X-RateLimit-*`, and how response headers work on the wire. Rate limiting is a headers contract; that topic is the grammar.
- `01-how-the-internet-works.md` — the encapsulation and request lifecycle a `429` rides on.

**Study alongside / next:**
- `15-rest-over-http.md` — where `429` fits among the status codes, and why using the *correct* status code (not `500`) matters semantically.
- `34-api-gateways.md` — the gateway is where global, distributed rate limiting usually lives (per-key, backed by Redis), offloaded from your app.
- `39-database-connections-over-network.md` — the fragile downstream resource rate limiting most often exists to protect.

**Related further out:**
- `16-authentication-over-http.md` — API keys and user identity are the *dimensions* you rate-limit on.
- `30-cdns.md` — the outermost, cheapest place to shed abusive traffic before it reaches your origin.

---

*Doc saved: `/docs/networking/18-rate-limiting-at-http-level.md`*
