# 94 — Design a URL Shortener (TinyURL)
## Category: HLD Case Study

---

## What is this?

A URL shortener takes a long, ugly link like `https://www.example.com/products/electronics/laptops?ref=summer_sale_2026&utm_source=newsletter&id=99213` and hands you back something tiny like `https://sho.rt/aX9kP2q`. When anyone opens the short link, your service looks up the original address and **redirects** the browser there.

Think of it as a **coat check at a restaurant.** You hand over your bulky coat (the long URL), you get a small numbered ticket (the short code), and later you present the ticket and get the coat back. The ticket is useless on its own — it only has meaning because the coat check keeps a table mapping ticket numbers to coats.

That's the whole system: a giant, extremely fast, never-allowed-to-lose-anything ticket table.

---

## Why does it matter?

This is the **canonical first system design interview question**, and it's asked everywhere — Google, Amazon, Meta, and almost every startup. Not because URL shortening is hard, but because it's *deceptively* simple. The product has exactly two operations, so there is nowhere to hide. The interviewer gets to see, in 45 minutes, whether you can:

- Do capacity math without panicking
- Notice that the read path and write path have wildly different traffic (100:1) and design them separately
- Pick a strategy for generating unique IDs in a distributed system — the actual meat of the problem
- Talk about caching, sharding, and availability with real reasoning instead of buzzwords

**At work**, the same skeleton shows up constantly: any service that mints a short opaque ID and resolves it back (payment links, invite codes, magic-link tokens, feature-flag keys, file share links in Dropbox/Drive, tracking pixels) is a URL shortener wearing a different hat.

**And there's a scary property:** links are forever. If your shortener goes down, every link anyone ever shared — in emails, in printed QR codes, in tweets from five years ago — is broken. You cannot roll back a link that's already printed on a billboard. That single fact drives most of the design decisions below.

---

## The core idea — explained simply

### The Coat Check Analogy

Imagine you run the coat check at the busiest theatre in the city.

- Someone hands you a coat. You must give them a ticket number **that nobody else has.** If two people get ticket #47, one of them is going home in a stranger's coat. That's a **collision**, and it's the single hardest problem in this system.
- On a normal night you take in maybe 200 coats — but you hand coats *back* thousands of times because people keep popping out for a smoke and coming back. **Retrieval massively outnumbers storage.** So you optimize the "give the coat back" path ruthlessly.
- The coats people ask for most often (the ones near the door, the regulars) you keep on a rack right at the counter instead of walking to the back room. That's your **cache**.
- Your ticket numbers must not be guessable. If tickets are numbered 1, 2, 3, 4… then a thief who gets ticket #500 knows that #499 and #501 exist and can just ask for them. **Sequential IDs are an enumeration vulnerability.**
- Instead of inventing a ticket number on the spot while a customer waits (slow, and two clerks might invent the same one), you pre-print a big roll of random ticket numbers in the morning and each clerk tears off a strip of 100. That is a **Key Generation Service**, and it's the answer the interviewer is fishing for.

Here's the mapping:

| Coat check | URL shortener |
|---|---|
| The coat | The long URL |
| The ticket number | The short code (e.g. `aX9kP2q`) |
| The ticket ↔ coat ledger | The `urls` table in the database |
| Giving a coat back | The `GET /{shortCode}` redirect (the read path) |
| Taking a coat in | The `POST /api/v1/urls` create (the write path) |
| Rack by the counter | Redis cache of hot links |
| Two clerks printing the same ticket | Collision in a distributed ID generator |
| Pre-printed roll of random tickets | Key Generation Service (KGS) |
| A clerk quits holding a strip of 100 tickets | Wasted key block — acceptable loss |

The rest of this doc is just working through that analogy with real numbers.

---

## Key concepts inside this topic

This is your first end-to-end design, so we're going to execute the **5-step framework from [93 — The HLD Approach Framework](./93-hld-approach-framework.md)** visibly and in order. Do it this way in every interview:

```
  STEP 1  Requirements      →  what are we even building? (5 min)
  STEP 2  Estimation        →  how big is it? (5 min)
  STEP 3  API + Data model  →  the contract and the storage (10 min)
  STEP 4  High-level design →  the big diagram (10 min)
  STEP 5  Deep dive + scale →  the hard parts (15 min)
```

Never skip step 1 to jump to the diagram. Interviewers fail candidates for that more than for anything else.

---

### 1. Requirements (Step 1)

**Clarifying questions to ask the interviewer.** Ask these out loud. They cost you 90 seconds and they buy you the whole scope of the problem:

1. "Do users need custom aliases, like `sho.rt/my-blog`?" (Changes the key generation design.)
2. "Do links expire? Is there a default TTL?" (Changes the data model and adds a cleanup job.)
3. "Do we need analytics — click counts, referrers, geography?" (Adds an entire async pipeline.)
4. "Do users have accounts, or is this anonymous?" (Affects rate limiting and the schema.)
5. "What's the expected traffic — how many new links per month?" (You need this for step 2; if they won't give it, propose 100M/month and get their nod.)
6. "Is it OK if the same long URL produces two different short codes for two different users?" (This is the question that kills the hashing approach — see section 4.)
7. "Do we need to delete/update links after creation?"

**Functional requirements (what the system DOES):**

| # | Requirement | Priority |
|---|---|---|
| F1 | Given a long URL, return a unique short URL | Core |
| F2 | Given a short URL, redirect the user to the original long URL | Core |
| F3 | Optionally let the user pick a custom alias | Nice-to-have |
| F4 | Optionally let the user set an expiry time | Nice-to-have |
| F5 | Basic analytics: click count, timestamp, referrer, country | Nice-to-have |

**Out of scope** (say this explicitly — narrowing scope is a senior signal): editing a link's target after creation, user-facing dashboards, teams/orgs, paid plans.

**Non-functional requirements (how the system BEHAVES) — these drive the architecture:**

| # | Requirement | Why it matters |
|---|---|---|
| N1 | **Extremely read-heavy — roughly 100:1 read:write** | One link is created once and clicked hundreds of times. The read path gets 99% of your engineering budget. |
| N2 | **Low redirect latency — p99 under 100ms** | A redirect is an invisible hop before the real page. If it costs 500ms, every link on the internet feels broken. |
| N3 | **High availability — target 99.99%+** | A dead shortener breaks *every link ever shared*. Links are forever. You cannot un-print a QR code. Availability beats consistency here. |
| N4 | **Short codes must not be guessable or enumerable** | If codes are sequential, anyone can walk the whole keyspace and read every private link people shortened. This is a real privacy breach, not a theoretical one. |
| N5 | Durable — never lose a mapping | Losing a row means permanently breaking a link. |

**The key insight from N1 + N3:** we will happily accept *eventual consistency* on the write path (a brand-new link being unreadable for 200ms is fine) in exchange for *bulletproof availability* on the read path. Recall the CAP trade-off: we're choosing **AP**.

---

### 2. Capacity estimation (Step 2)

Show every line. Round aggressively. Nobody wants five decimal places on a whiteboard.

**Given assumption:** 100 million new URLs created per month.

**Step 2a — Write QPS (queries per second):**

```
Seconds in a month = 30 days × 24 h × 3600 s
                   = 30 × 86,400
                   = 2,592,000 seconds

Write QPS = 100,000,000 / 2,592,000
          ≈ 38.6
          ≈ 40 writes/second
```

**Step 2b — Read QPS.** Apply the 100:1 read:write ratio from N1:

```
Read QPS = 40 × 100
         = 4,000 reads/second
```

That's the number that matters. **4,000 redirects per second.**

**Step 2c — Peak traffic.** Real traffic is bursty. Assume peak = 3× average:

```
Peak read QPS  = 4,000 × 3 = 12,000 reads/second
Peak write QPS =    40 × 3 =    120 writes/second
```

**Step 2d — Storage.** Estimate the size of one row:

```
short_code    7 chars                        ≈    7 bytes
long_url      average URL length             ≈  200 bytes  (allow up to 2 KB)
user_id       UUID                           ≈   16 bytes
created_at    timestamp                      ≈    8 bytes
expires_at    timestamp                      ≈    8 bytes
index / row overhead, padding, metadata      ≈  260 bytes
                                             ------------
Total per record                             ≈  500 bytes
```

```
Per month  = 100,000,000 records × 500 bytes
           = 50,000,000,000 bytes
           = 50 GB / month

Per year   = 50 GB × 12 = 600 GB / year

Over 5 yrs = 600 GB × 5 = 3,000 GB
           ≈ 3 TB
```

**3 TB over 5 years.** That is *small*. A single beefy Postgres box could technically hold it. This is the most important realization of the whole estimation: **the data is tiny; the traffic is the problem.**

**Step 2e — Bandwidth:**

```
Write bandwidth = 40 writes/s × 500 bytes  = 20,000 B/s   ≈  20 KB/s  (nothing)
Read bandwidth  = 4,000 reads/s × 500 bytes = 2,000,000 B/s ≈  2 MB/s  (nothing)
```

Bandwidth is a non-issue. A single 1 Gbps NIC laughs at 2 MB/s. Say so and move on — knowing what *isn't* a bottleneck is as valuable as knowing what is.

**Step 2f — Cache sizing.** Apply the **80/20 rule**: roughly 20% of the links generate 80% of the traffic. Cache that hot 20%.

```
Reads per day = 4,000 reads/s × 86,400 s
              = 345,600,000
              ≈ 346 million reads/day

Cache the 20% of daily reads that are hottest:
  = 346,000,000 × 0.20
  = 69,200,000 records

Memory needed = 69,200,000 × 500 bytes
              = 34,600,000,000 bytes
              ≈ 34.6 GB
              ≈ 35 GB
```

**35 GB of cache.** That fits in **one or two Redis nodes** (a single r6g.2xlarge has 64 GB). Cheap. And because link popularity follows a power law, the *real* hit rate you'll get from 35 GB is far better than 80% — probably 90%+.

**What the numbers MEAN architecturally** (say this part out loud — it's where you show judgment):

| Number | Consequence |
|---|---|
| 40 writes/s | Trivially small. One database primary handles this with room to spare. Don't over-engineer the write path. |
| 4,000–12,000 reads/s | Real, but not extreme. Solvable with a cache + a handful of stateless read servers behind a load balancer. |
| 3 TB / 5 years | Small enough that sharding is a *future* concern, not a day-one one. But we'll design for it. |
| 35 GB working set | **A cache can hold essentially all the hot data.** Most reads will never touch the DB. This is the single biggest lever in the design. |
| 2 MB/s | Bandwidth is irrelevant. Ignore it. |

Conclusion: **this is a caching problem and an ID-generation problem, not a storage problem.**

---

### 3. API design (Step 3)

Two endpoints. That's it.

**Create a short URL:**

```
POST /api/v1/urls
Headers: Authorization: Bearer <token>
Body:
{
  "longUrl":     "https://example.com/some/very/long/path?x=1",
  "customAlias": "my-blog",          // optional
  "expiresAt":   "2027-01-01T00:00:00Z"  // optional
}

201 Created
{
  "shortUrl":  "https://sho.rt/aX9kP2q",
  "shortCode": "aX9kP2q",
  "longUrl":   "https://example.com/some/very/long/path?x=1",
  "expiresAt": "2027-01-01T00:00:00Z"
}

409 Conflict   → customAlias already taken
400 Bad Request → longUrl is malformed or fails the safety check
429 Too Many Requests → rate limit exceeded
```

**Redirect:**

```
GET /{shortCode}

302 Found
Location: https://example.com/some/very/long/path?x=1

404 Not Found  → unknown or expired code
410 Gone       → the code existed but expired (nicer than 404 if you track it)
```

Note this endpoint lives at the **root** of the short domain, not under `/api/`. Every byte in the short URL counts, so you don't waste them on a path prefix.

**Delete (optional):** `DELETE /api/v1/urls/{shortCode}` → `204 No Content`.

#### 301 vs 302 — the guaranteed interview question

You must have a real answer here. This is not trivia; the choice has huge operational consequences.

| | **301 Moved Permanently** | **302 Found (Temporary)** |
|---|---|---|
| Browser behavior | Caches the redirect, often **indefinitely** | Does not cache (by default); asks your server every time |
| Load on your servers | **Massively reduced** — a repeat click never reaches you | Full load — every click is a request |
| Analytics | **Destroyed.** You never see the repeat clicks. Your click counts undercount badly. | Accurate. Every click hits your server and can be logged. |
| Ability to change the target later | **Effectively impossible.** The redirect is burned into millions of browser caches you cannot reach. | Easy. Change the DB row; the next click picks it up. |
| Ability to expire/revoke a link | Broken. A user whose browser cached the 301 keeps getting redirected even after you delete the row. | Works. Delete the row, next click 404s. |
| SEO | Passes link equity to the target (good if that's what you want) | Does not pass link equity |

**The trade:** a 301 trades away control and analytics in exchange for a huge reduction in traffic. A 302 keeps you in the loop on every single click.

**Recommendation:** use **302** by default. The reasons:

1. Analytics is a stated requirement (F5), and 301 silently breaks it.
2. **Safety and revocation.** Shorteners are abused by spammers and phishers. If you discover a link points to malware, you *must* be able to kill it. With 301, you can't — the redirect is already cached in browsers worldwide. That alone settles it.
3. Our capacity math says we can *afford* the traffic. 4,000 QPS served from cache is easy. We're not paying much for the extra load.

Then add the nuance that gets you the offer: "**If we were purely optimizing for cost and had no analytics or moderation requirements — say, a shortener for a static internal wiki — I'd use 301 with a short `Cache-Control: max-age`, e.g. `301` plus `max-age=3600`, to get *some* browser caching without permanently losing control.**" That last trick — a 302 with a small `max-age`, or a 301 with a bounded TTL — is the practical middle ground.

---

### 4. The core problem: generating the short code

This is where the interview actually happens. Everything else is plumbing. You need a function that, called 40 times a second forever, across many servers, produces a **short, unique, non-guessable** string.

First, the sizing math.

#### Base62 math — how many characters do we need?

Base62 uses `a–z` (26) + `A–Z` (26) + `0–9` (10) = **62 characters.** We avoid symbols like `+` and `/` (base64) because they need URL-escaping and look ugly.

```
62^1 =                62
62^2 =             3,844
62^3 =           238,328
62^4 =        14,776,336
62^5 =       916,132,832
62^6 =    56,800,235,584   ≈  56.8 billion
62^7 = 3,521,614,606,208   ≈   3.5 trillion
```

How many URLs will we actually create?

```
100M/month × 12 months × 5 years = 6,000,000,000 = 6 billion URLs in 5 years
```

- **6 characters** gives 56.8B slots for 6B URLs → we'd use ~10.5% of the keyspace. It *fits*, but a 10% dense keyspace is dangerous: a random guess hits a live link 1 time in 10. An attacker can trivially enumerate.
- **7 characters** gives 3.5 trillion slots for 6B URLs → we'd use **0.17%** of the keyspace. A random guess hits a live link about 1 time in 590. Combined with rate limiting, enumeration becomes impractical.

**Decision: 7 characters.** It comfortably covers 100M/month for decades and keeps the keyspace sparse enough to resist guessing. (Note that sparseness only helps if the codes are *random*. Sequential codes are guessable no matter how long they are.)

#### Base62 encode / decode in JavaScript

```js
// base62.js
const ALPHABET = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
const BASE = ALPHABET.length; // 62

/**
 * Convert a non-negative integer to a base62 string.
 * We use BigInt because IDs from a distributed counter can exceed
 * Number.MAX_SAFE_INTEGER (2^53) once you're minting billions of them.
 */
export function encodeBase62(num) {
  let n = BigInt(num);
  if (n === 0n) return ALPHABET[0];

  let out = '';
  const base = BigInt(BASE);
  while (n > 0n) {
    out = ALPHABET[Number(n % base)] + out; // prepend: most-significant digit first
    n = n / base;                            // BigInt division truncates
  }
  return out;
}

/** Inverse of encodeBase62. Returns a BigInt. */
export function decodeBase62(str) {
  const base = BigInt(BASE);
  let n = 0n;
  for (const ch of str) {
    const digit = ALPHABET.indexOf(ch);
    if (digit === -1) throw new Error(`Invalid base62 character: ${ch}`);
    n = n * base + BigInt(digit);
  }
  return n;
}

/** Left-pad to a fixed width so every code is exactly 7 chars. */
export function toShortCode(num, width = 7) {
  return encodeBase62(num).padStart(width, ALPHABET[0]);
}

// encodeBase62(1)            -> 'b'
// toShortCode(1)             -> 'aaaaaab'
// toShortCode(56800235583n)  -> '9999999'  (62^6 - 1, the largest 6-digit value)
// decodeBase62('aX9kP2q')    -> 1234...n
```

Now, the four candidate strategies. **Evaluate all of them out loud** — the interviewer wants your reasoning, not just your conclusion.

#### Approach A — Hash the URL and truncate

Take `MD5(longUrl)` or `SHA-256(longUrl)`, base62-encode it, keep the first 7 characters.

```js
import crypto from 'node:crypto';
import { encodeBase62 } from './base62.js';

function hashCode(longUrl, salt = '') {
  const digest = crypto.createHash('sha256').update(longUrl + salt).digest();
  // Take the first 8 bytes as a BigInt, then base62 it, then truncate to 7.
  const n = digest.readBigUInt64BE(0);
  return encodeBase62(n).slice(0, 7);
}

// Collision handling: retry with an incrementing salt until the insert succeeds.
async function createWithHash(db, longUrl) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const code = hashCode(longUrl, String(attempt));
    const inserted = await db.insertIfAbsent(code, longUrl); // atomic: INSERT ... ON CONFLICT DO NOTHING
    if (inserted) return code;
    // else: collision — someone already owns this code. Salt and retry.
  }
  throw new Error('Could not generate a unique code after 5 attempts');
}
```

**Pros:** Dead simple. Stateless — any server can compute a code with zero coordination. Deterministic.

**Cons — and they're serious:**
- **Collisions are guaranteed.** You threw away most of the hash bits by truncating to 7 chars. By the birthday paradox, with a 3.5-trillion keyspace you start seeing collisions after roughly `√(3.5×10^12) ≈ 1.9 million` URLs. At 100M/month you'll hit collisions constantly.
- **Collision handling costs a DB round-trip on every write** (you must check-and-insert atomically), and the retry loop makes the write path non-deterministic in latency.
- **The same URL always maps to the same code.** That sounds like a feature (dedup!) but it breaks the product: two users shortening the same URL get the *same* short code, so you cannot give them per-user analytics, per-user expiry, or per-user revocation. And user A can delete a link that user B is relying on.

**Verdict:** viable for a toy, wrong for this product.

#### Approach B — Base62 of an auto-incrementing counter

Keep a global counter. Every new URL gets `counter++`, base62-encoded.

```js
let counter = 1_000_000_000n; // start high so early codes aren't 1-2 chars
function nextCode() {
  return toShortCode(counter++);
}
```

**Pros:** **Zero collisions, ever** — by construction. Codes are as short as possible. Trivially simple.

**Cons — and one of them is fatal:**
- **The codes are sequential, and therefore guessable.** If I shorten a link and get `aaaabZ4`, I know `aaaabZ3` and `aaaabZ5` exist. I can write a 20-line script and enumerate **every link every user has ever created**, including the private Google Doc your CFO shortened. This directly violates N4. (Base62 obscures the sequence a little, but not meaningfully — the last character just cycles through the alphabet.)
- **A single counter is a bottleneck and a SPOF.** If it lives in one database row, every write in the system serializes on that row. If that DB dies, you cannot create any links at all.

**Verdict:** the enumeration problem alone disqualifies it for a public shortener. But note: if your codes are *not* meant to be secret (e.g. internal analytics links), this is genuinely a fine design, and it's what a lot of real systems quietly do.

#### Approach C — Key Generation Service (KGS) ← the usual best answer

Flip the problem around. **Stop generating keys at request time.** Instead, an offline service pre-generates a huge pile of *random* 7-character base62 strings, dedupes them, and stores them in a `keys` table. When an app server needs codes, it takes a **block** of them.

```
┌─────────────────────────────────────────────────────────────┐
│  KGS (Key Generation Service)                               │
│                                                             │
│   Offline generator ──> keys table                          │
│                          ┌──────────┬────────┐              │
│                          │   key    │ status │              │
│                          ├──────────┼────────┤              │
│                          │ aX9kP2q  │ used   │              │
│                          │ 7mQz1Bd  │ FREE   │              │
│                          │ pL0vC8x  │ FREE   │              │
│                          └──────────┴────────┘              │
│                                                             │
│   API:  allocateBlock(n=1000) -> ['7mQz1Bd', 'pL0vC8x', …]  │
│         (marks all n rows used ATOMICALLY, in one txn)      │
└─────────────────────────────────────────────────────────────┘
             │  block of 1000 keys
             ▼
   ┌───────────────────┐   ┌───────────────────┐
   │  API server #1    │   │  API server #2    │
   │  in-memory queue: │   │  in-memory queue: │
   │  [k1, k2, … k1000]│   │  [k1, k2, … k1000]│
   └───────────────────┘   └───────────────────┘
```

```js
// kgs-client.js — runs inside every API server
export class KeyPool {
  #keys = [];
  #refilling = null;

  constructor(kgsClient, { blockSize = 1000, lowWatermark = 200 } = {}) {
    this.kgs = kgsClient;
    this.blockSize = blockSize;
    this.lowWatermark = lowWatermark; // refill *before* we run dry
  }

  async take() {
    // Refill in the background as soon as we're running low, so a request
    // almost never has to wait on the KGS. This is the whole point of blocks.
    if (this.#keys.length < this.lowWatermark) this.#refillInBackground();

    if (this.#keys.length === 0) {
      await this.#refillInBackground(); // rare: we truly ran out, block on it
    }
    return this.#keys.pop();
  }

  #refillInBackground() {
    // Collapse concurrent refills into a single in-flight request.
    if (!this.#refilling) {
      this.#refilling = this.kgs
        .allocateBlock(this.blockSize)
        .then((block) => { this.#keys.push(...block); })
        .finally(() => { this.#refilling = null; });
    }
    return this.#refilling;
  }
}
```

The KGS side must mark keys used **atomically**, or two servers racing on `allocateBlock` will both get the same keys:

```sql
-- Postgres: SKIP LOCKED makes concurrent allocations non-blocking and correct.
UPDATE keys
   SET status = 'used', allocated_at = now()
 WHERE key IN (
   SELECT key FROM keys
    WHERE status = 'free'
    ORDER BY random()      -- or just any order; the keys are already random
    LIMIT 1000
    FOR UPDATE SKIP LOCKED
 )
RETURNING key;
```

**Now discuss the three things every interviewer probes:**

1. **"The KGS is now a single point of failure."** Correct. Mitigate it: run the KGS as a **replicated, multi-instance service** with a replicated key store (a primary with standbys, or Cassandra with RF=3). Also note that because servers hold a block of 1000 keys locally, **the KGS can be down for minutes and the site keeps working** — each API server has ~25 seconds of write capacity buffered per 1000 keys at 40 writes/s, and you'd size blocks so a full KGS outage is survivable. That buffering is a real availability win, not just a performance one.

2. **"How do you make sure two servers never get the same key?"** The `UPDATE … FOR UPDATE SKIP LOCKED … RETURNING` above is a single atomic transaction: a key is either marked used and returned to exactly one caller, or not touched at all. There is no window where two callers see it free.

3. **"What if a server dies holding an un-used block?"** You **lose those keys.** 1000 keys is nothing against a 3.5-trillion keyspace — you'd have to lose a block a second for centuries to matter. **Accept the waste. Say so confidently.** (Trying to reclaim them via a lease/TTL mechanism adds a lot of complexity for zero real benefit; mentioning that you *considered* and *rejected* reclamation is a strong signal.)

**Why KGS is usually the best answer:**

- Codes are **random**, so they are not enumerable (satisfies N4).
- Codes are **unique by construction** (the generator deduped them offline), so there are **no collisions at write time and no retry loop** — the write path is a single insert with predictable latency.
- The expensive part (generating and deduping keys) happens **offline**, completely off the request path.
- **No per-request coordination.** A server pulls a key from a local in-memory array. That's a nanosecond, not a network hop.

#### Approach D — Distributed counters (ZooKeeper ranges / Snowflake)

Two variants worth naming:

**D1 — ZooKeeper range allocation.** Keep Approach B's counter, but instead of one global counter, ZooKeeper hands each server a *range*: server 1 gets `[1 – 1,000,000]`, server 2 gets `[1,000,001 – 2,000,000]`, and so on. Each server increments locally within its range and asks ZooKeeper for a new range when it runs out. This removes the per-write bottleneck of Approach B. If a server dies mid-range, you burn the rest of that range — same acceptable waste as the KGS.

**D2 — Snowflake-style IDs.** A 64-bit ID composed of `[timestamp | machine_id | sequence]`. Guaranteed unique across machines with zero coordination.

**But note the problem with Snowflake here:** the IDs are **huge**. A 64-bit number is up to `2^63 ≈ 9.2 × 10^18`, and:

```
Base62 digits needed for a 64-bit number = ceil(log62(2^63))
                                         = ceil(63 × log(2) / log(62))
                                         = ceil(63 × 0.1680)
                                         = ceil(10.58)
                                         = 11 characters
```

An 11-character code like `sho.rt/2kL9xQ4mB7z` is **ugly and defeats the purpose of a shortener.** And because Snowflake IDs are timestamp-prefixed, they're still partially *guessable* — codes created around the same time share a prefix.

**Verdict on D:** range-allocated counters (D1) are a solid, simple choice if you don't need unguessable codes. Snowflake is the wrong tool for this specific job — right idea, wrong output size.

#### The summary table you should draw

| Approach | Collisions? | Short? | Guessable? | Coordination cost | Verdict |
|---|---|---|---|---|---|
| **A. Hash + truncate** | Yes — must retry | Yes (7) | No | DB check per write | Simple but breaks per-user semantics |
| **B. Global counter** | **Never** | Yes (shortest) | **YES — fatal** | Every write serializes | Disqualified for public links |
| **C. KGS with key blocks** | **Never** | Yes (7) | **No — random** | **None per-request** | **Best answer** |
| **D1. ZK range counter** | Never | Yes | Yes (sequential) | One call per range | Good if guessability is OK |
| **D2. Snowflake** | Never | **No — 11 chars** | Partially | None | Wrong output size |

**Pick C. Explain why. Mention D1 as the simpler alternative if unguessability weren't required.**

---

### 5. Data model (Step 4)

The schema is almost insultingly simple — and that's the point.

**`urls` table:**

| Column | Type | Notes |
|---|---|---|
| `short_code` | `VARCHAR(7)` | **PRIMARY KEY.** This is the only thing we ever look up by. |
| `long_url` | `VARCHAR(2048)` | The destination. 2048 is the practical browser URL limit. |
| `user_id` | `UUID` (nullable) | Who created it. Null for anonymous. |
| `created_at` | `TIMESTAMP` | |
| `expires_at` | `TIMESTAMP` (nullable) | Null = never expires. |
| `is_custom` | `BOOLEAN` | Custom aliases skip the KGS; worth flagging. |

**`keys` table (owned by the KGS):**

| Column | Type | Notes |
|---|---|---|
| `key` | `VARCHAR(7)` | PRIMARY KEY |
| `status` | `ENUM('free','used')` | Indexed. |
| `allocated_at` | `TIMESTAMP` | |

**Analytics is NOT in this table.** Click events go to a separate store (see deep dive (e)). Never put a mutable `click_count` column on the `urls` row — you'd be doing a write on the hot read path 4,000 times a second, and that write would contend on the exact rows that are most popular. That's a classic candidate mistake.

**Choosing the database — reason it out:**

Look at the access pattern. It is **100% "get one row by primary key."**

- No joins. Ever.
- No range scans, no `WHERE long_url LIKE …`, no sorting, no aggregations, no transactions across rows.
- Massive read volume, tiny write volume.
- 3 TB, growing predictably.

That is the **exact definition of a key-value workload.** A relational database's entire value proposition — joins, referential integrity, complex queries, ACID across tables — is machinery we will never touch. We'd be paying for it and using none of it. **There is no need for a relational schema here.**

| Option | Fit |
|---|---|
| **DynamoDB / Cassandra** | **Excellent.** Built for exactly this: single-key lookup, horizontal scaling by hash of the partition key, high availability, no joins. Cassandra gives you multi-datacenter replication out of the box, which directly serves N3 (availability) and N2 (low latency via geo-local reads). |
| **PostgreSQL / MySQL** | **Also fine, honestly.** 3 TB with a B-tree index on a 7-char PK is not scary. A single primary + read replicas would handle this. It's simpler to operate, and you get strong consistency for free (useful for custom-alias uniqueness). |
| **MongoDB** | Works, but buys you nothing over the two above for this shape of data. |

**Recommendation:** pick a **wide-column / KV store (Cassandra or DynamoDB)** for `urls`, because the workload is a pure KV lookup at high read scale and these systems scale horizontally without you doing any work.

Then add the honest nuance: "**If this were a startup shipping v1, I'd start with Postgres** — 40 writes/s and 3 TB is well inside its comfort zone, it's simpler to run, and I can migrate to Cassandra when I actually need to. Choosing Cassandra on day one for 40 writes/s is over-engineering." Interviewers love that answer, because it shows you know the difference between what scales and what's *needed*.

Keep the **`keys` table in a relational DB** regardless — `FOR UPDATE SKIP LOCKED` gives you exactly the atomic block-allocation primitive you need, and the KGS write volume is tiny.

---

### 6. High-level architecture (Step 5)

```
                          ┌──────────┐
                          │  Client  │
                          │(browser) │
                          └────┬─────┘
                               │  1. sho.rt/aX9kP2q
                               ▼
                          ┌──────────┐
                          │   DNS    │  resolves sho.rt → LB's anycast IP
                          └────┬─────┘
                               ▼
                     ┌───────────────────┐
                     │   Load Balancer   │  TLS termination, health checks
                     │   (L7, anycast)   │  routes by path
                     └────┬─────────┬────┘
              GET /{code} │         │ POST /api/v1/urls
             (READ PATH)  │         │ (WRITE PATH)
                          ▼         ▼
        ┌──────────────────────┐  ┌──────────────────────┐
        │  Redirect Servers    │  │    API Servers       │
        │  (stateless, N=20)   │  │  (stateless, N=3)    │
        │  scaled for 12k QPS  │  │  scaled for 120 QPS  │
        └────┬────────────┬────┘  └───┬──────────────┬───┘
             │            │           │              │
   2. GET    │            │  6. fire  │ b. take key  │ c. INSERT
      code   │            │   click   │              │
             ▼            │   event   ▼              │
     ┌───────────────┐    │      ┌─────────────┐     │
     │  Redis Cache  │    │      │     KGS     │     │
     │  ~35 GB       │    │      │ (replicated)│     │
     │  LRU, TTL 24h │    │      │  keys table │     │
     │  cache-aside  │    │      └─────────────┘     │
     └───────┬───────┘    │                          │
    3. MISS  │            │                          │
             ▼            │                          ▼
     ┌───────────────────────────────────────────────────────┐
     │            urls  DB  (Cassandra / DynamoDB)           │
     │   sharded by hash(short_code), RF=3, multi-AZ         │
     │  ┌─────────┬─────────┬─────────┬─────────┐            │
     │  │ Shard 0 │ Shard 1 │ Shard 2 │ Shard 3 │  …         │
     │  └─────────┴─────────┴─────────┴─────────┘            │
     └───────────────────────────────────────────────────────┘
                          │
                          │  (READ PATH ENDS: 302 back to client)
                          │
             ┌────────────┴───────────────────────────┐
             │        ASYNC ANALYTICS PIPELINE        │
             │  (fully off the redirect's hot path)   │
             └────────────────────────────────────────┘
                          │
     Redirect server ──6──▶ ┌──────────────┐
     (fire-and-forget)      │Kafka / Kinesis│  topic: click-events
                            └───────┬───────┘
                                    ▼
                            ┌───────────────┐
                            │Analytics      │
                            │Consumer       │
                            └───────┬───────┘
                                    ▼
                    ┌──────────────────────────────┐
                    │  Analytics store             │
                    │  (ClickHouse / BigQuery)     │
                    │  clicks(code, ts, ref, geo)  │
                    └──────────────────────────────┘
```

**Walk the WRITE path (creating a link) — 5 steps:**

1. Client sends `POST /api/v1/urls {longUrl}` with an auth token.
2. Load balancer routes it to an **API server** (there are only 3 — we only need 120 QPS of peak write capacity).
3. The API server **validates**: is `longUrl` a well-formed URL? Does it pass the safety check (not on a malware/phishing blocklist)? Is the user under their rate limit?
4. The API server takes a key from its **local in-memory key pool** (refilled in blocks from the KGS). No network call in the common case.
5. It **inserts** `{short_code, long_url, user_id, created_at, expires_at}` into the DB, and returns `201` with the short URL.

Note what is *not* here: no collision retry loop, no locking, no coordination. The write path is a single DB insert. That's the payoff from choosing the KGS.

**Walk the READ path (the redirect) — 6 steps. This is the path that matters:**

1. Browser requests `GET /aX9kP2q`.
2. LB routes to a **Redirect server** (stateless — any of the 20 will do).
3. The server checks **Redis**: `GET url:aX9kP2q`.
   - **HIT (~90%+ of the time):** it has the long URL. Skip to step 5. Total latency: ~1–2 ms.
   - **MISS:** go to step 4.
4. Read the row from the DB by primary key (~5–10 ms), then **write it back into Redis** with a TTL (this is **cache-aside**).
5. Check `expires_at`. If expired, return `410 Gone`. Otherwise return **`302 Found`** with `Location: <longUrl>`.
6. **Fire the click event onto Kafka — fire-and-forget, do NOT await it.** The response has already been sent. This is the design point most candidates miss: **the analytics write must never be on the critical path of the redirect.** If you `await` a database write to increment a click counter, you've just added 10 ms to every redirect *and* coupled your uptime to the analytics DB's uptime. If Kafka is down, you drop some click events — and that is completely fine, because nobody's link broke.

#### The redirect handler in Node.js

```js
// redirect-server.js
import express from 'express';
import { createClient } from 'redis';

const app = express();
const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

const CACHE_TTL_SECONDS = 24 * 60 * 60; // 24h — hot links stay hot
const NEGATIVE_TTL_SECONDS = 60;        // cache 404s briefly (see: cache penetration)

app.get('/:shortCode', async (req, res) => {
  const { shortCode } = req.params;

  // Cheap validation first — reject garbage before it ever touches Redis or the DB.
  if (!/^[0-9a-zA-Z]{1,10}$/.test(shortCode)) {
    return res.status(404).send('Not found');
  }

  try {
    // ---- 1. Cache-aside: try the cache first ----
    const cached = await redis.get(`url:${shortCode}`);

    if (cached === '__MISSING__') {
      // Negative cache hit: we already know this code doesn't exist.
      // Without this, an attacker hammering random codes bypasses the cache
      // entirely and DDoSes the database. This is "cache penetration".
      return res.status(404).send('Not found');
    }

    if (cached) {
      sendRedirect(res, JSON.parse(cached), shortCode, req);
      return;
    }

    // ---- 2. Cache miss: hit the DB ----
    const row = await db.getByShortCode(shortCode); // single PK lookup

    if (!row) {
      await redis.set(`url:${shortCode}`, '__MISSING__', { EX: NEGATIVE_TTL_SECONDS });
      return res.status(404).send('Not found');
    }

    // ---- 3. Populate the cache for next time ----
    await redis.set(
      `url:${shortCode}`,
      JSON.stringify({ longUrl: row.long_url, expiresAt: row.expires_at }),
      { EX: CACHE_TTL_SECONDS }
    );

    sendRedirect(res, row, shortCode, req);
  } catch (err) {
    // If Redis is down, we must NOT fail the redirect — fall through to the DB.
    // Availability (N3) beats efficiency. Degrade, don't die.
    console.error('redis/db error', err);
    const row = await db.getByShortCode(shortCode).catch(() => null);
    if (!row) return res.status(404).send('Not found');
    sendRedirect(res, row, shortCode, req);
  }
});

function sendRedirect(res, record, shortCode, req) {
  const longUrl = record.longUrl ?? record.long_url;
  const expiresAt = record.expiresAt ?? record.expires_at;

  // Lazy expiry: we check on read instead of running a sweeper over 6B rows.
  if (expiresAt && new Date(expiresAt) < new Date()) {
    return res.status(410).send('This link has expired');
  }

  // 302, not 301: we must be able to revoke malicious links and count clicks.
  res.redirect(302, longUrl);

  // Fire-and-forget AFTER the response is sent. Nothing below this line
  // can slow down or break the redirect the user already received.
  emitClickEvent({
    shortCode,
    ts: Date.now(),
    referrer: req.get('referer') ?? null,
    ua: req.get('user-agent') ?? null,
    ip: req.ip, // the consumer turns this into a country, then discards it
  }).catch((e) => console.error('analytics emit failed (non-fatal)', e));
}

app.listen(3000);
```

---

### 7. Deep dives

#### (a) Caching the hot links

**Why a cache works so well here:** link popularity follows a **power law** (a.k.a. Zipf distribution). A handful of links — the one in a viral tweet, the one in a marketing email blasted to 10M people — get millions of clicks, while the vast majority get 3 clicks and die. This means a **small cache captures a huge fraction of traffic.** Our estimate said 35 GB for the hot 20%; in practice you'll see 90%+ hit rates from far less.

- **Strategy: cache-aside** (also called lazy loading). The app checks the cache; on a miss it reads the DB and writes the result back. Simple, and the cache only ever holds data someone actually asked for. Recall [59 — Caching in Depth](./59-caching-in-depth.md) for the alternatives (read-through, write-through).
- **Eviction: LRU.** When Redis hits its memory limit (`maxmemory-policy allkeys-lru`), it evicts whatever hasn't been touched recently. That's exactly right for a power-law workload: cold links fall out, hot links stay resident.
- **TTL: 24 hours.** Even hot links go cold eventually, and a TTL bounds the damage from any stale entry.
- **Should you cache on write?** Tempting, but no — most created links are never clicked. You'd fill your cache with junk. Let the read path decide what's hot.

#### (b) Scaling the database with sharding

At 3 TB and 12,000 peak reads/s you can survive on one primary + replicas. At 100x that, you can't. **Shard by `hash(short_code)`.**

Recall [64 — Database Sharding](./64-database-sharding.md): you split the data across N machines by a **shard key**, and every query must be answerable from a single shard.

- **Shard key: `short_code`.** Perfect choice, because *every single query we make is a lookup by `short_code`*. There are no cross-shard queries. Ever. This is the dream scenario for sharding — most systems aren't this lucky.
- **Use consistent hashing** (recall [74 — Consistent Hashing](./74-consistent-hashing.md)) rather than `hash(code) % N`. With modulo, adding a 5th shard to a 4-shard cluster remaps ~80% of your keys and forces a catastrophic data migration. With consistent hashing, adding a shard moves only ~1/N of the keys.
- **Why not shard by `user_id`?** Because the redirect — the 99% path — doesn't know the user. You'd have to query every shard. Always shard by what your hot query filters on.
- Cassandra and DynamoDB do all of this for you: the partition key *is* the shard key, and they hash it internally.

#### (c) Handling custom aliases

Custom aliases (`sho.rt/my-blog`) bypass the KGS entirely — the user is supplying the key. Two problems:

1. **Uniqueness must be checked atomically.** Do NOT do `if (await exists(alias)) throw; else await insert(alias);` — that's a **TOCTOU race**: two users can both pass the `exists` check before either inserts. Let the **database's primary key constraint** be the arbiter:

```js
async function createCustomAlias(alias, longUrl, userId) {
  // Reserved words must never become aliases, or someone claims /api or /admin.
  const RESERVED = new Set(['api', 'admin', 'login', 'static', 'health']);
  if (RESERVED.has(alias.toLowerCase())) {
    throw new HttpError(400, 'That alias is reserved');
  }
  if (!/^[a-zA-Z0-9_-]{3,32}$/.test(alias)) {
    throw new HttpError(400, 'Alias must be 3-32 alphanumeric characters');
  }

  try {
    // The PK constraint does the locking for us. One INSERT wins, one throws.
    await db.insert({ short_code: alias, long_url: longUrl, user_id: userId, is_custom: true });
    return alias;
  } catch (err) {
    if (err.code === '23505') {            // Postgres unique_violation
      throw new HttpError(409, 'Alias already taken');
    }
    throw err;
  }
}
```

On DynamoDB the equivalent is a conditional write: `PutItem` with `ConditionExpression: 'attribute_not_exists(short_code)'`. On Cassandra it's a lightweight transaction: `INSERT ... IF NOT EXISTS` (which uses Paxos and is slow — but custom aliases are rare, so paying for consensus on that path is fine).

2. **Collision with generated keys.** A custom alias could theoretically match a key sitting unused in the KGS pool. Simplest fix: **make custom aliases structurally distinct** — require them to be at least 8 characters, or contain a `-`, while generated keys are always exactly 7 alphanumeric chars. Then the two namespaces can never overlap and you need no coordination at all. (Design the problem away rather than solving it — that's a senior move.)

#### (d) Expiry and cleanup

You have 6 billion rows and some of them have an `expires_at`. Two approaches:

| | **Lazy deletion (on read)** | **Background sweeper** |
|---|---|---|
| How | On every redirect, compare `expires_at` to now. If past, return `410` and (optionally) delete the row. | A cron job periodically scans for `expires_at < now()` and deletes in batches. |
| Cost | Free — you were reading the row anyway. | Expensive — a full scan over billions of rows. |
| Problem | Expired rows that nobody clicks live forever, wasting storage. | Load spikes; scanning a KV store by a non-key column is painful or impossible. |

**Use both, weighted toward lazy:**
- **Lazy deletion is the correctness mechanism.** It guarantees an expired link never redirects. It costs nothing. This is what the `sendRedirect` function above does.
- **A slow background sweeper is the housekeeping mechanism**, run at low priority off a replica, purely to reclaim storage. It doesn't need to be timely.
- **Better still: let the database do it.** DynamoDB has a native TTL attribute; Cassandra has per-row TTLs (`INSERT ... USING TTL 2592000`). Set the TTL at write time and the storage engine reclaims the row for you during compaction, for free. **Say this** — it shows you know your tools.
- Set the **Redis TTL shorter than the link's expiry** so a cached entry can't outlive the link it points to.

#### (e) Analytics without slowing the redirect

The requirement is click analytics. The constraint is a sub-100ms redirect. These fight each other, and **the redirect wins.**

**The wrong design (do not do this):**
```js
app.get('/:code', async (req, res) => {
  const row = await db.getByShortCode(req.params.code);
  await db.query('UPDATE urls SET click_count = click_count + 1 WHERE short_code = $1', [req.params.code]); // ← NO
  res.redirect(302, row.long_url);
});
```
Three things are wrong: (1) you added a synchronous DB **write** to the hottest read path in the system, doubling its latency; (2) that write **contends on exactly the rows that are most popular** — a viral link means thousands of concurrent updates to one row, which will lock up; (3) if the analytics DB is slow or down, **every redirect breaks**. You've coupled a critical path to a non-critical system.

**The right design:** the redirect server pushes a lightweight event onto a **message queue (Kafka/Kinesis)** after the response is already sent, and never waits for it.

```js
// analytics.js
import { Kafka } from 'kafkajs';

const producer = new Kafka({ brokers: [process.env.KAFKA_BROKERS] }).producer();
await producer.connect();

export async function emitClickEvent(event) {
  // acks: 0 — we do not wait for broker acknowledgement. A click event is not
  // worth a millisecond of a user's redirect. Losing 0.01% of events is fine;
  // adding 5ms to 4,000 redirects/sec is not.
  await producer.send({
    topic: 'click-events',
    acks: 0,
    messages: [{ key: event.shortCode, value: JSON.stringify(event) }],
  });
}
```

Downstream, a consumer batches events and writes them into an **OLAP store** (ClickHouse, BigQuery, Redshift) designed for aggregation. Counts are computed there, not in the hot table.

**Why this is the important design point:** you have decoupled a *nice-to-have* (analytics, F5) from a *must-have* (the redirect, F2 + N2 + N3). The analytics pipeline can be down for an hour and no user notices. That is what "async" actually buys you — not speed, but **blast-radius containment.**

---

### 8. Bottlenecks, failure modes, and how to scale further

**What breaks at 100x (400,000 reads/s, 10 billion new URLs/year)?**

| Component | Breaks at 100x? | Fix |
|---|---|---|
| Redirect servers | No — stateless | Just add more behind the LB. Autoscale on CPU. |
| Redis cache | Yes — 35 GB → 3.5 TB working set | **Shard Redis** (Redis Cluster) with consistent hashing. Add a geo-distributed layer: Redis in each region. |
| Database | Yes | Shard by `hash(short_code)` (deep dive (b)). Reads scale linearly with shards. |
| KGS | Maybe — 4,000 writes/s of key allocation | Increase the block size (10k instead of 1k) so KGS calls become 10x rarer. Pre-generate more keys offline. |
| Analytics pipeline | Yes — 400k events/s | Kafka scales horizontally by partition. Partition by `shortCode`. |
| Load balancer | Yes | Multiple LBs behind DNS round-robin / anycast; **put a CDN in front** — see below. |

**Failure mode 1 — Cache stampede on a viral link.**
A celebrity tweets a link. It's not in the cache. **10,000 requests arrive for the same missing key in the same 50ms**, all miss the cache, and all hammer the database for the same row simultaneously. The DB falls over, and now *every* link is broken because of one popular one.

**Fixes (name at least two):**
- **Request coalescing / single-flight:** the first request to miss takes a short-lived lock (`SET lock:code NX EX 5`); the other 9,999 wait a few ms and re-check the cache. Only one DB query happens.
- **Probabilistic early expiration:** refresh a hot key *before* its TTL fires, so it never expires under load.
- Because our data is immutable (a short code's target never changes), you can also just use a very long TTL and let the LRU handle eviction.

**Failure mode 2 — The KGS dies.**
No new links can be created. **Impact: writes stop; reads are completely unaffected.** That's exactly the right blast radius. Mitigations: (a) each API server holds a local block of 1000 keys, so it survives a KGS outage for minutes; (b) replicate the KGS across instances with a replicated key store; (c) as a last-resort fallback, generate a random 8-char code and use an atomic conditional insert — slower, but the site keeps taking writes.

**Failure mode 3 — The database is a SPOF.**
A single primary means one disk failure breaks every link ever created (N3 violated catastrophically). Mitigations: **replication factor of 3 across at least 3 availability zones**; automated failover; **read replicas** so a primary failure doesn't stop redirects; multi-region replicas for both latency and disaster recovery. And because the data is effectively **immutable and append-only**, replication lag is harmless — a replica can never serve a *wrong* answer, only a *missing* one for a brand-new link.

**Failure mode 4 — Abuse. This is a real, serious problem, not a footnote.**
URL shorteners are a spammer's favourite tool: they **hide** the destination. `sho.rt/aX9kP2q` could point to a phishing page that steals bank credentials, and the user has no way to tell before clicking. If you don't handle this, your domain gets blocklisted by Google Safe Browsing and Gmail, and then **every link your service ever created stops working.** Abuse is an availability threat.

- **Rate limiting** on `POST /api/v1/urls` — per user, per IP, per API key. A token-bucket limiter at the API gateway (e.g. 10 links/min per anonymous IP, 1000/day per authenticated user).
- **URL safety checking at write time** — check the submitted `longUrl` against the **Google Safe Browsing API** or a similar threat-intel feed before accepting it. Reject known-bad domains with a `400`.
- **Async re-scanning** — a link can be clean at creation and turn malicious a week later (attackers do this deliberately). Re-scan popular links periodically from a background job.
- **A kill switch** — you must be able to disable a `short_code` instantly. Write a `blocked` flag, purge it from the cache. **This is why you use 302, not 301** — with a 301 you fundamentally cannot revoke a link that browsers have cached.
- **An interstitial warning page** for links flagged as suspicious, instead of an immediate redirect.

**Scaling beyond that — the last 10%:**
- **Geo-distribution.** A redirect is one of the most latency-sensitive things on the internet. Put redirect servers + Redis in every region and route with GeoDNS or anycast.
- **CDN the redirect itself.** Because the mapping is immutable, you can serve the 302 from a CDN edge (Cloudflare Workers / Lambda@Edge with a KV store at the edge). Now the request never reaches your origin at all — this collapses your read load by an order of magnitude and gets you sub-20ms redirects worldwide. It's the endgame of this design.

---

## Visual / Diagram description

The big architecture diagram is in **Key concept 6** above — that's the one to memorize and be able to redraw from scratch. Two more diagrams worth internalizing:

**Diagram 2 — The read path, with latency budget.** Draw this and put numbers on every hop:

```
 Browser
   │
   │  ~20ms  (network, DNS cached)
   ▼
 ┌────────────────┐
 │ Load Balancer  │   ~1ms
 └───────┬────────┘
         ▼
 ┌────────────────┐
 │Redirect Server │   ~0.5ms  (stateless, no work)
 └───────┬────────┘
         │
         ├─── 90% ──▶ ┌──────────┐  ~1ms      ┐
         │            │  Redis   │            │  TOTAL (hit):   ~23ms
         │            └──────────┘            ┘
         │
         └─── 10% ──▶ ┌──────────┐  ~8ms      ┐
                      │    DB    │            │  TOTAL (miss):  ~30ms
                      └────┬─────┘            ┘
                           │ write back to Redis (~1ms)
                           ▼
         ┌──────────────────────────────────┐
         │ 302 Found + Location: <longUrl>  │  ◀── response sent HERE
         └──────────────────────────────────┘
                           │
                           ▼  (AFTER the response — costs the user 0ms)
                    ┌─────────────┐
                    │Kafka (acks:0)│
                    └─────────────┘
```

The point of drawing it this way: **the p99 budget is 100ms and we're spending ~30ms, most of it on network.** We have enormous headroom — which is exactly why we can afford a 302 on every click. Also notice that the analytics emit sits *below the response line*. Draw that line. It's the whole lesson.

**Diagram 3 — Key generation, at a glance:**

```
  OFFLINE (no user is waiting)          ONLINE (a user IS waiting)
  ────────────────────────────          ──────────────────────────
  random(7 chars) ──▶ dedupe ──▶ keys table ──block of 1000──▶ API server
                                                                  │
                                                            in-memory array
                                                                  │
                                                            .pop()  ← O(1),
                                                                      zero network
```

The insight the diagram encodes: **all the hard work happens where no user can feel it.** That is the central trick of the KGS, and it generalizes to a hundred other systems.

---

## Real world examples

### Bitly

Bitly is the archetype — billions of links, ~10 billion clicks per month. Publicly, they've described a system built around a key-value store (they've written about using a Redis-fronted, sharded backend) rather than a relational database, precisely because the workload is a pure key lookup. They also invest heavily in **spam and malware detection** on shortened links, since being blocklisted would break every link they've ever issued. And they serve **302s** for tracked links — because click analytics *is* their product, and a 301 would make it impossible.

### Twitter's `t.co`

Twitter wraps every link posted on the platform in a `t.co` short link. The design goals are revealing: it isn't primarily about saving characters — it's about **control and safety**. Because every outbound click passes through `t.co`, Twitter can check destinations against malware lists and **kill a malicious link after it has already been posted and shared millions of times.** That is only possible because the redirect is a live server-side lookup (a 301 would have burned the redirect into every browser cache). It's the single best real-world argument for the 302 recommendation above.

### Conceptual: internal shorteners (`go/links`)

Many large companies run an internal shortener where `go/payroll` resolves to an internal HR system. This is the same architecture with the non-functional requirements inverted: traffic is tiny, **custom aliases are the whole point** (nobody wants `go/aX9kP2q`), and guessability doesn't matter because the service is behind the corporate network. Notice how the *same* system design produces a completely different set of choices — you'd use a global counter or just the alias itself, a single Postgres box, and no KGS at all. **The requirements, not the product name, drive the design.**

---

## Trade-offs

**Short code generation:**

| Approach | Pros | Cons |
|---|---|---|
| Hash + truncate | Stateless, no coordination, deterministic | Collisions require a check-and-retry on every write; same URL → same code breaks per-user features |
| Global counter | Zero collisions, shortest possible codes | **Sequential = enumerable (security hole)**; single point of contention and failure |
| **KGS (key blocks)** | **Zero collisions, random/unguessable, zero per-request coordination, work is offline** | **An extra service to build and operate; it's a SPOF (must replicate); you waste a block of keys when a server dies** |
| Snowflake IDs | Zero collisions, no coordination at all | 11-char codes — too long for a *short*ener; timestamp prefix is partially guessable |

**301 vs 302:**

| | Pros | Cons |
|---|---|---|
| **301 Permanent** | Huge traffic reduction (browser caches forever); passes SEO link equity | Analytics destroyed; **cannot revoke a malicious link**; cannot change the target, ever |
| **302 Temporary** | Full analytics; full control; instant revocation | Every click hits your infrastructure (but our math says we can afford it) |

**Database choice:**

| | Pros | Cons |
|---|---|---|
| KV store (Cassandra/DynamoDB) | Scales horizontally for free; built for single-key lookups; multi-DC replication; native TTLs | Weaker consistency (makes atomic custom-alias uniqueness harder); more operational surface |
| Relational (Postgres/MySQL) | Simple, strong consistency, easy uniqueness constraints, plenty fast for 3 TB | Vertical scaling ceiling; you must build sharding yourself when you outgrow it |

**Rule of thumb:** **In a URL shortener, the read path is sacred.** Every design decision — 302 over a synchronous analytics write, cache-aside over write-through, async Kafka over a `click_count` UPDATE, KGS over collision retries — is you protecting the redirect. If a proposed feature would add a millisecond or a dependency to `GET /{shortCode}`, the answer is no. Push it off the hot path or don't do it.

---

## Common interview questions on this topic

### Q1: "How do you generate the short code, and how do you guarantee no collisions?"

**Hint:** Walk all four approaches (hash+truncate, counter, KGS, Snowflake) with their trade-offs, then commit to the **KGS**: pre-generate random unique 7-char base62 keys offline, hand them to app servers in blocks of ~1000, mark them used atomically (`UPDATE … FOR UPDATE SKIP LOCKED … RETURNING`). Collisions are impossible because uniqueness was established offline at generation time. Proactively raise the three follow-ups: the KGS is a SPOF (replicate it), keys must be marked used atomically (show the SQL), and a dying server wastes its block (that's fine — 1000 keys out of 3.5 trillion).

### Q2: "Should the redirect be a 301 or a 302? Why?"

**Hint:** **302.** A 301 gets cached by the browser essentially forever, which (a) destroys your click analytics because repeat clicks never reach you, and (b) — the killer — **means you can never revoke a link**, so if a spammer's link turns out to point to malware you are powerless. A 302 means every click hits your server, which costs you traffic but buys you analytics, control, and the ability to kill bad links. Our capacity math (4,000 QPS, 90% cache hit rate) says we can easily afford it. Mention the middle ground: a 302 with a small `Cache-Control: max-age` to get *some* browser caching without losing control forever.

### Q3: "How much storage do you need, and how big should the cache be?"

**Hint:** Do the math out loud. 100M URLs/month × 500 bytes = 50 GB/month → 600 GB/year → **3 TB over 5 years**. For the cache, apply 80/20: 4,000 reads/s × 86,400 s = **346M reads/day**; cache the hot 20% → 69M records × 500 B ≈ **35 GB**, which fits in a single Redis node. Then say the important thing: **the data is small; the traffic is the problem.** That reframing is what the question is actually testing.

### Q4: "The database is getting too big for one machine. How do you shard it?"

**Hint:** Shard by **`hash(short_code)`**, because *every* query in the system is a lookup by `short_code` — so every query is answerable from exactly one shard, and there are zero cross-shard queries. Use **consistent hashing** (topic 74), not `hash % N`, so that adding a shard remaps ~1/N of the keys instead of ~80% of them. Explicitly reject sharding by `user_id`: the redirect path doesn't know the user, so you'd have to scatter-gather across every shard on the hottest path in the system.

### Q5: "How do you track click analytics without slowing down the redirect?"

**Hint:** You **never** write to the database on the redirect path. Send the response first (`res.redirect(302, …)`), then **fire the click event onto Kafka as fire-and-forget** (`acks: 0`, no `await` blocking the response). A consumer aggregates into an OLAP store like ClickHouse. The reasoning matters more than the components: this decouples a nice-to-have (analytics) from a must-have (the redirect), so the analytics pipeline can be completely down and no user notices. Call out the anti-pattern explicitly — `UPDATE urls SET click_count = click_count + 1` on every redirect adds latency *and* creates brutal row-level contention on exactly your most popular links.

### Q6: "A link goes viral and it's not in the cache. What happens?"

**Hint:** **Cache stampede** (thundering herd): thousands of simultaneous requests all miss the cache for the same key and all slam the DB with the identical query, potentially taking it down — and taking every other link with it. Fix with **request coalescing / single-flight**: the first request that misses grabs a short Redis lock (`SET lock:code NX EX 5`) and does the DB read; the rest wait ~10ms and re-check the cache. Add probabilistic early expiration for hot keys. Bonus point: our data is immutable, so we can use very long TTLs and let LRU handle eviction.

---

## Practice exercise

### Design `snip.ly` — and defend three decisions

**Time: ~40 minutes. Produce a single page (paper or a text file) with four deliverables.**

You're designing a shortener for a marketing company. New constraints, deliberately different from the ones above:

- **500 million new links per month** (5x our number)
- Read:write ratio is **200:1** (their links get blasted out in email campaigns)
- **Every link has a mandatory 90-day expiry**
- **Custom aliases are a paid premium feature** and must work reliably
- Marketing clients demand **real-time click dashboards** (updated within ~1 minute)

**Deliverable 1 — Capacity estimation (show every line of arithmetic):**
- Write QPS, read QPS, and peak QPS at 3x
- Storage over 5 years (remember: links expire at 90 days — does that change your answer? Think carefully about steady-state vs cumulative)
- Cache size using the 80/20 rule
- One sentence: what do these numbers mean architecturally?

**Deliverable 2 — Short code length.** Given 500M/month with a 90-day expiry, how many *live* codes exist at steady state? Can you now safely use **6 characters** instead of 7? Compute the keyspace density and argue yes or no. (Careful: can you reuse an expired code? What breaks if you do?)

**Deliverable 3 — The architecture diagram.** Draw it with boxes and arrows, exactly like the one in Key concept 6. It must include the load balancer, the read path, the write path, the cache, the DB, the KGS, and the analytics pipeline. Label every arrow with what flows across it.

**Deliverable 4 — Defend three decisions in writing (3–4 sentences each):**
1. **301 or 302?** These are marketing links whose click counts *are the product*. Does that settle it? What would you tell a client who complains that the redirect adds 30ms?
2. **Which database, and why?** Justify against the actual access pattern, not against a brand name. Would you still say the same thing for a 10-person startup?
3. **Real-time dashboards within 1 minute** — is Kafka → ClickHouse enough? Where exactly does the 1-minute budget get spent, and what would you change if the requirement dropped to 1 *second*?

**The point of this exercise:** you should end up making at least one *different* choice than this doc did, because the requirements changed. If your design is identical, you weren't reading the requirements — you were pattern-matching. That is the failure mode the framework in topic 93 exists to prevent.

---

## Quick reference cheat sheet

- **The framework:** Requirements → Estimation → API + Data model → High-level design → Deep dives. In that order, every time. Never jump to the diagram.
- **The defining property:** URL shorteners are **100:1 read:write**. Design the read path first and protect it from everything else.
- **The math to memorize:** 100M/month ÷ 2.59M seconds ≈ **40 writes/s** → ×100 = **4,000 reads/s** → 500 B × 100M × 12 × 5 = **3 TB over 5 years** → 20% of 346M daily reads × 500 B ≈ **35 GB cache**.
- **Base62 sizing:** 62^6 ≈ **56.8 billion**, 62^7 ≈ **3.5 trillion**. Use **7 characters** — it covers 6B links over 5 years while keeping the keyspace 99.8% empty, so codes can't be guessed.
- **The core problem is ID generation**, not storage. Hash+truncate collides. A counter is enumerable and a SPOF. **The KGS is the answer.**
- **KGS in one line:** pre-generate random unique keys offline, hand out **blocks** of 1000 so there's no per-request coordination, mark them used **atomically**, replicate it, and accept that a dying server wastes its block.
- **301 vs 302:** 301 = browser caches forever = free traffic but **no analytics and no way to revoke a malicious link**. **302 = every click hits you = analytics + control.** Choose **302**.
- **Cache-aside + LRU + power-law popularity** = a small cache gets a huge hit rate. A few tens of GB serves 90%+ of 4,000 QPS.
- **Shard by `hash(short_code)`** using **consistent hashing** — every query is a single-key lookup, so there are zero cross-shard queries. Never shard by `user_id`.
- **Never write to the DB on the redirect path.** Send the 302, *then* fire the click event onto Kafka with `acks: 0`. Analytics can be down; redirects cannot.
- **Expiry: lazy deletion on read** for correctness (free), plus a slow background sweeper — or better, use a native DB TTL (DynamoDB TTL, Cassandra `USING TTL`).
- **Custom aliases:** let the **primary-key constraint** enforce uniqueness (catch the conflict), never `check-then-insert` — that's a TOCTOU race. Reserve words like `api` and `admin`.
- **Abuse is an availability risk:** rate limit writes, check URLs against Safe Browsing, re-scan popular links, and keep a kill switch. Get blocklisted and *every link you ever issued* dies.
- **Links are forever.** That single sentence justifies the availability target, the 302, the replication factor, and the kill switch. Say it in the interview.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [93 — The HLD Approach Framework](./93-hld-approach-framework.md) — the 5-step method this doc executes end to end; read it first, then come back and watch it applied |
| **Next** | [97 — Design Pastebin](./97-hld-pastebin.md) — the same skeleton (short key → stored blob) but the payload is large text, which changes the storage layer completely |
| **Related** | [74 — Consistent Hashing](./74-consistent-hashing.md) — how you actually distribute short codes across shards and Redis nodes without a mass remap when you add a machine |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — cache-aside, LRU eviction, TTLs, and the cache-stampede fix that keeps a viral link from taking down your database |
| **Related** | [64 — Database Sharding](./64-database-sharding.md) — choosing `short_code` as the shard key, and why sharding by `user_id` would destroy the redirect path |
