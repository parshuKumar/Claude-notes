# 60 — CDN — Content Delivery Networks
## Category: HLD Components

---

## What is this?

A **CDN (Content Delivery Network)** is a global network of servers — called **edge servers** or **PoPs** (Points of Presence) — that keep copies of your content physically close to your users. When someone in Mumbai loads your site, they download the logo, the CSS and the JavaScript from a machine in Mumbai, not from your origin server in Virginia.

Think of it like a book publisher. The publisher (your **origin server**) prints the book once. But readers don't fly to the printing press — the publisher ships crates of copies to bookshops in every city (the **edge servers**), and readers walk five minutes to their local shop. Same book, 10,000× less travel.

That's the whole idea. A CDN is a **cache with a passport** — the same caching you already know, but distributed across the planet so that distance stops being your enemy.

---

## Why does it matter?

Because **you cannot optimize your way out of geography.**

You can make your Node.js handler 10ms faster. You can add an index and shave 20ms off the query. You can double your CPU. None of that touches the 200ms your bytes spend physically travelling through undersea fibre. Distance is a hard floor set by physics, and the only lever you have is to *move the bytes closer*.

Concretely, without a CDN:

- Users far from your origin see a slow site, and there is nothing in your code you can fix to help them.
- Every single request for `logo.png` — the same 40KB, byte for byte — hits your origin. A million users means a million identical downloads you pay for in bandwidth and CPU.
- A traffic spike (a launch, a Hacker News front page, a TV ad) lands entirely on your origin and kills it.
- You pay cloud egress prices (~$0.09/GB on AWS) for traffic a CDN would serve for a fraction of that.

**The interview angle:** "How do you serve static assets globally?" is a warm-up question in almost every HLD interview. And in the big designs — Netflix, YouTube, Instagram, a news site — the CDN is not a footnote, it *is* the architecture. In a video-streaming design, 95%+ of your bytes never touch your servers at all.

**The real-work angle:** a CDN is usually the single highest-leverage change you can make to a slow website. It is often a one-afternoon change (point DNS at the CDN, set your cache headers correctly) that halves your page-load time and cuts your bandwidth bill.

---

## The core idea — explained simply

### The Physics Argument (do this arithmetic out loud in an interview)

Light in a vacuum travels at ~300,000 km/s. But your packets don't travel through a vacuum — they travel through **glass fibre**, where light slows to roughly **two-thirds of that: ~200,000 km/s.**

Now put a user in Mumbai and your server in Virginia (`us-east-1`, where an enormous fraction of the internet lives).

```
Great-circle distance Mumbai → Virginia   ≈ 12,500 km
A request needs to go there AND back       × 2
                                          ─────────────
Round-trip distance                        = 25,000 km

Time = distance / speed
     = 25,000 km / 200,000 km/s
     = 0.125 s
     = 125 ms
```

**125 milliseconds. That is the theoretical minimum for one round trip** — and it assumes a perfectly straight fibre line, zero routers, zero queuing, zero processing. Reality (fibre follows coastlines, packets hop through 15+ routers) is closer to **180–250ms**.

Now remember that opening an HTTPS connection is not one round trip:

| Step | Round trips | Cost at 125ms/RTT |
|---|---|---|
| TCP handshake (SYN, SYN-ACK, ACK) | 1 | 125 ms |
| TLS 1.3 handshake | 1 | 125 ms |
| HTTP request → first byte of response | 1 | 125 ms |
| **Total before a single byte of HTML arrives** | **3** | **375 ms** |

And your server hasn't done *any work yet*. Your beautifully optimized 5ms handler is a rounding error next to 375ms of pure travel time.

Now put an edge server in Mumbai, ~20 km from the user:

```
Round-trip distance = 40 km
Time = 40 / 200,000 = 0.0002 s = 0.2 ms  (theoretical)
Real-world, including ISP hops and last-mile Wi-Fi ≈ 5-15 ms
```

Same three round trips, but at ~10ms each: **~30ms total instead of 375ms.** A **12× improvement**, and you didn't write a single line of code.

> **The takeaway to say in an interview:** "No amount of server optimization fixes the speed of light. The only fix is to move the bytes closer to the user. That's what a CDN is for."

### The Coffee Chain Analogy

