# 20 — Caching in Depth

> **Phase 3 — Topic 7 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `14-http-headers-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine you keep a bottle of milk in your fridge, and the carton says **"best before 12 July."**

- **Today is 10 July** → the milk is *fresh*. You drink it without a second thought. You don't drive to the shop. (This is a **cache hit** — instant, no network.)
- **Today is 13 July** → the milk is *stale*. But instead of throwing it out, you phone the shop and ask: *"Is my carton still fine, or did you change the recipe?"* The shop says **"same milk, still fine"** — you keep drinking it, no new carton needed. (This is **revalidation** — a cheap check that saves you a full trip: HTTP `304 Not Modified`.)
- **The shop says "we changed it, here's a new carton"** → you get a fresh carton. (This is a **full miss** — HTTP `200` with a whole new body.)

Caching on the web is exactly this: **keep a copy, remember when it expires, and when it expires, ask a cheap yes/no question before fetching the whole thing again.** The whole game is deciding *how long* the copy stays fresh, and *who* is allowed to keep a copy.

---

## Where This Lives in the Network Stack

Caching is a **Layer 7 (Application / HTTP)** behavior. It is not a protocol of its own — it is a set of *rules encoded in HTTP headers* (`Cache-Control`, `ETag`, `Age`, `Vary`, `Last-Modified`) that every hop in the request path agrees to obey.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP)  ← Cache-Control / ETag / Vary   │  ← CACHING LIVES HERE
│  Layer 6 — Presentation  (TLS)                                  │
│  Layer 4 — Transport     (TCP)                                  │
│  Layer 3 — Network       (IP)                                   │
└─────────────────────────────────────────────────────────────────┘
```

But the *place* a cache sits is what matters. There are **five layers of caching** between a user's eyeball and your database, each owned by a different party:

```
   USER                                                    YOU
    │                                                        │
    ▼                                                        ▼
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐  ┌────────┐
│ 1. BROWSER │→ │ 2. CDN /   │→ │ 3. REVERSE │→ │ 4. APP    │→ │ 5.     │
│    CACHE   │  │    EDGE    │  │    PROXY   │  │    CACHE  │  │ ORIGIN │
│ (private,  │  │  (shared,  │  │ (shared,   │  │ (Redis,   │  │ (DB /  │
│  per-user) │  │  Fastly,   │  │  nginx,    │  │  Memcache,│  │  app)  │
│            │  │ CloudFront)│  │  Varnish)  │  │  in-proc) │  │        │
└────────────┘  └────────────┘  └────────────┘  └───────────┘  └────────┘
  on disk in     hundreds of      1 per region    your code       source
  the user's     edge PoPs        / data center   controls it     of truth
  laptop
  ─────────────  ──────────────   ─────────────   ────────────    ───────
  you control    you control      you control     you control 100%
  via headers    via headers +    fully           (it's your code)
  only           purge API
```

> **This topic goes deeper than the caching intro in Topic 14.** Topic 14 introduced `Cache-Control`, `ETag`, and `Last-Modified` as *headers*. Here we cover the full **freshness lifecycle**, **every directive**, the **304 revalidation round-trip**, **`stale-while-revalidate`**, **cache keys / `Vary`**, and **invalidation** — the operational reality of running caches in production.

---

## What Is This?

**HTTP caching** is the mechanism by which a stored copy of a response is reused to satisfy future requests *without re-executing the work that produced it*. The unit of caching is a **response** (a body + its headers), keyed by (roughly) the request method + URL.

Two fundamental concepts drive everything:

1. **Freshness** — a cached response has a **freshness lifetime**. While inside that lifetime, it is **fresh** and can be served instantly with *zero network contact* with the origin. Once past it, it is **stale**.

2. **Validation** — a stale response is not necessarily wrong. Before discarding it, a cache can **revalidate** it with the origin using a **validator** (an `ETag` or `Last-Modified` date). If nothing changed, the origin replies `304 Not Modified` — a tiny header-only response — and the cache resets the freshness clock and reuses the stored body. This is the single most important bandwidth-saving mechanism on the web.

There are two kinds of caches, and the distinction is critical:

| Cache type | Who it serves | Examples | May store `private` responses? |
|------------|---------------|----------|-------------------------------|
| **Private cache** | One single user | Browser disk cache | Yes |
| **Shared cache** | Many users | CDN, reverse proxy, corporate proxy | **No** |

