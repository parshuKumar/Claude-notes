# 97 — Design a Pastebin
## Category: HLD Case Study

---

## What is this?

A **pastebin** is a service where you dump a blob of text — a stack trace, a config file, a log excerpt, a snippet of code — and get back a short URL you can share. Anyone with the URL sees the text. That's it. Pastebin.com, GitHub Gist, Hastebin, and the `nc termbin.com 9999` trick all do the same job.

Think of it as a **coat check for text**. You hand over a coat (your text), you get a numbered ticket (a short URL), and anyone holding that ticket can collect the coat. The ticket is small and easy to pass around; the coat is bulky and stays in the back room.

That "the ticket is small, the coat is bulky" split is the entire design lesson of this document.

---

## Why does it matter?

On the surface a pastebin looks like a URL shortener with a bigger payload. It isn't — and the difference is exactly what interviewers are probing for.

A URL shortener stores **~100 bytes** per record. A pastebin stores **up to 10 MB** per record. That single change of scale flips a design decision that most candidates get wrong: **where does the payload live?** If you say "a TEXT column in Postgres" and move on, you have just failed the hardest part of the question without noticing.

**The interview angle:** Pastebin is the standard "level 2" HLD warm-up. It is asked *after* URL shortener because it reuses the ID-generation machinery (so you can say "same as the shortener, see below") and then adds one genuinely new axis: large opaque payloads, blob storage, expiry, and CDN caching. Interviewers use it to test whether you know *when to take data out of the database*.

**The real-work angle:** Every product eventually stores user-uploaded content — avatars, attachments, exports, CSV uploads, generated PDFs. The pastebin decision (metadata in the DB, bytes in object storage) is the exact pattern you will apply, and applying it wrongly is one of the most common causes of a database that becomes slow, expensive, and un-backupable at 2 AM.

---

## The core idea — explained simply

### The Warehouse and the Filing Cabinet

Imagine you run a storage business.

In the front office you have a **filing cabinet**. It holds index cards. Each card is small: an item ID, the shelf number where the item sits, who owns it, when it arrived, when it should be thrown away. You can flip through thousands of cards a second. The cabinet fits in the room and you photocopy it every night as a backup — the photocopy takes two minutes.

Out back you have a **warehouse**. It holds the actual items: boxes, furniture, crates. It is enormous, cheap per square metre, and it never needs to be searched — you never walk into the warehouse and say "find me all the brown boxes." You always arrive holding a shelf number from a card.

Now imagine you decide to skip the warehouse and **stuff the furniture into the filing cabinet**. The cabinet stops closing. Flipping through cards becomes slow because you have to shove a sofa aside to reach card #4,182. Your nightly photocopy now takes six hours because you're photographing furniture. And the cabinet — a precision, climate-controlled, expensive piece of equipment — is now being used as a shed.

That's what putting 10 MB pastes in a relational database does.

| Warehouse analogy | Pastebin reality |
|---|---|
| Filing cabinet | Postgres / MySQL `pastes` metadata table |
| Index card | One row: `paste_id`, `s3_key`, `expires_at`, `size_bytes` |
| Shelf number on the card | The S3 object key |
| Warehouse | S3 / GCS / Azure Blob object storage |
| The furniture | The paste content itself (up to 10 MB) |
| Photocopying the cabinet nightly | Your DB backup |
| "Find me all brown boxes" — never asked | You never query *inside* paste content |
| Cabinet jamming, photocopy taking 6h | Buffer-pool thrash, 3.6 TB/yr DB, slow restores |

The whole architecture falls out of one question: **does anything ever need to search, sort, or filter by the paste body?** No. Nobody queries `WHERE content LIKE '%password%'`. The body is **opaque bytes fetched by key**. And opaque bytes fetched by key is the literal definition of the workload object storage was invented for (recall [78 — Blob Storage](./78-blob-storage.md)).

Keep in the database only what you **query, filter, sort, or join on**. Everything else is a coat, and coats go in the back room.

---

## Key concepts inside this topic

We'll run the 5-step framework from [93 — HLD Approach Framework](./93-hld-approach-framework.md), visibly, in order: **Requirements → Estimation → API → Data model → Architecture → Deep dives → Bottlenecks.**

---

### 1. Requirements

**Clarifying questions to ask the interviewer first** (never start drawing boxes — recall topic 93):

1. Is the paste **text only**, or arbitrary files? *(Assume text. Files turn this into Dropbox.)*
2. What's the **max paste size**? *(Assume 10 MB — generous, covers a big log dump.)*
3. Can pastes be **edited** after creation? *(Assume NO. This is the single biggest simplification — see below.)*
4. Do we need **user accounts**? *(Assume optional: anonymous pastes allowed, logged-in users can list their own.)*
5. Do we need **syntax highlighting / analytics / search inside pastes**? *(Highlighting is client-side. Full-text search is explicitly out of scope — say so, it saves you an Elasticsearch cluster.)*
6. What **scale**? *(Assume 1M new pastes/day.)*

**Functional requirements:**

| # | Requirement |
|---|---|
| F1 | Create a paste from a block of text; get back a unique short URL |
| F2 | View a paste by its URL (raw text and rendered HTML views) |
| F3 | Optional expiry: `10 minutes`, `1 day`, `never` (extendable to a set of presets) |
| F4 | Optional custom URL (`/my-config`) if not already taken |
| F5 | Optional visibility: `public` (listed/searchable), `unlisted` (only via URL), `private` (owner only) |
| F6 | Delete a paste (owner only) |

**Non-functional requirements:**

| # | Requirement | Why it shapes the design |
|---|---|---|
| N1 | **Read-heavy, ~10:1** | Reads dominate → optimize the read path; caches and CDN, not write throughput |
| N2 | **Low read latency** (p99 < 200 ms) | Content should come from a cache/CDN edge, not a cold S3 GET every time |
| N3 | **High availability** on reads | A shared paste that 500s is worse than a slow one. Reads must survive DB hiccups |
| N4 | **Pastes are immutable** once created | See below — this is load-bearing |
| N5 | **Size limit: 10 MB** per paste | Bounds every downstream number; enforce at the edge, not after buffering |
| N6 | Durability of stored pastes | 11 nines from S3; we don't build our own replication |

**Why immutability changes everything.** Declare `N4` loudly in the interview. Because a paste can never change after it is written:

- **There is no update path.** No `PUT /pastes/{id}`, no read-modify-write, no optimistic locking, no version column, no lost-update race. An entire class of concurrency bugs simply does not exist.
- **You can cache forever.** A cached copy of paste `xK9mQ2` can never be stale, because `xK9mQ2` can never change. `Cache-Control: public, max-age=31536000, immutable` — a full year — is *provably correct*, not a gamble. No invalidation logic, no TTL tuning, no cache-stampede-after-update.
- **CDN edges are free wins.** A viral paste gets pulled to 300 edge locations once and served from there billions of times. Your origin sees ~300 requests. (Recall [60 — CDN](./60-cdn.md).)
- **Replication lag stops mattering for content.** A read replica can never serve a *stale version* of a paste — the only two states are "row exists" and "row doesn't exist yet." Worst case a brand-new paste 404s for 50 ms.

Whenever you can turn "mutable" into "immutable + create a new one," take the deal. It is the cheapest architectural win in this entire curriculum.

---

### 2. Capacity estimation

Show every line. Never hand-wave a number in an interview — derive it.

**Writes**

```
New pastes per day               = 1,000,000
Seconds in a day                 = 24 × 3600 = 86,400
Write QPS (average)              = 1,000,000 / 86,400
                                 ≈ 11.6  →  ~12 writes/sec
Peak write QPS (assume 3× avg)   ≈ 36 writes/sec
```

**Reads**

```
Read:Write ratio                 = 10 : 1
Reads per day                    = 10,000,000
Read QPS (average)               = 10,000,000 / 86,400
                                 ≈ 115.7  →  ~120 reads/sec
Peak read QPS (3× avg)           ≈ 360 reads/sec
```