Imagine you roast the world's best coffee beans in one warehouse in Virginia.

| In the analogy | In the system | Why it matters |
|---|---|---|
| The single roastery in Virginia | Your **origin server** | The one source of truth. Everything is created here. |
| Coffee shops in every city | **Edge servers / PoPs** | Close to customers. They hold *copies*, not originals. |
| A shop has the blend in stock | **Cache HIT** | Served in seconds. The roastery never hears about it. |
| A shop is out of stock, phones the roastery | **Cache MISS** | Slow *for that one customer*, who waits for the shipment. |
| The regional depot that shops order from | **Origin shield** | So 200 shops don't all phone the roastery at once. |
| "Best before" date on the bag | **`Cache-Control: max-age`** | How long a copy is considered fresh. |
| A recall notice: "throw out batch #47" | **Cache purge / invalidation** | Slow, manual, easy to get wrong. |
| Printing the batch number on the bag | **Versioned URLs** (`app.a1b2c3.js`) | A new batch is a *different product*. No recall needed. Ever. |
| A custom drink with your name on it | **Personalized / authenticated response** | Cannot be pre-stocked. Must be made fresh, per person. |

The last two rows are where most engineers get into trouble, so we'll spend real time on them.

---

## Key concepts inside this topic

### 1. Pull CDN vs Push CDN

There are exactly two ways content gets *into* the edge cache.

**Pull CDN (origin-pull)** — the lazy, default, 95%-of-the-time answer.

You change nothing about how you store files. You just point your DNS at the CDN. The first time anyone in Tokyo asks for `/logo.png`, the Tokyo edge doesn't have it, so it fetches it from your origin, stores it, and serves it. Every subsequent Tokyo user gets it from the edge.

- **Self-maintaining.** New file on your origin? It shows up at the edge automatically on first request.
- **Cost:** the *first* user in each region pays the slow path (an edge miss is slightly *slower* than no CDN, because of the extra hop).
- **Cache is demand-driven** — cold content simply expires and disappears, which is exactly what you want.

**Push CDN** — you upload content to the CDN yourself, ahead of time.

You (or your CI pipeline) explicitly publish files to the CDN's storage. The edge never asks your origin for anything, because your origin isn't in the picture.

