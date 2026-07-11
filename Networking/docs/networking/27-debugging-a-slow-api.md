# 27 — Debugging a Slow API

> **Phase 4 — Topic 7 of 7**
> Prerequisites: `21-curl-mastery.md`, `22-dig-and-dns-debugging.md`, `24-netstat-and-ss.md`

---

## ELI5 — The Simple Analogy

Imagine you order a pizza and it arrives an hour late. "The pizza place is slow!" you shout. But *is* it?

Break the hour down:
1. You looked up their phone number (**DNS**).
2. You dialed and waited for someone to pick up (**TCP connect**).
3. They confirmed your identity and read back a secret code (**TLS handshake**).
4. They took your order, then the kitchen actually cooked it (**server processing / TTFB**).
5. The driver drove it to your door (**content transfer**).

Maybe the kitchen was fast (5 minutes) but the driver got lost for 55 minutes. Or maybe the driver was instant but the kitchen was backed up. **You cannot fix "the pizza is slow" until you know which of the five steps ate the time.**

Debugging a slow API is exactly this: a single request's total time is made of distinct, separately-measurable phases. The whole skill is *attributing the delay to the right phase*, because each phase has completely different causes and fixes. Blaming "the network" when the kitchen is the problem wastes your whole afternoon.

---

## Where This Lives in the Network Stack

This is a **methodology topic**, not a protocol. It spans the entire stack, because a single HTTP request touches every layer — and each layer contributes a measurable slice of the total time:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   HTTP request/response   → TTFB + transfer   │  time_starttransfer, time_total
│  Layer 6 — Presentation  TLS handshake           → TLS phase         │  time_appconnect
│  Layer 4 — Transport     TCP 3-way handshake      → connect phase     │  time_connect
│  Layer 3 — Network       IP routing / distance   → RTT baseline      │  ping / traceroute
│  Layer 7 — Application   DNS resolution           → lookup phase      │  time_namelookup
└─────────────────────────────────────────────────────────────────────┘
```

Every tool from Phase 4 plugs into a specific phase here. This topic is the **integration layer** that turns `curl`, `dig`, `ss`, `traceroute`, and `tcpdump` from isolated commands into a single diagnostic pipeline.

---

## What Is This?

**"Debugging a slow API"** is a repeatable methodology: you take one number everyone complains about — "the API takes 2 seconds" — and decompose it into five measurable phases, identify which phase is guilty, then reach for the specific tool that diagnoses *that* phase.

The five phases of a single HTTPS request, in order:

```
   ┌──────── total request time ────────────────────────────────────────┐
   │                                                                     │
   │  DNS      TCP        TLS          server wait          transfer     │
   │ lookup  connect   handshake   (request→think→1st byte)  (body)      │
   ▼                                                                     ▼
   ├────────┼──────────┼────────────┼──────────────────────┼────────────┤
   │        │          │            │                      │            │
   0     name-      connect     app-              start-             total
        lookup                 connect          transfer
        (DNS)     (TCP done)  (TLS done)     (first byte in)

  Phase        curl variable          How to compute the phase's duration
  ─────────────────────────────────────────────────────────────────────
  DNS          time_namelookup        = time_namelookup            (from 0)
  TCP connect  time_connect           = time_connect     − time_namelookup
  TLS          time_appconnect        = time_appconnect  − time_connect
  (pre-xfer)   time_pretransfer       = time_pretransfer − time_appconnect
  server wait  time_starttransfer     = time_starttransfer − time_pretransfer
  transfer     time_total             = time_total       − time_starttransfer
```

Every curl timing variable is a **cumulative timestamp measured from t=0** (the moment curl starts). To get the duration of a single phase you subtract the previous milestone. That subtraction is the entire trick — miss it and you'll think TLS took 370ms when it actually took 185ms (the other 185ms was TCP that came before it).

---

## Why Does It Matter for a Backend Developer?

This is the single most common real production task you will do: someone says *"the API is slow"* and you have 10 minutes to find out why before the on-call escalates.

Without this methodology you will:
- **Optimize the wrong thing.** Spend a week adding a database index when the real problem was a cross-region DNS resolver.
- **Blame the network when it's your code** (TTFB is huge, everything else is <50ms → it's the server).
- **Blame your code when it's the network** (TTFB is fine, but TCP connect is 300ms → it's routing/distance).
- **Ship a "fix" that does nothing** because you measured once, got lucky, and never saw the p95 tail.

With it, you can walk into any latency incident, run *one* curl command, and within 30 seconds say: *"DNS and TLS are fine, TCP connect is 5ms, but TTFB is 2.1 seconds — this is server-side, pull up the APM and the slow-query log."* That sentence is worth more than a month of guessing.

---

## The Packet/Protocol Anatomy

Here is what actually happens on the wire for one `https://` request, annotated with which phase each packet belongs to and what makes it slow:

