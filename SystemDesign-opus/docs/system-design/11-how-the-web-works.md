# 11 — How the Web Works End to End
## Category: HLD Fundamentals

---

## What is this?

This is the **complete journey** of what happens between the moment you type `google.com` and press Enter, and the moment pixels appear on your screen. Roughly a dozen distinct systems participate, most of them invisible, and the whole thing finishes in about a quarter of a second.

Think of it like **sending a letter to someone whose address you don't know.** You check your own address book (browser cache). Then your family's address book (OS cache). Then you phone directory enquiries (DNS). Once you have the address, you call ahead to make sure they'll accept mail (TCP handshake), agree on a secret code so nobody can read it (TLS), and *then* finally send the letter. At the other end, a receptionist decides which department gets it (load balancer), a clerk does the work (app server), and possibly pulls a file from the archive (database) — unless the answer was already on a sticky note on their desk (cache).

---

## Why does it matter?

**This doc is the spine of the entire curriculum.** Every single component you will study for the next 80 documents is *one hop in this journey.* DNS, load balancers, reverse proxies, caches, CDNs, databases, connection pools — they're not a random list of topics. They are, in order, the things a request passes through.

If you can't narrate this journey, you cannot debug it. And debugging it is your actual job:
- "The site is slow." **Where?** DNS? TLS? The app? The DB? A p99 spike has to come from *somewhere on this path*, and there are only ~10 somewheres.
- "Users in Brazil see a 5-second load; users in Virginia see 300 ms." That's a *specific* hop.
- "It works on my machine." Your machine's DNS cache and warm TCP connection were doing a lot of quiet work.

**In interviews:** "What happens when you type google.com into your browser?" is one of the most-asked questions in all of software engineering — at Google, Amazon, Meta, and every startup that copied their loop. It is deliberately open-ended. A junior gives 4 steps. A senior gives 12, with **timings**, and knows which hop to optimize. That's what this doc makes you.

---

## The core idea — explained simply

### The Restaurant Delivery Analogy

You're hungry. You want food from a restaurant called "Google."

**1. "Do I already have their number?"** You check your phone's recent calls. (Browser cache.) If you ordered 2 minutes ago and still have the menu in hand, **you're done — no journey at all.** This is the fastest possible request: the one you never make.

**2. "Does anyone in the house have it?"** You check the family notepad by the door. (OS cache / hosts file.)

**3. "Call directory enquiries."** You don't know the number, so you call 118 (your ISP's DNS resolver). They don't know either, so *they* call the international registry, who says "ask the .com desk," who says "ask Google's own front desk," who finally gives the number. Directory enquiries then **writes the number down** so the next person who asks gets it instantly. (Recursive DNS resolution + caching.)

**4. "Ring them and confirm they're open."** You dial, they pick up, you both confirm you can hear each other. (TCP handshake.)

**5. "Agree on a private line."** You want to give your card details, so you both agree on a scrambling code first, and they show you ID proving they really are Google. (TLS handshake + certificate.)

**6. "Place the order."** Finally — the actual request. (HTTP GET.)

**7. Inside the restaurant.** The host at the door sends you to a free waiter (load balancer). The waiter passes the ticket to the kitchen (app server). The chef checks whether the dish is already made and sitting under the heat lamp (cache hit) — if yes, plate it and go. If not, they go to the pantry for ingredients (database query) and cook it.

**8. Food comes back.** And *then* you still have to actually eat it — unwrap the containers, lay it out, arrange the table. (The browser parsing HTML, fetching CSS/JS/images, and rendering.) Astonishingly often, **this last step takes longer than everything before it.**

### Mapping the analogy back

| Analogy | Technical step | Typical time |
|---|---|---|
| Recent calls on your phone | **Browser cache** | ~0 ms (hit = done) |
| Family notepad | **OS DNS cache / `/etc/hosts`** | ~0–1 ms |
| Calling directory enquiries | **DNS query to recursive resolver** | 1–20 ms (cached) / 20–120 ms (cold) |
| Enquiries calling the registry, then .com, then Google | **Root → TLD → authoritative nameserver** | ~100–200 ms total if fully cold |
| Ringing and confirming you can hear each other | **TCP 3-way handshake** | 1 RTT (~10–150 ms) |
| Agreeing on a scrambling code + checking their ID | **TLS 1.3 handshake + certificate validation** | 1 RTT (~10–150 ms) |
| Placing the order | **HTTP request** | ~0 ms to send |
| Host seating you at a free waiter | **Load balancer** | 1–5 ms |
| Waiter passing the ticket to the kitchen | **Reverse proxy → app server** | <1 ms |
| Dish already under the heat lamp | **Cache hit (Redis)** | ~1 ms |
| Chef going to the pantry | **Database query** | 5–50 ms |
| Unwrapping and laying out the food | **Browser parse + render** | 50–500 ms |

