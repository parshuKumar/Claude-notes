# 30 — CDNs

> **Phase 5 — Topic 3 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `06-dns-in-depth.md`, `20-caching-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine you run a popular bakery in one city, but customers everywhere want your bread.

Option A: everyone flies to your one shop. People near you get bread fast. People on the other side of the world spend a whole day traveling for one loaf. Your single shop gets swamped, the line goes around the block, and some days you sell out and turn people away.

Option B: you open **small warehouses in every major city**. Each warehouse keeps copies of your most popular loaves. When a local customer wants bread:
- If the local warehouse has it on the shelf → they grab it instantly (**cache HIT**).
- If the shelf is empty → the warehouse quickly phones your main bakery, gets a fresh loaf shipped, puts it on the shelf, and hands it over (**cache MISS → origin pull**). The *next* local customer who wants that loaf gets it instantly.

That network of local warehouses is a **CDN** (Content Delivery Network). Your main bakery is the **origin**. Each warehouse is an **edge node** in a **PoP** (Point of Presence). Customers are automatically routed to their nearest warehouse, they get bread faster, and your main bakery stops drowning in orders.

The whole trick rests on one unavoidable fact: **light is not infinitely fast**. A warehouse 10 km away will always beat a bakery 10,000 km away, no matter how good your code is.

---

## Where This Lives in the Network Stack

A CDN is not a single protocol — it's an **architecture** that operates mostly at Layer 7 (HTTP) but leans hard on Layer 3 routing (Anycast/BGP) and Layer 7 name resolution (DNS) to steer you to the right box.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP caching, TLS termination, WAF)   │  ← the CDN edge serves/caches here
│  Layer 6 — Presentation  (TLS handshake happens AT the edge)    │  ← cert served by CDN, not origin
│  Layer 5 — Session                                              │
│  Layer 4 — Transport     (TCP/QUIC terminated at the edge)      │  ← edge is your TCP peer, not origin
│  Layer 3 — Network       (Anycast IP, BGP picks nearest PoP)    │  ← "how do I reach the nearest edge?"
│  Layer 2 — Data Link                                            │
│  Layer 1 — Physical      (edge PoPs sit inside IXPs worldwide)  │  ← PoPs are physically near users
└─────────────────────────────────────────────────────────────────┘
```

The critical mental shift: when you put a CDN in front of your site, **the client's TCP/TLS/HTTP connection terminates at the edge PoP, not at your origin server.** Your browser is talking to a Cloudflare/Fastly/CloudFront box a few milliseconds away. That box *may* talk to your origin behind the scenes — or it may not need to at all (on a HIT).

Routing you to the nearest edge is a Layer 3 problem (Anycast) or a Layer 7 problem (DNS geo-routing). The actual content serving is a Layer 7 caching problem, governed by the same `Cache-Control` / `ETag` rules from Topic 20.

---

## What Is This?

A **CDN** is a globally distributed network of caching servers (**edge nodes**), grouped into **Points of Presence** (**PoPs**) placed in data centers and Internet Exchange Points around the world. Its job:

1. **Serve content from close to the user** — cutting latency by shrinking physical distance (Topic 33).
2. **Offload the origin** — most requests never reach your server, saving CPU, bandwidth, and money.

Core vocabulary you must know cold:

| Term | Meaning |
|------|---------|
| **Origin** | Your actual server(s) — the source of truth for the content |
| **Edge node / edge server** | A CDN cache server that sits near users and serves content |
| **PoP (Point of Presence)** | A physical location (data center/IXP) housing many edge nodes |
| **Cache HIT** | The edge already had the content → served instantly, origin untouched |
| **Cache MISS** | The edge didn't have it → it fetches from origin (**origin pull**), caches, then serves |
| **Origin pull** | Edge fetches from origin *lazily* on the first miss (default model) |
| **Origin push** | You proactively upload content to the CDN ahead of demand |
| **TTL** | How long the edge keeps a cached object before revalidating (from `Cache-Control`) |
| **Purge / invalidation** | Forcibly evicting content from all edges before its TTL expires |
| **Origin shield** | A designated mid-tier PoP that all other PoPs pull through, protecting origin |

```
                         THE CDN WORLD MAP (simplified)

     [Edge: London]      [Edge: Frankfurt]        [Edge: Tokyo]
          ▲                    ▲                       ▲
          │ ~5ms               │ ~8ms                  │ ~4ms
     ┌────┴────┐          ┌────┴────┐             ┌────┴────┐
     │ User=UK │          │ User=DE │             │ User=JP │
     └─────────┘          └─────────┘             └─────────┘

   All three edges, on a MISS, pull from:

                    ┌──────────────────────────┐
                    │   ORIGIN (us-east-1, VA)  │  ← one server, far from most users
                    └──────────────────────────┘

   The UK user gets bytes from London (5ms), NOT from Virginia (~80ms).
   Virginia is only touched on a cache MISS — and even then, once per object per PoP.
```