```
CLIENT                                                      SERVER
  │                                                            │
  │  ── DNS PHASE ──  (UDP :53, or DoH/DoT)                    │
  │   query  api.myapp.com  ─────────►  resolver               │   SLOW IF: cold cache, low TTL,
  │   answer 142.250.80.46  ◄─────────  resolver               │            far/overloaded resolver
  │                                                            │
  │  ── TCP CONNECT PHASE ──  (1 RTT)                          │
  │   SYN         ───────────────────────────────────────────►│   SLOW IF: physical distance (RTT),
  │   SYN-ACK     ◄───────────────────────────────────────────│            routing detour, firewall
  │   ACK         ───────────────────────────────────────────►│            queueing, SYN backlog full
  │                                                            │
  │  ── TLS PHASE ──  (1 RTT for TLS 1.3, 2 for TLS 1.2)       │
  │   ClientHello ───────────────────────────────────────────►│   SLOW IF: TLS 1.2 (extra RTT),
  │   ServerHello + Certificate ◄──────────────────────────────│            large/deep cert chain,
  │   Finished    ───────────────────────────────────────────►│            OCSP stapling stall
  │                                                            │
  │  ── SERVER WAIT PHASE (this is TTFB) ──                    │
  │   GET /checkout HTTP/2  ──────────────────────────────────►│   ┌──────────────────────────────┐
  │                                                            │   │ request travels (½ RTT)       │
  │                                            SERVER THINKS   │   │ + DB query / downstream call  │  SLOW IF:
  │                                                            │   │ + business logic / GC / lock  │  server-side
  │   HTTP/2 200  (first byte) ◄────────────────────────────────│   │ + first byte travels (½ RTT)  │
  │                                                            │   └──────────────────────────────┘
  │  ── TRANSFER PHASE ──                                      │
  │   [body bytes] ◄──────────────────────────────────────────│   SLOW IF: big payload, no gzip,
  │   [body bytes] ◄──────────────────────────────────────────│            low bandwidth, TCP
  │   [body bytes] ◄──────────────────────────────────────────│            retransmissions/loss
  │                                                            │
```

**Critical nuance the diagram makes visible:** the "server wait" phase (TTFB) is *not* pure server CPU time. It contains **one network round-trip** (request out, first byte back) **plus** the server's actual processing. That's why to isolate *pure* server processing you compare TTFB against a health endpoint on the *same host* — same RTT, so the difference is server work. More on that in "How It Works."

---

## How It Works — Step by Step

### Step 1 — Measure: get the phase breakdown in one command

```bash
curl -o /dev/null -s -w "\
 DNS lookup:    %{time_namelookup}s\n\
 TCP connect:   %{time_connect}s\n\
 TLS handshake: %{time_appconnect}s\n\
 Pretransfer:   %{time_pretransfer}s\n\
 TTFB:          %{time_starttransfer}s\n\
 Total:         %{time_total}s\n" \
 https://api.myapp.com/checkout
```

### Step 2 — Compute the per-phase deltas (do this in your head)

```
Given curl output:
  time_namelookup   = 0.021
  time_connect      = 0.055     → TCP     = 0.055 − 0.021 = 0.034 (34ms)
  time_appconnect   = 0.089     → TLS     = 0.089 − 0.055 = 0.034 (34ms)
  time_pretransfer  = 0.089     → (~0, negligible)
  time_starttransfer= 2.140     → TTFB    = 2.140 − 0.089 = 2.051 (2051ms) ← GUILTY
  time_total        = 2.160     → transfer= 2.160 − 2.140 = 0.020 (20ms)

  DNS = 21ms · TCP = 34ms · TLS = 34ms · SERVER = 2051ms · transfer = 20ms
```

### Step 3 — Route to the guilty phase with the decision tree

```
                    ┌─────────────────────────────┐
                    │  Which phase is the biggest? │
                    └──────────────┬──────────────┘
        ┌──────────────┬───────────┼────────────┬───────────────┐
        ▼              ▼           ▼            ▼               ▼
   DNS is slow    TCP connect   TLS is slow  TTFB is slow   transfer slow
  (namelookup)    is slow      (appconnect  (starttransfer  (total −
                (connect −      − connect)   − pretransfer)  starttransfer)
                 namelookup)
        │              │           │            │               │
        ▼              ▼           ▼            ▼               ▼
  Resolver / TTL  Network       Cert chain / SERVER-SIDE   Payload /
  problem.        distance /    handshake /  App code,     bandwidth /
                  routing /     TLS version. DB query,     compression.
  → dig,          firewall.                  downstream    Is gzip on?
    topic 22      → ping,       → openssl    call, cold    Payload huge?
                    traceroute,   s_client,  start, lock   → check
  Fixes: raise      mtr,          topic 09   contention.   Content-
  TTL, use          topic 25                               Encoding,
  faster/closer                  Fixes: TLS  → APM, logs,  Accept-
  resolver, cache Fixes: move    1.3, shorter slow-query   Encoding,
  DNS in-app.     region, CDN,   chain, OCSP  log, ss for  topic 33
                  fix BGP/peering stapling.   pool state,
                  edge PoP.                   topic 39 for
                                              DB conns.
```