- **No cold-start miss.** Content is warm before the first user ever arrives.
- **You own the lifecycle:** you must upload, update, and delete. Forget to delete and you pay storage forever.
- Great for **large files** (a 4GB game patch — you really don't want that streaming through an origin-pull) and **predictable launches** (a movie drops at midnight; you pre-position it in every PoP a week early).

| | **Pull CDN (origin-pull)** | **Push CDN** |
|---|---|---|
| Who puts content at the edge? | The CDN, lazily, on first miss | You, ahead of time |
| Setup effort | Very low — change DNS, set headers | Higher — build an upload/publish step |
| First user in a region | Pays the slow path (cache miss) | Fast — already warm |
| Content lifecycle | Automatic (TTL expiry) | Manual (you upload and delete) |
| Load on origin | Low, but non-zero (misses + revalidation) | Zero |
| Storage cost | You pay for what's requested | You pay for everything you uploaded |
| Best for | Websites, APIs, images, JS/CSS — the normal case | Huge files, video libraries, scheduled launches |
| Risk | Thundering herd on a cold cache | Stale content you forgot to re-upload |

> **Interview line:** "I'd default to a pull CDN because it's self-maintaining. I'd reach for push only for large, predictable assets — like pre-positioning a movie in every PoP before a launch."

### 2. The cache HIT / MISS request flow

This is the flow you must be able to draw. See the diagrams section for the full picture, but the mental model is:

- **HIT:** user → nearest edge → done. Your origin never learns the request happened.
- **MISS:** user → nearest edge → (edge fetches from origin, and this is the full physics penalty) → edge stores it → serves user.

The number that matters is the **cache hit ratio**. If 97% of requests are hits, your origin sees 3% of the traffic. Do that arithmetic in an interview:

```
Traffic:            50,000 requests/second for static assets
Cache hit ratio:    97%
Origin traffic:     50,000 × 0.03 = 1,500 req/s

Without a CDN, you'd need to size your origin fleet for 50,000 req/s.
With one, you size it for 1,500 — a 33× reduction in servers.
```

### 3. What belongs on a CDN (and what doesn't)

**Put on the CDN — content that is identical for every user:**

- Images, thumbnails, avatars
- CSS and JavaScript bundles
- Fonts (`.woff2`)
- Video and audio segments (HLS/DASH `.ts` / `.m4s` chunks — this is the *dominant* CDN use case by bytes)
- Downloads: PDFs, installers, firmware
- Public API responses that change slowly (a country list, a public product catalogue)

**Do NOT put on the CDN:**

- **Personalized responses.** `/api/me`, a logged-in dashboard, a cart. If the edge caches Alice's response and serves it to Bob, you have just leaked Alice's data to Bob. This is a real, recurring, career-limiting bug. Mark these `Cache-Control: private, no-store`.
- **Rapidly-changing data.** A live stock price, a live score. The TTL would have to be so short the cache is pointless.
- **Anything with a `Set-Cookie` or an `Authorization` header** — unless you have very deliberately configured the cache key to include the user identity.

> **The nuance to mention (it scores points):** this line is blurring. **Edge compute** (Cloudflare Workers, Lambda@Edge, Fastly Compute) lets you run code *at the PoP*, so you can cache the expensive shared shell of a page and assemble the personalized bits at the edge. **ESI (Edge Side Includes)** is the old-school version of the same idea: cache the page, punch a hole in it, and fill the hole per user. So "personalized ⇒ uncacheable" is a good default, not a law.

### 4. HTTP caching headers, done properly

The CDN doesn't guess. **You tell it what to do with headers**, and getting these right is most of the job.

```js
import express from "express";
const app = express();

// Fingerprinted build assets: the filename contains a content hash,
// so this exact URL can NEVER mean anything different. Cache forever.
app.use("/static", express.static("dist", {
  setHeaders: (res) => {
    // immutable = "don't even bother revalidating me, I will never change"
    res.setHeader("Cache-Control", "public, max-age=31536000, immutable"); // 1 year
  },
}));

// The HTML entry point: it must NOT be cached long, because it's the
// document that points at the new hashed bundle names after a deploy.
app.get("/", (req, res) => {
  // no-cache does NOT mean "don't cache" — see below. It means
  // "you may store this, but revalidate with me before every reuse".
  res.setHeader("Cache-Control", "public, no-cache");
  res.send(html);
});

// A shared, non-personalized API response.
app.get("/api/products", async (req, res) => {
  // max-age   → what BROWSERS should do (30s)
  // s-maxage  → what SHARED caches (the CDN) should do (5 min) — overrides max-age for them
  // stale-while-revalidate → after it expires, serve the stale copy INSTANTLY
  //   for up to 60s while refreshing in the background. Nobody ever waits.
  res.setHeader("Cache-Control", "public, max-age=30, s-maxage=300, stale-while-revalidate=60");
  res.json(await getProducts());
});

// A personalized response. Poison for a shared cache.
app.get("/api/me", requireAuth, (req, res) => {
  res.setHeader("Cache-Control", "private, no-store");
  res.json(req.user);
});
```

**The directives, precisely:**

| Directive | What it actually means |
|---|---|
| `public` | Any cache — including the CDN — may store this. |
| `private` | Only the **user's own browser** may store it. Shared caches (the CDN) must not. Use for anything user-specific. |
| `max-age=N` | Fresh for N seconds. Applies to all caches. |
| `s-maxage=N` | Fresh for N seconds **in shared caches only** (the CDN). Overrides `max-age` there. This is how you say "browsers cache for 30s, CDN caches for 5 minutes". |
| `no-cache` | **The most misunderstood directive in HTTP.** It does **not** mean "do not cache". It means: *store it, but you must revalidate with the origin before serving it.* You still get the bandwidth win of a `304 Not Modified`. |
| `no-store` | *This* is "do not cache". Don't write it to disk, don't write it to RAM, forget it exists. Use for genuinely sensitive responses. |
| `immutable` | "This URL's content will never change. Don't revalidate even on a hard reload." Pair with a 1-year `max-age` on hashed filenames. |
| `stale-while-revalidate=N` | For N seconds *after* expiry, serve the stale copy immediately while fetching a fresh one in the background. Turns a slow miss into a fast hit for everyone but the background fetch. |

**Revalidation: ETag and Last-Modified**

When a cached copy expires, the cache doesn't have to re-download it — it can *ask if it's still good*.

```
Origin's original response:
    HTTP/1.1 200 OK
    ETag: "a1b2c3d4"            ← a fingerprint of the content
    Last-Modified: Tue, 07 Jul 2026 10:00:00 GMT
    Cache-Control: public, max-age=60

60 seconds later, the cache revalidates:
    GET /api/products
    If-None-Match: "a1b2c3d4"       ← "I have this version; still current?"
    If-Modified-Since: Tue, 07 Jul 2026 10:00:00 GMT

If nothing changed, the origin replies:
    HTTP/1.1 304 Not Modified       ← ~200 bytes, NO body
```

A `304` still costs you a round trip, but it costs you almost **zero bytes**. On a 2MB response that's a 10,000× bandwidth saving. Express and most frameworks generate `ETag`s for you automatically.

### 5. Cache invalidation — and the versioned-URL trick

Phil Karlton's line — *"There are only two hard things in Computer Science: cache invalidation and naming things"* — exists for a reason.

**Option A: explicit purge.** You call the CDN's API and say "delete `/app.js` from every PoP on Earth."

```js
// Cloudflare-style purge. It works. It's also the option you should avoid.
async function purge(urls) {
  const res = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/purge_cache`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.CF_API_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ files: urls }),
    }
  );
  if (!res.ok) throw new Error(`Purge failed: ${res.status}`);
}
```

Why this is bad as a *strategy*:

- **Slow.** Propagating to hundreds of PoPs takes seconds to minutes. Not atomic.
- **Rate-limited.** Most CDNs cap purges (e.g. a few thousand URLs/day on cheaper plans).
- **Error-prone.** Miss one URL and a stale asset lingers for its full TTL. Users see a half-updated site.
- **It creates a thundering herd** — you just emptied every edge cache at once, and now every PoP stampedes your origin.

**Option B: never invalidate anything — the versioned-URL trick. This is the right answer.**

Put a **hash of the file's content into its filename**:

```
dist/
  app.a1b2c3d4.js      ← hash of the file contents
  vendor.9f8e7d6c.js
  main.4b5c6d7e.css