Today's big CDNs (Cloudflare, Akamai, Fastly, AWS CloudFront, Google Cloud CDN, Bunny) operate hundreds of PoPs across dozens of countries.

---

## Why Does It Matter for a Backend Developer?

Even if "the frontend team owns the CDN," it lands squarely on your desk the moment something breaks. You need this to:

- **Cut real latency** — a user in Sydney fetching your JS bundle from Virginia pays ~250ms RTT *per round trip*. From a Sydney edge, ~5ms. You cannot fix this in code (Topic 01/33 — it's physics).
- **Survive traffic spikes** — a Reddit hug-of-death sends 50,000 req/s. If 98% are cache HITs served by edges, your origin sees 1,000 req/s. Without a CDN, your origin dies.
- **Slash egress cost** — cloud providers charge for bytes *leaving* your origin. Serving a 2 MB video 10 million times from origin = 20 TB of egress = a large bill. Cached at the edge, origin serves it a handful of times.
- **Not leak user data** — the single most dangerous CDN bug: an authenticated, personalized response gets cached at a *shared* edge and served to the wrong user. You must understand `Cache-Control: private` vs `public` (Topic 20) to prevent it.
- **Ship deploys correctly** — "users still see the old app after deploy" is almost always a CDN caching problem. Understanding TTLs, purging, and content-hashed filenames is how you fix it.
- **Get free infrastructure wins** — CDNs also do TLS termination, HTTP/3, DDoS absorption, and compression at the edge. Even for uncacheable APIs, terminating TCP/TLS near the user speeds up the handshake dramatically.

If you don't understand CDNs, you will eventually either serve stale content, leak private data, or pay for bandwidth you didn't need to.

---

## The Packet/Protocol Anatomy

A CDN doesn't invent a new wire protocol. It rides on the HTTP caching headers you already know (Topic 20) plus a few CDN-specific response headers that tell you what the edge did.

### Request as seen by the edge
```
GET /assets/app.9f2a1c.js HTTP/2
Host: www.myapp.com
Accept-Encoding: gzip, br            ← edge may serve a pre-compressed variant
If-None-Match: "abc123"              ← conditional request → edge can 304
Cookie: session=...                  ← edge MAY vary/ bypass cache on this (danger!)
```

### Response from the edge (the headers that matter)
```
HTTP/2 200
content-type: application/javascript
cache-control: public, max-age=31536000, immutable   ← origin's caching instructions
age: 3487                            ← seconds this object has sat in the edge cache
etag: "abc123"                       ← validator for revalidation
vary: Accept-Encoding                ← "cache a separate copy per Accept-Encoding"
x-cache: HIT                         ← generic: did the edge serve from cache?
cf-cache-status: HIT                 ← Cloudflare's version of the same signal
x-served-by: cache-lhr6san-LHR       ← Fastly: which PoP/edge served it (London here)
```

### The response headers that reveal cache state (per vendor)

| Header | Vendor | Values you'll see |
|--------|--------|-------------------|
| `x-cache` | Generic / CloudFront / Varnish | `HIT`, `MISS`, `RefreshHit` |
| `cf-cache-status` | Cloudflare | `HIT`, `MISS`, `EXPIRED`, `REVALIDATED`, `BYPASS`, `DYNAMIC` |
| `x-served-by` / `x-cache-hits` | Fastly | PoP identifiers + hit counter |
| `age` | Standard (RFC 9111) | Seconds since the object was stored/validated at the edge |
| `x-amz-cf-pop` | CloudFront | Which edge PoP handled it, e.g. `LHR62-C1` |

```
DE-ENCAPSULATION of a cached asset request (what the edge does on a HIT):

┌──────────────────────────────────────────────────────────┐
│ TCP/QUIC segment  ← terminated AT THE EDGE (near user)   │
│  ┌────────────────────────────────────────────────────┐  │
│  │ TLS record     ← edge presents YOUR cert & decrypts │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │ HTTP GET /assets/app.9f2a1c.js               │  │  │
│  │  │   1. Edge computes cache key (host+path+Vary)│  │  │
│  │  │   2. Looks up key in local cache             │  │  │
│  │  │   3. HIT → serve bytes, add age + x-cache:HIT│  │  │
│  │  │      MISS → pull from origin, store, serve   │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
      On a HIT, the origin is NEVER contacted. Zero packets leave the PoP toward origin.
```

The **cache key** is the fingerprint the edge uses to find an object. By default it's roughly `scheme + host + path + query`, expanded by any `Vary` headers. Getting the cache key wrong is the root of most CDN bugs (serving one user's data to another, or never getting a HIT).