### Step 4 — Isolate server processing from network time

TTFB includes one network RTT. To prove the delay is *server-side* and not network, run three comparisons:

```
1. Compare against a trivial endpoint on the SAME host (same RTT):
     curl -w "%{time_starttransfer}\n" https://api.myapp.com/health   → 0.09s
     curl -w "%{time_starttransfer}\n" https://api.myapp.com/checkout → 2.14s
   Same DNS/TCP/TLS/RTT. Difference (2.05s) is PURE server work on /checkout.

2. Compare from near vs far (bypass DNS with --resolve to keep it fair):
     From same-region box:  TTFB 2.10s   ← server work dominates
     From cross-region box: TTFB 2.30s   ← only +200ms, still server-bound
   If far TTFB were 10× near, the network would be the story. It isn't.

3. Bypass DNS entirely to rule DNS in or out:
     curl --resolve api.myapp.com:443:203.0.113.5 -w "%{time_namelookup}\n" ...
   time_namelookup drops to ~0 → any leftover slowness is NOT DNS.
```

### Step 5 — Narrow the guilty phase with its dedicated tool

```
DNS slow    → dig api.myapp.com          (see query time, TTL, which NS answered)
TCP slow    → traceroute / mtr api...    (find the hop where latency jumps)
TLS slow    → openssl s_client -connect  (inspect cert chain depth, TLS version)
TTFB slow   → server APM + logs + ss     (slow query? pool exhausted? GC pause?)
transfer    → curl -w size + Content-Encoding (payload size, gzip on/off)
```

### Step 6 — Measure MANY times, not once (find the p95 tail)

One measurement lies. Caching, warm connections, and GC pauses mean the *distribution* matters more than any single sample. Loop it (see "Exact Syntax") and read p50 vs p95. A p50 of 40ms with a p95 of 1,800ms is a completely different bug (intermittent — GC, cold pool, lock) than a consistent 1,800ms (structural — missing index, sync downstream call).

---

## Exact Syntax Breakdown

### The `-w` (write-out) timing template, dissected

```
curl  -o /dev/null   -s      -w "FORMAT"        URL
│     │              │       │                  │
│     │              │       │                  └── the endpoint under test
│     │              │       └── write-out: print these variables AFTER transfer
│     │              └── silent: hide the progress meter (keeps output clean)
│     └── dump the response BODY to the void (we only want timings, not JSON)
└── the request
```

### Every timing variable (all are cumulative seconds from t=0)

| Variable | Milestone it marks | Phase you get by subtracting |
|----------|--------------------|------------------------------|
| `time_namelookup` | DNS resolution finished | **DNS** = itself |
| `time_connect` | TCP 3-way handshake finished | **TCP** = connect − namelookup |
| `time_appconnect` | TLS handshake finished (0 on plain HTTP) | **TLS** = appconnect − connect |
| `time_pretransfer` | ready to send request (post-TLS/proxy) | pre-xfer gap = pretransfer − appconnect |
| `time_starttransfer` | first response byte received (**TTFB**) | **server wait** = starttransfer − pretransfer |
| `time_total` | last byte received, connection done | **transfer** = total − starttransfer |
| `time_redirect` | total time spent in all prior redirects | add if chasing 3xx with `-L` |
| `size_download` | bytes of body received | pairs with transfer phase |
| `speed_download` | avg download speed (bytes/sec) | pairs with transfer phase |
| `http_code` | final HTTP status | sanity: are you even getting 200? |
| `num_connects` | new TCP connections opened | 0 on a reused keep-alive conn |

### Reusable template file (cleaner than a giant inline string)

Save this once as `curl-timing.txt`:

```
    time_namelookup:  %{time_namelookup}s\n
       time_connect:  %{time_connect}s\n
    time_appconnect:  %{time_appconnect}s\n
   time_pretransfer:  %{time_pretransfer}s\n
 time_starttransfer:  %{time_starttransfer}s\n
                     ----------\n
         time_total:  %{time_total}s\n
```

Then run:

```bash
curl -w "@curl-timing.txt" -o /dev/null -s https://api.myapp.com/checkout
```

The `@` means "read the write-out format from this file." Keep it in your dotfiles — this is the command you'll run hundreds of times.

### Loop for percentiles (p50 / p95)