A shared cache serving one user's personalized response to another user is a **data leak**. This distinction is the reason `private` and `public` exist.

---

## Why Does It Matter for a Backend Developer?

Caching is the highest-leverage performance tool you have. Get it right and you:

- **Cut origin traffic by 90%+.** A CDN with a 95% hit rate means your origin sees 1 in 20 requests. This is the difference between 2 servers and 40.
- **Slash latency.** A cache hit at an edge PoP 10ms away beats a full origin round-trip of 300ms — with *zero* code change.
- **Survive outages.** `stale-if-error` lets a CDN keep serving your homepage while your origin is on fire.
- **Save real money.** Egress bandwidth and compute are billed. Cache hits are nearly free.

Get it **wrong** and you get the two nightmares of caching:

- **Stale nightmare:** you deploy a fix, but users keep seeing the old broken version for hours because you set `max-age=31536000` on a URL that never changes. You *cannot* take it back — the copy lives in a million browsers.
- **Leak nightmare:** a shared cache stores `{"balance": "$4,201", "user": "alice"}` and serves it to Bob because you forgot `Cache-Control: private`.

Both are caused by not understanding the directives below. That is why this topic exists.

---

## The Packet/Protocol Anatomy

Caching is negotiated entirely through **HTTP headers**. Here is a real annotated exchange. First, the **origin response** that establishes the cache policy and the validators:

```
HTTP/1.1 200 OK
Date: Sat, 12 Jul 2026 10:00:00 GMT      ← when the origin generated this
Content-Type: application/json
Cache-Control: public, max-age=300        ← FRESHNESS: any cache, fresh for 300s
ETag: "a1b2c3d4"                           ← VALIDATOR (strong): opaque version id
Last-Modified: Sat, 12 Jul 2026 09:55:00 GMT  ← VALIDATOR (fallback): mtime
Vary: Accept-Encoding                      ← CACHE KEY: also key on encoding
Age: 0                                      ← seconds this has spent in caches
Content-Length: 342

{"users":[{"id":1,"name":"Alice"}]}        ← the body (the expensive part)
```

Every field above is a lever. `Cache-Control` sets freshness and *who* may store. `ETag`/`Last-Modified` enable the cheap `304` check. `Vary` controls the cache key. `Age` reports staleness.

Now the **conditional request** a cache sends to revalidate a stale copy — it echoes the validators back:

```
GET /users HTTP/1.1
Host: api.myapp.com
If-None-Match: "a1b2c3d4"                   ← "only send body if ETag changed"
If-Modified-Since: Sat, 12 Jul 2026 09:55:00 GMT  ← fallback condition
```

And the two possible replies. **Nothing changed** — the money-saver:

```
HTTP/1.1 304 Not Modified                   ← NO BODY. Just headers. ~200 bytes.
Date: Sat, 12 Jul 2026 10:06:00 GMT
ETag: "a1b2c3d4"                             ← same version
Cache-Control: public, max-age=300           ← freshness clock RESETS
```

**Something changed** — a full replacement:

```
HTTP/1.1 200 OK                              ← full new body
Cache-Control: public, max-age=300
ETag: "z9y8x7w6"                             ← NEW version id
Content-Length: 361

{"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]}
```

---

## How It Works — Step by Step

The full decision a cache makes on every request. Here is the **freshness → validation → fetch** funnel:

```
                    ┌─────────────────────────────────┐
   Request  ───────►│ Do I have a stored response for  │
                    │ this cache key? (method + URL +  │
                    │ Vary headers)                    │
                    └───────────────┬─────────────────┘
                          NO │      │ YES
              ┌──────────────┘      ▼
              │            ┌──────────────────────────┐
              │            │ Is it still FRESH?        │
              │            │ (Age < freshness lifetime)│
              │            └──────┬─────────────┬──────┘
              │              YES  │             │ NO (stale)
              │                   ▼             ▼
              │           ┌──────────┐   ┌─────────────────────┐
              │           │ SERVE    │   │ Revalidate: send a   │
              │           │ from     │   │ conditional GET with │
              │           │ cache    │   │ If-None-Match /      │
              │           │ (HIT,    │   │ If-Modified-Since    │
              │           │ 0 network)│  └─────────┬───────────┘
              │           └──────────┘             │
              ▼                          ┌─────────┴──────────┐
      ┌──────────────┐              304 │                    │ 200
      │ MISS: fetch  │                  ▼                    ▼
      │ full 200 from│          ┌──────────────┐    ┌──────────────┐
      │ origin,      │          │ Reuse stored │    │ Replace stored│
      │ store it     │          │ body, reset  │    │ body with new │
      └──────────────┘          │ freshness    │    │ 200, store    │
                                └──────────────┘    └──────────────┘
```