---

## How It Works — Step by Step

### Part A — Getting routed to the nearest edge

There are two dominant strategies. Big CDNs often combine both.

**Strategy 1 — Anycast (Layer 3 / BGP)**

The *same* IP address is announced from every PoP simultaneously. The internet's own routing protocol (BGP) delivers your packets to the *topologically nearest* announcement. You don't choose — the network does.

```
CDN announces 104.18.5.10 from London, Frankfurt, Tokyo, all at once.

  User in UK  ──► packets to 104.18.5.10 ──► BGP routes to LONDON PoP  (nearest)
  User in JP  ──► packets to 104.18.5.10 ──► BGP routes to TOKYO PoP   (nearest)

  Same destination IP. Different physical box answers. That's Anycast.
```

**Strategy 2 — DNS-based geo-routing (Layer 7 / Topic 06)**

The CDN runs the authoritative DNS for your hostname. When a resolver asks "what is `www.myapp.com`?", the CDN's nameserver looks at *where the query came from* and returns the IP of the closest PoP.

```
UK user's resolver asks CDN DNS:  "A record for www.myapp.com?"
CDN DNS sees resolver is in Europe → answers: 151.101.0.10  (London edge)

JP user's resolver asks CDN DNS:  "A record for www.myapp.com?"
CDN DNS sees resolver is in Asia  → answers: 151.101.64.10 (Tokyo edge)

Same name. Different IP per region. That's DNS geo-routing.
```

**Anycast vs DNS geo-routing — the trade-off:**

| | Anycast | DNS geo-routing |
|---|---------|-----------------|
| Steering layer | Layer 3 (BGP) | Layer 7 (DNS) |
| Granularity | Network topology (not always geographic) | Geographic-ish (based on resolver location) |
| Failover speed | Fast — BGP withdraws a dead PoP in seconds | Slow — bound by DNS TTL; clients cache the old IP |
| DDoS behavior | Excellent — attack is spread across all PoPs | Weaker — one IP can be targeted |
| Accuracy problem | Rare | "Resolver location ≠ user location" (public DNS like 8.8.8.8 can misroute) → mitigated by EDNS Client Subnet |

In practice: Cloudflare and Fastly lean heavily on Anycast; classic Akamai historically leaned on DNS steering; CloudFront and Google Cloud CDN blend both. As a backend dev you rarely configure this — but when a user is "routed to the wrong region," this is the layer you're debugging.

### Part B — The HIT path (the happy path)

```
User (UK) ──HTTPS──► London Edge
                        │
                        │ 1. Terminate TLS (edge holds the cert)
                        │ 2. Compute cache key: myapp.com + /assets/app.9f2a1c.js
                        │ 3. Object present AND fresh (age < max-age)?  YES
                        │ 4. Serve from RAM/SSD.  Add: age, x-cache: HIT
                        ▼
User (UK) ◄──200 + bytes── London Edge     Total: ~5-15ms. Origin never touched.
```

### Part C — The MISS path (origin pull)

```
User (UK) ──HTTPS──► London Edge
                        │
                        │ 1. Terminate TLS, compute cache key
                        │ 2. Object NOT in cache (or expired) → MISS
                        │
                        │ 3. ORIGIN PULL: edge opens its own connection to origin
                        ▼
                   ┌──────────────────────┐
                   │ ORIGIN (us-east-1)   │
                   │  returns 200 + body  │
                   │  + Cache-Control:    │
                   │    public, max-age=  │
                   │    31536000          │
                   └──────────────────────┘
                        │
                        │ 4. Edge reads Cache-Control → stores object with TTL
                        │ 5. Serve to user.  Add: x-cache: MISS, age: 0
                        ▼
User (UK) ◄──200 + bytes── London Edge

  ⚠ This user pays the full origin RTT (~80ms+). But the NEXT UK user gets a HIT (~5ms).
    That's the "first request warms the cache" effect. Each PoP warms independently.
```

### Part D — Sequence diagram: HIT vs MISS side by side

```
        CACHE MISS (first request to a PoP)          CACHE HIT (subsequent)

User        Edge         Origin              User        Edge         Origin
 │  GET  ───►│                                │  GET  ───►│
 │           │  key lookup → MISS             │           │  key lookup → HIT
 │           │──── GET (origin pull) ───►│    │           │  (no origin call)
 │           │◄─── 200 + Cache-Control ──│    │           │
 │           │  store w/ TTL                  │           │
 │◄─ 200 ────│  x-cache: MISS, age:0         │◄─ 200 ────│  x-cache: HIT, age:3487
 │           │                                │           │
 ~80-300ms (paid once per PoP)                 ~5-15ms (paid by everyone else)
```

### Part E — Origin shield / tiered caching