```bash
# Fire 50 requests, capture TTFB only, sort, pick percentiles
for i in $(seq 1 50); do
  curl -o /dev/null -s -w "%{time_starttransfer}\n" https://api.myapp.com/checkout
done | sort -n | awk '
  { a[NR]=$1 }
  END {
    print "p50:", a[int(NR*0.50)]"s";
    print "p95:", a[int(NR*0.95)]"s";
    print "max:", a[NR]"s";
  }'
```

**Reading the result:**
```
consistent slowness:   p50 1.9s  p95 2.1s   → structural (missing index, sync call)
variance / tail:       p50 0.04s p95 1.8s   → intermittent (GC, cold pool, lock, cold start)
```

---

## Example 1 — Basic

**Goal:** run the breakdown against a public endpoint and read which phase dominates.

```bash
curl -o /dev/null -s -w "\
 DNS:      %{time_namelookup}s\n\
 TCP:      %{time_connect}s\n\
 TLS:      %{time_appconnect}s\n\
 TTFB:     %{time_starttransfer}s\n\
 Total:    %{time_total}s\n" \
 https://api.github.com/zen
```

**Output:**
```
 DNS:      0.028s
 TCP:      0.061s
 TLS:      0.128s
 TTFB:     0.191s
 Total:    0.192s
```

**Now compute the phases (subtract the previous milestone):**
```
 DNS       = 0.028                 = 28ms
 TCP       = 0.061 − 0.028         = 33ms   ← one RTT to GitHub
 TLS       = 0.128 − 0.061         = 67ms   ← ~2×RTT (TLS 1.2-ish setup)
 server    = 0.191 − 0.128         = 63ms   ← request RTT + tiny server work
 transfer  = 0.192 − 0.191         =  1ms   ← body is a one-line quote
```

**Read it:** nothing is wrong here — 192ms total, all phases small and proportional to distance. TLS is the single biggest slice (67ms), which is normal for a fresh HTTPS connection. Now run it a **second** time immediately:

```
 DNS:      0.001s    ← cached! DNS dropped from 28ms to 1ms
 TCP:      0.034s
 TLS:      0.101s
 TTFB:     0.166s
 Total:    0.167s
```

DNS collapsed because your resolver cached the answer. **This is exactly why measuring once lies** — the first request paid the DNS tax, the second didn't. Lesson learned before you ever touch production.

---

## Example 2 — Production Scenario

### Case A — "The checkout API is slow" (the classic server-side bug)

**The page:** on-call gets paged. Checkout p95 is 2.3s, normally 120ms. Revenue graph is dipping. You have curl and 10 minutes.

**Step 1 — one command to localize the phase:**
```bash
curl -w "@curl-timing.txt" -o /dev/null -s https://api.shop.com/checkout
```
```
    time_namelookup:  0.019s
       time_connect:  0.041s
    time_appconnect:  0.074s
   time_pretransfer:  0.074s
 time_starttransfer:  2.281s     ← !!!
                     ----------
         time_total:  2.298s
```

**Step 2 — compute deltas:**
```
 DNS       = 19ms      ✅ fine
 TCP       = 22ms      ✅ fine (server is nearby, 1 RTT)
 TLS       = 33ms      ✅ fine (TLS 1.3, one RTT)
 SERVER    = 2207ms    🚨 the entire delay lives here
 transfer  = 17ms      ✅ fine (small JSON)
```
Verdict in 30 seconds: **DNS/TCP/TLS are all <75ms. This is 100% server-side.** No point running traceroute — the network is innocent.

**Step 3 — prove it's the server, not a network RTT hiding in TTFB.** Compare against the health endpoint on the same host:
```bash
curl -o /dev/null -s -w "health TTFB: %{time_starttransfer}s\n"   https://api.shop.com/health
curl -o /dev/null -s -w "checkout TTFB: %{time_starttransfer}s\n" https://api.shop.com/checkout
```
```
health TTFB:   0.076s     ← same DNS/TCP/TLS/RTT path
checkout TTFB: 2.281s
```
Same network path, same RTT. The 2.2s difference is **pure application work inside `/checkout`**. Network fully ruled out.

**Step 4 — check whether it's consistent or a tail:**
```bash
for i in $(seq 1 30); do
  curl -o /dev/null -s -w "%{time_starttransfer}\n" https://api.shop.com/checkout
done | sort -n | awk '{a[NR]=$1} END{print "p50:",a[15]; print "p95:",a[29]; print "max:",a[30]}'
```
```
p50: 2.190
p95: 2.410
max: 2.520
```
**Consistent**, not a tail — every single request is ~2.2s. That rules out intermittent causes (GC, cold start, occasional lock) and points at **structural** work on every request: a slow query or a synchronous downstream call.