### Flow (a) — Fresh cache HIT (the goal: no network at all)

```
Browser                 (nothing else is contacted)
   │
   │  need /logo.png
   │  Check disk cache → found, Age=42s, max-age=86400 → 42 < 86400 → FRESH
   │
   ▼
[SERVE from disk]   ← 0 RTT. No packet leaves the machine. ~0ms.
```

In Chrome DevTools this shows as **"(disk cache)"** or **"(memory cache)"** with `Size` = *(from cache)* and near-0 time.

### Flow (b) — Conditional revalidation → 304 (stale, but unchanged)

```
Browser                          Origin (or CDN)
   │                                   │
   │  need /users, cached copy STALE   │
   │  (Age=350s > max-age=300)         │
   │                                   │
   │  GET /users                       │
   │  If-None-Match: "a1b2c3d4"  ──────►│  compares ETag: unchanged
   │                                   │
   │◄──── 304 Not Modified ────────────│  header-only, ~200 bytes
   │      (no body!)                   │
   │                                   │
   ▼                                   
[SERVE stored body]  ← 1 RTT, but body (maybe 500KB) NOT re-sent.
                       Freshness clock reset for another 300s.
```

**Cost: one round-trip and ~200 bytes**, instead of re-downloading the whole body. This is the win.

### Flow (c) — Full MISS → 200 (not cached, or changed)

```
Browser                          Origin
   │                                   │
   │  GET /users  (no cached copy,     │
   │              or ETag changed) ────►│  runs handler, DB query,
   │                                   │  serialization...
   │                                   │
   │◄──── 200 OK + full body ──────────│  entire 500KB payload
   │      Cache-Control: max-age=300   │
   │      ETag: "z9y8x7w6"             │
   │                                   │
   ▼
[STORE + SERVE]   ← 1 RTT + full transfer + server work. Slowest path.
```

### How freshness lifetime is actually computed

A cache determines the freshness lifetime in this priority order:

```
1. Cache-Control: s-maxage=N   → N seconds  (SHARED caches only; wins for CDN/proxy)
2. Cache-Control: max-age=N    → N seconds
3. Expires: <http-date>        → Expires minus Date header
4. (none of the above)         → HEURISTIC: often 10% of (Date − Last-Modified)
```

Then: **`response is fresh  ⟺  Age < freshness_lifetime`.** The `Age` header is how many seconds the response has already lived in caches; a CDN increments it as the object sits in its store.

---

## Exact Syntax Breakdown

### Reading cache headers with `curl -I`

```
curl  -I            https://api.myapp.com/users
│     │             │
│     │             └── URL to fetch
│     └── -I = HEAD request: fetch headers ONLY, no body
└── transfer tool
```
Look at the response for `Cache-Control`, `ETag`, `Age`, `Vary`, `X-Cache`.

### Forcing a 304 with a conditional request

```
curl  -I  -H 'If-None-Match: "a1b2c3d4"'   https://api.myapp.com/users
      │   │              │
      │   │              └── the ETag value you got from a previous request
      │   └── -H adds a request header
      └── HEAD (headers only)

# If the resource is unchanged  → HTTP/1.1 304 Not Modified
# If it changed                 → HTTP/1.1 200 OK (with a new ETag)
```

### The key headers you read, and what each means

```
Cache-Control: public, max-age=300, s-maxage=600   ← the policy (see full table below)
ETag: "a1b2c3d4"                                    ← strong validator (byte-exact)
ETag: W/"a1b2c3d4"                                  ← WEAK validator (W/ prefix = "semantically equal")
Age: 42                                             ← seconds spent in shared caches
Vary: Accept-Encoding, Origin                       ← extra dimensions of the cache key
X-Cache: HIT                                         ← NON-STANDARD, CDN-specific: served from edge
X-Cache: MISS                                        ← went to origin
```