```

Now serve those with `Cache-Control: public, max-age=31536000, immutable` (one year).

You never purge, because **you never need to**. Change one character of source code and the build produces `app.e5f6a7b8.js` — a **different URL**, which no cache anywhere has ever seen, so it's fetched fresh. The old `app.a1b2c3d4.js` sits harmlessly in caches until it expires and is evicted. Users mid-session on the old HTML keep working, because their JS file still exists.

The only file you must keep short-lived is the **HTML entry point**, because it's the thing that names the hashed bundles:

```html
<!-- index.html — Cache-Control: public, no-cache  (revalidate every time) -->
<script src="/static/app.a1b2c3d4.js"></script>
<link rel="stylesheet" href="/static/main.4b5c6d7e.css" />
```

A tiny, cheap, always-revalidated HTML file points at enormous, cached-forever assets. That's the whole design. Webpack, Vite, esbuild, Next.js, Rails Sprockets — **every modern build tool does this**, and now you know why.

> **Say this in an interview:** "I'd avoid purging entirely by using content-hashed filenames with a one-year immutable TTL, and keeping only the HTML short-lived. A new deploy produces new URLs, so invalidation becomes a non-problem." That single sentence signals real production experience.

The same trick works for user content without a build step — just add a version query param or a new key:

```js
// Bad: overwrite the same key. Now you must purge, and you'll race with readers.
const key = `avatars/${userId}.jpg`;