**Be honest about what this means.** A modern Node.js API server handles a few thousand simple requests per second. **360 peak reads/sec is nothing.** Two or three API boxes behind a load balancer, one Postgres primary, one Redis — done. This is a genuinely *modest* system.

Contrast with [94 — URL Shortener](./94-hld-url-shortener.md): the same 1M new items/day, but a shortened link gets clicked *hundreds* of times (it's pasted into tweets, emails, ads), so a shortener at this creation rate runs at ~10,000+ reads/sec. A paste is typically read by **ten people** — the person you sent it to, and a few teammates. **Fewer reads per item = a much smaller system.** Saying this out loud is a strong signal: it shows you derive scale rather than cargo-culting "we'll need Cassandra."

The pressure in pastebin is **not QPS. It's bytes.**

**Storage**

```
Average paste size               = 10 KB       (median is far smaller, ~2 KB;
                                                10 KB average accounts for the
                                                fat tail of log dumps)
Content written per day          = 1,000,000 × 10 KB
                                 = 10,000,000 KB
                                 = 10 GB / day

Per year                         = 10 GB × 365
                                 = 3,650 GB  ≈  3.6 TB / year

Over 5 years (ignoring expiry)   = 3.6 TB × 5  ≈  18 TB
With S3 replication/overhead     ≈ 18–20 TB
```

Now the metadata, which is what actually lives in your database:

```
Metadata row size (paste_id 8B + s3_key 40B + user_id 16B
                   + timestamps 16B + visibility 1B + size 4B
                   + view_count 8B + index/row overhead ≈ 100B)
                                 ≈ 200 bytes / row

Metadata per day                 = 1,000,000 × 200 B = 200 MB / day
Metadata per year                = 200 MB × 365 ≈ 73 GB / year
Metadata over 5 years            ≈ 365 GB
```

**Look at those two numbers side by side. This is the whole doc in one comparison:**

| Where the bytes go | 5-year size | Fits on one Postgres box? |
|---|---|---|
| Paste **content** | **~18 TB** | Absolutely not, and you'd hate yourself |
| Paste **metadata** | **~365 GB** | Yes, comfortably. One primary + replicas |

**Bandwidth**

```
Write bandwidth  = 12 writes/s × 10 KB   = 120 KB/s   ≈ 1 Mbps
Read bandwidth   = 120 reads/s × 10 KB   = 1,200 KB/s ≈ 1.2 MB/s ≈ 10 Mbps
Peak read        ≈ 30 Mbps
```

Trivial for a single load balancer. But note: bandwidth is where the **money** is. Egress from S3 direct to the internet is roughly $0.09/GB, whereas egress *through a CDN* is cheaper and, more importantly, the CDN serves most requests without touching S3 at all. 10 GB/day of reads served at ~90% cache hit ratio means your origin egress drops by ~10×.

**Conclusion from the numbers:** QPS is small — no exotic sharding, no eventual-consistency gymnastics, no Kafka. The design pressure is entirely **storage volume and byte egress**. Therefore the interesting decision is *where the bytes live*. Which is exactly step 3.

---

### 3. The key design decision — where does the TEXT live?

This is the centerpiece. Two options.

#### Option A — content in a database column

```sql
CREATE TABLE pastes (
  paste_id    VARCHAR(12) PRIMARY KEY,
  content     TEXT,              -- ← up to 10 MB lives right here
  expires_at  TIMESTAMPTZ,
  ...
);
```

It works. Day one it's the simplest thing that could possibly work: one write, one read, one transaction, no second system. And for a toy with 10,000 pastes, it's the *right* call — don't over-engineer.

At 1M pastes/day it becomes a slow-motion disaster:

**1. The table bloats.** 3.6 TB/year of content sitting in your primary datastore. Postgres will move large values out-of-line into a TOAST table, which softens (but does not remove) the pain; MySQL with `LONGTEXT` is worse. You are now operating a multi-terabyte OLTP database whose "hot" working set is 0.1% of its size.

**2. It destroys the buffer pool.** This is the killer, and the one candidates miss. Your database keeps a fixed-size in-memory cache of pages (the buffer pool / shared buffers) — say 64 GB. That cache exists to keep **indexes and hot rows** in RAM. When you read one 10 MB paste, you drag ~10 MB of pages through that cache, **evicting index pages to make room**. A handful of large reads per second and your index pages are constantly being flushed and re-read from disk. Your *unrelated* metadata queries — the ones that were 0.5 ms — become 20 ms. One user pasting a giant log file degrades latency for everybody. This is *cache pollution*, and it is why databases and large blobs are a bad marriage (recall [59 — Caching in Depth](./59-caching-in-depth.md): a cache is only useful if the hot set fits).

**3. Backups become enormous and slow.** A 3.6 TB/year database means `pg_dump` runs for hours, WAL archives balloon, replication of a new replica takes a day, and — the part that actually hurts — your **restore time** goes from 20 minutes to many hours. Your recovery objective is now governed by data nobody queries. Meanwhile a 365 GB metadata-only DB restores fast.

**4. Cost per GB is 10–25× higher.** Provisioned SSD storage on a managed DB (RDS gp3 and friends) runs roughly **$0.10–0.125 per GB-month** — and you pay for it again on every replica and every snapshot. S3 Standard is about **$0.023 per GB-month**, S3 Infrequent Access about **$0.0125**, and object storage is single-copy-priced (the replication is included, not billed per replica).

**5. You cannot put a CDN in front of a SQL query.** But you *can* put one in front of an object URL. Option A forces every single read through your API and your database. Option B lets 90% of reads never reach your infrastructure at all.

#### Option B — content in blob storage, metadata in the DB (recommended)

```
Database row (≈200 bytes):
  paste_id = "xK9mQ2vB4nRt"
  s3_key   = "pastes/2026/07/xK9mQ2vB4nRt"
  expires_at, visibility, size_bytes, user_id, view_count

S3 object at that key:  the 10 KB (or 10 MB) of text.
```

The DB holds the **index card**. S3 holds the **furniture**.

#### The cost arithmetic (do this out loud)

Take the 5-year content volume, ~18 TB = ~18,000 GB.

```
Option A — content on managed-DB SSD:
   18,000 GB × $0.10/GB-month                 = $1,800 / month
   × 1 read replica (you need one)            = $3,600 / month
   + snapshot storage (~$0.095/GB-mo, 1 copy) ≈ $1,710 / month
   ------------------------------------------------------------
   ≈ $5,300 / month  (and this is before the latency damage)

Option B — content in S3 Standard:
   18,000 GB × $0.023/GB-month                = $414 / month
   (replication included; no per-replica charge; no snapshot copy)
   metadata DB: 365 GB × $0.10 × 2 copies     ≈ $73 / month
   ------------------------------------------------------------
   ≈ $487 / month

   Add an S3 lifecycle rule moving pastes older than 30 days
   to Infrequent Access ($0.0125/GB-mo) and the storage line
   drops to roughly $230/month.
```

**Roughly a 10× cost difference**, and Option B is *also* the faster, more available, more scalable, easier-to-back-up design. That's rare — most trade-offs cost you something. This one costs you exactly one thing:

**The trade-off: an extra network hop.** Reading a paste in Option A is one query. In Option B it is: query the DB for metadata (~1 ms), then GET the object from S3 (~20–50 ms). You added ~30 ms to the read path.

**How you pay that back:**
- **Cache the content in Redis** for small pastes (< 100 KB, which is ~99% of them). A cache hit removes the S3 hop entirely: ~1 ms.
- **Put the content behind a CDN.** Immutability (N4) means you can cache at the edge for a year. A CDN hit never touches S3 *or* your API.
- **Serve the browser a pre-signed S3/CDN URL** and let the client fetch the bytes directly — your API server never proxies 10 MB through its event loop.

Net: the "extra hop" is paid on a cache miss only, which is a small minority of reads. **Recommend Option B. Every real pastebin does this.**

> **The general rule to state in the interview:** *Databases are for data you query. Object storage is for bytes you retrieve by key.* If you never `WHERE` on it, `ORDER BY` it, or `JOIN` on it, it does not belong in a row.

---

### 4. API design

```
POST /api/pastes
  Body:  { content: string,          // required, ≤ 10 MB
           expiresIn: "10m"|"1d"|"never",   // default "never"
           visibility: "public"|"unlisted"|"private",  // default "unlisted"
           customUrl?: string,        // optional, [a-zA-Z0-9_-]{4,32}
           title?: string }
  201 →  { pasteId: "xK9mQ2vB4nRt",
           url: "https://pste.io/xK9mQ2vB4nRt",
           expiresAt: "2026-07-15T09:12:00Z" }
  400 →  content empty / too large
  409 →  customUrl already taken
  429 →  rate limited

GET /api/pastes/{id}
  200 →  { pasteId, title, content, createdAt, expiresAt,
           sizeBytes, visibility, viewCount }
         // OR, for large pastes:
         { pasteId, ..., contentUrl: "<pre-signed CDN URL>" }
  404 →  not found OR expired (deliberately indistinguishable)
  403 →  private paste, not the owner

GET /api/pastes/{id}/raw
  200 →  text/plain body, served/redirected to CDN

DELETE /api/pastes/{id}
  204 →  deleted (owner only)
  403 →  not the owner
```

**Design notes worth saying aloud:**
- **404 for expired, not 410 Gone.** A `410` confirms "this ID existed" — an information leak that helps an enumeration attacker map the keyspace. Expired, deleted, and never-existed should be indistinguishable.
- `GET` returns either inline `content` (small pastes) or a `contentUrl` (large pastes). Don't stream 10 MB through your Node API when a pre-signed URL lets the browser talk to S3/CDN directly.
- Creation is not idempotent, which is fine — pastes are cheap and duplicates are harmless. If you want idempotency, accept a client-supplied `Idempotency-Key` header.

---

### 5. Data model

```sql
CREATE TABLE pastes (
  paste_id     VARCHAR(12)  PRIMARY KEY,   -- random base62 key, see §6
  s3_key       VARCHAR(128) NOT NULL,      -- "pastes/2026/07/xK9mQ2vB4nRt"
  user_id      UUID         NULL,          -- NULL = anonymous paste
  title        VARCHAR(200) NULL,
  created_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
  expires_at   TIMESTAMPTZ  NULL,          -- NULL = never expires
  visibility   SMALLINT     NOT NULL,      -- 0=public 1=unlisted 2=private
  size_bytes   INTEGER      NOT NULL,
  status       SMALLINT     NOT NULL,      -- 0=PENDING 1=ACTIVE 2=DELETED
  view_count   BIGINT       NOT NULL DEFAULT 0
);

CREATE INDEX idx_expires  ON pastes (expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_user     ON pastes (user_id, created_at DESC);
```

Notice what is **not** here: `content`. That's the point.

**Which database, and why.**

The access pattern is overwhelmingly **`SELECT * FROM pastes WHERE paste_id = ?`** — a single-row lookup on a primary key. Almost any store does this well. So choose on the *secondary* requirements:

| Option | Verdict |
|---|---|
| **Postgres / MySQL** | **Pick this.** 365 GB of metadata over 5 years fits on one box with room to spare. You get transactions (useful for the `customUrl` uniqueness check — a `UNIQUE` constraint gives you a race-free "is this taken?" for free), the `idx_expires` partial index makes the expiry sweeper a cheap range scan, and `idx_user` gives you "list my pastes" without a second system. Add read replicas for the read path. |
| **DynamoDB / Cassandra** | Defensible, and the right answer if the numbers were 100× bigger. But at 12 writes/s and 365 GB, you're buying horizontal scale you do not need and giving up transactions and ad-hoc queries. Interviewers *like* candidates who decline unnecessary NoSQL — but be ready to justify it with the numbers above. |
| **Redis as the source of truth** | No. It's the cache, not the store. Metadata must survive a restart. |

**The honest framing:** "The metadata is small and relational, the access pattern is key lookup plus two simple secondary queries, and the write rate is 12/s. Postgres handles this on a single primary for years. I'll add read replicas because the workload is 10:1 read-heavy, and I'll revisit sharding by `paste_id` hash if we ever 50×."

---

### 6. Unique ID generation

You already did this analysis in [94 — URL Shortener](./94-hld-url-shortener.md). The three candidates were:

1. **Base62 of an auto-increment counter** — shortest possible IDs, guaranteed unique, but *sequential and guessable*.
2. **Hash the content (MD5/SHA) and take a prefix** — deterministic and dedupes identical pastes, but needs collision handling.
3. **A Key Generation Service (KGS)** — pre-generates random unused keys into a table, hands them out in batches. No collisions at request time, one extra service to run.

Don't re-derive it. But there is **one new requirement here that the shortener didn't have**, and it disqualifies option 1 outright:

> **Paste IDs must be UNGUESSABLE.**

An **unlisted** paste's *only* protection is that nobody knows its URL. There is no ACL, no login, no permission check — the URL **is** the password. If your IDs are `1, 2, 3, ...` base62-encoded (`b, c, d, ...`), an attacker writes a 10-line script, walks the keyspace from the beginning, and **downloads every unlisted paste ever created.** People paste AWS keys, `.env` files, database dumps, and private stack traces. Sequential IDs turn your pastebin into a public breach.

So: **random keys, with enough entropy that enumeration is infeasible.**

#### How long must the ID be?

Base62 (`a–z A–Z 0–9`) gives log₂(62) ≈ **5.95 bits per character**.

```
Length 6:   62^6  ≈ 5.7 × 10^10   (~36 bits)
Length 8:   62^8  ≈ 2.2 × 10^14   (~48 bits)
Length 10:  62^10 ≈ 8.4 × 10^17   (~60 bits)
Length 12:  62^12 ≈ 3.2 × 10^21   (~71 bits)
```

Now the attacker's model. Over 5 years we have:

```
Live pastes ≈ 1M/day × 365 × 5 ≈ 1.8 × 10^9  (1.8 billion)
```

The chance a *random guess* hits a real paste is `live_pastes / keyspace` — the **hit density**:

| Length | Keyspace | Hit density | Guesses to find ONE real paste |
|---|---|---|---|
| 6 | 5.7 × 10¹⁰ | 1 in 32 | **3 guesses.** Catastrophic |
| 8 | 2.2 × 10¹⁴ | 1 in 122,000 | ~122k guesses — trivial for a botnet |
| 10 | 8.4 × 10¹⁷ | 1 in 4.7 × 10⁸ | ~470 million guesses |
| 12 | 3.2 × 10²¹ | 1 in 1.8 × 10¹² | **~1.8 trillion guesses** |

Add rate limiting — say an aggressive attacker sustains **1,000 guesses/sec** from a distributed botnet before your WAF notices:

```
Length 12:  1.8 × 10^12 guesses ÷ 1,000/s
          = 1.8 × 10^9 seconds
          ≈ 57 years  ...to find ONE single paste.
```

**Verdict: 12 random base62 characters (~71 bits).** Generate with a CSPRNG — `crypto.randomBytes`, never `Math.random()` (which is seeded, predictable, and has been the root cause of real token-prediction breaches).

Collision probability with 1.8×10⁹ keys in a 3.2×10²¹ space is, by the birthday bound, around `(1.8e9)² / (2 × 3.2e21)` ≈ **0.05%** across the whole 5 years — and the `PRIMARY KEY` constraint catches even that: on a duplicate-key error, generate a new key and retry. One retry, ever, statistically.

```js
import crypto from 'node:crypto';

const ALPHABET = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
const ID_LENGTH = 12; // ~71 bits — see the enumeration math above

// Rejection sampling: 256 % 62 != 0, so naively doing byte % 62 would make the
// first 8 letters ~1.6% more likely than the rest. Tiny bias, but bias in a
// security-relevant keyspace is exactly the thing you don't hand-wave.
export function generatePasteId(length = ID_LENGTH) {
  const max = 256 - (256 % ALPHABET.length); // 248
  let out = '';
  while (out.length < length) {
    for (const byte of crypto.randomBytes(length)) {
      if (byte >= max) continue;          // reject, keeps the distribution uniform
      out += ALPHABET[byte % ALPHABET.length];
      if (out.length === length) break;
    }
  }
  return out;
}
```

**Custom URLs** live in the same keyspace: `POST` with `customUrl: "my-config"` just tries to `INSERT` with `paste_id = 'my-config'`. The `PRIMARY KEY` constraint makes the "is it taken?" check race-free — no `SELECT ... IF NOT EXISTS` two-step, no lock. A unique-violation error becomes a `409 Conflict`. Reserve a denylist (`admin`, `api`, `login`, `about`) so nobody claims a route you need.

---

### 7. High-level architecture

```
                    ┌──────────┐
                    │  Client  │  (browser / curl / CLI)
                    └────┬─────┘
                         │  1. GET /xK9mQ2vB4nRt
                         ▼
                 ┌───────────────┐
                 │      CDN      │  ← ~90% of reads STOP HERE
                 │  (CloudFront) │    Cache-Control: max-age=1y, immutable
                 └───────┬───────┘    (safe ONLY because pastes are immutable)
                         │  cache miss
                         ▼
                 ┌───────────────┐
                 │ Load Balancer │
                 └───────┬───────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     ┌─────────┐   ┌─────────┐    ┌─────────┐
     │  API    │   │  API    │    │  API    │   Node.js / Express
     │ server  │   │ server  │    │ server  │   (stateless, autoscaled)
     └────┬────┘   └────┬────┘    └────┬────┘
          │             │              │
          │      ┌──────┴──────┐       │
          ▼      ▼             ▼       ▼
   ┌────────────────┐   ┌──────────────────┐
   │  Redis Cache   │   │  Metadata DB     │
   │  (hot pastes:  │   │  Postgres        │
   │  metadata +    │   │  primary + 2     │
   │  small bodies) │   │  read replicas   │
   └────────────────┘   └────────┬─────────┘
                                 │  200-byte rows ONLY
                                 │  (paste_id, s3_key, expires_at, …)
                                 │
   ┌─────────────────────────────┴──────────────────────────────┐
   │                                                            │
   ▼                                                            ▼
┌──────────────────────┐                          ┌──────────────────────┐
│   Blob Storage (S3)  │                          │  Expiry Sweeper      │
│   the paste BYTES    │◀── deletes expired ──────│  + Orphan Cleaner    │
│   ~18 TB over 5 yrs  │    objects & orphans     │  (cron, every 5 min) │
│   + lifecycle rules  │                          └──────────────────────┘
└──────────┬───────────┘
           │  origin-fetch on CDN miss (pre-signed URL)
           └────────────────────────────────▶ back to CDN
```

**Read the diagram as two paths.**

**Write path (`POST /api/pastes`) — and the order matters:**

```
1. LB → API server
2. Validate:  size ≤ 10 MB?  content non-empty?  rate limit OK?
3. Generate paste_id (12 random base62 chars)
4. INSERT metadata row with status = PENDING     ← DB first!
5. PUT content to S3 at pastes/YYYY/MM/{paste_id}
6. UPDATE metadata row  SET status = ACTIVE
7. Return { pasteId, url }
```

**Read path (`GET /api/pastes/{id}`):**

```
1. CDN edge — hit? serve, done (~10 ms, never touches you)
2. Miss → LB → API server
3. Redis:  GET paste:meta:{id}    → hit? skip step 4
4. Postgres (read replica): SELECT metadata WHERE paste_id = ?
5. status != ACTIVE, or expires_at < now()?  → 404 (LAZY expiry)
6. visibility = private and requester != owner? → 403
7. Content:
     small paste (<100 KB)?  Redis GET paste:body:{id}
                             → miss → S3 GET → warm Redis
     large paste?            return a pre-signed CDN URL; the
                             client fetches bytes directly (your
                             Node process never buffers 10 MB)
8. Fire-and-forget: increment view count (async, see §9)
9. Respond with Cache-Control: public, max-age=31536000, immutable
```

#### The write-ordering failure case (the detail that wins points)

Two writes happen — one to S3, one to the DB — and **they are not in a single transaction.** There is no distributed transaction between Postgres and S3. So one of them can fail. Which order should you use?

**Order 1 — S3 first, then DB.** If the S3 PUT succeeds and then the DB `INSERT` fails (DB failover, connection pool exhausted, process crashed), you have an **orphaned object**: 10 MB sitting in S3 that no row points to. It is invisible, unreachable, undeletable through the app — and you are **paying for it forever**. Do that a million times and you're renting a landfill. Note the failure is *silent*: nothing is broken from the user's point of view (they got a 500 and retried), so nobody notices until the bill arrives.

**Order 2 — DB first (as `PENDING`), then S3, then confirm `ACTIVE`.** If the S3 PUT fails, you have a `PENDING` row with no object. The read path already ignores anything that isn't `ACTIVE`, so the paste simply 404s — **correct behavior**. And now the garbage is a **200-byte row** you can find with a trivial query, instead of a 10 MB object you can only find by diffing an entire S3 bucket listing against your table.

**Recommend Order 2, plus a sweeper as a backstop.** Both failure modes still need cleanup — you can never fully avoid orphans, because the process can die *between* the S3 PUT and the `ACTIVE` update. So run a background job:

```js
// Runs every 5 minutes. Two cheap queries, both index-backed.
async function cleanupSweep(db, s3) {
  // A) PENDING rows older than 15 minutes: the write never completed.
  //    Delete any object that may exist, then the row. S3 DELETE is
  //    idempotent, so "object was never created" is a no-op, not an error.
  const stale = await db.query(
    `SELECT paste_id, s3_key FROM pastes
      WHERE status = 0 AND created_at < now() - interval '15 minutes'
      LIMIT 1000`
  );
  for (const row of stale.rows) {
    await s3.deleteObject({ Bucket: BUCKET, Key: row.s3_key });
    await db.query('DELETE FROM pastes WHERE paste_id = $1', [row.paste_id]);
  }

  // B) Expired ACTIVE pastes: delete the bytes, tombstone the row.
  //    (Keeping a tiny tombstone row means we never re-issue that ID.)
  const expired = await db.query(
    `SELECT paste_id, s3_key FROM pastes
      WHERE status = 1 AND expires_at IS NOT NULL AND expires_at < now()
      LIMIT 1000`
  );
  for (const row of expired.rows) {
    await s3.deleteObject({ Bucket: BUCKET, Key: row.s3_key });
    await db.query('UPDATE pastes SET status = 2 WHERE paste_id = $1', [row.paste_id]);
  }
}
```

That partial index `idx_expires ... WHERE expires_at IS NOT NULL` is what makes query (B) cheap: it only indexes the minority of pastes that *can* expire, so the sweep is a small range scan, not a full-table crawl over 1.8 billion rows.

#### The create and read handlers (Node.js, with pre-signed S3 URLs)

```js
import express from 'express';
import crypto from 'node:crypto';
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand }
  from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const app = express();
const s3 = new S3Client({ region: 'us-east-1' });
const BUCKET = 'pastes-prod';
const MAX_BYTES = 10 * 1024 * 1024;     // 10 MB — requirement N5
const INLINE_LIMIT = 100 * 1024;        // cache/inline bodies under 100 KB
const CDN_HOST = 'https://cdn.pste.io';

const STATUS = Object.freeze({ PENDING: 0, ACTIVE: 1, DELETED: 2 });
const VISIBILITY = Object.freeze({ PUBLIC: 0, UNLISTED: 1, PRIVATE: 2 });
const TTL = Object.freeze({ '10m': 600, '1d': 86400, never: null });

// Reject oversized bodies at the parser, before we buffer 10 MB in the heap.
app.use(express.json({ limit: '10mb' }));

// ---------------------------------------------------------------- CREATE
app.post('/api/pastes', rateLimit({ perIpPerHour: 20 }), async (req, res) => {
  const { content, expiresIn = 'never', visibility = 'unlisted', customUrl, title } = req.body;

  if (typeof content !== 'string' || content.length === 0)
    return res.status(400).json({ error: 'content_required' });

  const sizeBytes = Buffer.byteLength(content, 'utf8');
  if (sizeBytes > MAX_BYTES)
    return res.status(400).json({ error: 'content_too_large', maxBytes: MAX_BYTES });
  if (!(expiresIn in TTL))
    return res.status(400).json({ error: 'bad_expiry' });

  const pasteId = customUrl ? sanitizeCustomUrl(customUrl) : generatePasteId();
  const now = new Date();
  const ttl = TTL[expiresIn];
  const expiresAt = ttl ? new Date(now.getTime() + ttl * 1000) : null;
  const s3Key = `pastes/${now.getUTCFullYear()}/${String(now.getUTCMonth() + 1).padStart(2, '0')}/${pasteId}`;

  // STEP 1: metadata row FIRST, as PENDING.
  // If we crash after this, the worst artifact is a 200-byte row the
  // sweeper reclaims — not an unreferenced 10 MB object we pay for forever.
  try {
    await db.query(
      `INSERT INTO pastes (paste_id, s3_key, user_id, title, created_at,
                           expires_at, visibility, size_bytes, status)
       VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9)`,
      [pasteId, s3Key, req.userId ?? null, title ?? null, now,
       expiresAt, VISIBILITY[visibility.toUpperCase()], sizeBytes, STATUS.PENDING]
    );
  } catch (err) {
    // The PRIMARY KEY constraint is our race-free uniqueness check for
    // customUrl — no SELECT-then-INSERT window to lose.
    if (err.code === '23505') return res.status(409).json({ error: 'url_taken' });
    throw err;
  }

  // STEP 2: the bytes go to S3.
  await s3.send(new PutObjectCommand({
    Bucket: BUCKET,
    Key: s3Key,
    Body: content,
    ContentType: 'text/plain; charset=utf-8',
    // Objects inherit the paste's own expiry via a tag; an S3 lifecycle
    // rule on this tag deletes them even if our sweeper is down.
    Tagging: expiresIn === 'never' ? 'ttl=none' : `ttl=${expiresIn}`,
  }));

  // STEP 3: only now is the paste readable.
  await db.query('UPDATE pastes SET status = $1 WHERE paste_id = $2',
    [STATUS.ACTIVE, pasteId]);

  res.status(201).json({
    pasteId,
    url: `https://pste.io/${pasteId}`,
    expiresAt: expiresAt?.toISOString() ?? null,
  });
});