`X-Cache` is **not** a standard header — different CDNs spell it differently (`X-Cache: Hit from cloudfront`, Fastly's `X-Cache: HIT`, Varnish's `X-Cache: HIT`). But `Age` *is* standard: **`Age > 0` almost always means a shared cache served you.**

---

## Example 1 — Basic

Fetch a real asset and read its caching story off the wire.

```bash
# 1. Read the cache policy of a static asset on a CDN
curl -sI https://cdn.jsdelivr.net/npm/react@18/package.json | grep -iE 'cache-control|etag|age|x-cache|vary'
```

Annotated output:
```
cache-control: public, max-age=604800, s-maxage=86400, immutable
                └─ any cache   └─ 7d in browser  └─ 1d at CDN   └─ never revalidate
etag: W/"a1b2c3..."          ← weak validator
age: 3517                    ← this object has lived 3517s (~1h) in the CDN edge
vary: Accept-Encoding        ← gzip and brotli copies are cached separately
x-cache: HIT                 ← this came from the edge, not the origin
```

```bash
# 2. Now grab the ETag and prove a 304 round-trip
ETAG=$(curl -sI https://cdn.jsdelivr.net/npm/react@18/package.json | grep -i etag | cut -d' ' -f2 | tr -d '\r')

curl -sI -H "If-None-Match: $ETAG" https://cdn.jsdelivr.net/npm/react@18/package.json | head -1
```
Expected:
```
HTTP/2 304          ← the server agreed nothing changed. No body sent. Bandwidth saved.
```

You just watched the exact mechanism that keeps the web fast: a `304` instead of re-downloading the file.

---

## Example 2 — Production Scenario

### "We shipped a fix an hour ago, but users are still running the OLD JavaScript"

**The situation.** You deploy a critical bug fix. Your build outputs `app.js`, served by your reverse proxy with:

```
Cache-Control: public, max-age=31536000    ← 1 YEAR. Meant to make assets fast.
```

Support is flooded. Half of users still hit the bug. The fix is live on the server — `curl` from your laptop shows the new code. But real users' browsers show the old code, and there is **no way to reach into their browser cache and delete it.** The response said "this is good for a year," and the browser believed you.

**Why this happens — the diagram:**

```
DEPLOY: server now has NEW app.js
   │
   ▼
User's browser:  "app.js? I have a copy, Age=3600s, max-age=31536000.
                  3600 < 31536000 → FRESH. Serve from disk."
   │
   ▼
[OLD app.js served]   ← the browser NEVER asks the server. It won't for a year.
```

The URL `app.js` never changed, so the browser has no reason to re-fetch. **A long `max-age` on a stable URL is a time-bomb you cannot defuse.**

**Diagnose it:**
```bash
# Confirm the origin has the fix but is telling clients to cache for a year
curl -sI https://myapp.com/app.js | grep -iE 'cache-control|etag'
# cache-control: public, max-age=31536000    ← the culprit
```

**The correct fix: content-hash filenames (cache busting) + immutable.**

The rule is: **the URL must change whenever the content changes.** Build tools (Webpack, Vite, esbuild) do this automatically — they emit a **fingerprinted** filename containing a hash of the file's contents:

```
Before:   app.js
After:    app.a1b2c3d4.js      ← hash changes when content changes
```

Now cache configuration splits cleanly into two tiers:

```nginx
# 1. Fingerprinted assets: cache FOREVER, never revalidate.
#    Safe, because a new build produces app.e5f6a7b8.js — a brand-new URL.
location ~ \.[0-9a-f]{8}\.(js|css)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}

# 2. The HTML that REFERENCES those assets: never trust the cache blindly.
#    no-cache = "store it, but revalidate every time before using it."
location = /index.html {
    add_header Cache-Control "no-cache";
    # optionally: "no-store" if the HTML must never be cached at all
}
```

**Why this works:**
```
New deploy → app.e5f6a7b8.js (new URL) + index.html points to it
   │
   ▼
Browser requests index.html → no-cache → revalidates → gets NEW html (200, cheap)
   │  new html references app.e5f6a7b8.js
   ▼
Browser has never seen app.e5f6a7b8.js → cache MISS → fetches NEW bundle
   │
   ▼
[NEW code runs immediately]   ← old app.a1b2c3d4.js just ages out harmlessly, unused
```