---

## Key concepts inside this topic

### 1. Step 0 — The request you never make (browser cache)

Before *anything* touches the network, the browser asks: **do I already have this?**

Chrome checks, in order:
1. **Memory cache** — resources from this page session. Instant.
2. **Disk cache** — resources with a valid `Cache-Control: max-age`. ~1–10 ms.
3. **Service worker** — if the page registered one, JS can intercept and serve from IndexedDB. This is how PWAs work offline.
4. **Push/prefetch cache** — things speculatively fetched.

If a response has `Cache-Control: max-age=31536000, immutable` (one year, common for versioned assets like `app.a3f9c1.js`), the browser **does not make a network request at all**. Zero DNS, zero TCP, zero server load.

If the resource is stale, the browser can still avoid downloading it with a **conditional request**:

```
GET /style.css HTTP/1.1
If-None-Match: "a3f9c1"          ← the ETag the server gave last time

HTTP/1.1 304 Not Modified        ← ~200 bytes instead of 200 KB
```

**Nothing you build will ever be faster than the request that doesn't happen.** That's why caching gets its own doc ([59 — Caching in Depth]) and why HTTP cache headers are the cheapest performance win in existence.

### 2. Step 1–2 — DNS resolution (name → IP)

Your browser has `google.com`. TCP needs `142.250.183.14`. Bridging that gap is **DNS**.

The lookup goes through a cascade of caches, and stops at the first hit:

```
1. Browser DNS cache      (Chrome: ~1 min)          ~0 ms
2. OS DNS cache           (systemd-resolved, etc.)  ~0-1 ms
3. /etc/hosts file        (manual overrides)        ~0 ms
4. Router cache                                     ~1-5 ms
5. ISP recursive resolver (or 8.8.8.8, 1.1.1.1)     ~10-50 ms  ← usually stops here
6. Root nameserver        (".")                     +~30-80 ms ┐
7. TLD nameserver         (".com")                  +~30-80 ms │ only if
8. Authoritative NS       ("ns1.google.com")        +~30-80 ms ┘ fully cold
```

**"Recursive" means the resolver does the walking for you.** Your browser asks one question and gets one answer; the resolver may have asked three others behind the scenes:

```
  YOU                RECURSIVE RESOLVER              THE DNS HIERARCHY
   │                    (8.8.8.8)                            │
   │  "google.com?"          │                               │
   ├────────────────────────▶│                               │
   │                         │  "google.com?"          ┌─────▼──────┐
   │                         ├────────────────────────▶│ ROOT (.)   │
   │                         │◀── "ask .com at x.x" ───┤ 13 clusters│
   │                         │                         └────────────┘
   │                         │  "google.com?"          ┌────────────┐
   │                         ├────────────────────────▶│ TLD (.com) │
   │                         │◀ "ask ns1.google.com" ──┤ Verisign   │
   │                         │                         └────────────┘
   │                         │  "google.com?"          ┌────────────┐
   │                         ├────────────────────────▶│ AUTHORITAT.│
   │                         │◀── "142.250.183.14" ────┤ ns1.google │
   │                         │                         └────────────┘
   │◀── "142.250.183.14" ────┤  (and CACHES it for TTL seconds)
   │      + TTL=300          │
```

Two things to carry forward:

- **DNS runs over UDP port 53.** Why UDP? Because a query and a reply is *one exchange* — paying a TCP handshake for it would double the cost. (Falls back to TCP for responses > 512 bytes, and DNS-over-HTTPS/TLS now exists for privacy.)
- **TTL is your deploy lever and your trap.** A record with `TTL=300` means resolvers cache it for 5 minutes. If you're about to fail over to a new IP, **lower the TTL to 60s hours in advance** — otherwise the world keeps hitting your dead IP for however long the old TTL says. And note: some clients ignore TTL and cache longer anyway. **Never rely on DNS as your only failover mechanism.**