// Good: a new upload is a new URL. Nothing to purge, and it's atomic.
const key = `avatars/${userId}/${crypto.randomUUID()}.jpg`;
await s3.putObject({ Bucket, Key: key, Body: buffer });
await db.users.update(userId, { avatarUrl: `https://cdn.example.com/${key}` });
```

### 6. Origin shield — protecting your origin from the edges

Here's a failure mode people miss. You have 300 PoPs. You deploy a new asset. Traffic arrives worldwide. **All 300 PoPs miss simultaneously and all 300 hammer your origin for the same file.** That's a *thundering herd* — your CDN, the thing meant to protect you, just DDoS'd you.

The fix is an **origin shield**: one designated mid-tier cache (usually a PoP near your origin) that *all other PoPs* fetch through.

```
300 edges  ──▶  1 origin shield  ──▶  origin
```

Now your origin sees **one** request instead of 300. The shield also does **request coalescing** — if 50 edges ask for the same missing file at the same time, the shield makes exactly one origin request and fans the response back out.

Every serious CDN offers this (CloudFront "Origin Shield", Fastly "Shielding", Cloudflare "Argo Tiered Cache"). Turn it on. Mentioning it unprompted is a strong signal in an interview.

### 7. Anycast — how the user finds the nearest edge

How does a Mumbai user's browser end up talking to the Mumbai PoP without knowing it exists? Two mechanisms:

- **DNS-based routing:** the CDN's DNS server sees roughly where the resolver is and returns the IP of a nearby PoP. Different users get different answers for `cdn.example.com`. (Akamai's classic approach.)
- **Anycast:** the *same* IP address is announced from every PoP simultaneously via BGP. Internet routing then naturally delivers your packet to the topologically closest one. One IP, hundreds of machines. (Cloudflare's approach.)

Anycast has a bonus superpower: it's the best DDoS defence there is. A 5 Tbps attack against one anycast IP is automatically *spread across every PoP on the planet*, and each one absorbs a small, survivable slice.

---

## Visual / Diagram description

### Diagram 1 — Edge PoPs around the world

```
                    ORIGIN (us-east-1, Virginia)
                    ┌──────────────────────────┐
                    │  App servers + Database  │
                    │  The ONE source of truth │
                    └────────────▲─────────────┘
                                 │ (only cache MISSES get here)
                    ┌────────────┴─────────────┐
                    │      ORIGIN SHIELD       │  ← mid-tier cache
                    │  coalesces edge misses   │    protects the origin
                    └────────────▲─────────────┘
        ┌──────────────┬─────────┴────────┬──────────────┐
        │              │                  │              │
  ┌─────┴─────┐  ┌─────┴─────┐      ┌─────┴─────┐  ┌─────┴─────┐
  │ PoP       │  │ PoP       │      │ PoP       │  │ PoP       │
  │ London    │  │ São Paulo │      │ Mumbai    │  │ Tokyo     │
  │ [cache]   │  │ [cache]   │      │ [cache]   │  │ [cache]   │
  └─────▲─────┘  └─────▲─────┘      └─────▲─────┘  └─────▲─────┘
        │ 8ms          │ 12ms             │ 10ms         │ 9ms
     ┌──┴──┐        ┌──┴──┐            ┌──┴──┐        ┌──┴──┐
     │users│        │users│            │users│        │users│
     └─────┘        └─────┘            └─────┘        └─────┘

   Users always talk to their NEAREST PoP (via anycast or geo-DNS).
   The long, expensive fibre hop to Virginia happens rarely — only on a miss.
```

### Diagram 2 — Cache HIT (the common case, ~97% of requests)

```
  ┌──────────┐   1. GET /static/app.a1b2c3.js      ┌────────────────┐
  │  User    │ ─────────────────────────────────▶  │  Mumbai PoP    │
  │ (Mumbai) │            ~10 ms                   │  [HIT — cached]│
  │          │ ◀─────────────────────────────────  │                │
  └──────────┘   2. 200 OK + file       ~10 ms     └────────────────┘
                                                            │
                                                            ✗
                                              ORIGIN IS NEVER CONTACTED

  TOTAL: ~20 ms.  Origin load: ZERO. Origin bandwidth: ZERO.