`immutable` is the finishing touch: it tells the browser *"do not even send a revalidation request on reload — this URL's bytes will never change."* Without it, a user hitting Reload would fire a wasteful `If-None-Match` for every fingerprinted asset.

### Variant: a CDN leaked a personalized response

Same class of bug, opposite symptom. Your `/account` endpoint returned:

```
HTTP/1.1 200 OK
Cache-Control: max-age=60          ← no "private"! and it's user-specific
Set-Cookie: session=...
{"user":"alice","balance":"$4,201"}
```

The CDN (a **shared** cache) stored Alice's response and served it to Bob 30 seconds later. **Root cause: a personalized response with no `private` and no `no-store`, cached by a shared cache.** Fix:

```
Cache-Control: private, no-store    ← never store user data in a shared cache
Vary: Cookie                        ← belt-and-suspenders: key on the session cookie
```

---

## Common Mistakes

### Mistake 1: Confusing `no-cache` with `no-store` (the classic)

This is the single most misunderstood pair in HTTP.

```
Cache-Control: no-cache
   → "You MAY store this. But you MUST revalidate with the origin
      (send If-None-Match) before serving it. If the origin says 304,
      serving the cached copy is fine."
   → It is NOT "don't cache." It means "always check first."
   → Perfect for HTML: cached, but never stale.

Cache-Control: no-store
   → "Do NOT write this to disk or memory. AT ALL. Fetch fresh every time."
   → Truly no caching. For sensitive data: bank balances, auth tokens, PII.
```

Mnemonic: **no-cache = "check before use"**, **no-store = "never keep it."** If you meant "don't cache my secret data" and wrote `no-cache`, you just cached the secret. Use `no-store`.

---

### Mistake 2: Caching a private/personalized response in a shared cache

```
# WRONG — a CDN will happily serve Alice's cart to Bob
Cache-Control: max-age=300
{"cart": [...], "user": "alice"}

# RIGHT — private keeps it in the browser only; shared caches must skip it
Cache-Control: private, max-age=300
# or for truly sensitive data:
Cache-Control: no-store
```
Rule: **any response that varies per user must be `private` (or `no-store`), never bare `max-age` on a shared path.**

---

### Mistake 3: A long `max-age` on a URL that never changes

```
# WRONG — you can never take this back for up to a year
GET /app.js  →  Cache-Control: max-age=31536000
```
If the URL is stable, a long TTL means **you cannot invalidate it.** Either (a) use fingerprinted URLs (`app.a1b2c3.js`) so a new deploy = a new URL, then `max-age=31536000, immutable` is safe; or (b) use a short TTL / `no-cache` on stable URLs. Never a long TTL on a stable URL you might need to change.

---

### Mistake 4: Forgetting `Vary` — serving the wrong representation

```
# Server returns gzip to a client that sent Accept-Encoding: gzip
# CDN caches it WITHOUT Vary: Accept-Encoding...
# ...then serves the gzip bytes to a client that DIDN'T ask for gzip → broken.

# FIX: tell caches what the response depends on
Vary: Accept-Encoding
```
Common `Vary` values: `Accept-Encoding` (compression), `Origin` (CORS), `Accept` (content negotiation), `Accept-Language` (i18n). **Missing `Vary` = the cache keys only on the URL and mixes up representations.** (But beware `Vary: *` or `Vary: User-Agent` — they explode the cache into near-uniqueness and destroy your hit rate.)

---

### Mistake 5: Not using ETags / 304 — re-sending unchanged bodies forever

```
# WRONG — no validator. When max-age expires, the cache must re-download
# the ENTIRE body even if it's identical to what it already had.
Cache-Control: max-age=60
(no ETag, no Last-Modified)

# RIGHT — add a validator so expiry triggers a cheap 304 instead of a full 200
Cache-Control: max-age=60
ETag: "a1b2c3d4"
```
Without a validator, every expiry is a full re-transfer. With one, most expiries are a 200-byte `304`. On a 2MB image refreshed hourly, that is the difference between 2MB and 0.2KB per revalidation.

---

### Mistake 6: Assuming a purge is instant across all edges