**Step 5 — narrow inside the server.** Pull the APM trace / slow-query log for `/checkout`:
```
APM span waterfall for POST /checkout (2.21s):
  ├─ auth-middleware ................ 4ms
  ├─ SELECT cart WHERE user_id=$1 ... 3ms
  ├─ SELECT * FROM orders           
  │    WHERE email = $1 ............. 2181ms   ← 🚨 sequential scan, 40M rows
  └─ serialize + respond ........... 9ms
```
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE email = 'a@b.com';
--  Seq Scan on orders  (cost=0..982431 rows=1)  actual time=2180ms
--  Rows Removed by Filter: 39,999,999
```

**Root cause:** a recently-added lookup queries `orders.email`, which has **no index**. Every checkout does a full sequential scan of 40M rows.

**Fix:**
```sql
CREATE INDEX CONCURRENTLY idx_orders_email ON orders (email);
```
Re-measure: `time_starttransfer` drops to `0.061s`. Checkout p95 back to 120ms.

**The lesson:** the network methodology got you from "the API is slow" to "it's server-side" in under a minute, so you never wasted time on traceroute, DNS, or TLS. Then server tools (APM + `EXPLAIN`) finished the job. Curl told you *which room* the fire was in; APM told you *which wire*.

---

### Case B — Same symptom, completely different fault (cross-region routing)

Another team reports their `/checkout` is slow *only* for users in Singapore. Same first command, from a Singapore box:

```
    time_namelookup:  0.021s
       time_connect:  0.331s     ← 🚨 TCP connect is 310ms
    time_appconnect:  0.662s     ← TLS = another 331ms (1 RTT)
 time_starttransfer:  0.995s     ← TTFB = another RTT
         time_total:  1.010s
```
```
 DNS       = 21ms      ✅
 TCP       = 310ms     🚨 this is one full round-trip — huge
 TLS       = 331ms     (1 RTT, but that RTT is 310ms because of distance)
 SERVER    = 333ms     (mostly the RTT again; real server work is ~20ms)
 transfer  = 15ms
```

Here **TCP connect alone is 310ms** — that is a single RTT, and it's enormous. Since RTT ≈ 310ms shows up in *every* phase (TCP, TLS, and TTFB all pay one RTT), the culprit is **network distance/routing**, not the server. Confirm with the path:

```bash
mtr --report --report-cycles 20 api.shop.com
```
```
HOST: sg-box                     Loss%  Avg   Best  Wrst
 1. sg-edge-router                0.0%   1.2    0.9   2.1
 2. singapore-ix                  0.0%   3.4    2.8   6.0
 5. 203.0.113.9 (us-east transit) 0.0% 218.0  210.1 240.2   ← jump: SG → Virginia
 9. api.shop.com (us-east-1)      0.0% 311.4  305.0 330.7
```

**Root cause:** the service only runs in `us-east-1`. Singapore users' packets physically cross the Pacific — ~310ms RTT is the speed of light in fiber, unbeatable by code. TTFB looked "slow" only because it inherits that RTT (see topic 33 for the physics).

**Fix:** none of it is server code. Either deploy a replica in `ap-southeast-1`, or put a CDN/edge PoP in Singapore so the TCP/TLS handshake terminates locally. TCP connect drops from 310ms to 4ms.

**The contrast that matters:** Case A and Case B had the *identical* user complaint ("checkout is slow") but the curl breakdown pointed at opposite fixes — a database index vs. a new region. **Same symptom, different phase, different world.** That is the entire value of this methodology.

---

## Common Mistakes

### Mistake 1: Blaming the network when TTFB proves it's the server

```
❌ "The API is slow, must be the network — let me run traceroute."
   curl breakdown:  DNS 20ms · TCP 30ms · TLS 35ms · TTFB 2100ms · xfer 15ms
```
**Reality:** TCP+TLS being 65ms *proves* the network path is fine — you completed a full handshake in 65ms. The 2.1s is entirely server-side. Running traceroute here is wasted effort.
```
✅ TTFB dominates + connect/TLS are small → go straight to APM / slow-query log / ss.
   Never traceroute when TCP connect is already fast.
```

### Mistake 2: Blaming the server when connect proves it's the network

```
❌ "TTFB is 900ms, our app must be slow — let me profile the code."
   curl breakdown:  DNS 20ms · TCP 300ms · TLS 300ms · TTFB 900ms · xfer 15ms
```
**Reality:** TCP connect (before any of your code runs) is *already* 300ms. That RTT is baked into TLS and TTFB too. Your app might only be doing 20ms of work — the other 880ms is distance/routing. Profiling the code finds nothing.
```
✅ If TCP connect is large, one RTT is inflating every later phase. Fix the network
   (region, CDN, routing) — subtract the RTT before judging the server.