If you have 200 PoPs and each pulls from origin on its own first miss, that's 200 origin pulls for one object. **Origin shield** designates one mid-tier PoP that every other PoP pulls *through*. Now the origin sees at most one pull; the other 199 PoPs pull from the shield.

```
Without shield:                        With origin shield:

  PoP1 ─┐                                 PoP1 ─┐
  PoP2 ─┼─► ORIGIN (200 pulls)            PoP2 ─┼─► SHIELD PoP ─► ORIGIN (1 pull)
  ...   │                                 ...   │
  PoP200┘                                 PoP200┘

  Cuts origin load, egress cost, and "thundering herd" on cache expiry.
```

---

## Exact Syntax Breakdown

### `curl -I` — read cache headers without downloading the body
```
curl  -I     https://www.myapp.com/assets/app.9f2a1c.js
│     │      │
│     │      └── URL to fetch
│     └── -I = HEAD request; print response headers only, no body
└── the tool
```
Look in the output for `x-cache` / `cf-cache-status` / `age`. Run it **twice** — the first may be a `MISS`, the second a `HIT`, proving caching works.

### `curl --resolve` — force a request to a specific edge IP
```
curl  -I  --resolve  www.myapp.com:443:151.101.64.10   https://www.myapp.com/asset.js
│         │          │             │   │
│         │          │             │   └── the edge IP you want to hit (e.g. Tokyo PoP)
│         │          │             └── port
│         │          └── hostname (keeps TLS/SNI + Host header correct)
│         └── override DNS for this hostname:port only
└── ...
```
This lets you test *one specific PoP* and bypass whatever your local DNS/Anycast would pick — essential for "is only the Tokyo edge stale?" debugging.

### `dig` — see what the CDN's DNS actually returns
```
dig  +short  www.myapp.com
│    │       │
│    │       └── name to resolve
│    └── terse output: just the answer records
└── DNS lookup tool
```
A CDN-fronted hostname almost always resolves to a **CNAME** pointing at the CDN, then to one or more edge **A/AAAA** records. Seeing `...cloudfront.net` or `...fastly.net` or `...cloudflare.com` in the CNAME chain confirms a CDN is in front.

### `curl -w` — measure where the time goes (from Topic 01)
```
curl -o /dev/null -s -w "TCP:%{time_connect}s TLS:%{time_appconnect}s TTFB:%{time_starttransfer}s\n" URL
```
On a CDN HIT, `TTFB` should be barely above `time_appconnect` (edge just hands you bytes). A high gap between TLS-done and first-byte means the edge is doing an origin pull (MISS).

---

## Example 1 — Basic: Prove a CDN is Caching

Let's use a real CDN-fronted site. Cloudflare fronts many sites; jQuery's CDN and Cloudflare's own assets are easy to observe.

```bash
# Ask twice. First request may populate the edge; second should HIT.
curl -sI https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js | grep -iE 'cf-cache-status|age|cache-control'
curl -sI https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js | grep -iE 'cf-cache-status|age|cache-control'
```

Annotated output:
```
# --- First call ---
cache-control: public, max-age=30672000, s-maxage=...   ← origin says: cache ~1 year
cf-cache-status: HIT                                     ← already warm (popular file)
age: 5821                                                ← been in this edge for 5821s

# --- Second call ---
cf-cache-status: HIT
age: 5822                                                ← age ticked up by ~1s: same cached copy
```

What the fields tell you:
- `cache-control: public, max-age=30672000` — the origin authorized shared caching for ~355 days. This is why fingerprinted library files use huge TTLs.
- `cf-cache-status: HIT` — Cloudflare served it from the edge; your bytes never went to origin.
- `age` — increments across calls, confirming it's the *same* cached object aging in place, not a fresh fetch.

Now watch a MISS turn into a HIT by requesting something less popular (a unique query string busts the cache key):
```bash
curl -sI "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js?bust=$RANDOM" | grep -i cf-cache-status
# Likely:  cf-cache-status: MISS      ← new cache key → edge had to fetch
curl -sI "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js?bust=SAMEVALUE" | grep -i cf-cache-status
curl -sI "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js?bust=SAMEVALUE" | grep -i cf-cache-status
# First SAMEVALUE:  MISS   ← populated
# Second SAMEVALUE: HIT    ← now cached under that exact key
```

The lesson: **the query string is part of the cache key by default.** Change it and you get a fresh cache slot (a MISS). This is both a debugging trick and a common accidental cause of low hit rates.

---

## Example 2 — Production Scenario: "Users Still See the Old App After Deploy" (and a data-leak near-miss)

**The situation.** You deploy a new version of `www.myapp.com`. QA says the new feature is live. But support tickets pour in: users still see the *old* UI. Force-refresh fixes it for some, not others. Meanwhile a second, scarier ticket lands: "I logged in and saw *someone else's* account name in the header."