```
# You call the CDN purge API and immediately test → still stale. Panic.
# Reality: a global CDN has hundreds of PoPs. A purge PROPAGATES —
# usually seconds, sometimes tens of seconds, and not perfectly atomic.
```
A **hard purge** evicts the object (next request is a full MISS to origin). A **soft purge** marks it stale (next request revalidates, and with `stale-while-revalidate` serves stale meanwhile — no origin thundering-herd). Never assume "purge → done everywhere in the same instant." Design for eventual consistency, and prefer soft purge for hot objects.

---

## Hands-On Proof

```bash
# 1. See a full cache policy + validators on a real CDN-served file
curl -sI https://cdn.jsdelivr.net/npm/vue@3/dist/vue.global.js \
  | grep -iE 'cache-control|etag|age|vary|x-cache'

# 2. Prove the 304 round-trip yourself (grab ETag, send it back)
URL=https://cdn.jsdelivr.net/npm/vue@3/dist/vue.global.js
ETAG=$(curl -sI "$URL" | awk -F': ' 'tolower($1)=="etag"{print $2}' | tr -d '\r')
curl -sI -H "If-None-Match: $ETAG" "$URL" | head -1     # → HTTP/2 304

# 3. Prove Last-Modified validation (the other validator)
LM=$(curl -sI "$URL" | awk -F': ' 'tolower($1)=="last-modified"{print $2}' | tr -d '\r')
curl -sI -H "If-Modified-Since: $LM" "$URL" | head -1    # → 304 Not Modified

# 4. Watch Age climb (run twice, a few seconds apart) — proof a shared cache is holding it
curl -sI "$URL" | grep -i '^age'; sleep 5; curl -sI "$URL" | grep -i '^age'

# 5. Compare HIT vs MISS: hit an obscure query string to force a MISS, then repeat for a HIT
curl -sI "$URL?bust=$RANDOM" | grep -iE 'x-cache|age'    # likely MISS, age: 0
curl -sI "$URL"              | grep -iE 'x-cache|age'    # likely HIT, age: >0

# 6. See how a browser caches: open DevTools → Network → reload.
#    Size column "(disk cache)" / "(memory cache)" = a fresh HIT, 0 bytes over the wire.
#    Status 304 = a revalidation. Status 200 = a full miss.
```

---

## Practice Exercises

### Exercise 1 — Easy: Read a cache policy off the wire
```bash
# Pick any CDN-hosted asset and inspect its headers:
curl -sI https://cdn.jsdelivr.net/npm/lodash@4/lodash.min.js \
  | grep -iE 'cache-control|etag|age|vary|x-cache'

# Answer from the output:
# 1. What is the max-age (in browser) and s-maxage (at the CDN)? Which applies where?
# 2. Is this response cacheable by a SHARED cache? (Is it public? private? immutable?)
# 3. Does Age > 0? What does that tell you about where this response came from?
# 4. What request header must a cache key on, according to Vary?
```

### Exercise 2 — Medium: Trigger and confirm a 304
```bash
URL=https://cdn.jsdelivr.net/npm/axios@1/dist/axios.min.js

# Step 1: capture the ETag
curl -sI "$URL" | grep -i etag

# Step 2: send it back with If-None-Match and confirm you get a 304 (no body)
# (fill in the ETag value from step 1)
curl -si -H 'If-None-Match: PASTE_ETAG_HERE' "$URL" | head -5

# Answer:
# 1. What status code did you get? Was there a body?
# 2. Change one character of the ETag and re-run. What status now, and why?
# 3. Roughly how many bytes did the 304 save vs a full 200 for this ~30KB file?
```

### Exercise 3 — Hard (Production Simulation): Design the cache policy for a full app
```
You are deploying a single-page app. Write the exact Cache-Control (and any
Vary / fingerprinting) for EACH of these, and justify it in one line:

  a) /index.html                → references the fingerprinted bundles
  b) /assets/app.a1b2c3d4.js    → content-hashed JS bundle
  c) /api/products              → public product list, changes ~hourly, same for all users
  d) /api/me                    → the logged-in user's profile (personalized)
  e) /api/checkout (POST)       → a payment mutation

# For each, decide: max-age? s-maxage? public vs private? no-cache vs no-store?
# immutable? ETag? Vary? Then answer:
# 1. Which of these may a CDN (shared cache) store? Which absolutely must it NOT?
# 2. For (c), you want the CDN to serve stale instantly during a 5-min origin outage
#    while refreshing in the background. Which two directives do you add?
# 3. After a data fix, you must purge (c) from the CDN. Hard purge or soft purge — why?
```