Cold DNS: **~100–200 ms.** Warm DNS: **~0–20 ms.** ([54 — DNS Deep Dive] goes much further.)

### 3. Step 3–4 — TCP handshake, then TLS handshake

Now you have an IP. Before one byte of HTTP moves, two handshakes fire.

> Recall from [10 — Networking Fundamentals]: TCP is connection-oriented, so it must establish the connection first (**SYN → SYN-ACK → ACK = 1 RTT**), and TLS 1.3 needs **1 more RTT** to agree on encryption keys and prove the server's identity via a certificate.

```
  CLIENT                                                    SERVER
     │  SYN                    ─────────────────────────────▶ │  ┐
     │  ◀───────────────────── SYN-ACK                        │  │ 1 RTT
     │  ACK                    ─────────────────────────────▶ │  ┘
     │                                                        │
     │  ClientHello (ciphers, key-share)  ──────────────────▶ │  ┐
     │  ◀── ServerHello + CERTIFICATE + Finished              │  │ 1 RTT
     │  Finished + [encrypted data can now flow] ───────────▶ │  ┘
```

Two details people miss:

**Certificate validation is not free and not automatic-magic.** The browser checks: is this cert signed by a CA in my trust store? Is it expired? Does the domain match? Is it revoked (OCSP / OCSP stapling)? A cert chain that requires an *extra network fetch* to validate can add 100+ ms — which is why **OCSP stapling** (server pre-fetches the revocation proof and hands it to you) exists.

**Where the handshake terminates decides how fast this is.** If you use a CDN, the TCP+TLS handshakes terminate at an **edge PoP 10 ms away**, not at your origin 200 ms away. The CDN then talks to your origin over a **pre-warmed, already-open** connection. You just turned 2 × 200 ms into 2 × 10 ms. **That's often the single biggest latency win available to you**, and it's why CDNs are useful even for *uncacheable* content.

Cost: **~2 RTTs.** Same-region: ~10–30 ms. Cross-continent: **~200–300 ms.**

### 4. Step 5–6 — The HTTP request meets the load balancer

Finally the actual request travels:

```
GET / HTTP/1.1
Host: google.com
User-Agent: Mozilla/5.0 ...
Accept: text/html,...
Cookie: SID=abc123...
```

The first thing it hits is **not your app.** It's a load balancer.

- A **Layer 4 LB** (AWS NLB, HAProxy in TCP mode) forwards packets based on IP:port. Fast, dumb, ~microseconds of added latency.
- A **Layer 7 LB** (Nginx, AWS ALB, Envoy) *parses the HTTP*. It can route `/api/*` to the API pool and `/static/*` to the asset pool, do sticky sessions, terminate TLS, and rate limit. Costs ~1–5 ms.

The LB picks a healthy backend using round robin / least connections / IP hash, and **health checks** ensure it never sends you to a dead box. ([55 — Load Balancing].)

Behind the LB often sits a **reverse proxy** (usually Nginx). The distinction confuses everyone, so:

| | Load balancer | Reverse proxy |
|---|---|---|
| **Core job** | Distribute traffic across N identical backends | Sit in front of backends and *do things* to requests |
| **Also does** | Health checks, failover | TLS termination, gzip/brotli, static file serving, caching, header rewriting, buffering slow clients |
| **Can it be one box?** | **Yes.** Nginx is routinely both. The distinction is *roles*, not machines. |

One underrated reverse-proxy job: **buffering slow clients.** A user on 2G takes 8 seconds to receive a 200 KB response. If your Node process streams directly to them, that's 8 seconds of a held connection. Nginx accepts the response from Node in 2 ms, frees Node immediately, and dribbles it out to the slow client itself. ([57 — Reverse Proxy], [58 — API Gateway].)

Cost: **~1–5 ms.**

### 5. Step 7 — The app server, the cache, and the database

Your Node process finally sees the request. Typical path:

