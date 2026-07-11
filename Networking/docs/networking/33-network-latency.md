# 33 — Network Latency

> **Phase 5 — Topic 6 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `08-tcp-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine you and a friend are talking, but you live on opposite sides of a huge canyon. You shout a question across the canyon. It takes **2 seconds** for your voice to travel over, and **2 seconds** for their answer to come back. That 4-second round trip is fixed — it's just how far apart you are.

Now imagine you buy a **megaphone**. Can you shout *more words at once*? Yes! But does each word arrive any **faster**? No. It still takes 2 seconds to cross the canyon.

- The **4-second round trip** is **latency** — the delay for one message to make the journey. It's set by distance.
- The **megaphone** is **bandwidth** — how much you can send at once.
- How much you actually get across in a minute of back-and-forth is **throughput**.

Here's the punchline every engineer must feel in their bones: **a bigger megaphone does not shrink the canyon.** If your app is slow because the server is far away, buying "more internet" (bandwidth) won't help. You have to send **fewer messages back and forth**, or move your friend **closer**.

---

## Where This Lives in the Network Stack

Latency is not a single protocol — it is a **property of every layer**. Each layer adds its own delay to the total.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application  (HTTP)   → app logic, DB queries, TTFB   │  ← round trips added HERE
│  Layer 6 — Presentation (TLS)    → handshake = 1 extra RTT       │  ← round trips added HERE
│  Layer 5 — Session               → connection setup/reuse        │
│  Layer 4 — Transport   (TCP)     → 3-way handshake = 1 RTT       │  ← round trips added HERE
│  Layer 3 — Network     (IP)      → routing/processing delay      │  ← processing delay
│  Layer 2 — Data Link   (Ethernet)→ queuing in switch buffers     │  ← queuing delay
│  Layer 1 — Physical    (fiber)   → PROPAGATION DELAY (the floor) │  ← speed of light lives HERE
└─────────────────────────────────────────────────────────────────┘
```

The **immovable floor** lives at Layer 1: light travels through fiber at a fixed speed. Everything above it (handshakes, chatty protocols, N+1 queries) *multiplies* that floor by adding round trips. Your job as a backend developer is to fight the layers you *can* fight (4–7) because the one at the bottom (1) is physics.

---

## What Is This?

**Latency** is the time delay for a single bit of data to travel from source to destination. Measured one-way, or more commonly as **Round-Trip Time (RTT)** — the time for a signal to go there *and* the acknowledgment to come back.

Three words get confused constantly. Nail the distinction now:

| Term | Definition | Unit | Analogy |
|------|-----------|------|---------|
| **Latency** | Delay for one bit to travel A→B | milliseconds (ms) | How long one car takes to drive the highway |
| **Bandwidth** | Maximum capacity of the link | bits/sec (Mbps/Gbps) | Number of lanes on the highway |
| **Throughput** | *Actual* data rate achieved | bits/sec | Cars per hour that actually arrive |

**The highway analogy, fully:**

```
LATENCY   = time for ONE car to drive from city A to city B
            (set by distance + speed limit — adding lanes changes NOTHING)

BANDWIDTH = how many lanes the highway has
            (8 lanes lets more cars travel side by side)

THROUGHPUT= cars that actually reach B per hour
            (limited by lanes, BUT also by traffic jams (congestion),
             on-ramps (handshakes), and the drive time itself)
```

A 10 Gbps link between New York and Sydney still has ~160ms of latency. The fat pipe means you can push a *lot* of data once it's flowing — but the *first byte* still takes 80ms to arrive. **Bandwidth is a capacity problem. Latency is a distance problem. They are not the same knob.**

---

## Why Does It Matter for a Backend Developer?

Because latency is the single most misdiagnosed performance problem in backend engineering. Without understanding it you will:

- **Waste money** "upgrading bandwidth" to fix a distance problem it can't touch.
- **Ship chatty APIs** that feel instant in your local dev environment (0.1ms RTT) and fall apart in production across regions (100ms RTT × 30 calls = 3 seconds).
- **Write N+1 database queries** that are invisible on localhost and catastrophic when the DB is one network hop away.
- **Optimize the wrong percentile** — tune p50 to look great in dashboards while p99 quietly ruins the experience for your most important users.
- **Misunderstand what a CDN does** — expect it to speed up your uncacheable, compute-heavy dynamic endpoint (it won't).
- **Fail system design interviews** — "just add a cache" and "add more servers" are non-answers when the interviewer is probing whether you understand the speed of light.

The engineers who reason correctly about latency are the ones who can look at a 2-second page load and say, in 30 seconds, "that's 20 sequential round trips over a 100ms link — batch them" instead of guessing for two days.

---

## The Packet/Protocol Anatomy

Latency is not stored in a header field — it is the **sum of four physical delays** that every packet accumulates at **every hop** along its path. Memorize these four. This is the entire topic in one diagram.

```
TOTAL ONE-WAY LATENCY  =  Σ (over every hop)  of the four components below:

┌────────────────────────────────────────────────────────────────────────┐
│ 1. PROPAGATION DELAY   = distance / signal_speed                         │
│    "How long light takes to physically cross the wire."                  │
│    Fiber ≈ 200,000 km/s.  THIS IS THE PHYSICS FLOOR.                     │
│    ► Can you reduce it? ONLY by moving endpoints closer. No code helps.   │
│                                                                          │
│ 2. TRANSMISSION DELAY  = packet_size / bandwidth                         │
│    "How long to push all the bits onto the wire (serialization)."        │
│    1500 bytes over 1 Gbps = 12,000 bits / 1e9 = 12 microseconds.         │
│    ► Can you reduce it? YES — more bandwidth, or smaller packets.         │
│                                                                          │
│ 3. PROCESSING DELAY    = time router/switch/OS spends per packet         │
│    "Read header, do routing lookup, verify checksum, decrypt."           │
│    Typically microseconds per hop; adds up over many hops.               │
│    ► Can you reduce it? Somewhat — fewer hops, faster hardware.           │
│                                                                          │
│ 4. QUEUING DELAY       = time waiting in a buffer before being sent      │
│    "The packet sits in a router's queue because the link is busy."       │
│    HIGHLY VARIABLE. 0ms when idle, hundreds of ms under congestion.      │
│    ► Can you reduce it? Partly — avoid congestion, fix bufferbloat, QoS.  │
└────────────────────────────────────────────────────────────────────────┘
```

**A worked breakdown of a single hop** (laptop → router over Wi-Fi, sending a 1500-byte packet on a 100 Mbps link, 10 km away):

```
Propagation:   10 km / 200,000 km/s   = 0.00005 s  = 50 µs   (physics floor)
Transmission:  12,000 bits / 100 Mbps = 0.00012 s  = 120 µs  (serialization)
Processing:    router header lookup    ≈ 0.00001 s  = 10 µs
Queuing:       link idle, no wait      ≈ 0 s        = 0 µs
                                        ─────────────────────
One-hop delay:                          ≈ 180 µs
```

Now multiply that intuition across ~10–15 hops and thousands of kilometers of fiber, and you get real-world RTT. The single biggest term over long distances is almost always **#1, propagation** — the one you cannot code your way out of.

---

## How It Works — Step by Step

### The physics: why there is a floor

```
Speed of light in VACUUM:   ~299,792 km/s   (call it 300,000)
Speed of light in FIBER:    ~200,000 km/s   (~2/3 of c)
```

Why is light *slower* in glass? Fiber's glass core has a **refractive index** of ~1.47–1.5. Light's effective speed = c / index ≈ 300,000 / 1.5 ≈ **200,000 km/s**. This is not an engineering imperfection you can optimize away — it is a material property of glass. No cable, no protocol, no CPU makes light go faster through fiber.

### Worked example 1 — NYC ↔ London (5,585 km great-circle)

```
Theoretical one-way (fiber):  5,585 km / 200,000 km/s  = 27.9 ms
Theoretical RTT:              × 2                        = 55.8 ms
Even at VACUUM light speed:   5,585 / 300,000 × 2        = 37.2 ms  ← hard floor
Real measured RTT:            ~70–76 ms

WHY REAL > THEORETICAL:  cables don't run straight (they follow ocean floors
and existing rights-of-way), plus ~10–15 router hops each adding processing
and queuing delay. Reality is ~1.3–1.4× the straight-line theoretical minimum.
```

### Worked example 2 — NYC ↔ Sydney (15,990 km)

```
Theoretical one-way (fiber):  15,990 / 200,000  = 80.0 ms
Theoretical RTT:              × 2                = 160 ms
Real measured RTT:            ~200–230 ms
```

### Worked example 3 — Tokyo ↔ Virginia (expanding Topic 01)

Topic 01 quoted "~10,000 km → ~100ms one-way → ~200ms RTT." Let's make it precise:

```
Great-circle distance:        ~10,900 km
Theoretical one-way (fiber):  10,900 / 200,000  = 54.5 ms
Theoretical RTT:              × 2                = 109 ms

But the ACTUAL cable path is longer: trans-Pacific submarine cable to the
US west coast, then terrestrial fiber across the continent to Ashburn, VA.
Real routed distance ≈ 14,000–15,000 km of fiber.

Real one-way:                 ~90–100 ms
Real measured RTT:            ~150–170 ms
```

The hard truth Topic 01 stated stands: **no code fixes the speed of light.** You can shave the handshake round trips, but the propagation floor between two fixed points is set the moment you choose where your server lives.

### Worked example 4 — Intra-datacenter

```
Distance:                     ~10–500 m
Propagation delay:            500 m / 200,000 km/s = 0.0000025 s = 2.5 µs (!)
Real measured RTT:            ~0.25–0.5 ms

Propagation is NEGLIGIBLE here. Same-datacenter RTT is dominated by switch
PROCESSING and QUEUING delay + OS network stack overhead, NOT distance.
```

### Why RTT dominates everything

Every conversational step over the network pays **one full RTT**. They stack up:

```
Cold HTTPS request to a server 1 RTT (say 100ms) away:

  DNS lookup ............ 1 RTT  →  100 ms   (if not cached)
  TCP 3-way handshake ... 1 RTT  →  100 ms
  TLS 1.3 handshake ..... 1 RTT  →  100 ms
  HTTP request/response . 1 RTT  →  100 ms  (+ server processing)
                          ─────────────────
  Before ANY byte of body: ~400 ms, and your server did ~0 work.

N sequential round trips  =  N × RTT.  This is THE formula.
```

Chatty protocols and N+1 queries are so damaging precisely because they turn one logical operation into **N sequential RTTs**. Over a 1ms in-region link nobody notices. Over a 100ms cross-region link it's a disaster. The distance didn't change your code — it changed the *cost* of your code's round trips by 100×.

---

## Exact Syntax Breakdown

### `ping` — the raw RTT measurement

```
ping        -c 5        api.myapp.com
│           │           │
│           │           └── target host (resolved to IP, then ICMP echo sent)
│           └── send 5 packets then stop (omit on Windows; use -n 5 there)
└── send ICMP Echo Request, measure time until Echo Reply returns = RTT
```

Annotated output:

```
PING api.myapp.com (203.0.113.10): 56 data bytes
64 bytes from 203.0.113.10: icmp_seq=0 ttl=52 time=71.2 ms   ← RTT for packet 0
64 bytes from 203.0.113.10: icmp_seq=1 ttl=52 time=70.8 ms   ← ttl=52 means
64 bytes from 203.0.113.10: icmp_seq=2 ttl=52 time=94.1 ms   ←   started at 64,
64 bytes from 203.0.113.10: icmp_seq=3 ttl=52 time=70.5 ms   ←   12 hops crossed
64 bytes from 203.0.113.10: icmp_seq=4 ttl=52 time=70.6 ms
--- api.myapp.com ping statistics ---
5 packets transmitted, 5 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 70.5/75.4/94.1/9.3 ms
                                 │    │    │    └── jitter indicator
                                 │    │    └── worst case (that 94ms = queuing spike)
                                 │    └── typical RTT
                                 └── best case ≈ your propagation floor + fixed cost
```

**Read `min`, not `avg`.** The `min` is the closest you'll get to the pure propagation + processing floor (least queuing). The gap between `min` and `max` is **jitter** — variable queuing delay.

### `curl -w` — per-phase latency breakdown (ties to Topic 27)

```bash
curl -o /dev/null -s -w "\
DNS:      %{time_namelookup}s
TCP:      %{time_connect}s
TLS:      %{time_appconnect}s
TTFB:     %{time_starttransfer}s
Total:    %{time_total}s
" https://api.myapp.com/users
```

Each `%{...}` is a **cumulative** stopwatch from t=0. The *delta* between adjacent phases is what each stage cost:

```
time_namelookup   ── DNS done here          → DNS cost = time_namelookup
time_connect      ── TCP handshake done      → TCP cost = time_connect - time_namelookup
time_appconnect   ── TLS handshake done      → TLS cost = time_appconnect - time_connect
time_starttransfer── first response byte     → server  = time_starttransfer - time_appconnect
time_total        ── last byte received      → download= time_total - time_starttransfer
```

### Theoretical vs measured RTT — the sanity-check calc

```bash
# Rule of thumb you can do in your head:
#   min RTT (ms)  ≈  (distance_km × 2)  /  200   [because 200,000 km/s = 200 km/ms]
#
# Example: server 3,000 km away
#   (3000 × 2) / 200 = 30 ms  ← this is the FLOOR. If ping shows 32ms, physics is
#                                the whole story. If it shows 300ms, something is wrong.
```

---

## Example 1 — Basic

Measure RTT to three targets at different distances and *see* the physics.

```bash
# Nearby (same country / cloud region)
ping -c 4 1.1.1.1

# Farther (typically another continent depending on where you are)
ping -c 4 google.com

# Compare against the theoretical floor for a known distance.
```

Expected shape of output (from a machine in the US):

```
--- 1.1.1.1 ping statistics ---          ← Cloudflare, anycast, very close
round-trip min/avg/max = 3.1/4.0/6.2 ms   ← ~3ms = same-metro fiber

--- google.com ping statistics ---
round-trip min/avg/max = 14.2/15.1/21.0 ms ← ~14ms = cross-region, still domestic
```

**What to notice:**

```
3ms RTT   → distance ≈ (3 × 200) / 2  = 300 km of fiber away (or less, +fixed cost)
14ms RTT  → distance ≈ (14 × 200) / 2 = 1400 km of fiber away
```

Now the key experiment — **prove bandwidth doesn't fix latency.** A tiny `ping` packet (64 bytes) and a huge download to the *same host* have the *same* first-byte delay:

```bash
# The ping RTT and the curl TCP-connect time to the same host will match closely,
# regardless of how fat your internet connection is. Latency ≠ bandwidth.
ping -c 3 github.com
curl -o /dev/null -s -w "TCP connect: %{time_connect}s\n" https://github.com
```

Your 1 Gbps home fiber and a friend's 50 Mbps DSL, pinging the same server, will report **nearly identical RTT**. The pipe size is irrelevant to the round-trip delay.

---

## Example 2 — Production Scenario

**The situation:** Your product page calls an internal aggregation service that, to build the response, makes **20 sequential calls** (some to a DB, some to other microservices) — each waiting for the previous one's result. In staging (everything in `us-east-1`, ~1ms RTT) the page renders in ~25ms. Users love it. You launch in Europe, serving them from the same `us-east-1` backend. Now European users report the page takes **~2 seconds**. The team's first instinct: "upgrade the instance / add bandwidth." That is the wrong fix. Here's the diagnosis.

**Step 1 — Measure RTT from the affected region.**

```bash
# Run from a EU box (or use a VPN / cloud shell in eu-west-1) against the backend:
ping -c 5 backend.internal.myapp.com
```
```
round-trip min/avg/max = 88.4/91.2/103.7 ms   ← ~90ms RTT US-East ↔ Europe. Physics.
```

**Step 2 — Do the round-trip math.** The service is doing 20 *sequential* round trips:

```
In-region (staging):   20 calls × 1 ms RTT   =   20 ms   ✅ feels instant
Cross-region (prod):   20 calls × 90 ms RTT  = 1800 ms   ❌ unusable

The code did not change. The RTT went from 1ms to 90ms. 20 × RTT is the whole story.
```

**Step 3 — Confirm it's round trips, not payload/bandwidth.** Check `curl` timing — TTFB is huge but download is tiny:

```bash
curl -o /dev/null -s -w "TTFB: %{time_starttransfer}s  Total: %{time_total}s  Size: %{size_download}B\n" \
  https://backend.internal.myapp.com/product/123
```
```
TTFB: 1.842s  Total: 1.851s  Size: 4210B     ← 4 KB payload. Bandwidth is NOT the issue.
                                                 A 4KB response over even 10 Mbps = ~3ms.
```

A 4 KB response could be served over a *dial-up* modem in well under a second. The 1.8s is **entirely** the 20 stacked round trips. Adding a 10 Gbps NIC changes nothing.

**Step 4 — Fix by removing round trips (not by adding bandwidth).** Two levers:

```
LEVER A — Collapse 20 sequential round trips into 1 (or a few):
  • Batch the DB calls: 20 queries → 1 query with an IN (...) or a JOIN.
  • Parallelize independent service calls: Promise.all() instead of await-in-loop.
  • Add a batch/BFF endpoint so the client makes 1 request, not 20.

  20 sequential × 90ms  = 1800 ms
   1 round trip × 90ms  =   90 ms   ← 20× faster, same distance, same bandwidth.

  If some calls MUST stay separate but are independent, parallel round trips
  cost max(RTT), not sum(RTT):
   20 parallel × 90ms   ≈   90 ms
```

```
LEVER B — Move the data closer (shrink the RTT itself):
  • Deploy a read replica of the DB in eu-west-1 (Topic 39 / 01).
  • Run a regional instance of the aggregation service in Europe.
  • Cache/edge the parts that CAN be cached (Topic 30 CDNs).

  If the whole call chain runs in-region again: 20 × 1ms = 20 ms.
```

**The before/after:**

```
BEFORE:  EU user ──90ms──► us-east-1 (×20 sequential)  = 1800 ms
AFTER-A: EU user ──90ms──► us-east-1 (×1 batched)      =   90 ms
AFTER-B: EU user ──1ms──►  eu-west-1 (×20 sequential)  =   20 ms
WRONG:   "upgrade to 10 Gbps / bigger instance"        = 1800 ms (unchanged) ❌
```

**The lesson:** the fix for a latency problem is **fewer round trips** and/or **less distance** — never "more bandwidth." Bandwidth solves *how much*, not *how far*.

---

## Common Mistakes

### Mistake 1: Conflating latency and bandwidth

```
❌ "Our transfers are slow, let's buy a 10 Gbps link."
```
If the problem is a single small request that's slow to *start*, the bottleneck is latency, and bandwidth won't touch it. A 10 Gbps pipe to Sydney still has ~160ms RTT.
```
✅ Diagnose first: is the FIRST byte slow (latency) or is the download rate slow
   for LARGE payloads (bandwidth)? curl's TTFB vs (total - TTFB) tells you which.
   Small payload + high TTFB = latency. Large payload + low rate = bandwidth.
```

---

### Mistake 2: Thinking a faster server or more bandwidth fixes distance latency

```
❌ "Users in Australia are slow — let's upgrade the US server to more CPU/RAM."
```
The server was never the bottleneck. Tokyo↔Virginia is ~150ms of physics. A 2× faster CPU turns 5ms of processing into 2.5ms — invisible next to 150ms of propagation.
```
✅ Move compute/data closer: regional deploy, read replica, CDN/edge (Topics 30, 01, 39).
   You cannot make the ocean narrower; you can put a server on the other side of it.
```

---

### Mistake 3: Ignoring the round-trip cost of chatty designs (N+1)

```javascript
// ❌ N+1: one query for the list, then one query PER item — sequential round trips
const orders = await db.query("SELECT * FROM orders WHERE user_id = ?", [userId]);
for (const order of orders) {
  order.items = await db.query("SELECT * FROM items WHERE order_id = ?", [order.id]);
  // ↑ each iteration = 1 more RTT. 50 orders = 50 sequential round trips.
}
// On localhost (0.1ms RTT): 50 × 0.1ms = 5ms — invisible.
// DB one hop away (2ms RTT): 50 × 2ms  = 100ms.
// DB cross-region (90ms RTT): 50 × 90ms = 4.5 SECONDS.
```
```javascript
// ✅ One round trip, regardless of distance:
const orders = await db.query("SELECT * FROM orders WHERE user_id = ?", [userId]);
const items  = await db.query(
  "SELECT * FROM items WHERE order_id IN (?)", [orders.map(o => o.id)]);
// 2 round trips total. 90ms distance now costs 180ms, not 4.5s.
```

---

### Mistake 4: Optimizing p50 while p99 kills UX

```
❌ "Average latency is 40ms, we're fine." → averages hide the tail.
```
If a page fans out to 20 backend calls and each has a 40ms p50 but a 900ms p99, the *page* almost certainly hits at least one slow call — so the user-perceived latency tracks p99, not p50. Averages lie because a few very slow requests are exactly the ones users complain about.
```
✅ Track p95/p99. In fan-out, user-perceived latency ≈ the SLOWEST dependency.
   With 20 parallel calls each at 5% chance of being "slow", P(at least one slow)
   = 1 − 0.95^20 ≈ 64%. The tail becomes the norm.
```

---

### Mistake 5: Forgetting handshake round trips

```
❌ Opening a fresh HTTPS connection per request in a loop / no connection reuse.
```
Every cold connection pays DNS + TCP (1 RTT) + TLS (1 RTT) *before* the request. Over a 100ms link that's ~300ms of pure overhead per call, repeated needlessly.
```
✅ Reuse connections: HTTP keep-alive, connection pools, HTTP/2 multiplexing,
   TLS 1.3 (1-RTT) / 0-RTT resumption (Topics 08, 09, 39). Amortize the handshake
   once, then every subsequent request pays only 1 RTT.
```

---

### Mistake 6: Assuming a CDN fixes uncacheable dynamic compute

```
❌ "We put CloudFront in front of it, so the slow /checkout API is fixed now."
```
A CDN shortens the *network path* and serves *cached* content from the edge. For an **uncacheable, compute-heavy** dynamic endpoint, the request still travels edge → origin, and your origin still does all the slow work. The CDN helped the miles, not the milliseconds of computation.
```
✅ CDN helps: static assets, cacheable responses, TLS termination near the user,
   and a warm keep-alive backbone to origin. It does NOT reduce origin compute
   time for a cache miss (Topic 30). Fix the origin separately.
```

---

## Hands-On Proof

```bash
# 1. Measure RTT and read min (floor) vs max (queuing spikes)
ping -c 10 1.1.1.1
ping -c 10 google.com
# Compute the theoretical floor: (min_RTT_ms × 200) / 2 ≈ distance in km.

# 2. Prove bandwidth ≠ latency: tiny ping and small curl to same host ≈ same delay
ping -c 3 github.com
curl -o /dev/null -s -w "TCP connect: %{time_connect}s\n" https://github.com

# 3. Full per-phase latency breakdown (which RTT is which — ties to Topic 27)
curl -o /dev/null -s -w "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" https://api.github.com/zen

# 4. Watch latency GROW hop by hop (each router = propagation + processing added)
traceroute 1.1.1.1        # macOS/Linux
tracert 1.1.1.1           # Windows
# The line where RTT jumps 50–150ms = a long-haul (often undersea) fiber link.

# 5. See queuing/jitter under load: continuous ping while downloading a big file
#    (bufferbloat demo — RTT balloons when your uplink saturates)
ping google.com           # in one terminal, keep it running
curl -o /dev/null https://speed.hetzner.de/100MB.bin   # in another; watch ping RTT rise

# 6. Isolate handshake cost: compare cold connection vs reused (keep-alive)
curl -o /dev/null -s -w "1st (cold): TLS %{time_appconnect}s Total %{time_total}s\n" https://api.github.com/zen
curl -o /dev/null -s -w "reuse (2 reqs, same conn): Total %{time_total}s\n" \
  https://api.github.com/zen https://api.github.com/zen   # 2nd request skips handshake
```

---

## Practice Exercises

### Exercise 1 — Easy: Estimate a distance from a ping

```bash
ping -c 10 <any-website-or-server>

# From the output:
# 1. Read the MIN round-trip time (not avg). Why min and not avg?
# 2. Using the rule (distance_km ≈ min_RTT_ms × 200 / 2), estimate how many km
#    of fiber separate you from that server.
# 3. Look at max − min. What physical delay component does that gap represent?
# 4. Ping it again over Wi-Fi vs wired (if possible). Which is more stable? Why?
```

### Exercise 2 — Medium: Prove bandwidth doesn't fix latency

```bash
# Pick a server on another continent (or use a cloud shell in a far region).
# 1. ping it, record min RTT.
# 2. curl a SMALL endpoint from it, record time_starttransfer (TTFB).
# 3. curl a LARGE file (a few MB) from the same host, record time_total and size.
curl -o /dev/null -s -w "TTFB: %{time_starttransfer}s Total: %{time_total}s Size: %{size_download}B Speed: %{speed_download}B/s\n" <large-file-url>

# Answer:
# a) How much of the small request's time is just the RTT-based handshakes?
# b) For the large file, is total time dominated by RTT or by download rate?
# c) If you doubled your bandwidth, which of the two requests would speed up, and why?
```

### Exercise 3 — Hard: Model and fix a chatty cross-region endpoint

```bash
# Scenario: an endpoint makes N sequential internal calls. You measure:
#   - in-region RTT: 1 ms
#   - cross-region RTT: 95 ms
#   - per-call server processing: 3 ms

# 1. Write the formula for total latency as a function of N and RTT (sequential).
# 2. Compute total in-region and cross-region latency for N = 15.
# 3. You can either (a) parallelize the 15 calls, or (b) batch them into 1 call,
#    or (c) move the backend in-region. Compute the resulting latency for EACH
#    fix in the cross-region case. Which single fix helps most? Which combo is best?
# 4. Now suppose each call has p50 = 3ms but p99 = 120ms of processing. For 15
#    PARALLEL calls, estimate P(the request hits at least one p99 call) and explain
#    why the endpoint's tail latency is far worse than any single call's p50.
#    (Hint: P(at least one) = 1 − 0.99^15.)
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **In one sentence each, define latency, bandwidth, and throughput. Which one does "upgrade to a 10 Gbps link" improve, and which does it NOT touch?**

2. **Name the four components of network latency. Which one is the physics floor you cannot reduce with code, and which one is the most variable?**

3. **A server is 6,000 km away by fiber. What is the theoretical minimum RTT? (Show the arithmetic.) Why is the real measured RTT typically ~1.3–1.4× higher?**

4. **Your endpoint makes 25 sequential DB calls. It's 30ms on localhost and 4 seconds in production. The DB moved from same-host to a cross-region replica at 95ms RTT. Explain the math, and give the two categories of fix (neither of which is "more bandwidth").**

5. **Why is light slower in fiber (~200,000 km/s) than in vacuum (~300,000 km/s), and roughly what fraction of c is that?**

6. **Your dashboard shows p50 = 45ms and the team is celebrating, but users complain about slowness. A page fans out to 30 backend calls. Explain how p99 can dominate user-perceived latency even when p50 looks great.**

7. **A cold HTTPS request pays DNS + TCP + TLS + the request itself. Over a 100ms-RTT link, roughly how long before the first response byte arrives, and name two techniques that eliminate the handshake round trips on subsequent requests.**

---

## Quick Reference Card

**The four components of latency:**

| Component | Formula | Typical size | Can you reduce it? |
|-----------|---------|--------------|--------------------|
| Propagation | distance / 200,000 km/s | dominates long distance | ❌ Only by moving closer |
| Transmission | packet_size / bandwidth | µs on fast links | ✅ More bandwidth / smaller packets |
| Processing | per-hop router/OS work | µs per hop | ~ Fewer hops, faster HW |
| Queuing | wait in buffers | 0 → hundreds of ms | ~ Avoid congestion, QoS |

**Latency numbers every programmer should know** (order-of-magnitude, Jeff Dean-style):

| Operation | Latency | Human scale (if L1 = 1 second) |
|-----------|---------|-------------------------------|
| L1 cache reference | 0.5 ns | 1 second |
| L2 cache reference | 7 ns | 14 seconds |
| Main memory reference | 100 ns | 3.3 minutes |
| SSD random read | 16 µs | ~9 hours |
| Datacenter round trip | 0.5 ms | ~11.5 days |
| Disk seek | 2 ms | ~46 days |
| CA → Netherlands → CA (RTT) | 150 ms | ~9.5 years |

```
Visual scale (log-ish) — every step is roughly an order of magnitude:

  L1  0.5ns  │▏
  L2    7ns  │▎
  RAM 100ns  │█
  SSD  16µs  │████████
  DC-RT 0.5ms│██████████████
  disk   2ms │████████████████
  CA↔NL 150ms│██████████████████████████████████████████████████
             └────────────────────────────────────────────────────►
             Network round trips are the SLOWEST thing in this table.
             A single cross-continent RTT ≈ 1.5 MILLION main-memory reads.
```

**Geographic RTT cheat sheet** (extends Topic 01):

```
Same server (loopback):      <0.1 ms
Same data center:            0.25–0.5 ms
Same city / AZ:              1–5 ms
Cross-country (US):          30–60 ms
US ↔ Europe:                 70–100 ms
US ↔ Asia (Tokyo):           130–170 ms
US ↔ Australia:              180–230 ms
Anywhere ↔ anywhere (worst): ~250–300 ms
```

**Mental arithmetic:**
```
200,000 km/s = 200 km/ms   →   min RTT (ms) ≈ (distance_km × 2) / 200
N sequential round trips    =   N × RTT      ← the formula that explains chatty APIs
Parallel round trips        =   max(RTT)     ← Promise.all beats await-in-loop
```

**Key commands:**
```bash
ping -c 5 HOST                                     # RTT: read MIN (floor) & jitter
traceroute HOST                                    # per-hop latency growth
curl -o /dev/null -s -w "%{time_starttransfer}\n" URL   # TTFB (RTTs + server)
curl ... -w "TCP:%{time_connect} TLS:%{time_appconnect}" # isolate handshake RTTs
```

---

## When Would I Use This at Work?

### Scenario 1: "The app is fine for us but our customers overseas say it's slow"

You immediately reach for `ping` and `curl -w` from (or toward) the affected region. If `min` RTT is ~90ms, that's the Atlantic — a distance problem, fixed by a regional deploy / read replica / CDN, **not** a bigger server. If RTT is 5ms but TTFB is 2s, it's your origin code. You've split "latency vs compute" in two minutes instead of two days.

### Scenario 2: "This page got slow after we added a feature"

You suspect the new feature introduced sequential round trips. You count the backend/DB calls in the request path, multiply by the RTT to those dependencies, and compare to the observed slowdown. If a new N+1 query added 30 sequential calls over a 2ms link, that's ~60ms — you found it by arithmetic, then you batch or parallelize.

### Scenario 3: Capacity/architecture planning and design interviews

When someone proposes "just add bandwidth" or "put a CDN on the dynamic API," you can articulate precisely what those do and don't fix. You reason about where to place data (replicas near users), how to collapse round trips (batching, HTTP/2, GraphQL/BFF), and which SLO percentile to defend (p99, not p50). This is the difference between hand-waving and engineering.

### Scenario 4: Debugging tail latency in a fan-out service

Your service calls 20 downstreams in parallel and its p99 is terrible even though each downstream's p50 is great. You now know user-perceived latency ≈ the slowest dependency, so you attack the tail: hedged requests, timeouts with fallbacks, and fixing the one slow dependency rather than the fast majority.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the physical path (undersea cables, IXPs) and the original Tokyo↔Virginia physics example this topic expands.
- `08-tcp-in-depth.md` — the 3-way handshake that costs 1 RTT, and why connection reuse matters.

**Study alongside / after:**
- `27-debugging-a-slow-api.md` — the `curl -w` methodology to attribute latency to DNS/TCP/TLS/TTFB/transfer.
- `30-cdns.md` — moving content closer to shrink the propagation floor (and what a CDN does *not* fix).
- `35-network-in-system-design.md` — the latency numbers and bandwidth math applied to design trade-offs and interviews.
- `39-database-connections-over-network.md` — connection pooling to amortize handshake round trips and avoid per-query RTT storms.

**The one-line summary to carry forward:** *Bandwidth is how much; latency is how far. Fewer round trips and less distance — never more bandwidth — fix a latency problem, because no code changes the speed of light.*

---

*Doc saved: `/docs/networking/33-network-latency.md`*