<details>
<summary>Reference answer (peek only after attempting)</summary>

```
a) index.html          Cache-Control: no-cache            (store, but always revalidate → new deploys picked up)
b) app.a1b2c3d4.js     Cache-Control: public, max-age=31536000, immutable   (URL changes on rebuild → safe forever)
c) /api/products       Cache-Control: public, s-maxage=3600, stale-while-revalidate=300, stale-if-error=86400
                       + ETag + Vary: Accept-Encoding      (CDN caches it; serves stale on refresh/outage)
d) /api/me             Cache-Control: private, no-store    (never in a shared cache; personalized)
e) POST /checkout      Cache-Control: no-store             (mutations are never cacheable)

1. CDN may store (b) and (c). It must NOT store (d) or (e).
2. stale-while-revalidate (background refresh) + stale-if-error (serve stale during outage).
3. Soft purge — mark stale so the next request revalidates (and SWR serves stale meanwhile),
   avoiding a thundering herd of misses hammering the just-recovered origin.
```
</details>

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **A user reports seeing an old version of your JS after a deploy. The server has the new file. What is the most likely cause, and why can't you "just clear their cache"?**

2. **Explain the difference between `no-cache` and `no-store` in one sentence each. Which one do you use for a bank balance, and why is the other one dangerous there?**

3. **A response has BOTH an `ETag` and a `Last-Modified`. A cache sends BOTH `If-None-Match` and `If-Modified-Since`. Which one does the origin honor, and which does it ignore?**

4. **What is the difference between `max-age` and `s-maxage`? Give a scenario where you'd set `s-maxage=3600` but `max-age=60`.**

5. **Your `/products` endpoint is served through a CDN. The origin goes down for 3 minutes. What single directive keeps the CDN serving the last-known-good response during the outage, and what directive makes it refresh in the background once the origin is back?**

6. **Why does a shared cache need `Vary: Accept-Encoding`? What breaks without it?**

7. **A `304 Not Modified` costs one full round-trip — so how does it actually save anything compared to a `200`?**

---

## Quick Reference Card

### Response `Cache-Control` directives

| Directive | Meaning | Applies to |
|-----------|---------|------------|
| `max-age=N` | Fresh for N seconds | all caches |
| `s-maxage=N` | Fresh for N seconds; **overrides `max-age`** | **shared caches only** |
| `public` | Any cache may store (even with `Authorization`) | all |
| `private` | Only a **private** (browser) cache may store | browser only |
| `no-cache` | May store, but **must revalidate before every use** | all |
| `no-store` | **Never store.** No disk, no memory. | all |
| `must-revalidate` | Once stale, MUST revalidate; **must not serve stale** (return 504 if origin down) | all |
| `proxy-revalidate` | Like `must-revalidate` but **shared caches only** | shared |
| `immutable` | Body won't change; **don't even revalidate on reload** | all |
| `stale-while-revalidate=N` | Serve stale up to N s **while refreshing in background** | all |
| `stale-if-error=N` | Serve stale up to N s **if origin errors / is down** | all |

### Request `Cache-Control` directives (client → cache)

| Directive | Meaning |
|-----------|---------|
| `no-cache` | Force revalidation even if fresh (hard reload) |
| `no-store` | Don't store the response |
| `max-age=N` | Give me a response no older than N seconds |
| `max-stale=N` | I'll accept a stale response, up to N s past expiry |
| `min-fresh=N` | Give me a response that stays fresh at least N more s |
| `only-if-cached` | Return the cached copy or `504` — **don't contact origin** |

### Validators & the 304 flow

| Header | Direction | Purpose |
|--------|-----------|---------|
| `ETag: "abc"` | response | Strong validator (byte-exact version id) |
| `ETag: W/"abc"` | response | **Weak** validator (semantically equal, not byte-exact) |
| `Last-Modified: <date>` | response | Fallback validator (modification time) |
| `If-None-Match: "abc"` | request | "Send body only if ETag differs" → **wins over If-Modified-Since** |
| `If-Modified-Since: <date>` | request | "Send body only if changed since" → **ignored if If-None-Match present** |
| `Age: N` | response | Seconds spent in shared caches (`>0` ⇒ served by a shared cache) |
| `Vary: H1, H2` | response | Extra request headers that form the cache key |