// ------------------------------------------------------------------ READ
app.get('/api/pastes/:id', async (req, res) => {
  const { id } = req.params;

  // Layer 1: Redis metadata cache (recall 59 — cache-aside).
  let meta = await redis.get(`paste:meta:${id}`).then(v => v && JSON.parse(v));
  if (!meta) {
    const { rows } = await dbReplica.query(
      'SELECT * FROM pastes WHERE paste_id = $1', [id]);
    meta = rows[0];
    if (meta) await redis.setex(`paste:meta:${id}`, 3600, JSON.stringify(meta));
  }

  // LAZY expiry: never trust the sweeper for correctness. Check on every read.
  // Note we return 404 (not 410) so an attacker can't distinguish
  // "expired" from "never existed" and map the keyspace.
  const gone = !meta
    || meta.status !== STATUS.ACTIVE
    || (meta.expires_at && new Date(meta.expires_at) < new Date());
  if (gone) return res.status(404).json({ error: 'not_found' });

  if (meta.visibility === VISIBILITY.PRIVATE && meta.user_id !== req.userId)
    return res.status(403).json({ error: 'forbidden' });

  // Fire-and-forget view count. Never block the read on analytics (§9).
  redis.incr(`paste:views:${id}`).catch(() => {});

  // Immutability (N4) is what makes this header provably correct: the bytes
  // behind this ID can NEVER change, so a one-year edge cache can never be stale.
  res.set('Cache-Control', 'public, max-age=31536000, immutable');

  // Big paste: hand back a pre-signed URL and let the client fetch the bytes
  // straight from CDN/S3. Our Node process never buffers 10 MB.
  if (meta.size_bytes > INLINE_LIMIT) {
    const signed = await getSignedUrl(
      s3,
      new GetObjectCommand({ Bucket: BUCKET, Key: meta.s3_key }),
      { expiresIn: 3600 }
    );
    return res.json({ ...publicFields(meta), contentUrl: signed });
  }

  // Small paste: serve inline, from Redis if we can.
  let content = await redis.get(`paste:body:${id}`);
  if (content === null) {
    const obj = await s3.send(new GetObjectCommand({ Bucket: BUCKET, Key: meta.s3_key }));
    content = await obj.Body.transformToString();
    // TTL on the cache entry never outlives the paste itself.
    const ttl = meta.expires_at
      ? Math.max(1, Math.floor((new Date(meta.expires_at) - Date.now()) / 1000))
      : 86400;
    await redis.setex(`paste:body:${id}`, Math.min(ttl, 86400), content);
  }
  res.json({ ...publicFields(meta), content });
});
```

> **A note on very large uploads:** for a real 10 MB limit you'd skip proxying the body through Node entirely on the *write* side too — return a **pre-signed PUT URL** from `POST /api/pastes`, let the browser upload directly to S3, and have the client call back to confirm. Same PENDING → ACTIVE state machine, but your API servers move zero bytes. Mention this; it's the "how would you handle 1 GB files?" follow-up in disguise.

---

### 8. Expiry and cleanup

You promised `expiresIn: "10m" | "1d" | "never"`. How does a paste actually die? Two strategies — this comparison is a **classic interview follow-up**.

#### Strategy A — Lazy deletion

On every read, check `expires_at`. If it's in the past, return 404 and (optionally) enqueue a delete.

| Pros | Cons |
|---|---|
| Trivially simple — three lines in the read handler | Expired data **lingers forever** if nobody reads it |
| Always **correct**: a paste is never served past its expiry, even if every background job is down | You keep paying S3 for content that is logically deleted |
| Zero moving parts, nothing to monitor | Your DB row count grows without bound |
| No thundering-herd of deletes at midnight | GDPR/"delete my data" promises are not actually kept |

The fatal flaw: **most expired pastes are never read again.** That's *why* they expired. So lazy deletion alone means your storage bill grows monotonically forever, and you're storing data you told users you'd destroy.

#### Strategy B — Active cleanup

A background sweeper (or an S3 lifecycle policy) proactively deletes expired content.

| Pros | Cons |
|---|---|
| Storage actually shrinks; the bill reflects live data | Another job to run, monitor, and alert on |
| The "10 minutes and it's gone" promise is real | If the sweeper dies silently, you don't notice for weeks |
| Table stays small → indexes stay fast | Delete batches can spike DB/S3 load if unthrottled |
| S3 **lifecycle rules** need no code at all | Lifecycle rules are coarse (by prefix/tag/age, not per-object timestamp) |

#### Recommendation: run BOTH

This is the answer.

- **Lazy check for correctness.** The read handler *always* compares `expires_at` to `now()`. This is your source of truth. It means the system is correct **even if the sweeper has been broken for a month** — you never serve expired content. Correctness must not depend on a cron job.
- **Active sweeper for the bill.** A job every 5 minutes deletes batches of expired pastes (the code in §7). It reclaims S3 bytes and keeps the table from becoming a graveyard.
- **S3 lifecycle rules as a safety net.** Tag objects on upload with their TTL bucket (`ttl=10m`, `ttl=1d`) and configure S3 to expire tagged objects after 1 day / 7 days. Now even if your sweeper *and* your alerting fail, AWS deletes the bytes for you. Belt, braces, and a second pair of braces — and the third layer costs zero code and zero compute.

> **The principle:** *correctness in the read path, cost-control in the background.* Never make a background job responsible for a guarantee you make to users.

**Also expire the cache.** When you cache a paste body in Redis, set the Redis TTL to `min(paste_ttl, cache_ttl)` — otherwise you'll cheerfully serve a 10-minute paste for an hour from Redis while the DB says it's dead. (The code in §7 does this.)

---

### 9. Deep dives

#### Deep dive 1 — Caching hot pastes (the viral paste)

Normally reads are spread thin — 1.8 billion pastes, 120 reads/sec. Then someone posts a paste to Hacker News and **one** `paste_id` takes 5,000 requests/sec while every other paste takes zero. Your average QPS is unchanged. Your database is on fire.

The fix is layered (recall [59 — Caching in Depth](./59-caching-in-depth.md)):

| Layer | What it holds | Hit latency | Notes |
|---|---|---|---|
| **CDN edge** | Rendered page + raw bytes | ~10 ms | Absorbs ~99% of a viral spike. The origin sees one request *per edge location*, not per user |
| **Redis** | `paste:meta:{id}` + `paste:body:{id}` (< 100 KB) | ~1 ms | Catches CDN misses and API-client traffic |
| **In-process LRU** | Top ~1,000 pastes, per API box | ~0.01 ms | Optional. Kills the last hop for a truly hot key |
| **Postgres replica** | Everything | ~2 ms | Should almost never be reached for a hot paste |

Because pastes are immutable, **there is no invalidation problem** — the hardest part of caching (recall: "the two hard problems are naming things, cache invalidation, and off-by-one errors") *disappears*. You never invalidate. You only expire, and expiry is a fixed timestamp known at write time.

The one real risk is a **cache stampede**: the hot paste's Redis entry expires and 5,000 concurrent requests all miss and all hammer S3 simultaneously. Guard it with a **single-flight lock** — the first miss takes a short Redis lock and fetches; the rest wait ~20 ms and read the freshly-warmed key.

```js
async function getBodySingleFlight(id, s3Key) {
  const cached = await redis.get(`paste:body:${id}`);
  if (cached !== null) return cached;

  // Only ONE request per node gets to hit S3; the herd waits for it.
  const lock = await redis.set(`lock:body:${id}`, '1', 'NX', 'EX', 10);
  if (!lock) {
    await new Promise(r => setTimeout(r, 50));
    return getBodySingleFlight(id, s3Key);   // re-check; it should be warm now
  }
  const obj = await s3.send(new GetObjectCommand({ Bucket: BUCKET, Key: s3Key }));
  const body = await obj.Body.transformToString();
  await redis.setex(`paste:body:${id}`, 3600, body);
  await redis.del(`lock:body:${id}`);
  return body;
}
```

#### Deep dive 2 — Serving content via CDN, and why immutability makes it trivial

Recall [60 — CDN](./60-cdn.md): a CDN is a fleet of caches near your users. The perennial headache with CDNs is **staleness** — you update a resource, but edges keep serving the old one until the TTL lapses, so you set short TTLs (poor hit rate) or build a purge pipeline (complexity, and purges take seconds-to-minutes to propagate globally).

**None of that applies here.** `pste.io/xK9mQ2vB4nRt` maps to exactly one immutable byte-string, forever. There is no "new version." So:

```
Cache-Control: public, max-age=31536000, immutable
```

One year. The `immutable` directive tells the browser not to even send a revalidation request on reload. This isn't an optimization gamble — it is **provably correct** given requirement N4. It is the single biggest performance win in the design, and it is *free* because of a decision you made in the requirements phase. This is why interviewers want you to state immutability explicitly: it unlocks everything downstream.

Two caveats worth naming:
- **Expiring pastes need shorter edge TTLs.** A paste with `expiresIn: "10m"` must not sit in a CDN for a year. Set `max-age = min(paste_ttl, 1 year)`. Pastes that never expire get the full year.
- **Private pastes must not be CDN-cached.** Send `Cache-Control: private, no-store` for `visibility = private`, and serve them only through the API with an auth check. A public CDN edge caching a private paste is a data breach, and it's an easy mistake — get the branch right.

#### Deep dive 3 — View counts without slowing reads

You want a `view_count`. The naive version is a landmine:

```js
// DON'T. On a viral paste this is 5,000 UPDATEs/sec on ONE row.
await db.query('UPDATE pastes SET view_count = view_count + 1 WHERE paste_id = $1', [id]);
```

Every one of those updates takes a **row lock**. They serialize. Your reads now queue behind a write lock on the hottest row in the database — and worse, this write **only happens on a CDN miss**, so your counts are wrong anyway.

Make analytics **asynchronous and approximate**:

1. **Increment in Redis** on the read path — `INCR paste:views:{id}` — which is atomic, lock-free, and sub-millisecond. Fire-and-forget: `.catch(() => {})`. A dropped view count is not an incident.
2. **Flush periodically.** A cron job every 60 seconds reads the Redis counters and folds them into Postgres in one batched statement. 5,000 individual updates become **one** update with `+5000`.

```js
// Every 60s: drain the Redis counters into the durable store.
async function flushViewCounts() {
  const keys = await redis.keys('paste:views:*');       // use SCAN in prod
  for (const key of keys) {
    const id = key.split(':')[2];
    // GETDEL is atomic: read and reset in one step, so views arriving
    // during the flush are counted in the NEXT window, never lost or doubled.
    const delta = await redis.getdel(key);
    if (!delta) continue;
    await db.query(
      'UPDATE pastes SET view_count = view_count + $1 WHERE paste_id = $2',
      [Number(delta), id]
    );
  }
}
```

Accept the trade: view counts are eventually consistent, up to 60 seconds stale, and lose a few counts if Redis dies. **That is completely fine** — nobody has ever filed a P1 because a view counter read 4,981 instead of 4,983. And for views served straight from the CDN, you can't count them in your app at all: parse the **CDN access logs** offline if you need true numbers.

**Never let a nice-to-have metric sit on the critical path of a must-have read.**

#### Deep dive 4 — Abuse and moderation

An anonymous, free, unauthenticated text-hosting service with public URLs is a **magnet for abuse**. Real pastebins deal with all of this daily:

| Abuse | Countermeasure |
|---|---|
| **Malware / phishing payloads** — attackers host base64 droppers and have malware fetch them | Scan on upload (ClamAV, VirusTotal API) asynchronously; quarantine (`status = QUARANTINED`) rather than blocking the write path. Honour takedown requests and publish an abuse contact |
| **Leaked credentials** — AWS keys, `.env` files, DB dumps, pasted by careless devs | Run secret-detection regexes (the `gitleaks` / `truffleHog` rulesets) on upload. On a hit: auto-set `visibility = private`, warn the user, and optionally notify the provider — AWS has a program that auto-revokes keys found in public pastes |
| **Spam / SEO link farms** — thousands of pastes full of backlinks | Rate limit hard (20 pastes/hour/IP, 5/min burst); CAPTCHA on anonymous creation above a threshold; `rel="nofollow noindex"` on rendered output |
| **Enumeration scraping** — botnets walking the keyspace for unlisted secrets | 12-char random IDs (§6) + aggressive 404-rate limiting per IP. **A client generating a high ratio of 404s is not a user — it's a scanner.** Ban on that signal |
| **Storage exhaustion** — a script creating 10 MB pastes in a loop | Per-IP byte quota, not just a request quota. 20 requests/hour × 10 MB = 200 MB/hour/IP is still a lot — cap total bytes too |
| **Illegal content** | Hash-matching against known-bad databases; a report button; a documented, staffed takedown process. This is a legal requirement, not a nice-to-have |

The architectural point: **all of this belongs off the write path.** Scanning synchronously would add seconds to every `POST`. Write the paste as `PENDING`, publish an event, let scanners run in the background, and flip to `ACTIVE` or `QUARANTINED`. You already built that state machine in §7 for a completely different reason — reuse it. Good state machines pay for themselves twice.

---

### 10. Bottlenecks, failure modes, and how to scale further

| # | Bottleneck / failure | Symptom | Mitigation |
|---|---|---|---|
| 1 | **Viral paste** | One key at 5,000 rps; DB replicas saturate | CDN + Redis + in-process LRU; single-flight to prevent stampede |
| 2 | **S3 latency spike or partial outage** | Read p99 → 2s; some reads 500 | Serve from Redis where possible; S3 cross-region replication; return a clear "content temporarily unavailable" rather than hanging |
| 3 | **Metadata DB primary down** | Writes fail entirely | Reads keep working from replicas + cache (**reads are the SLA**). Automatic failover; accept a short write outage. Availability requirement N3 is about *reads* |
| 4 | **Orphaned S3 objects** | Storage bill grows with no visible cause | DB-first `PENDING` write ordering + sweeper (§7) + S3 lifecycle rules |
| 5 | **Sweeper falls behind or dies silently** | Expired data lingers; bill creeps | Lazy read-time expiry keeps you *correct* regardless. Alert on sweeper lag and on `count(expired AND status=ACTIVE)` |
| 6 | **Redis eviction of hot bodies** | S3 read amplification, latency spike | Size Redis for the working set; `allkeys-lru`; cache only bodies < 100 KB so one 10 MB paste can't evict 1,000 small ones |
| 7 | **Node event loop blocked by large bodies** | Whole API box goes slow under a few big pastes | Never buffer 10 MB in the handler — pre-signed PUT/GET URLs, client talks to S3 directly |
| 8 | **DB growth past a single box** (a 50× future) | Slow queries, painful vacuums | Shard `pastes` by `hash(paste_id)`. Trivial here: every query is by primary key, there are **no cross-shard joins**, and IDs are already random so the hash distributes perfectly |
| 9 | **Storage cost creep** | The 18 TB line item | S3 lifecycle: Standard → Infrequent Access at 30 days → Glacier at 1 year. Most pastes are read in the first 48 hours and never again |
| 10 | **Abuse / legal takedown** | Malware hosting, leaked keys | §9: async scanning, rate limits, quarantine state, takedown process |

**How to scale 50× (50M pastes/day):** writes go 12/s → 600/s (still small), reads 120/s → 6,000/s (mostly absorbed by the CDN), storage 18 TB → 900 TB. Shard the metadata DB by `hash(paste_id)` into 8–16 shards; the ID is random so distribution is uniform and no query crosses a shard. S3 needs **no change at all** — it is already effectively infinite, and your keys (random 12-char IDs) already spread across its internal partitions.

**The scaling shape to remember:** blob storage scales itself; the metadata DB is the thing you eventually have to shard; and the read path is defended by caching, not by more servers.

---

## Visual / Diagram description

The full architecture diagram is in §7 above. Here is the **decision** the whole design turns on, drawn as one picture — this is the thing to put on the whiteboard first:

```
   OPTION A — content in the DB (WRONG at scale)
   ┌────────────────────────────────────────────┐
   │           Postgres  (3.6 TB / year)        │
   │  ┌──────────────────────────────────────┐  │
   │  │ paste_id │ expires │  content        │  │
   │  │ xK9mQ2   │  never  │ [ 10 MB blob ]  │  │  ← buffer pool thrashed
   │  │ pL3nW8   │  1 day  │ [ 8 MB blob  ]  │  │  ← backups take hours
   │  │ ...                                   │  │  ← $0.10/GB, ×replicas
   │  └──────────────────────────────────────┘  │  ← CDN can't cache a query
   └────────────────────────────────────────────┘


   OPTION B — content in S3, metadata in the DB (RIGHT)
   ┌───────────────────────────┐        ┌───────────────────────────┐
   │  Postgres (365 GB/5 yrs)  │        │      S3 (18 TB/5 yrs)     │
   │ ┌───────────────────────┐ │        │  ┌─────────────────────┐  │
   │ │paste_id│expires│s3_key│ │        │  │ pastes/2026/07/     │  │
   │ │xK9mQ2  │ never │ ────────────────────▶  xK9mQ2  [10 MB]   │  │
   │ │pL3nW8  │ 1 day │ ────────────────────▶  pL3nW8  [ 8 MB]   │  │
   │ └───────────────────────┘ │  s3_key│  └─────────────────────┘  │
   │   ~200 bytes per row      │ points │   $0.023/GB, infinite,    │
   │   index fits in RAM       │   to   │   CDN-frontable, backed   │
   │   backup in minutes       │  bytes │   up by AWS for you       │
   └───────────────────────────┘        └───────────────────────────┘
        the FILING CABINET                    the WAREHOUSE