Two classic CDN bugs at once. Let's diagnose both with `curl`.

### Bug 1 — Stale asset because the filename never changed

Your HTML references `/js/app.js` (a *stable* filename) and the origin set a 1-year TTL on it.

```bash
curl -sI https://www.myapp.com/js/app.js | grep -iE 'cache-control|age|etag|x-cache|cf-cache-status'
```
```
cache-control: public, max-age=31536000     ← 1 YEAR. Set months ago.
age: 2419200                                 ← 28 days old and still "fresh"
etag: "v1-old-hash"                          ← still the OLD build
cf-cache-status: HIT                         ← edge is happily serving the old file
```

**Root cause.** The filename `app.js` never changes, but you gave it a 1-year TTL. The edge has no reason to re-check the origin — from its perspective the object is fresh for another 337 days. Your new deploy sits at the origin, unreachable, because nobody is asking for it.

**Two fixes, one right and one a band-aid:**

Band-aid (do now to stop the bleeding) — **purge**:
```bash
# Cloudflare API: purge one URL from every edge
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE/purge_cache" \
  -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  --data '{"files":["https://www.myapp.com/js/app.js"]}'
```
Purge propagates across all PoPs in seconds-to-minutes (not instant — see Common Mistakes).

Real fix (so this never happens again) — **content-hashed filenames + `immutable`:**
```
Before:  <script src="/js/app.js">            TTL 1yr → can't safely cache long
After:   <script src="/js/app.9f2a1c3b.js">   TTL 1yr → SAFE

Every build emits a new hash in the filename. A new deploy → a new URL →
a guaranteed cache MISS on the new file, while old URLs keep serving old
clients mid-session. You NEVER purge; you just stop referencing the old name.
```
```
cache-control: public, max-age=31536000, immutable
                                          └── tells browsers/edges: never even revalidate
```
This is why every modern bundler (Vite, webpack, esbuild) fingerprints filenames. The HTML that references them gets a *short* TTL (or `no-cache`), the fingerprinted assets get a *forever* TTL.

### Bug 2 — Authenticated response cached in a shared edge (data leak)

The scary ticket. Check the personalized endpoint:
```bash
curl -sI https://www.myapp.com/api/me -H "Cookie: session=userA" | grep -iE 'cache-control|vary|cf-cache-status|set-cookie'
```
```
cache-control: public, max-age=60          ← ☠ PUBLIC + cacheable on a per-user endpoint
cf-cache-status: HIT                        ← ☠ a SHARED edge is serving this to everyone
vary: accept-encoding                       ← ☠ does NOT vary on Cookie/Authorization
```

**Root cause.** `/api/me` returns user-specific data but was marked `Cache-Control: public, max-age=60`. The edge cached user A's response under the key `host+/api/me` and now serves it to user B, C, D... Because `Vary` doesn't include the session, everyone shares one cache slot. **This is a data breach.**

**The fix (origin side):** private/authenticated responses must never be shared-cacheable.
```
cache-control: private, no-store        ← private = browser only; no-store = don't cache at all
# (For per-user-cacheable-in-browser: cache-control: private, max-age=0, must-revalidate)
```
`private` tells shared caches (CDN edge, corporate proxy) "do not store this — browser-local only." For anything with `Authorization` or a session cookie, the safe default is `private, no-store` unless you deliberately designed per-user edge caching keyed on the session (advanced, error-prone).

Verify the fix:
```bash
curl -sI https://www.myapp.com/api/me -H "Cookie: session=userA" | grep -iE 'cache-control|cf-cache-status'
# cache-control: private, no-store
# cf-cache-status: BYPASS      ← edge now refuses to cache it. Correct.
```

### Bonus — the egress-cost angle

Same investigation often surfaces a cost problem: a large `/downloads/report.pdf` served with `Cache-Control: no-cache` was hitting origin on *every* request → 8 TB/month egress. It's public and immutable, so:
```
cache-control: public, max-age=86400        ← let edges cache for a day
```
plus enabling **origin shield** collapsed origin pulls from ~1M/day to a few hundred. Egress dropped ~95%.

**The lesson:** the same header (`Cache-Control`) that leaks data when wrong on private endpoints *saves you money* when right on public ones. Reading `cf-cache-status`/`x-cache` + `age` with `curl -I` diagnoses all three problems.

---

## Common Mistakes

### Mistake 1: Caching private/authenticated content in a shared edge
```
Wrong:  GET /api/me → 200
        cache-control: public, max-age=60      ← shared edge caches per-user data
Result: User A's data served to User B. Data breach. (See Example 2, Bug 2.)
```
**Fix:** Any response gated by a session/`Authorization` header must be `Cache-Control: private, no-store` (or at minimum `private`). Never `public` on personalized responses. When in doubt, `no-store`.