### Cheat block

```
# Fingerprinted static asset (safe forever)
Cache-Control: public, max-age=31536000, immutable

# HTML entry point (cached but always fresh)
Cache-Control: no-cache

# Public API, CDN-cacheable, resilient
Cache-Control: public, s-maxage=3600, stale-while-revalidate=60, stale-if-error=86400
ETag: "..."   Vary: Accept-Encoding

# Personalized / sensitive — never in a shared cache
Cache-Control: private, no-store

# --- inspect from the terminal ---
curl -sI URL                                   # read Cache-Control / ETag / Age / X-Cache
curl -sI -H 'If-None-Match: "abc"' URL         # force a 304
curl -sI -H 'If-Modified-Since: <date>' URL     # 304 via Last-Modified

# no-cache  = store, but CHECK before use   (revalidate)
# no-store  = NEVER keep it                 (truly no cache)
# max-age   = browser freshness             s-maxage = CDN freshness (overrides max-age)
# ETag wins over Last-Modified when both are present
# "There are only two hard things in CS: cache invalidation and naming things." — Phil Karlton
```

---

## When Would I Use This at Work?

### Scenario 1: "Static assets are slow and our origin bandwidth bill is huge"
Put a CDN in front, fingerprint your assets (`app.[hash].js`), and serve them with `public, max-age=31536000, immutable`. Set the HTML to `no-cache`. Your origin now serves only HTML + API; JS/CSS/images are served from the edge with a 99% hit rate. Bandwidth bill drops, page loads get faster, and deploys are still instant because the HTML always revalidates.

### Scenario 2: "A deploy shipped broken JS and users are stuck on it"
You now recognize this instantly as a **stable-URL + long-max-age** trap. Short-term: you can't reach into browsers, but you *can* change the HTML to reference a new fingerprinted URL and purge the HTML/CDN. Long-term: adopt content-hash filenames so this can never recur. This is the #1 caching incident in real teams.

### Scenario 3: "The CDN is serving one user's data to another"
A shared cache stored a personalized response. You audit for any endpoint returning user-specific data with a bare `max-age` and no `private`. Fix: `private` (or `no-store` for anything sensitive) on every personalized response, and `Vary: Cookie` as a safety net. This is a security incident, not just a bug.

### Scenario 4: "Our homepage went down when the origin had an outage"
You add `stale-if-error=86400` so the CDN keeps serving the last-known-good page for up to a day during origin failures, and `stale-while-revalidate=60` so users never wait on a blocking revalidation. The site stays up (in read-only form) even when the backend is on fire.

### Scenario 5: "We updated pricing but the CDN still shows old prices"
You reach for the **purge/invalidation** API — choosing a **soft purge** for a hot page so the recovered/updated origin isn't hammered by a wall of misses, and remembering that propagation across edges takes seconds, not zero. You design the release so a brief window of staleness is acceptable, because you now know invalidation is never truly instant.

---

## Connected Topics

**Study before this:**
- `14-http-headers-in-depth.md` — Introduced `Cache-Control`, `ETag`, `Last-Modified` as headers. This topic is the full operational deep-dive on top of that foundation.
- `01-how-the-internet-works.md` — Why an edge cache 10ms away beats an origin 300ms away (latency and physics).

**Study next / alongside:**
- `29-reverse-proxies.md` — Nginx/Varnish as a **shared cache** in front of your app: where `s-maxage`, `proxy-revalidate`, and purge actually run.
- `30-cdns.md` — The edge/PoP layer in depth: origin pull, hit vs miss, purge propagation, and how the directives here map onto real CDN behavior.
- `18-rate-limiting-at-http-level.md` — Caching and rate limiting are the two levers that protect your origin from load.

**Mental model to carry forward:** caching is a *contract* between your origin and every hop in the path, written entirely in headers. Freshness decides when to ask; validators decide how cheaply to ask; `Vary` decides who shares an answer; and invalidation — the hard part — decides how you take an answer back.

---

*Doc saved: `/docs/networking/20-caching-in-depth.md`*