```

**What the picture says:** the arrow labelled `s3_key` is the whole design. It is a *pointer*. The database stores pointers and the things you query; object storage stores the bytes those pointers point at. Once you internalize that arrow, you have the answer for pastebin, for file upload, for image hosting, for video, for document storage, and for every "user gives us a big thing" feature you will ever build.

---

## Real world examples

### GitHub Gist

Gist is a pastebin whose backing store is **Git itself** — every gist is a real, cloneable Git repository. That's an unusual choice with a clear motive: Git gives them versioning and diffs for free (so Gists, unlike our design, *are* mutable and keep history). The trade is exactly the one we've been discussing in reverse: they accepted a heavier storage substrate to buy revision history. Note how the requirement ("gists can be edited and show history") *forced* the storage decision. Requirements first, always — this is the framework from topic 93 doing real work.

### Pastebin.com

The archetype. Public/unlisted/private visibility levels, an expiry dropdown with exactly the presets we listed, syntax highlighting, and — tellingly — a large public **abuse and scraping problem**: security researchers famously monitor Pastebin's public firehose for leaked credential dumps, and it is a well-documented distribution channel for malware configs. That real-world history is precisely why §6 (unguessable IDs) and §9 (moderation) exist in this design rather than being afterthoughts.

### The general pattern: metadata-in-DB, bytes-in-blob-store

This isn't a pastebin trick — it is the near-universal architecture for user-uploaded content. Conceptually: Slack stores a file's name, uploader, and channel in its datastore and the file bytes in object storage; Instagram stores photo metadata in a database and the image bytes in a blob store behind a CDN; Dropbox stores file metadata (name, path, chunk list) in a database and the file chunks in object storage. Describe it as the standard pattern rather than claiming specific internal details — but state it with confidence, because **any system that stores big user blobs in its primary OLTP database is, by industry consensus, doing it wrong.**

---

## Trade-offs

### Content storage

| Approach | Pros | Cons |
|---|---|---|
| **DB TEXT/BLOB column** | One system; transactional with metadata; dead simple | Table bloat; buffer-pool pollution; slow, huge backups; 10× the cost; can't CDN it |
| **Blob storage + metadata DB** | Cheap ($0.023/GB); infinite scale; CDN-frontable; small fast DB; free durability | Extra network hop on cache miss; **no cross-system transaction** (orphans possible); two systems to operate |

### Expiry

| Approach | Pros | Cons |
|---|---|---|
| **Lazy only** | Trivial; always correct | Storage grows forever; deletion promises broken |
| **Sweeper only** | Bill matches live data | If the job dies, you serve expired content — a correctness bug |
| **Both (recommended)** | Correct *and* cheap; lifecycle rules as a third net | One more job to monitor |

### ID scheme

| Approach | Pros | Cons |
|---|---|---|
| **Sequential base62** | Shortest IDs; no collisions | **Guessable → unlisted pastes are public.** Disqualifying |
| **Random 12-char base62** | ~71 bits; enumeration takes ~57 years at 1k guesses/s | Slightly longer URLs; a vanishingly rare collision retry |
| **Content hash** | Free deduplication of identical pastes | Identical content ⇒ identical URL: a leak, and it breaks per-paste expiry |

**The sweet spot:** metadata in Postgres, bytes in S3, 12 random base62 characters, immutable pastes with a one-year CDN `max-age`, lazy expiry for correctness plus a sweeper and S3 lifecycle rules for the bill. That design serves 1M pastes/day on a handful of small boxes and scales 50× by sharding one table.

**Rule of thumb:** **if you never query it, it does not belong in your database.**

---

## Common interview questions on this topic

### Q1: "Would you store the paste content in the database or in blob storage? Defend it."

**Hint:** Blob storage, and give *numbers*, not opinions. 10 MB rows bloat the table to ~3.6 TB/year; large values evict index pages from the buffer pool so *unrelated* queries slow down; backups and restores go from minutes to hours; DB SSD is ~$0.10/GB-month against S3's $0.023 — and you pay the DB price again per replica and per snapshot. The cost delta alone is ~10×. Name the one real cost you're accepting — an extra network hop on read — and immediately explain how you pay it back with Redis and a CDN. Close with the general rule: *the database holds what you query; object storage holds bytes you fetch by key.*

### Q2: "Can I just use sequential IDs like the URL shortener does?"

**Hint:** No — and explain *why this system is different*. An unlisted paste's only protection **is** its URL, so a guessable ID means the paste is effectively public. With sequential IDs an attacker enumerates the entire corpus with a for-loop and harvests every leaked API key ever pasted. Do the entropy math: 12 random base62 chars ≈ 71 bits, keyspace 3.2×10²¹, ~1.8B live pastes ⇒ ~1 in 1.8 trillion guesses hits, ≈ 57 years at 1,000 guesses/sec. Then mention `crypto.randomBytes`, never `Math.random()`.

### Q3: "The S3 upload succeeds but the metadata write fails. What happens?"

**Hint:** You get an **orphaned object** — bytes you pay for forever that nothing references, and the failure is *silent*. Fix it by inverting the order: write the metadata row **first** as `PENDING`, then upload, then flip to `ACTIVE`. Now a crash leaves a 200-byte row (findable with an index scan) instead of a 10 MB unreferenced object (findable only by diffing the whole bucket). The read path ignores non-`ACTIVE` rows, so a partial write simply 404s — which is correct. Back it with a sweeper that reaps `PENDING` rows older than 15 minutes. Bonus: name *why* the problem exists at all — there is no distributed transaction across Postgres and S3.

### Q4: "How do you handle expiry — and what if the cleanup job is down for a week?"

**Hint:** That question is a trap testing whether your **correctness** depends on a cron job. It must not. Lazy expiry — checking `expires_at` on every read — means you never serve an expired paste no matter what the background jobs are doing. The sweeper exists purely to control the **storage bill**, and S3 lifecycle rules are a zero-code backstop under it. Recommend all three layers and say the principle: *correctness in the read path, cost control in the background.*

### Q5: "A paste hits the front page and takes 5,000 requests/sec. Walk me through what happens."

**Hint:** Almost nothing, and that's the point. The CDN absorbs ~99% — the origin sees roughly one request per edge location, not 5,000/sec. CDN misses hit Redis (~1 ms). Because pastes are **immutable**, `max-age=31536000, immutable` is provably safe and there is **no invalidation problem at all**. The two real risks: a **cache stampede** when the Redis key expires (fix: single-flight lock), and the `view_count` `UPDATE` serializing on one row lock (fix: `INCR` in Redis, batch-flush to Postgres every 60s, accept eventual consistency — nobody pages you over an off-by-two view counter).

---

## Practice exercise

### Build the "where do the bytes live?" case, end to end

**Time: ~35 minutes. Produce a one-page written answer plus one diagram — the deliverable an interviewer would actually accept.**

Your product manager wants to raise the paste size limit from **10 MB to 100 MB** (people want to share whole log archives). Volume stays at 1M pastes/day, but the average paste size rises from 10 KB to **50 KB**.

Do all five:

1. **Redo the estimation.** New storage/day, /year, and over 5 years. New read bandwidth at 10:1. Show every line of arithmetic.
2. **Cost it both ways.** Compute the 5-year monthly storage cost for Option A (DB at $0.10/GB-month, plus one replica) and Option B (S3 Standard at $0.023/GB-month). State the multiple.
3. **Kill the proxy.** At 100 MB, buffering the body in your Node handler is no longer merely wasteful — it will OOM your box under concurrency. Redesign `POST /api/pastes` to return a **pre-signed S3 PUT URL** so the browser uploads directly. Write the handler *and* the confirm endpoint the client calls afterwards. Keep the `PENDING → ACTIVE` state machine.
4. **Fix the cache.** Your Redis inline-cache limit is 100 KB. With 50 KB average pastes, what fraction of pastes now qualify — and what breaks if you naively raise the limit to 100 MB? (Think about what a single cached object does to the LRU set.)
5. **Draw it.** Redraw the §7 architecture with the direct-to-S3 upload path. Mark exactly which arrows carry the 100 MB and which carry only metadata. **Every arrow touching an API server should carry only metadata.** If one doesn't, your design is wrong — find it.

---

## Quick reference cheat sheet

- **The one lesson:** metadata in the database, **bytes in blob storage**. The DB row holds an `s3_key` *pointer*, not the content.
- **Why:** 10 MB rows bloat the table, **pollute the buffer pool** (slowing unrelated queries), make backups/restores glacial, cost ~10× more, and **cannot be fronted by a CDN**.
- **The rule:** if you never `WHERE`, `ORDER BY`, or `JOIN` on it, it does not belong in a row.
- **Immutability is the superpower.** No update path, no cache invalidation, no lost updates, and a provably-safe `max-age=31536000, immutable`. Declare it in the requirements phase and reap it everywhere downstream.
- **The numbers:** 1M pastes/day = **~12 writes/s**, **~120 reads/s** (10:1), **10 GB/day**, **~3.6 TB/year**, **~18 TB over 5 years**. Metadata is only ~365 GB.
- **This is a modest system.** The pressure is **bytes, not QPS** — no Cassandra, no Kafka, no exotic sharding. Say so; it shows you derive rather than cargo-cult.
- **vs. a URL shortener:** same creation rate, far fewer reads *per item*, so a much smaller read tier. Fewer reads per item = smaller system.
- **IDs must be UNGUESSABLE.** An unlisted paste's only protection is its URL. **12 random base62 chars ≈ 71 bits** ⇒ ~57 years to find one paste at 1,000 guesses/sec. `crypto.randomBytes`, never `Math.random()`.
- **Write order: DB (`PENDING`) → S3 → DB (`ACTIVE`).** There's no transaction across Postgres and S3, so a crash leaves *something*. Make that something a 200-byte row, not a 10 MB orphan.
- **Expiry: lazy AND active.** Lazy check on read = correctness (survives a dead cron). Sweeper + S3 lifecycle rules = the storage bill. Never let a cron job own a user-facing guarantee.
- **404, not 410,** for expired pastes — don't confirm to an enumerator that an ID once existed.
- **View counts:** `INCR` in Redis, batch-flush to Postgres every 60s. Never `UPDATE ... SET view_count = view_count + 1` on the hot path — it's a row lock on your hottest row.
- **Never proxy big bodies through Node.** Pre-signed S3 URLs let the client talk to S3 directly; your API servers move only metadata.
- **Abuse is a first-class requirement:** rate limit by requests *and bytes*, secret-scan asynchronously into a `QUARANTINED` state, ban clients with a high 404 ratio (that's a scanner, not a user).

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [94 — Design a URL Shortener](./94-hld-url-shortener.md) — the ID-generation analysis reused here; pastebin is that system plus a large payload and an unguessability requirement |
| **Next** | [78 — Blob Storage](./78-blob-storage.md) — the deep dive on the object-storage layer this entire design rests on: keys, durability, lifecycle rules, and pre-signed URLs |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — cache-aside, TTLs, stampedes, and why immutable content makes cache invalidation vanish |
| **Related** | [60 — CDN](./60-cdn.md) — how edge caching absorbs a viral paste, and why `max-age=31536000, immutable` is provably safe here |
| **Related** | [93 — HLD Approach Framework](./93-hld-approach-framework.md) — the 5-step framework (requirements → estimation → API → data model → architecture) executed in this doc |