```

### Diagram 3 — Cache MISS (the first request in a region, ~3%)

```
  ┌──────────┐  1. GET /static/app.a1b2c3.js   ┌───────────┐
  │  User    │ ──────────────────────────────▶ │ Mumbai PoP│
  │ (Mumbai) │           ~10 ms                │  [MISS]   │
  └──────────┘                                 └─────┬─────┘
                                                     │ 2. fetch from shield
                                                     │    ~180 ms  ◀── the physics
                                                     ▼
                                              ┌──────────────┐
                                              │ Origin Shield│
                                              │ (Virginia)   │
                                              └──────┬───────┘
                                                     │ 3. ~2 ms (same region)
                                                     ▼
                                              ┌──────────────┐
                                              │   ORIGIN     │
                                              │  ~15 ms work │
                                              └──────┬───────┘
                                                     │ 4. 200 OK + Cache-Control
                                                     ▼
                                              ┌───────────┐
                                              │ Mumbai PoP│  5. STORE for 1 year
                                              └─────┬─────┘
  ┌──────────┐                                      │
  │  User    │ ◀────────────────────────────────────┘
  └──────────┘   6. 200 OK   ~10 ms

  TOTAL: 10 + 180 + 2 + 15 + 2 + 180 + 10  ≈  400 ms   (slower than no CDN!)
  But this happens ONCE. Every Mumbai user after this gets Diagram 2: ~20 ms.