---

### Mistake 2: Long TTL on an asset whose URL never changes
```
Wrong:  /js/app.js  with  cache-control: public, max-age=31536000
Result: You deploy. Edges keep serving the OLD app.js for up to a year.
        You can't invalidate without a manual purge on every deploy.
```
**Fix:** Fingerprint the filename (`app.9f2a1c.js`) so a new build = a new URL = automatic MISS on the new file. Give fingerprinted assets long TTL + `immutable`; give the referencing HTML a short TTL or `no-cache`. Purging becomes unnecessary.

---

### Mistake 3: Assuming a purge is instant and global
```
Wrong belief: "I purged /js/app.js, so every user worldwide now gets the new file instantly."
Reality:      Purge propagates PoP-by-PoP over seconds to a few minutes.
              A browser that already downloaded it keeps its OWN copy until ITS max-age expires.
              Purging the edge does NOT reach into users' browser caches.
```
**Fix:** Don't rely on purge timing for correctness. Use versioned URLs so old and new coexist safely. Treat purge as a best-effort cleanup, not an atomic global switch.

---

### Mistake 4: Forgetting `Vary`, so the wrong variant is served
```
Wrong:  Origin serves gzip and brotli from the same URL but omits `Vary: Accept-Encoding`.
Result: Edge caches the brotli copy, then serves those raw brotli bytes to a client
        that only sent `Accept-Encoding: gzip` → garbled/broken response.
Same class of bug: language/currency by cookie without `Vary`, or mobile vs desktop HTML.
```
**Fix:** Add `Vary: Accept-Encoding` (and any other header the response body depends on) so the edge keys a separate cache entry per variant. Keep `Vary` minimal — `Vary: User-Agent` or `Vary: Cookie` explodes the cache into near-uncacheable fragments.

---

### Mistake 5: Believing a CDN "fixes" a slow origin for dynamic, uncacheable content
```
Wrong belief: "Put it behind a CDN and it'll be fast."
Reality:      If every response is unique/uncacheable, every request is a MISS →
              origin pull → CDN adds a hop and does NOT speed up your slow endpoint.
              (It still helps with TLS/TCP termination near the user + DDoS, but the
               slow origin work still happens on every request.)
```
**Fix:** Fix the origin (caching layer, DB indexes, precomputation — Topic 20/27). Use the CDN for what it's good at: static assets, cacheable API responses (short TTL + `stale-while-revalidate`), and edge termination. Don't expect it to mask a slow DB query.

---

### Mistake 6: Not setting `Cache-Control`, letting the CDN guess
```
Wrong:  Origin returns assets with NO Cache-Control header.
Result: The CDN applies its OWN default TTL (varies by vendor/config — could be
        "don't cache," could be "cache for 2 hours"). Behavior is unpredictable and
        changes if you switch CDNs. You've handed correctness to a default you didn't set.
```
**Fix:** Always set explicit `Cache-Control` at the origin. Be intentional: `public, max-age=…, immutable` for fingerprinted assets; `private, no-store` for personalized; `public, max-age=60, stale-while-revalidate=300` for cacheable-but-changing HTML/JSON.

---

## Hands-On Proof

```bash
# 1. Confirm a CDN sits in front (look for cloudfront/fastly/cloudflare in the CNAME chain)
dig +short www.cloudflare.com
dig CNAME www.github.com +short          # GitHub Pages / Fastly, etc.

# 2. Read the cache status headers (run twice — watch MISS become HIT / age climb)
curl -sI https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js \
  | grep -iE 'cf-cache-status|age|cache-control|etag|vary'

# 3. Force a fresh cache key with a random query → observe a MISS
curl -sI "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js?x=$RANDOM" \
  | grep -i cf-cache-status

# 4. Hit a SPECIFIC edge IP directly, bypassing your local routing
EDGE=$(dig +short cdnjs.cloudflare.com | head -1)
curl -sI --resolve cdnjs.cloudflare.com:443:$EDGE \
  https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js | grep -i cf-cache-status

# 5. Compare edge vs origin latency (CDN-fronted asset should be low + stable)
curl -o /dev/null -s -w "TCP:%{time_connect}s TLS:%{time_appconnect}s TTFB:%{time_starttransfer}s\n" \
  https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js

# 6. See which PoP served you (vendor-dependent header)
curl -sI https://www.fastly.com/ | grep -iE 'x-served-by|x-cache|age'

# 7. Prove a personalized endpoint is NOT shared-cached (should be BYPASS/DYNAMIC/no-store)
curl -sI https://github.com/settings/profile | grep -iE 'cache-control|cf-cache-status|x-cache'
```

Watch for: `cf-cache-status` flipping MISS→HIT on repeat, `age` incrementing on repeats, and a CNAME chain that ends in a CDN's domain.