```

### Mistake 3: Measuring once instead of many times (missing the p95 tail)

```
❌ curl ...once... → 40ms → "It's fast, works on my machine, closing the ticket."
```
**Reality:** the user's complaint is about the *tail*. One warm request hides a p95 of 1.8s caused by GC pauses or a cold connection pool.
```
✅ Loop it 30–50× and read p50 vs p95:
     p50 40ms / p95 1800ms  → intermittent (GC, cold pool, lock contention)
     p50 1900ms / p95 2100ms → consistent (missing index, sync downstream call)
   The shape of the distribution IS the diagnosis.
```

### Mistake 4: Testing only from your laptop (not from where users are)

```
❌ "From my laptop it's 50ms, users must be exaggerating."
```
**Reality:** you're in the same region as the server. Users in Sydney cross an ocean. Your laptop can never reproduce a routing/distance problem.
```
✅ Measure from where users actually are: a box in the user's region, a
   synthetic monitor, or --resolve to a specific edge PoP. Compare near vs far.
```

### Mistake 5: Forgetting DNS/connection caching makes the 2nd request artificially fast

```
❌ Run 1: DNS 28ms.  Run 2: DNS 1ms.  → "DNS is fine!" (you only looked at run 2)
```
**Reality:** the first (uncached) request paid the real DNS cost. In production, low TTLs and many distinct clients mean cold lookups happen constantly. Keep-alive also hides fresh TCP/TLS cost after request one.
```
✅ Test cold: fresh resolver cache, or force a new connection each time. The
   FIRST request's numbers are the honest ones for cache-miss latency.
```

### Mistake 6: Changing multiple variables at once (not isolating)

```
❌ "I bumped the pool size AND added an index AND switched region — it's faster now!"
   ...which one fixed it? You'll never know, and two of those changes may be risky.
```
```
✅ Change ONE thing, re-measure, keep or revert. Science, not vibes.
```

### Mistake 7: Ignoring payload size / compression for the transfer phase

```
❌ TTFB is 40ms but Total is 1.8s → "server's fine, ¯\_(ツ)_/¯"
   You skipped the transfer phase (total − starttransfer = 1760ms).
```
**Reality:** the body is a 12 MB uncompressed JSON with no gzip. First byte was instant; the *bytes* took forever.
```
✅ Check size + encoding:
     curl -so /dev/null -w "size:%{size_download} enc-check-headers\n" URL
     curl -sI -H "Accept-Encoding: gzip" URL | grep -i content-encoding
   No gzip on a big payload → enable compression (topic 33) before blaming anything.
```

---

## Hands-On Proof

```bash
# 0. One-time setup: save the reusable timing template
cat > curl-timing.txt <<'EOF'
    time_namelookup:  %{time_namelookup}s\n
       time_connect:  %{time_connect}s\n
    time_appconnect:  %{time_appconnect}s\n
 time_starttransfer:  %{time_starttransfer}s\n
                     ----------\n
         time_total:  %{time_total}s\n
EOF

# 1. Full phase breakdown against any endpoint
curl -w "@curl-timing.txt" -o /dev/null -s https://api.github.com/zen

# 2. Bypass DNS with --resolve, prove namelookup goes to ~0
IP=$(dig +short api.github.com | head -1)
curl --resolve "api.github.com:443:$IP" \
     -o /dev/null -s -w "no-DNS namelookup: %{time_namelookup}s\n" \
     https://api.github.com/zen

# 3. Isolate server vs network: compare a light endpoint to a heavy one on same host
curl -o /dev/null -s -w "light TTFB: %{time_starttransfer}s\n" https://httpbin.org/get
curl -o /dev/null -s -w "heavy TTFB: %{time_starttransfer}s\n" https://httpbin.org/delay/2

# 4. p50/p95 over 30 samples (find the tail)
for i in $(seq 1 30); do
  curl -o /dev/null -s -w "%{time_starttransfer}\n" https://api.github.com/zen
done | sort -n | awk '{a[NR]=$1} END{print "p50:",a[15]"s"; print "p95:",a[29]"s"}'

# 5. Confirm which DNS server answered and how long DNS actually took (topic 22)
dig api.github.com | grep -E "Query time|SERVER|ANSWER SECTION" -A1

# 6. See TCP connection states / TIME_WAIT / pool pressure (topic 24)
ss -tan state established '( dport = :443 )' | head

# 7. Check the transfer phase: payload size + whether gzip is on
curl -sI -H "Accept-Encoding: gzip" https://api.github.com/zen | grep -i "content-encoding\|content-length"
```

---

## Practice Exercises

### Exercise 1 — Easy: Read a breakdown and name the guilty phase
```bash
curl -w "@curl-timing.txt" -o /dev/null -s https://httpbin.org/delay/2
```
```
# Answer from the output:
# 1. Compute each phase (subtract the previous milestone).
# 2. Which phase is by far the largest? (Hint: /delay/2 sleeps 2s server-side.)
# 3. Is DNS/TCP/TLS a problem here? Justify with the numbers.
# 4. In one sentence, is this a network or a server problem — and how do you KNOW?
```

### Exercise 2 — Medium: Isolate server processing from network RTT
```bash
# Measure TTFB for a light and a heavy endpoint on the SAME host:
curl -o /dev/null -s -w "light: %{time_starttransfer}s\n" https://httpbin.org/get
curl -o /dev/null -s -w "heavy: %{time_starttransfer}s\n" https://httpbin.org/delay/3