```js
// Express route — the "hot path" of almost every web app on Earth.
app.get('/api/user/:id', async (req, res) => {
  const key = `user:${req.params.id}`;

  // 1. CACHE FIRST. A Redis hit is ~1ms round trip inside the datacenter.
  //    A Postgres query with an index is ~5-20ms. That's 5-20x.
  const cached = await redis.get(key);
  if (cached) {
    res.set('Cache-Control', 'private, max-age=60');
    return res.json(JSON.parse(cached));   // ~1ms path. Done.
  }

  // 2. CACHE MISS → go to the source of truth.
  //    This borrows a connection from a POOL — because opening a fresh
  //    Postgres connection costs a TCP handshake + auth (~5-30ms) that
  //    would dwarf the query itself. (See 65 - Connection Pooling.)
  const { rows } = await pool.query('SELECT id, name, email FROM users WHERE id = $1', [
    req.params.id,
  ]);
  if (rows.length === 0) return res.status(404).json({ error: 'not found' });

  // 3. Populate the cache for next time. This is the "cache-aside" pattern.
  //    TTL is your safety net against stale data — the invalidation you forgot.
  await redis.setex(key, 300, JSON.stringify(rows[0]));

  res.json(rows[0]);
});
```

The numbers that make this design obvious:

| Operation | Time | Note |
|---|---|---|
| Redis GET (same datacenter) | **~1 ms** | Dominated by the network RTT, not Redis itself (~0.1 ms of work) |
| Postgres indexed SELECT | **~5–20 ms** | B-tree lookup + network + result serialization |
| Postgres query with no index (seq scan, 1M rows) | **~500 ms–2 s** | The classic "why is prod slow" |
| Opening a *new* Postgres connection | **~5–30 ms** | Why connection pools exist |
| Your JS business logic | **~0.1–5 ms** | Almost never the bottleneck. Stop micro-optimizing it. |

**The pattern to internalize:** at every layer, someone is asking *"do I already have this?"* Browser cache → CDN cache → reverse-proxy cache → Redis → database buffer pool → disk. It's caches all the way down, and each hit skips everything below it.

### 6. Step 8 — The response comes back, and the browser does the *other* half of the work

The server sends `200 OK` plus an HTML document. **You are barely halfway done.**

```
1. Browser receives HTML bytes                             ← TTFB happens here
2. Parses HTML → builds the DOM tree
3. Hits <link rel="stylesheet"> → BLOCKS rendering, fetches CSS
   → new request → possibly new DNS + TCP + TLS if a new domain!
4. Hits <script src="..."> → BLOCKS parsing (unless async/defer)
5. Parses CSS → builds the CSSOM tree
6. DOM + CSSOM  →  RENDER TREE
7. LAYOUT   ("reflow") — compute the box/position of every element
8. PAINT    — fill in pixels, layer by layer
9. COMPOSITE — assemble layers on the GPU → PIXELS ON SCREEN
10. Meanwhile: images, fonts, and XHR/fetch calls stream in and
    trigger more layout/paint cycles.
```

Three facts with real design consequences:

- **CSS is render-blocking. Synchronous JS is parser-blocking.** A single `<script src>` in your `<head>` without `defer` halts DOM construction until it downloads *and executes*. This is why "put scripts at the bottom" / `defer` is universal advice.
- **Every unique domain in your HTML costs a fresh DNS + TCP + TLS.** Loading fonts from one CDN, analytics from another, and images from a third can add **3 × ~300 ms** on a cold cross-continent load. This is why `<link rel="preconnect">` exists.
- **A modern SPA often does the whole journey twice**: load the HTML shell (fast), then the JS bundle (slow), then the JS makes API calls (another full round trip) before showing anything. Server-side rendering exists precisely to collapse that.

**Time: 50–500 ms**, and on a heavy JS app it can dwarf every network hop combined.

### 7. Adding it all up — the full budget

**Scenario A — Cold load, user in Mumbai, server in Virginia (RTT ≈ 200 ms), no CDN.**

| # | Step | Time |
|---|---|---|
| 1 | Browser cache check | ~0 ms |
| 2 | OS cache check | ~1 ms |
| 3 | DNS resolution (cold, full recursive walk) | **~150 ms** |
| 4 | TCP 3-way handshake (1 RTT) | **~200 ms** |
| 5 | TLS 1.3 handshake (1 RTT) | **~200 ms** |
| 6 | HTTP request travels to server (½ RTT) | ~100 ms |
| 7 | Load balancer + reverse proxy | ~3 ms |
| 8 | App server: cache miss → DB query → render | ~25 ms |
| 9 | HTTP response travels back (½ RTT) | ~100 ms |
| 10 | Browser parse + subresources + render | **~300 ms** |
| | **TOTAL** | **≈ 1,080 ms** |

**One point one seconds. And your application code was 25 ms of it — 2%.**