---

## Practice Exercises

### Exercise 1 — Easy: Spot the HIT and read the age
```bash
# Request the same cached asset three times, 2s apart. Record cf-cache-status and age each time.
for i in 1 2 3; do
  curl -sI https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js \
    | grep -iE 'cf-cache-status|^age'
  sleep 2
done

# Answer:
# 1. Is the status HIT or MISS? What does that tell you about the origin's involvement?
# 2. By roughly how much does `age` grow between calls? Why?
# 3. What single URL change would force a MISS on the next request?
```

### Exercise 2 — Medium: Prove versioned filenames beat purging
```bash
# Using any CDN-fronted asset with a query string as a stand-in for a "version":
# a) Request ?v=1 twice — note MISS then HIT (v=1 is now cached).
curl -sI "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js?v=1" | grep -i cf-cache-status
curl -sI "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js?v=1" | grep -i cf-cache-status
# b) Now request ?v=2 — a "new deploy."
curl -sI "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js?v=2" | grep -i cf-cache-status

# Answer:
# 1. Did ?v=2 come back as MISS or HIT? Why did you NOT need to purge anything?
# 2. Explain how this maps to shipping app.<hash>.js on every build.
# 3. Which cache-control directive would let you also skip browser revalidation of v=1 forever?
```

### Exercise 3 — Hard (Production Simulation): Design the caching policy for a full site
```
You run www.shop.com behind a CDN. Write the exact Cache-Control (and Vary if needed)
for EACH of these responses, and say what cf-cache-status you'd expect:

  a) /assets/app.4f9a2c.js        (fingerprinted JS bundle)
  b) /index.html                  (references the fingerprinted assets; changes each deploy)
  c) /api/products                (public catalog, changes a few times per hour)
  d) /api/cart                    (per-user, requires session cookie)
  e) /api/me                      (per-user profile, requires Authorization header)
  f) /images/logo.png             (public, essentially never changes, stable URL)

# For each: is it safe in a SHARED edge cache? public or private? What TTL?
# Then answer:
# 1. Which of these, if mis-marked `public`, would cause a data breach?
# 2. Which benefits most from stale-while-revalidate, and why?
# 3. Your origin egress bill is high. Which TWO responses are the biggest levers, and how?
# 4. You must ship a hotfix to (b) immediately. What TTL on /index.html makes that safe
#    WITHOUT a purge, and why can't you give it max-age=31536000?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **A user in Tokyo requests your JS bundle. The Tokyo edge returns `x-cache: MISS`, then a second Tokyo user gets `HIT`. Meanwhile a London user's first request is also a `MISS`. Explain why the London user didn't benefit from the Tokyo user warming the cache.**

2. **Anycast and DNS geo-routing both steer users to a nearby edge. Which one fails over faster when a PoP dies, and why? Which is bound by TTL?**

3. **You see `cache-control: public, max-age=31536000` on `/js/app.js` (a stable filename). What deployment problem is guaranteed, and what's the fix that avoids ever purging?**

4. **An endpoint returns per-user data but ships `Cache-Control: public, max-age=60` with no `Vary` on the session. Walk through exactly how user A's data ends up on user B's screen.**

5. **You purge `/logo.png` from the CDN. A user who loaded the page 30 seconds ago still sees the old logo on refresh. Why didn't the purge reach them?**

6. **Your API is 100% dynamic, per-user, and uncacheable. Your boss says "put it behind a CDN to make it fast." Is that wrong? What (if anything) does the CDN still buy you here?**

7. **What is `age` in a cache response, what is `Vary` for, and what does `cf-cache-status: BYPASS` (or `DYNAMIC`) tell you about how the edge treated the request?**

---

## Quick Reference Card

| Concept | What it is | Key point to remember |
|---------|-----------|----------------------|
| Edge / PoP | Cache server near the user | Terminates TCP/TLS/HTTP; your real peer, not origin |
| Origin | Your actual server | Touched only on a MISS (once per PoP per object) |
| Cache HIT | Served from edge | Fast (~5-15ms), origin untouched |
| Cache MISS | Not in edge → origin pull | First request per PoP pays full origin RTT |
| Origin pull | Lazy fetch on first miss | Default model; no upload needed |
| Origin push | You upload to the CDN | Rare; for large/predictable content |
| Origin shield | Mid-tier PoP all others pull through | Collapses N origin pulls into ~1 |
| Anycast | Same IP from many PoPs, BGP picks | Layer 3; fast failover; great vs DDoS |
| DNS geo-routing | CDN DNS returns nearest edge IP | Layer 7; bound by DNS TTL on failover |
| Cache key | host + path + query (+ Vary) | Get it wrong → wrong data or 0% hit rate |
| Purge | Force-evict before TTL | NOT instant, NOT global, doesn't touch browsers |
| Fingerprinted URL | `app.<hash>.js` + `immutable` | Beats purging: new build = new URL = auto-MISS |

**Cache-status headers cheat block:**
```
x-cache / cf-cache-status / x-served-by   → did the edge serve from cache?
  HIT         served from edge cache (origin untouched)
  MISS        not cached → edge pulled from origin, then cached
  EXPIRED     was cached but stale → revalidated with origin
  REVALIDATED origin returned 304, edge reused stored body
  BYPASS      edge was told NOT to cache (e.g. private/no-store)
  DYNAMIC     content not eligible for caching by config