# Answer:
# 1. Both share the same DNS/TCP/TLS/RTT. What does (heavy − light) represent?
# 2. Why can't you read "pure server time" off a single TTFB number alone?
# 3. Now bypass DNS with --resolve and confirm time_namelookup drops to ~0.
#    Did total time change meaningfully? What does that tell you about DNS here?
```

### Exercise 3 — Hard (Production Simulation): Consistent vs tail, then localize
```bash
# Step 1: capture 50 TTFB samples and get p50/p95/max
for i in $(seq 1 50); do
  curl -o /dev/null -s -w "%{time_starttransfer}\n" https://api.github.com/zen
done | sort -n | awk '{a[NR]=$1} END{print "p50:",a[25]; print "p95:",a[48]; print "max:",a[50]}'

# Step 2: run mtr to the same host to see if the path has loss/latency
mtr --report --report-cycles 20 api.github.com

# Questions:
# 1. Is the slowness consistent (p50 ≈ p95) or a tail (p95 ≫ p50)?
#    Which class of causes does each shape point to?
# 2. If p95 ≫ p50 but mtr shows 0% loss and stable latency, is the tail
#    network or server? What server-side causes create a fat tail?
# 3. Design the exact next command you'd run IF the breakdown had instead shown
#    TCP connect = 280ms. (Which tool, which phase, why?)
# 4. You "fixed" it by bumping the DB pool AND adding an index AND enabling gzip.
#    p95 improved. What is wrong with this experiment, and how would you redo it?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the mapped section.

1. **A single HTTPS request decomposes into five measurable phases. Name them in order, and give the curl variable whose subtraction yields each phase's duration.**

2. **`curl -w` shows `time_appconnect = 0.240` and `time_connect = 0.055`. How long did the TLS handshake actually take, and why is it a mistake to read `time_appconnect` as "TLS took 240ms"?**

3. **TTFB (`time_starttransfer − time_pretransfer`) is 900ms. Why is this NOT necessarily 900ms of server CPU time? How do you isolate the pure server-processing portion?**

4. **Two incidents both report "checkout is slow." In incident A, TCP connect is 8ms and TTFB is 2.1s. In incident B, TCP connect is 300ms and TTFB is 950ms. For each, name the guilty phase, the likely root cause, and the single next tool you'd run.**

5. **You run the timing command once and get 40ms. The user insists it's slow. What is the flaw in your measurement, what do you run instead, and what does p50 ≈ p95 vs p95 ≫ p50 each tell you?**

6. **Map each symptom to its tool: (a) DNS phase is slow, (b) TCP connect is slow, (c) TLS phase is slow, (d) TTFB is slow but everything else is fast, (e) transfer phase is slow.**

7. **Your first curl shows DNS = 28ms; the second, immediately after, shows DNS = 1ms. Which number is the "honest" one for a real user hitting you cold, and why does the second request lie?**

---

## Quick Reference Card

### Phase → variable → cause → tool

| Phase | curl variable (delta) | Slow means | Diagnose with | Topic |
|-------|-----------------------|------------|---------------|-------|
| DNS | `time_namelookup` | resolver far/overloaded, low TTL, cold cache | `dig` (Query time, SERVER, TTL) | 22 |
| TCP connect | `time_connect − time_namelookup` | distance/RTT, routing detour, firewall, SYN backlog | `ping`, `traceroute`, `mtr` | 25 |
| TLS | `time_appconnect − time_connect` | TLS 1.2 extra RTT, deep cert chain, OCSP stall | `openssl s_client` | 09 |
| Server wait (TTFB) | `time_starttransfer − time_pretransfer` | slow query, sync downstream, pool exhaustion, GC, cold start, lock | APM, logs, `ss`, `EXPLAIN` | 39 |
| Transfer | `time_total − time_starttransfer` | big payload, no gzip, low bandwidth, retransmits | `curl -w size`, check `Content-Encoding` | 33 |

### Symptom → verdict cheat block

```
DNS/TCP/TLS all small + TTFB huge   → SERVER-SIDE. Go to APM/logs. Do NOT traceroute.
TCP connect large                   → NETWORK/DISTANCE. One RTT inflates every phase. traceroute/mtr.
TLS >> TCP (roughly 2× the RTT)     → TLS 1.2 or heavy cert chain. openssl s_client.
TTFB fine + Total huge              → TRANSFER. Check payload size + gzip. total − starttransfer.
p50 ≈ p95 (consistent)              → STRUCTURAL: missing index, sync call on every request.
p95 ≫ p50 (fat tail)               → INTERMITTENT: GC pause, cold pool, cold start, lock.
2nd request much faster than 1st    → CACHING artifact. Trust the COLD (first) numbers.
```