**Scenario B — Warm load, same user, but with a CDN edge PoP 10 ms away (RTT ≈ 20 ms to edge).**

| # | Step | Time |
|---|---|---|
| 1–2 | Browser + OS cache | ~0 ms |
| 3 | DNS (cached locally) | **~2 ms** |
| 4 | TCP handshake **to the edge** | **~20 ms** |
| 5 | TLS handshake **to the edge** | **~20 ms** |
| 6 | HTTP request → edge | ~10 ms |
| 7 | Edge cache **HIT** (static) — or edge→origin over a warm, pre-opened connection | ~1 ms |
| 8 | (skipped — cache hit) | 0 ms |
| 9 | Response → user | ~10 ms |
| 10 | Browser render (assets also cached) | ~120 ms |
| | **TOTAL** | **≈ 183 ms** |

**Same page. Same code. ~6× faster.** Nothing about your business logic changed. **This table is the entire argument of system design in miniature:** the wins are in the architecture around your code, not in your code.

---

## Visual / Diagram description

### Diagram 1: The complete journey (draw this from memory — it's the spine)

```
 ┌──────────┐
 │  YOU     │  type "google.com" + Enter
 │ (browser)│
 └────┬─────┘
      │
      │ ① Browser cache?  ──── HIT → DONE. 0ms. (the fastest request is none)
      │ ② OS cache / hosts? ── HIT → skip to ⑤
      │            MISS
      ▼
 ┌──────────────────────────────────────────────────────────────┐
 │  ③ DNS RESOLUTION   (UDP :53)                    ~1-150ms    │
 │                                                              │
 │   resolver ──▶ ROOT (.)   ──▶ "ask .com"                     │
 │   (8.8.8.8) ──▶ TLD (.com) ──▶ "ask ns1.google.com"          │
 │             ──▶ AUTH NS    ──▶ "142.250.183.14"  TTL=300     │
 └──────────────────────────┬───────────────────────────────────┘
                            │  now we have an IP
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │  ④ TCP HANDSHAKE      SYN → SYN-ACK → ACK        1 RTT       │
 │  ⑤ TLS HANDSHAKE      ClientHello → Cert → Fin   1 RTT       │
 │                       (cert proves it IS google.com)         │
 └──────────────────────────┬───────────────────────────────────┘
                            │  ⑥ GET / HTTP/1.1  (encrypted)
                            ▼
 ╔══════════════════════════════════════════════════════════════╗
 ║                    THE DATACENTER                            ║
 ║                            │                                 ║
 ║                 ┌──────────▼──────────┐                      ║
 ║                 │  ⑦ LOAD BALANCER    │  health checks,      ║
 ║                 │     (L4 or L7)      │  round-robin  ~2ms   ║
 ║                 └──────────┬──────────┘                      ║
 ║                            │                                 ║
 ║                 ┌──────────▼──────────┐                      ║
 ║                 │  ⑧ REVERSE PROXY    │  TLS term, gzip,     ║
 ║                 │     (Nginx)         │  slow-client buffer  ║
 ║                 └──────────┬──────────┘                      ║
 ║                            │                                 ║
 ║          ┌─────────────────┼─────────────────┐               ║
 ║          ▼                 ▼                 ▼               ║
 ║     ┌────────┐        ┌────────┐        ┌────────┐           ║
 ║     │ APP #1 │        │ APP #2 │        │ APP #3 │  Node     ║
 ║     └───┬────┘        └────────┘        └────────┘           ║
 ║         │  ⑨ "do I already have this?"                       ║
 ║         ├──────────────▶ ┌─────────┐   HIT → return  ~1ms    ║
 ║         │                │  REDIS  │                         ║
 ║         │                └─────────┘                         ║
 ║         │  MISS                                              ║
 ║         └──────────────▶ ┌──────────────┐                    ║
 ║                          │  POSTGRES    │   ~5-20ms          ║
 ║                          │  (+ replicas)│                    ║
 ║                          └──────────────┘                    ║
 ╚══════════════════════════╤═══════════════════════════════════╝
                            │  ⑩ 200 OK  <html>...
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │  ⑪ BROWSER RENDER                              ~50-500ms     │
 │                                                              │
 │   HTML ──parse──▶ DOM ──┐                                    │
 │                          ├──▶ RENDER TREE ──▶ LAYOUT ──▶     │
 │   CSS  ──parse──▶ CSSOM ─┘                     PAINT  ──▶    │
 │                                              COMPOSITE ──▶   │
 │   JS   ──blocks parsing unless async/defer──▶  PIXELS 🎉     │
 └──────────────────────────────────────────────────────────────┘
```