```

**What to notice:** a miss is *worse* than having no CDN at all — you paid for an extra hop. The entire value proposition is that misses are rare. That's why cache hit ratio is the metric you monitor, why long TTLs matter, and why an origin shield exists.

---

## Real world examples

### Netflix — Open Connect

Netflix doesn't rent a CDN. It **built one and gave the hardware away.** Open Connect Appliances (OCAs) are Netflix-built storage servers that Netflix ships, for free, to **ISPs**, who rack them **inside their own datacenters**.

The consequence is remarkable: when you press play in Bangalore, the video bytes come from a box sitting in *your ISP's* building — often a single network hop away. They never cross the public internet backbone, never cross an ocean, and never touch AWS.

The push/pull split is instructive. Netflix's control plane (login, search, recommendations, the "play" button) runs on AWS. But the *video bytes* — which are >95% of all traffic — are **pushed** to OCAs overnight during off-peak hours, guided by Netflix's own predictions of what each region will watch tomorrow. It is the textbook case for a push CDN: enormous files, and highly predictable demand.

### Cloudflare — anycast at the edge

Cloudflare runs a large anycast network: the same IP is announced from every PoP, and BGP delivers each user to the nearest one. Free tiers, a very aggressive DDoS-absorption story (an attack gets spread across the whole network by construction), and **Workers** — V8 isolates that run *your JavaScript at the edge*, in a few milliseconds, with no cold start.

Workers are what makes the "personalized content can't be cached" rule negotiable. You can cache the expensive, shared page shell at the edge and have a Worker stitch in the per-user pieces, so the user gets an edge-speed response for a page that is nonetheless unique to them.

### Akamai — the original, and the one behind the scenes

Akamai came out of MIT research in the late 1990s and effectively invented the commercial CDN. It has an unusually large footprint, with servers deployed deep inside ISP networks worldwide rather than only in big datacenters. It's the CDN quietly behind an enormous amount of banking, airline, government, and large-media traffic — including huge live events, where the whole game is absorbing a spike of millions of concurrent viewers that would obliterate any origin.

---

## Trade-offs

| Benefit | What it costs you |
|---|---|
| Dramatically lower latency for distant users | An extra hop on a miss — a cold cache is *slower* than no CDN |
| Origin load drops by 10–100× | A whole new system to configure, debug and monitor |
| Bandwidth bill drops (CDN egress is cheap) | A new bill, and egress overage pricing that can surprise you |
| Absorbs traffic spikes and DDoS automatically | Cache-poisoning risk: one bad `Cache-Control` header can leak user data |
| Free TLS termination at the edge; HTTP/3 for free | Debugging is harder — "is it my bug, or a stale edge copy in Sydney?" |
| Survives an origin outage (`stale-if-error`) | Staleness: a change may take up to your TTL to be visible everywhere |

**Cost model.** The reason CDNs win on money as well as speed:

| Path | Typical egress price | 100 TB/month |
|---|---|---|
| Direct from AWS EC2/S3 to the internet | ~$0.085/GB | ~$8,500 |
| Via CloudFront (and origin→CDN transfer is free) | ~$0.02–0.08/GB, falling steeply with volume | ~$2,000–7,000 |
| Cloudflare (bandwidth not metered on most plans) | flat fee | ~$0–200 |

Bandwidth-heavy products (video, images, downloads) can cut their bill by an order of magnitude. The CDN often *pays for itself* before you even count the latency win.

**When a CDN is NOT worth it:**

- **All your users are in one city, next to your datacenter.** Distance is already ~0. A CDN adds a hop and buys nothing.
- **Everything you serve is personalized and uncacheable.** An internal admin dashboard with 40 users. Your hit ratio would be 0%. (Though you may *still* want the CDN for TLS termination, DDoS protection and a warm TCP connection at the edge — that "acceleration" alone can be worth it.)
- **Truly real-time data.** A trading feed or a multiplayer game's state. A cache is the wrong tool.
- **Very low traffic.** A hobby blog with 100 visitors a month — the operational complexity outweighs the benefit. (Though the free tiers make this argument weak.)

**Rule of thumb:** if you serve static assets to users in more than one country, put a **pull CDN** in front of them, use **content-hashed filenames with a one-year immutable TTL**, keep the **HTML short-lived**, and turn on the **origin shield**. That covers 90% of real systems, and it's the answer that lands in an interview.

---

## Common interview questions on this topic

### Q1: "Why can't I just make my servers faster instead of adding a CDN?"

**Hint:** Do the physics arithmetic out loud. Light in fibre travels at ~200,000 km/s; a Mumbai↔Virginia round trip is ~25,000 km ≈ **125ms minimum**, and an HTTPS request needs ~3 round trips ≈ 375ms *before your server executes a single instruction*. Server speed is a term you can shrink; distance is a floor you cannot. The only fix is to move the bytes closer to the user.

### Q2: "How do you invalidate content on a CDN?"

**Hint:** The trap is that they expect you to say "call the purge API". Say that purging exists, is slow, is rate-limited, is non-atomic, and causes a thundering herd — then say **you avoid it entirely with content-hashed URLs** (`app.a1b2c3.js`) cached for a year as `immutable`, with only the small HTML entry point kept short-lived (`no-cache`, so it revalidates and returns a cheap `304`). A new build makes a new URL, so there is nothing to invalidate. Purge only for emergencies — a leaked secret, a legal takedown.

### Q3: "What does `Cache-Control: no-cache` do?"

**Hint:** It does **not** mean "don't cache" — that's `no-store`. `no-cache` means "you may store this, but you must **revalidate** with the origin before serving it." The revalidation uses `ETag`/`If-None-Match` and typically returns a `304 Not Modified` with no body — so you still save nearly all the bandwidth, you just pay one round trip. Getting this right is a small but real credibility signal.

### Q4: "Pull or push CDN for a video-streaming service?"

**Hint:** Both, for different things. **Push** the video segments — they're huge, demand is predictable, and you can pre-position tomorrow's popular titles in each region overnight (this is literally Netflix Open Connect). **Pull** for thumbnails, CSS/JS, and the app shell — small, long-tailed, and self-maintaining. Then mention that the *catalogue/metadata API* is personalized and shouldn't be CDN-cached at all, except perhaps via edge compute.

### Q5: "Your cache hit ratio is 60%. How do you improve it?"

**Hint:** Work through the usual suspects: (1) TTLs too short — raise `s-maxage`, and add `stale-while-revalidate` so expiry doesn't cause a miss; (2) **cache key too specific** — query strings like `?utm_source=twitter` or a `Vary: User-Agent` header can fragment one asset into thousands of distinct cache entries, so normalize the key and strip tracking params; (3) no **origin shield**, so every PoP misses independently; (4) unhashed filenames forcing short TTLs; (5) cookies on static-asset responses making them accidentally uncacheable. Then say you'd look at the CDN's per-path hit-ratio report rather than guess.

---

## Practice exercise

### Build and instrument a mini-CDN

Build a two-process system in Node that demonstrates the whole model. ~30–40 minutes.

**1. The origin (`origin.js`, port 4000).** Serve one endpoint, `GET /assets/:file`. Make it **deliberately slow** — `await sleep(200)` — to simulate a transcontinental round trip plus real work. Log every request it receives; this log is the point of the exercise. Set `Cache-Control: public, max-age=10` and a stable `ETag` on the response.

**2. The edge (`edge.js`, port 3000).** An in-memory cache in front of the origin:

- On a request, look up the URL in a `Map`.
- **HIT** (entry exists and hasn't expired): return it immediately, with a header `X-Cache: HIT`.
- **MISS**: fetch from the origin, store it with an expiry computed from the origin's `max-age`, return it with `X-Cache: MISS`.
- **Request coalescing (the interesting part):** if 10 requests for the same missing key arrive at once, make **exactly one** origin fetch. Store the in-flight `Promise` in a second `Map` keyed by URL, and have the other 9 `await` the same promise. Delete the entry when it settles.

**3. Prove it works.** Fire 50 concurrent requests for the same asset:

```bash
seq 1 50 | xargs -P 50 -I{} curl -s -o /dev/null -D - http://localhost:3000/assets/app.js | grep X-Cache | sort | uniq -c
```

**What to produce:**

- The origin's log should show **exactly ONE** request, not 50. If it shows 50, your coalescing is broken — that's the thundering herd, and you've just reproduced it.
- A one-paragraph note on what happens after the 10-second TTL expires and 50 more requests arrive.

**Stretch goals:** (a) implement `stale-while-revalidate` — serve the stale entry instantly while refreshing in the background, and show that *no* user ever waits after the first fill; (b) add `If-None-Match` revalidation so an expired-but-unchanged entry costs a `304` and no body transfer; (c) add a second edge process on port 3001 and make both fetch through a shared "shield" process — then show the origin still sees only one request.

---

## Quick reference cheat sheet

- **The physics:** light in fibre ≈ 200,000 km/s. Mumbai↔Virginia round trip ≈ 25,000 km ≈ **125 ms minimum**. You cannot optimize past it. Move the bytes closer. That is the entire reason CDNs exist.
- **Three round trips** (TCP + TLS + HTTP) before the first byte — so multiply that 125ms by ~3.
- **Pull CDN** = lazy, fetches from origin on first miss, self-maintaining. **The default.**
- **Push CDN** = you upload ahead of time. For huge files and predictable launches. You own the lifecycle.
- **HIT** = user → edge → done (~20ms, origin untouched). **MISS** = user → edge → origin → edge → user (~400ms). Misses must be rare; **hit ratio is the metric you watch.**
- **`no-cache` ≠ don't cache.** It means *revalidate before use*. `no-store` means don't cache. Everyone gets this wrong; don't.
- **`max-age`** = browsers. **`s-maxage`** = shared caches (the CDN), and it wins there.
- **`public`** = the CDN may store it. **`private`** = browser only. Use `private, no-store` for anything authenticated — a mis-cached personalized response leaks one user's data to another.
- **`ETag` + `If-None-Match` → `304 Not Modified`:** one round trip, ~zero bytes.
- **`stale-while-revalidate=N`:** serve stale instantly, refresh in the background. Nobody waits at the TTL boundary.
- **Never purge — version instead.** `app.a1b2c3.js` + `max-age=31536000, immutable`. New build ⇒ new URL ⇒ nothing to invalidate. Keep only the HTML short-lived. This is what Webpack/Vite/Next do, and it's the right interview answer.
- **Origin shield:** a mid-tier cache so 300 PoPs missing at once become 1 origin request. Prevents the thundering herd.
- **Anycast:** one IP announced from every PoP; BGP routes you to the nearest. Also the best DDoS defence there is.
- **Cost:** CDN egress is typically far cheaper than cloud origin egress (~$0.085/GB on AWS vs cents, or flat-rate). For media-heavy products the CDN often pays for itself.
- **Skip the CDN** when all users are local, everything is personalized, or the data is truly real-time — but consider it anyway for TLS termination and DDoS absorption.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [59 — Caching in Depth](./59-caching-in-depth.md) — a CDN *is* a cache; TTLs, eviction, invalidation and the thundering herd all carry over directly. |
| **Next** | [61 — Databases: SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — once the CDN has absorbed your static traffic, the database is your next bottleneck. |
| **Related** | [06 — Latency and Throughput](./06-latency-and-throughput.md) — the latency-numbers-every-engineer-should-know that make the physics argument concrete. |
| **Related** | [11 — How the Web Works](./11-how-the-web-works.md) — DNS, TCP, TLS and HTTP headers, which are exactly the machinery a CDN plugs into. |
| **Related** | [57 — Reverse Proxy](./57-reverse-proxy.md) — a CDN edge is, structurally, a globally-distributed caching reverse proxy. |