### The commands you actually run

```bash
# The one command (save curl-timing.txt in dotfiles)
curl -w "@curl-timing.txt" -o /dev/null -s https://HOST/path

# Inline, no template file
curl -o /dev/null -s -w "DNS:%{time_namelookup} TCP:%{time_connect} TLS:%{time_appconnect} TTFB:%{time_starttransfer} Total:%{time_total}\n" https://HOST/path

# Bypass DNS (rule DNS in/out, keep TLS hostname)
curl --resolve HOST:443:1.2.3.4 -o /dev/null -s -w "%{time_namelookup}\n" https://HOST/path

# Isolate server: light vs heavy endpoint, same host
curl -o /dev/null -s -w "%{time_starttransfer}\n" https://HOST/health
curl -o /dev/null -s -w "%{time_starttransfer}\n" https://HOST/slow

# p50/p95 over N samples
for i in $(seq 1 50); do curl -o /dev/null -s -w "%{time_starttransfer}\n" https://HOST/path; done \
  | sort -n | awk '{a[NR]=$1} END{print "p50:",a[int(NR*.5)]; print "p95:",a[int(NR*.95)]}'

# Phase-specific follow-ups
dig HOST                    # DNS phase   (topic 22)
mtr --report HOST           # TCP phase   (topic 25)
openssl s_client -connect HOST:443   # TLS phase (topic 09)
ss -tan state established '( dport = :443 )'   # pool/TIME_WAIT (topic 24)
sudo tcpdump -ni any host HOST and tcp   # retransmits/resets (topic 23)
```

### The deltas, one more time (memorize this)

```
DNS      = time_namelookup
TCP      = time_connect       − time_namelookup
TLS      = time_appconnect    − time_connect
server   = time_starttransfer − time_pretransfer
transfer = time_total         − time_starttransfer
```

---

## When Would I Use This at Work?

### Scenario 1: The on-call page — "API p95 spiked to 3s"
Your first move is *always* the curl `-w` breakdown, not a code deploy or a server restart. Thirty seconds later you can say "TTFB owns it, network is clean" or "TCP connect is 300ms, this is regional." That single sentence routes the incident to the right team (backend vs. platform/network) and stops five engineers from investigating five wrong things.

### Scenario 2: "It's fast for us, slow for the customer"
You reproduce from the customer's region (or with a synthetic monitor there), run the breakdown, and immediately see whether they're paying a cross-ocean RTT (fix: CDN/edge/region — topic 33) or hitting the same server slowness you do (fix: the code). This kills the endless "works on my machine" argument with numbers.

### Scenario 3: Pre-launch performance review
Before shipping a new endpoint, you loop the timing command 50× for p50/p95, check the transfer phase for a missing gzip, and confirm TTFB is dominated by intended work and not an N+1 query. You catch the missing index in staging instead of at 2am in production.

### Scenario 4: Capacity / pool incidents
When TTFB is a *fat tail* (p95 ≫ p50) and `ss` shows connections piling up in `TIME_WAIT` or the DB pool saturated, you connect this methodology to connection-pool exhaustion (topic 39) — the request isn't slow because the query is slow, it's slow because it *waited for a free connection*. Same TTFB symptom, entirely different fix (raise/tune the pool, add PgBouncer).

---

## Connected Topics

**Study before this (the tools this methodology orchestrates):**
- `21-curl-mastery.md` — the `-w` write-out format and every timing variable used here
- `22-dig-and-dns-debugging.md` — when the DNS phase is guilty
- `24-netstat-and-ss.md` — connection states, TIME_WAIT, and pool pressure behind slow TTFB
- `25-ping-traceroute-mtr.md` — localizing a slow TCP-connect phase to a specific hop

**Study alongside / next:**
- `23-tcpdump-and-wireshark.md` — when TTFB or transfer hides TCP retransmissions/resets
- `09-tls-ssl-in-depth.md` — diagnosing a slow TLS handshake phase
- `33-network-latency.md` — the physics behind large TCP-connect times and the transfer phase
- `39-database-connections-over-network.md` — the #1 cause of server-side TTFB: slow queries and pool exhaustion

**This topic is the capstone of Phase 4.** Every debugging tool you learned in topics 21–26 plugs into exactly one phase of the timeline here. Master this decomposition and "the API is slow" stops being scary — it becomes a five-minute, five-phase checklist.

---

*Doc saved: `/docs/networking/27-debugging-a-slow-api.md`*