### Diagram 2: The same journey as a latency budget bar (this is what you optimize)

```
 COLD LOAD, Mumbai → Virginia, no CDN.  Total ≈ 1080ms
 ├───────────┬────────────┬────────────┬───┬─┬──┬────────────────┤
 │ DNS 150ms │ TCP 200ms  │ TLS 200ms  │REQ│LB│DB│  RENDER 300ms  │
 │           │            │            │200│3 │25│                │
 ╰─── 14% ───┴─── 18% ────┴─── 18% ────┴18%┴  ┴2%┴───── 28% ──────╯
                                                ▲
                                    YOUR CODE IS 2% OF THE PAGE LOAD.

 Optimizations, ranked by how much they actually buy you:
   1. CDN edge     → kills 3 long RTTs (DNS/TCP/TLS to origin)  ~-550ms  ★★★
   2. Browser cache→ kills the ENTIRE request on repeat visit    ~-1000ms ★★★
   3. HTTP/2 or /3 → parallelize subresources, fewer handshakes  ~-150ms  ★★
   4. Keep-alive   → kills TCP+TLS on every request AFTER #1     ~-400ms  ★★
   5. Redis cache  → turns 20ms into 1ms                         ~-19ms   ★
   6. Faster JS    → turns 3ms into 1ms                          ~-2ms    ·
```

**Redraw diagram 2 whenever someone wants to spend a sprint micro-optimizing a function.** The bar chart is the argument.

---

## Real world examples

### 1. Google — engineering every hop

Google has optimized *literally every step* in this doc, because at their scale each millisecond is measurable revenue.

- **DNS:** they run `8.8.8.8`, one of the largest public resolvers on Earth — partly a public good, partly a way to control and observe step 3.
- **TCP + TLS:** they invented **SPDY → HTTP/2** and **QUIC → HTTP/3** specifically to collapse handshake RTTs (QUIC merges transport + crypto into 1 RTT, and 0 RTT on reconnect).
- **Network:** they run their own global backbone, so once a packet hits a Google edge PoP (usually a few ms from you), it travels on **Google's private network**, not the public internet, for the rest of the trip.
- **Render:** the classic Google homepage was deliberately tiny — the goal was for the search box to be interactive before you finished glancing at it.

### 2. Netflix — moving the server *inside* your ISP

Netflix took step ④/⑤ to its logical extreme with **Open Connect**: they ship physical caching appliances **into ISP datacenters**, so the movie you stream comes from a box that is *inside your ISP's building*, sometimes one network hop from you.

The result: the TCP handshake, TLS handshake, and every byte of video have an RTT measured in **single-digit milliseconds**, and Netflix traffic never crosses the expensive public internet backbone at all. This is the CDN idea, pushed as far as it can physically go. Note what it optimizes: not the app server, not the database — **the hops.**

### 3. Cloudflare — being step 4 through 8 for millions of sites

When you put a site behind Cloudflare, your DNS `A` record points at **Cloudflare's** IP, not yours. So:

- The **TCP + TLS handshakes terminate at a Cloudflare PoP** near the user (typically <20 ms away) instead of at your origin (potentially 200 ms away). You just saved ~2 × 180 ms **even for content that can't be cached**.
- Cloudflare then reaches your origin over an **already-warm, already-authenticated** connection it keeps open.
- Static assets are served from the edge cache — steps 7 through 9 never happen at all.
- Bonus: they absorb DDoS traffic at the edge, so it never reaches your load balancer.

This is the concrete demonstration that **latency is architecture**, not code.

---

## Trade-offs