age: N        seconds this object has aged in the edge cache
```

**Cache-Control decision cheat block:**
```
Fingerprinted asset (app.9f2a1c.js):  public, max-age=31536000, immutable
Referencing HTML (index.html):        no-cache   (or public, max-age=60)
Public cacheable API (/products):     public, max-age=60, stale-while-revalidate=300
Per-user / authenticated response:    private, no-store        ← NEVER public
Large public static (video, pdf):     public, max-age=86400    + enable origin shield
```

**Key commands:**
```bash
curl -I URL                                   # read cache headers (run twice: MISS→HIT)
curl --resolve host:443:EDGE_IP URL           # hit one specific edge/PoP
dig +short host                               # see CDN CNAME + edge IPs
curl -o /dev/null -s -w "%{time_starttransfer}s\n" URL   # TTFB (HIT should be tiny)
# purge (Cloudflare example):
curl -X POST .../zones/$ZONE/purge_cache -H "Authorization: Bearer $T" \
  --data '{"files":["https://host/path"]}'
```

---

## When Would I Use This at Work?

### Scenario 1: "Latency is fine in the US but terrible in Australia"
You put static assets and cacheable responses behind a CDN. AU users now fetch from a Sydney PoP (~5ms) instead of your `us-east-1` origin (~250ms RTT). For the uncacheable API calls, the CDN still terminates TLS/TCP near the user, so even those get a faster handshake. You verify with `curl --resolve` against a Sydney edge and compare `time_starttransfer`.

### Scenario 2: "We deployed but users see the old site"
You recognize this instantly as a caching problem. `curl -I` shows `cf-cache-status: HIT`, a big `age`, and a stale `etag` on a stable filename with a long `max-age`. Short-term you purge; long-term you switch to content-hashed filenames + `immutable` on assets and `no-cache` on the HTML so this never recurs.

### Scenario 3: "Security found that user A saw user B's data"
You check the personalized endpoint with `curl -I`, see `public` + `HIT` + no `Vary` on the session, and immediately know a shared edge is fanning out one user's response. You ship `Cache-Control: private, no-store` on all authenticated responses and confirm `cf-cache-status: BYPASS`. Then you audit every endpoint that reads a session cookie or `Authorization` header.

### Scenario 4: "Our cloud egress bill tripled"
You find large public objects served with `no-cache`, forcing an origin pull on every request. You make them `public, max-age=…` so edges absorb the traffic, and enable **origin shield** so 200 PoPs stop each hammering the origin independently. Origin egress drops dramatically; the CDN eats the fan-out.

### Scenario 5: "We're getting DDoSed / traffic-spiked"
A viral post drives 50k req/s. Because your assets are cacheable and served from edges via Anycast, the flood spreads across every PoP and mostly hits cache, not origin. Your origin sees a trickle. This is the CDN acting as a shock absorber — the same property that makes Anycast-based CDNs strong against volumetric DDoS.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the physics of distance/latency that make edges necessary
- `06-dns-in-depth.md` — DNS geo-routing, CNAME chains, and TTL that steer users to a PoP
- `20-caching-in-depth.md` — every `Cache-Control` directive, `ETag`/`Vary`, and `stale-while-revalidate` the edge obeys

**Study alongside / after:**
- `12-http3-and-quic.md` — CDNs terminate HTTP/3 (QUIC) at the edge for faster, migration-friendly connections
- `33-network-latency.md` — the speed-of-light budget that a CDN spends wisely by shortening distance
- `28-load-balancers.md` / `29-reverse-proxies.md` — an edge node is itself a caching reverse proxy + LB; same header mechanics (`X-Forwarded-For`, etc.)
- `32-firewalls-and-security-groups.md` — locking your origin so it only accepts traffic from the CDN (so attackers can't bypass the edge)

**The one-sentence takeaway:** A CDN moves your content physically closer to users (Anycast/DNS routing) and serves most requests from edge caches governed by `Cache-Control` — cutting latency and origin load — but it is only as safe and effective as the caching headers *you* set at the origin.

---

*Doc saved: `/docs/networking/30-cdns.md`*