| Optimization | You gain | You give up |
|---|---|---|
| **Aggressive browser caching** (`max-age=1yr`) | Repeat visits cost ~0 ms and ~0 server load | Users can be stuck on stale assets. **Fix:** content-hashed filenames (`app.a3f9c1.js`) — the *name* changes, so the cache is naturally busted |
| **Low DNS TTL** (60 s) | Fast failover to a new IP | More DNS queries; slightly slower cold loads; more load on your NS |
| **High DNS TTL** (24 h) | Fast, cheap resolution | Failover takes up to a day to propagate. **Never rely on DNS alone for failover** |
| **CDN** | Kills 2–3 long RTTs; absorbs traffic and DDoS; huge p99 win | Cost; cache invalidation is genuinely hard; a stale edge can serve wrong content globally |
| **TLS everywhere** | Privacy, integrity, HTTP/2+, SEO, browser features that require it | ~1 extra RTT on cold connections; cert management. **(Not optional in 2026 — just pay it and use TLS 1.3.)** |
| **Server-side rendering** | Content visible on first paint; no waiting for a JS bundle + API call | Server does more work per request; harder caching; more infra |
| **Client-side SPA** | Cheap, cacheable static shell; snappy in-app navigation | Slow *first* load (HTML → JS bundle → API call = 3 sequential round trips before anything shows) |
| **Adding a Redis cache** | 5–20× faster reads; protects the DB | Cache invalidation, stale reads, one more thing that can be down, and a new failure mode (cache stampede) |

**Rule of thumb:** Optimize in this order — **(1) don't make the request** (browser cache), **(2) make the request travel less far** (CDN/edge), **(3) don't hit the database** (Redis), **(4) make the database faster** (index it), **(5) *then*, and only then, make your code faster.** Almost every engineer instinctively starts at #5. That's where the least time is.

---

## Common interview questions on this topic

### Q1: "What happens when you type google.com and hit Enter?"
**Hint:** The classic. Narrate all 11 steps *in order*, with rough timings, and — critically — **say where each one could fail or be optimized.** Browser cache → OS cache → recursive DNS (root → TLD → authoritative, over UDP:53) → TCP 3-way handshake → TLS handshake + cert validation → HTTP request → load balancer → reverse proxy → app server → cache/DB → response → parse/render. Bonus points: mention that DNS returns a *CDN edge* IP, that the handshakes terminate at the edge, and that browser rendering is often the *biggest* single chunk.

### Q2: "Where would you look first if a page suddenly got 2 seconds slower?"
**Hint:** Don't guess — **bisect the journey.** Use the browser waterfall (or `curl -w`) to isolate the hop: `time_namelookup` (DNS), `time_connect` (TCP), `time_appconnect` (TLS), `time_starttransfer` (TTFB = server think time), `time_total`. If TTFB jumped, it's the server/DB. If `time_connect` jumped, it's the network or the LB. If total-minus-TTFB jumped, it's response size or rendering. **The value of knowing all 11 steps is that you can binary-search them.**

### Q3: "Why do we need both a load balancer and a reverse proxy? Isn't that the same thing?"
**Hint:** They overlap heavily and are frequently the *same box* (Nginx does both). The distinction is *role*: an LB's job is **choosing a backend** (round robin, least connections, health checks, failover). A reverse proxy's job is **doing things to requests** (TLS termination, compression, caching, header rewriting, buffering slow clients so your app isn't held hostage by a 2G user). In a big architecture you'll often see an L4 LB → an L7 proxy/gateway → app servers, each layer earning its keep.

### Q4: "How does DNS know which IP to give a user in Tokyo vs one in Berlin?"
**Hint:** **GeoDNS / anycast.** The authoritative nameserver looks at the resolver's IP (or the EDNS Client Subnet extension, which passes along the *user's* subnet) and returns a different `A` record per region. Alternatively — and more commonly at scale — **anycast**: the *same* IP is announced from dozens of datacenters worldwide, and BGP routes each user to the topologically nearest one. Cloudflare and Google both do this. It's how one IP address can be "in" 300 cities.

### Q5: "Your app code takes 20 ms but users say the page takes 3 seconds. What's happening?"
**Hint:** Your code is 0.7% of the problem. Suspects, in rough order of likelihood: cold DNS/TCP/TLS to a far-away origin (~500 ms); a 3 MB JS bundle that must download *then parse* *then* make its own API calls before anything paints; render-blocking CSS/JS in `<head>`; resources spread across 5 domains, each paying a fresh DNS+TCP+TLS; unoptimized images; no CDN. Draw the latency-budget bar. **The fix is almost never in the app server.**

---

## Practice exercise

### The Waterfall Autopsy

**Part A — Measure a real cold load (10 min).**

Run this against a site you don't visit often (so caches are cold), **twice in a row**:

```bash
curl -w "
  dns_lookup:   %{time_namelookup}s
  tcp_connect:  %{time_connect}s
  tls_done:     %{time_appconnect}s
  ttfb:         %{time_starttransfer}s
  total:        %{time_total}s
  size:         %{size_download} bytes
" -o /dev/null -s https://en.wikipedia.org/wiki/Systems_design
```

These timings are **cumulative from t=0**, so subtract to get each phase:
- DNS = `time_namelookup`
- TCP = `time_connect` − `time_namelookup`
- TLS = `time_appconnect` − `time_connect`
- Server think time = `time_starttransfer` − `time_appconnect`
- Download = `time_total` − `time_starttransfer`

**Write down all five numbers for both runs.** What changed on run 2, and why?

**Part B — Do the same in the browser (10 min).**
Open DevTools → Network, tick "Disable cache", hard-reload the same page. Click the main document → **Timing** tab. Find: *Queueing, DNS Lookup, Initial connection, SSL, Waiting (TTFB), Content Download.* Map each one to a step in Diagram 1.

Then untick "Disable cache" and reload. **How many requests hit the network at all?**

**Part C — Draw and defend (15 min).**
1. On paper, redraw Diagram 1 **from memory**. Label each hop with the time *you actually measured* in Part A.
2. Now answer, in writing: **"If I could make exactly ONE change to cut this page's load time, what would it be, and how many milliseconds would it save?"**
3. Then the harder one: **"Which hops would a CDN eliminate entirely, and which would it not touch at all?"**

**Deliverable:** your annotated diagram plus 5 sentences of analysis. If your one-change answer was "optimize the backend code," go back and look at your TTFB number again.

---

## Quick reference cheat sheet

- **The 11 hops, in order:** browser cache → OS cache → DNS → TCP handshake → TLS handshake → HTTP request → load balancer → reverse proxy → app server → cache/DB → response → browser render.
- **The fastest request is the one you never make.** Browser cache beats everything. `Cache-Control: max-age` + content-hashed filenames.
- **DNS is recursive:** resolver → root (.) → TLD (.com) → authoritative NS. Runs over **UDP:53**. Cold ≈ 150 ms, cached ≈ 0–20 ms.
- **DNS TTL is your failover lever** — but clients lie about it. **Never use DNS as your only failover mechanism.**
- **TCP handshake = 1 RTT. TLS 1.3 handshake = 1 RTT.** So a cold HTTPS connection costs **~2 RTTs before your code runs.**
- **A CDN's biggest win is terminating those handshakes at an edge 10 ms away** instead of an origin 200 ms away — **even for uncacheable content.**
- **L4 LB** = routes by IP:port (fast, dumb). **L7 LB** = parses HTTP, routes by URL/header/cookie (~1–5 ms).
- **Reverse proxy jobs:** TLS termination, gzip, static files, caching, and **buffering slow clients** so your app isn't held hostage.
- **Cache-aside is the default app pattern:** check Redis (~1 ms) → miss → query Postgres (~5–20 ms) → write back to Redis with a TTL.
- **Connection pools exist** because opening a fresh Postgres connection (~5–30 ms) costs more than the query.
- **Browser render is often the single biggest chunk (50–500 ms).** CSS blocks rendering; sync JS blocks parsing. Use `defer`/`async`.
- **Every extra domain in your HTML = a fresh DNS + TCP + TLS.** Use `<link rel="preconnect">` or just use fewer domains.
- **Full cold cross-continent load ≈ 1,000 ms. Your app code is ~2% of it.** Optimize hops, not functions.
- **Optimization order:** (1) don't request → (2) request from closer → (3) don't hit the DB → (4) index the DB → (5) *finally*, your code.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [10 — Networking Fundamentals](./10-networking-fundamentals.md) — TCP, TLS, and HTTP; the protocols this journey is built from |
| **Next** | [12 — Numbers Every Engineer Should Know](./12-numbers-every-engineer-should-know.md) — the latency table that makes every timing in this doc precise |
| **Related** | [54 — DNS Deep Dive](./54-dns-deep-dive.md) — step 3, in full: recursive resolvers, record types, TTL strategy, GeoDNS |
| **Related** | [55 — Load Balancing](./55-load-balancing.md) — step 7, in full: algorithms, L4 vs L7, health checks |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — steps 1, 8 and 9: it's caches all the way down |
| **Related** | [60 — CDN](./60-cdn.md) — the single biggest lever on this entire journey |
