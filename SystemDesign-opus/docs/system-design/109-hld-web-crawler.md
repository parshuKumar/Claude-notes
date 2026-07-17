# 109 — Design a Web Crawler
## Category: HLD Case Study

---

## What is this?

A **web crawler** (also called a *spider* or *bot*) is a program that starts from a handful of URLs, downloads those pages, finds the links inside them, and then downloads *those* pages too — recursively, forever, across the whole web. It is how Google, Bing, and the Internet Archive discover and copy the billions of pages that make the web searchable.

Think of it as a **librarian with an infinite to-do list**. You hand them a few book titles (seed URLs). Each book they read cites other books; they add every citation to the to-do list, fetch those, and keep going. The web crawler is that process running on thousands of machines, politely, for years.

---

## Why does it matter?

**Interview angle.** "Design a web crawler" is a top-tier distributed-systems question because it forces you to reason about *scale you cannot fit on one machine* (billions of pages), a *shared work queue* that many workers pull from (the URL frontier), *deduplication at scale* (Bloom filters), and *being a good citizen* (politeness). It is a graph-traversal problem wearing a distributed-systems costume. Interviewers love it because a naive answer ("a queue and a `for` loop") collapses instantly under the follow-up questions.

**Real-work angle.** If you ever build a scraper, a search index, an SEO tool, a price-monitor, or a data-ingestion pipeline, you *are* building a crawler. Get politeness wrong and you will DoS a small website, get your IP banned, and possibly receive a legal letter. A crawler that hammers one server 1,000 times per second is indistinguishable from an attack. The difference between "a crawler" and "an attack" is entirely in the engineering you are about to learn.

---

## The core idea — explained simply

### The analogy: a swarm of polite mail carriers sharing one sorting office

Imagine a giant postal sorting office. Letters (URLs) pour in. A rule book says: *each carrier owns specific neighbourhoods, and no carrier may knock on the same house more than once every 10 seconds.* Carriers pick up letters, walk to the address (download the page), read the new addresses written inside (extract links), drop those new letters back at the sorting office, and file a copy of what they found (store the page).

Two things make this hard at web scale:

1. **The to-do pile never stops growing** and cannot fit in one room — it must be spread across many buildings.
2. **You must be polite** — one street (one website's server) can only take so many knocks before it collapses or calls the police.

| Postal-office part | Crawler component | Job |
|---|---|---|
| The letters pouring in | URLs | The units of work |
| The sorting office / to-do pile | **URL Frontier** | Prioritised, polite queue of URLs to fetch |
| A mail carrier | **Fetcher / worker** | Downloads one page |
| Reading addresses inside a letter | **Parser** | Extracts text + outbound links |
| "Have I delivered here already?" list | **Dedup (Bloom filter)** | Skip URLs already seen |
| "Is this the same flyer again?" check | **Content dedup (hash / SimHash)** | Skip identical page content |
| The archive room | **Blob storage + DB** | Persist pages and metadata |
| The 10-second knock rule | **Politeness policy** | Per-host rate limiting + robots.txt |

Everything below is just filling in these boxes carefully. The whole design is a giant **BFS (breadth-first search) over the web graph**, done politely, at scale, without falling into traps.

---

## Key concepts inside this topic

We will execute the standard **5-step HLD framework**: (1) Requirements, (2) Capacity estimation, (3) the crawl loop / API, (4) Data model, (5) High-level architecture — then deep dives and bottlenecks.

### 1. Requirements (functional + non-functional + clarifying questions)

**Clarifying questions to ask the interviewer first:**
- Are we crawling for a *search index* (need text + link graph), an *archive* (need raw HTML), or *monitoring* (need change detection)? This changes what we store.
- HTML only, or images / PDFs / video too? (Assume: HTML text focus, skip binaries.)
- One-time crawl or continuous with re-crawling for freshness? (Assume: continuous.)
- Scale target? (Assume: **1 billion pages/month**, growing to tens of billions stored.)

**Functional requirements:**
1. Given a set of **seed URLs**, download the pages.
2. **Extract** text and outbound links from each page.
3. **Recursively crawl** discovered links (the BFS).
4. **Store** pages (for a search index or archive).
5. Respect **robots.txt**.
6. **Re-crawl** pages periodically to stay fresh.

**Non-functional requirements (these are what the interview is really about):**
- **Massive scale** — billions of pages; nothing fits on one machine.
- **High throughput** — hundreds of pages/second sustained.
- **Politeness** — never hammer one host. This is a hard constraint, not a nice-to-have.
- **Robustness** — the web is hostile: crawler traps, malformed HTML, 5 GB "files", slow servers, redirect loops. The crawler must survive all of it.
- **Freshness** — important pages re-crawled more often.
- **Extensibility** — easy to add a new parser (say, for PDFs) later.

### 2. Capacity estimation (show the math)

**Throughput.** 1 billion pages per month. A month is about `30 × 24 × 3600 ≈ 2.5 million seconds`.

```
pages/second = 1,000,000,000 / 2,500,000 = 400 pages/second (sustained average)
```

Real traffic is bursty, so design for a **peak of ~2,000 pages/second** (a 5× headroom factor).

**Storage.** Average HTML page ≈ 100 KB.

```
per month = 1e9 pages × 100 KB = 1e14 bytes = 100 TB / month
per year  = 100 TB × 12       = 1.2 PB / year
```

So we are firmly in **petabyte** territory. This alone kills any single-machine design and mandates distributed blob storage.

**Bandwidth.** 400 pages/s × 100 KB = `40 MB/s = 320 Mbps` sustained ingest (much higher at peak). Non-trivial but fine for a datacenter.

**Machine count.** Suppose one crawler machine can politely fetch ~100 pages/second (limited by network round-trips and politeness delays, not CPU).

```
machines = 2,000 peak / 100 per machine = ~20 fetcher machines
```

Add machines for parsing, dedup stores, and the frontier. The point of the arithmetic: **~400 pages/s and 1.2 PB/year is why this must be a distributed system** — you cannot do it on one box.

**Dedup memory.** We must remember which URLs we have seen. 1 billion URLs. A hash set of full URLs at ~100 bytes each = `100 GB` — too big for one machine's RAM. This is exactly why we reach for a **Bloom filter** (section 6): ~1.2 GB for 1 billion URLs at a 1% false-positive rate. That is a 80× saving and fits in memory.

### 3. The high-level crawl loop (the cycle)

Everything the crawler does is one loop, repeated billions of times:

```
   ┌──────────────────────────────────────────────────────────┐
   │                                                            │
   ▼                                                            │
URL FRONTIER ──▶ FETCHER ──▶ PARSER ──▶ DEDUP ──▶ store page   │
(pick next   (download,   (extract    (seen     to blob store  │
 polite URL)  robots+rate   text +     before?)                │
              limited)      links)        │                    │
                                          │ new URLs           │
                                          └────────────────────┘
                                             add back to frontier
```

Read it as a cycle: the Frontier hands out a URL → the Fetcher downloads it (respecting robots.txt and the per-host delay) → the Parser pulls out text and links → Dedup drops URLs/content we have already seen → the page is stored, and genuinely new URLs flow **back into the Frontier**. That feedback edge is what makes it a recursive traversal — a **BFS over the web graph**, where each node is a page and each edge is a hyperlink.

### 4. The URL Frontier — the heart of the design

The Frontier is *not* a simple FIFO queue. It must satisfy two goals that **fight each other**:

- **Priority** — crawl important/fresh pages first. A news homepage should be crawled far more often than a 2009 archive page.
- **Politeness** — a bounded, respectful request rate *per host*, with a delay between consecutive hits to the same domain.

The classic solution is a **two-level queue design** (from the Mercator crawler):

```
                        THE URL FRONTIER
   New URLs in
       │
       ▼
 ┌──────────────────────┐
 │  Prioritiser          │  assign priority 1..N (freshness,
 └──────────────────────┘  page rank, update frequency)
       │
       ▼
 ┌────────── FRONT QUEUES (priority) ──────────┐
 │  F1 (high)  F2        F3       ...  FN (low) │  one queue per priority level
 └──────────────────────────────────────────────┘
       │   biased picker: pull more often from high-priority queues
       ▼
 ┌────────── BACK QUEUES (politeness) ──────────┐
 │  B1        B2        B3       ...  BM         │  one queue PER HOST
 │  (host a)  (host b)  (host c)     (host z)    │  each mapped to exactly one host
 └──────────────────────────────────────────────┘
       │
       ▼
 ┌──────────────────────┐
 │  Heap of (nextTime,   │  min-heap: which host is allowed to be
 │           backQueue)  │  crawled next, ordered by "not before" time
 └──────────────────────┘
       │
       ▼   worker pops the earliest-ready host, fetches ONE url,
           then reinserts that host at (now + crawlDelay)
    FETCHERS
```

**How it works:**
- **Front queues** enforce *priority*. A URL's computed priority decides which front queue it lands in. A biased picker pulls from high-priority queues more often.
- **Back queues** enforce *politeness*. There is exactly **one back queue per host**, and a routing table maps each host → its back queue. Because a host has only one queue, only one URL from that host is ever in flight, so a per-host delay is trivially enforceable.
- The **min-heap** tracks, for each back queue, the earliest time it may next be crawled (`lastCrawlTime + crawlDelay`). A worker always pops the host that is ready soonest, fetches one URL, then pushes that host back onto the heap with an updated "not before" time.

**Distributing the frontier — the key insight.** Partition back queues across machines by **hashing the host** (recall **[74 — Consistent Hashing](./74-consistent-hashing.md)**). Because *all* URLs for `example.com` hash to the *same* worker, that worker sees the entire request stream for that host and can enforce the per-host delay **locally**, with no cross-machine coordination. This host-partitioning trick is what makes politeness scale. If you instead sharded URLs randomly, two machines could hit the same host simultaneously and you would be back to accidental DoS.

### 5. Politeness in detail

Politeness is the single non-negotiable requirement. It has several parts:

1. **robots.txt** — before crawling any host, fetch `http://host/robots.txt`, parse it, and **cache it** (per host, with a TTL of, say, 24 hours). Honour `Disallow` rules (never fetch forbidden paths) and `Crawl-delay` (minimum seconds between requests).
2. **Default courteous delay** — even without a `Crawl-delay`, impose your own default (e.g. 1 request/second per host). Better: adapt the delay to the server's response time — if a server takes 2 s to respond, it is struggling; back off.
3. **Descriptive User-Agent** — identify yourself: `MyCrawler/1.0 (+https://example.com/bot-info)`. This lets site owners contact you instead of blocking you.
4. **Honour 429 and rate limits** (recall **[70 — Rate Limiting](./70-rate-limiting.md)**) — a `429 Too Many Requests` or `Retry-After` header means slow down *now*.
5. **Exponential backoff on errors** — on 5xx/timeouts, retry with growing delays, then give up. Do not retry-storm a server that is already down.

Say it plainly in the interview: **without this, your crawler is a distributed denial-of-service attack.** Politeness is what makes it legal and welcome.

### 6. Deduplication — two kinds, both essential

**(a) URL dedup** — do not crawl the same URL twice. With billions of URLs you cannot keep them all in RAM as strings (we computed ~100 GB above). Use a **Bloom filter**: a bit array plus *k* hash functions.

> A Bloom filter is a probabilistic set. `add(x)` sets *k* bits; `has(x)` checks those *k* bits. It has **false positives** (it may say "seen" for a URL you never saw — so you *rarely* skip a genuinely new URL) but **never false negatives** (if it says "not seen", it is definitely new). For a crawler that trade-off is perfect: occasionally missing one new page out of a billion is fine; re-crawling the same page millions of times is not. Back it with a persistent key-value store so state survives restarts.

**(b) Content dedup** — many *different* URLs return *identical or near-identical* content: mirror sites, `?sessionid=...` URLs, printer-friendly versions, `www` vs non-`www`. Storing and indexing the same page 50 times wastes storage and pollutes search results.
- For **exact** duplicates: hash the page body (e.g. SHA-256) and keep a set of seen content hashes. Same hash → skip.
- For **near** duplicates (a page that differs only by a timestamp or an ad): use a **similarity hash such as SimHash**. SimHash produces fingerprints where *similar documents get similar fingerprints* (small Hamming distance). If a new page's SimHash is within, say, 3 bits of an existing one, treat it as a duplicate.

Both are cheap fingerprints standing in for expensive full-text comparison at scale.

### 7. Data storage

| What | Where | Why |
|---|---|---|
| Raw HTML / page bodies | **Blob storage** (recall **[78 — Blob Storage](./78-blob-storage.md)**), e.g. S3/HDFS | Huge, write-once, read-rarely; petabyte-scale; cheap per GB |
| URL metadata + crawl state (last-crawled, status, priority, content hash) | Distributed DB / KV store | Frequent point lookups and updates; needs to scale horizontally |
| Bloom-filter bits / seen-URL set | In-memory + persistent backing store | Fast membership checks |
| The **link graph** (page → outbound links) | Column store / graph store | Feeds PageRank-style ranking and re-crawl priority |
| Parsed text | Handed to the **search indexing pipeline** (recall **[79 — Search Systems](./79-search-systems.md)**) | Builds the inverted index that makes pages findable |

The parsed text does not stay in the crawler — it is emitted onto a stream (recall **[67 — Message Queues](./67-message-queues.md)**) that the indexing pipeline consumes, decoupling crawling from indexing.

### 8. Extensibility

Make the **Parser** and **storage** pluggable. A parser interface (`canHandle(contentType)` + `parse(body)`) lets you add a PDF or image parser later without touching the crawl loop. The frontier, fetcher, and dedup stay unchanged. This is the "extensibility" NFR paid off.

---

## Visual / Diagram description

The full distributed architecture, tying every component together:

```
        SEED URLS
           │
           ▼
   ┌───────────────────────────────────────────────────────────┐
   │                    URL  FRONTIER  (sharded by host hash)    │
   │   ┌──────────┐   ┌──────────┐          ┌──────────┐         │
   │   │ shard 1  │   │ shard 2  │   ...    │ shard N  │         │
   │   │ front+   │   │ front+   │          │ front+   │         │
   │   │ back Qs  │   │ back Qs  │          │ back Qs  │         │
   │   └────┬─────┘   └────┬─────┘          └────┬─────┘         │
   └────────┼──────────────┼─────────────────────┼──────────────┘
            │              │                      │
            ▼              ▼                      ▼
      ┌──────────┐   ┌──────────┐          ┌──────────┐
      │ Fetcher  │   │ Fetcher  │   ...    │ Fetcher  │  worker pool
      │ +robots  │   │ +robots  │          │ +robots  │
      │ +delay   │   │ +delay   │          │ +delay   │
      └────┬─────┘   └────┬─────┘          └────┬─────┘
           │              │                      │
           ▼              ▼                      ▼
      ┌─────────────────────────────────────────────────┐
      │  DNS RESOLVER (with aggressive cache)            │  recall [54]
      └─────────────────────────────────────────────────┘
           │
           ▼
      ┌──────────┐        ┌──────────────┐        ┌──────────────┐
      │  PARSER  │──text─▶│ SEARCH INDEX │        │ BLOB STORE   │
      │ extract  │        │  PIPELINE    │◀──raw──│  (pages)     │
      │ text+links│       │  [79] via MQ │        │  [78]        │
      └────┬─────┘        └──────────────┘        └──────────────┘
           │ links
           ▼
      ┌──────────────────────────────┐
      │  DEDUP                         │
      │  ├─ URL Bloom filter (seen?)   │  [rare false positives OK]
      │  └─ Content hash / SimHash     │  [drop duplicate bodies]
      └────┬─────────────────────────┘
           │ genuinely new URLs
           └────────────────▶ back to URL FRONTIER  (loop closes)
```

**Reading it:** Seed URLs prime the Frontier. The Frontier is *sharded by host hash* so each shard owns whole domains. Fetcher workers pull ready URLs, resolve DNS (through a cache to avoid resolving billions of hostnames), download politely, and pass bodies to the Parser. The Parser splits into two flows: **raw HTML → blob store**, **extracted text → search-index pipeline** via a message queue. Extracted links go through **Dedup**; survivors loop back into the Frontier. The loop is the BFS; every box is a place it could break, which the next section addresses.

---

## Real world examples

### Google (Googlebot)

Google's crawler discovers pages by following links and by reading sitemaps. It maintains a massive URL frontier and, per Google's own published guidance, adjusts its **crawl rate** based on how fast a site responds and on the site's configured limits — slowing down when servers show strain. It caches robots.txt and honours `Disallow`. It re-crawls based on estimated **change frequency**: high-churn pages (news) get visited far more often than static ones. The extracted content feeds Google's search index (representative of the widely-documented Mercator-style architecture).

### Internet Archive (Wayback Machine)

The Internet Archive runs crawls (using the open-source **Heritrix** crawler) not to build a search index but to **archive** the web — it stores the raw responses in WARC files (a blob-storage format) so pages can be replayed years later. This is the "archive" flavour of our requirements: the emphasis is on faithful raw storage and breadth over ranking. Heritrix's design centres on exactly the frontier + politeness machinery described here.

### Common Crawl

Common Crawl is a non-profit that crawls billions of pages monthly and **publishes the dataset publicly** (petabytes of WARC files on S3-style blob storage). It is a real, open example of the capacity numbers we estimated — billions of pages, petabyte storage — and many companies build search/ML products on top of its output instead of crawling themselves.

---

## Trade-offs

**URL dedup: Bloom filter vs. exact hash set**

| | Bloom filter | Exact set (hash / DB) |
|---|---|---|
| Memory | Tiny (~1.2 GB / 1B URLs) | Huge (~100 GB / 1B URLs) |
| Errors | Rare false positives (skip a new page) | None |
| Speed | Very fast, in-memory | Slower if disk-backed |
| Verdict | **Default at web scale** | Only if zero misses are required |

**Politeness delay: aggressive vs. gentle**

| | Short delay (fast) | Long delay (polite) |
|---|---|---|
| Throughput per host | High | Low |
| Risk of banning / DoS | High | Low |
| Coverage speed | Faster | Slower |
| Verdict | Fine for big robust hosts | **Safer default; adapt per host** |

**Freshness: crawl everything often vs. adaptive**

| | Uniform re-crawl | Adaptive (by change rate) |
|---|---|---|
| Simplicity | Simple | Needs change estimation |
| Wasted fetches | Many (static pages) | Few |
| Freshness of hot pages | Poor | Good |
| Verdict | Toy systems | **Production choice** |

**The sweet spot:** shard the frontier by host, enforce per-host politeness locally, dedup URLs with a Bloom filter and content with SimHash, store raw pages in blob storage, and schedule re-crawls adaptively by estimated change rate. That combination is essentially every production crawler.

---

## Common interview questions on this topic

### Q1: "How do you crawl politely at scale without accidentally DoS-ing a site?"
**Hint:** Partition the URL frontier by **host** (hash the domain — consistent hashing). All URLs for one domain land on one worker, which keeps exactly one back queue per host and enforces `crawl-delay` locally via a min-heap of "not-before" times. Plus robots.txt caching, adaptive delays, and honouring 429/`Retry-After`. The host-partitioning is the key structural answer.

### Q2: "You have a billion URLs — how do you avoid crawling the same one twice?"
**Hint:** You cannot store a billion URL strings in RAM (~100 GB). Use a **Bloom filter** (~1.2 GB at 1% FP): false positives mean you *rarely* skip a genuinely new URL (acceptable), false negatives are impossible so you *never* re-crawl (the property you need). Back it with a persistent store so it survives restarts.

### Q3: "What is a crawler trap and how do you defend against it?"
**Hint:** Pages that generate infinite unique URLs — bottomless calendars (`?date=...`), session-ID links, deliberately hostile mazes. Defenses: max crawl **depth**, max URL **length**, a per-domain **page cap**, and detecting near-duplicate content (SimHash) so a trap that emits the "same" page under new URLs gets pruned.

### Q4: "How do you keep the index fresh?"
**Hint:** Estimate each page's **change frequency** from its history (a news homepage changes hourly; an archive page never does). Store `lastCrawled` and an estimated change rate; schedule re-crawls adaptively and raise priority for frequently-changing, important pages. Do not re-crawl everything uniformly — it wastes most of your budget on static pages.

### Q5: "A worker crashes mid-crawl. What happens to its URLs?"
**Hint:** Two problems. (1) **In-flight URLs** are lost unless the frontier is **durable** — persist the queue (e.g. on a message queue / durable log) so a URL isn't removed until the page is successfully stored. (2) The dead worker's **host partitions** must be reassigned to surviving workers — consistent hashing minimises how many hosts move, and the new owner resumes their back queues.

---

## Practice exercise

**Build a mini polite crawler in Node.js (~30–40 min).**

Write a single-process crawler that:
1. Takes 2–3 **seed URLs** on the same domain (use a site you control, or `example.com`).
2. Maintains an in-memory **frontier** (a simple array is fine) and a **Bloom filter** (or a `Set` for the mini version) for URL dedup.
3. Enforces a **per-host delay** of at least 1 second between fetches to the same host (track `lastFetchTime[host]`).
4. Fetches each page, extracts `href` links with a regex or a parser, filters to same-domain links, dedups them, and pushes new ones onto the frontier.
5. Stops after **50 pages** or **depth 3**, whichever comes first (your trap defense).
6. Prints, for each page: the URL, its depth, byte size, and how many *new* links it discovered.

**Produce:** the working script plus a short note answering — *what breaks first if you remove the per-host delay, and what breaks first if you remove the dedup?* (Answers: you DoS the host; you loop forever on a trap or cycle.)

---

## Quick reference cheat sheet

- **Crawler = distributed BFS over the web graph**, done politely, at scale, without falling into traps.
- **The 5 loop stages:** Frontier → Fetcher → Parser → Dedup → Store, with new URLs looping back to the Frontier.
- **URL Frontier** is the heart: two-level design — **front queues = priority**, **back queues = one per host = politeness**, plus a min-heap of "not-before" times.
- **Shard the frontier by HOST hash** (consistent hashing) so one worker owns a domain and enforces politeness locally. This is *the* distributed-politeness insight.
- **Politeness is mandatory:** robots.txt (cached), `crawl-delay`, descriptive User-Agent, honour 429/`Retry-After`, exponential backoff. Without it you are a DoS attack.
- **URL dedup = Bloom filter** — rare false positives (skip a new URL), never false negatives (never re-crawl). ~1.2 GB for 1B URLs vs ~100 GB exact.
- **Content dedup = content hash** (exact) **+ SimHash** (near-duplicates) — mirrors, session-IDs, print versions.
- **Storage:** raw pages → blob store [78]; metadata/state → distributed DB; link graph → for ranking; text → search index [79] via a queue [67].
- **Crawler traps:** infinite calendars, session-ID mazes. Defend with max depth, max URL length, per-domain caps, near-dup detection.
- **DNS is a hidden bottleneck** — resolving billions of hostnames is expensive; cache aggressively.
- **Capacity anchors:** 1B pages/month ≈ **400 pages/s** sustained; **100 TB/month** → **PB/year**; ~20 fetcher machines.
- **Durability:** make the frontier persistent so a worker crash does not lose in-flight URLs.

---

## Deep dives

### Deep dive A — Node.js polite fetcher (robots.txt + per-host delay)

This shows the politeness machinery: cache robots.txt per host, honour `crawl-delay` and `Disallow`, and never hit the same host faster than allowed.

```javascript
import { setTimeout as sleep } from 'node:timers/promises';

const USER_AGENT = 'MyCrawler/1.0 (+https://example.com/bot-info)';
const DEFAULT_DELAY_MS = 1000; // courteous default: max 1 req/s per host

class PoliteFetcher {
  constructor() {
    this.robotsCache = new Map(); // host -> { rules, crawlDelayMs, fetchedAt }
    this.lastHitAt = new Map();   // host -> timestamp of last request
  }

  // Fetch and cache robots.txt for a host (24h TTL). WHY cache: robots.txt
  // rarely changes, and re-fetching it for every URL would itself be impolite.
  async getRobots(host) {
    const cached = this.robotsCache.get(host);
    if (cached && Date.now() - cached.fetchedAt < 24 * 3600 * 1000) return cached;

    let rules = { disallow: [], crawlDelayMs: DEFAULT_DELAY_MS };
    try {
      const res = await fetch(`https://${host}/robots.txt`, {
        headers: { 'User-Agent': USER_AGENT },
      });
      if (res.ok) rules = this.parseRobots(await res.text());
    } catch {
      // No robots.txt or network error -> use polite defaults, do not assume "allow all aggressively"
    }
    const entry = { ...rules, fetchedAt: Date.now() };
    this.robotsCache.set(host, entry);
    return entry;
  }

  // Minimal robots.txt parser: honour Disallow paths and Crawl-delay for our UA / '*'.
  parseRobots(text) {
    const disallow = [];
    let crawlDelayMs = DEFAULT_DELAY_MS;
    let appliesToUs = false;
    for (const raw of text.split('\n')) {
      const line = raw.split('#')[0].trim();       // strip comments
      if (!line) continue;
      const [field, ...rest] = line.split(':');
      const key = field.trim().toLowerCase();
      const value = rest.join(':').trim();
      if (key === 'user-agent') {
        appliesToUs = value === '*' || USER_AGENT.startsWith(value);
      } else if (appliesToUs && key === 'disallow' && value) {
        disallow.push(value);
      } else if (appliesToUs && key === 'crawl-delay') {
        crawlDelayMs = Math.max(DEFAULT_DELAY_MS, Number(value) * 1000);
      }
    }
    return { disallow, crawlDelayMs };
  }

  isAllowed(rules, path) {
    return !rules.disallow.some((prefix) => path.startsWith(prefix));
  }

  // The polite fetch: robots check + per-host spacing + 429 handling.
  async fetch(urlString) {
    const url = new URL(urlString);
    const host = url.host;
    const rules = await this.getRobots(host);

    if (!this.isAllowed(rules, url.pathname)) {
      return { skipped: true, reason: 'disallowed by robots.txt' };
    }

    // Enforce per-host delay: wait until (lastHit + crawlDelay) has passed.
    const last = this.lastHitAt.get(host) ?? 0;
    const waitMs = last + rules.crawlDelayMs - Date.now();
    if (waitMs > 0) await sleep(waitMs);
    this.lastHitAt.set(host, Date.now());

    const res = await fetch(urlString, {
      headers: { 'User-Agent': USER_AGENT },
      redirect: 'follow',
    });

    // Honour rate limiting (recall [70 - Rate Limiting]): back off on 429.
    if (res.status === 429) {
      const retryAfter = Number(res.headers.get('retry-after')) || 60;
      this.lastHitAt.set(host, Date.now() + retryAfter * 1000); // push next hit out
      return { skipped: true, reason: `429, retry after ${retryAfter}s` };
    }

    // Guard against huge files: cap body size (trap/robustness defense).
    const len = Number(res.headers.get('content-length') || 0);
    if (len > 5_000_000) return { skipped: true, reason: 'file too large' };

    const contentType = res.headers.get('content-type') || '';
    if (!contentType.includes('text/html')) {
      return { skipped: true, reason: `non-HTML: ${contentType}` };
    }

    return { skipped: false, status: res.status, body: await res.text() };
  }
}

// --- demo ---
const fetcher = new PoliteFetcher();
const r1 = await fetcher.fetch('https://example.com/');
console.log('first fetch:', r1.skipped ? r1.reason : `${r1.body.length} bytes`);
const r2 = await fetcher.fetch('https://example.com/about'); // will wait ~1s (same host)
console.log('second fetch:', r2.skipped ? r2.reason : `${r2.body.length} bytes`);
```

### Deep dive B — Node.js Bloom filter for URL dedup

A compact Bloom filter: a bit array plus *k* hash functions. `add` sets *k* bits; `has` checks them. It answers "have I seen this URL?" in ~1.2 GB for a billion URLs instead of ~100 GB.

```javascript
import { createHash } from 'node:crypto';

class BloomFilter {
  // n = expected number of items, p = target false-positive rate.
  // WHY these formulas: they give the optimal bit-array size m and hash count k.
  constructor(n = 1_000_000, p = 0.01) {
    this.size = Math.ceil((-n * Math.log(p)) / (Math.log(2) ** 2)); // m bits
    this.numHashes = Math.max(1, Math.round((this.size / n) * Math.log(2))); // k
    this.bits = new Uint8Array(Math.ceil(this.size / 8)); // packed bit array
  }

  // Derive k independent hashes from two base hashes (double hashing).
  *indexes(item) {
    const h = createHash('sha256').update(item).digest();
    const h1 = h.readUInt32BE(0);
    const h2 = h.readUInt32BE(4);
    for (let i = 0; i < this.numHashes; i++) {
      yield ((h1 + i * h2) >>> 0) % this.size; // >>>0 keeps it an unsigned 32-bit int
    }
  }

  add(item) {
    for (const idx of this.indexes(item)) {
      this.bits[idx >> 3] |= 1 << (idx & 7); // set the bit
    }
  }

  // false positives possible (says seen when not), false negatives impossible.
  has(item) {
    for (const idx of this.indexes(item)) {
      if ((this.bits[idx >> 3] & (1 << (idx & 7))) === 0) return false; // definitely new
    }
    return true; // probably seen
  }
}

// Dedup gate used by the crawler: returns true only for genuinely-new URLs.
class UrlDedup {
  constructor() {
    this.seen = new BloomFilter(1_000_000, 0.01);
  }
  // Normalise first so http/https and trailing slashes don't count as different URLs.
  normalise(u) {
    const url = new URL(u);
    url.hash = '';                       // drop #fragments
    if (url.pathname !== '/' && url.pathname.endsWith('/')) {
      url.pathname = url.pathname.slice(0, -1);
    }
    return `${url.host}${url.pathname}${url.search}`;
  }
  isNew(u) {
    const key = this.normalise(u);
    if (this.seen.has(key)) return false; // already crawled (or rare false positive)
    this.seen.add(key);
    return true;
  }
}

// --- demo ---
const dedup = new UrlDedup();
console.log(dedup.isNew('https://example.com/page'));  // true  (new)
console.log(dedup.isNew('https://example.com/page/')); // false (same after normalise)
console.log(dedup.isNew('https://example.com/other')); // true  (new)
```

### Deep dive C — Node.js two-level URL frontier (priority + per-host politeness)

This is the heart of the design in code: front queues for **priority**, one back queue **per host** for politeness, and a min-heap picking whichever host is ready to be crawled soonest. It shows *why* one-host-one-queue makes per-host delays trivial to enforce.

```javascript
// A tiny binary min-heap keyed by "readyAt" time. WHY a heap: we constantly need
// "which host is allowed to be crawled next?" — a heap answers that in O(log n).
class MinHeap {
  constructor() { this.items = []; }
  get size() { return this.items.length; }
  push(item) {
    this.items.push(item);
    let i = this.items.length - 1;
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (this.items[p].readyAt <= this.items[i].readyAt) break;
      [this.items[p], this.items[i]] = [this.items[i], this.items[p]];
      i = p;
    }
  }
  pop() {
    const top = this.items[0];
    const last = this.items.pop();
    if (this.items.length) {
      this.items[0] = last;
      let i = 0;
      for (;;) {
        const l = 2 * i + 1, r = 2 * i + 2;
        let s = i;
        if (l < this.items.length && this.items[l].readyAt < this.items[s].readyAt) s = l;
        if (r < this.items.length && this.items[r].readyAt < this.items[s].readyAt) s = r;
        if (s === i) break;
        [this.items[s], this.items[i]] = [this.items[i], this.items[s]];
        i = s;
      }
    }
    return top;
  }
}

class UrlFrontier {
  // NUM_PRIORITIES front queues (0 = highest). One back queue per host.
  constructor({ numPriorities = 3, defaultDelayMs = 1000 } = {}) {
    this.frontQueues = Array.from({ length: numPriorities }, () => []);
    this.backQueues = new Map();     // host -> array of URLs (politeness lane)
    this.heap = new MinHeap();       // {host, readyAt} ordered by readyAt
    this.hostsInHeap = new Set();    // hosts currently scheduled
    this.defaultDelayMs = defaultDelayMs;
  }

  priorityOf(url) {
    // Toy heuristic: news/homepages are high priority, deep paths are low.
    if (url.pathname === '/' || url.host.includes('news')) return 0;
    return Math.min(this.frontQueues.length - 1, url.pathname.split('/').length - 1);
  }

  add(urlString) {
    const url = new URL(urlString);
    this.frontQueues[this.priorityOf(url)].push(urlString);
    this._drainFrontToBack();
  }

  // Move URLs from priority front queues into per-host back queues, biased to high priority.
  _drainFrontToBack() {
    for (let p = 0; p < this.frontQueues.length; p++) {
      while (this.frontQueues[p].length) {
        const urlString = this.frontQueues[p].shift();
        const host = new URL(urlString).host;
        if (!this.backQueues.has(host)) this.backQueues.set(host, []);
        this.backQueues.get(host).push(urlString);
        if (!this.hostsInHeap.has(host)) {      // schedule host as ready now
          this.heap.push({ host, readyAt: Date.now() });
          this.hostsInHeap.add(host);
        }
      }
    }
  }

  // Return the next URL that is polite to fetch right now, or null if none is ready yet.
  next() {
    if (this.heap.size === 0) return null;
    if (this.heap.items[0].readyAt > Date.now()) return null; // earliest host still cooling down
    const { host } = this.heap.pop();
    this.hostsInHeap.delete(host);
    const queue = this.backQueues.get(host);
    const urlString = queue.shift();
    if (queue.length) {                          // host has more work: reschedule after the delay
      this.heap.push({ host, readyAt: Date.now() + this.defaultDelayMs });
      this.hostsInHeap.add(host);
    }
    return urlString;
  }
}

// --- demo ---
const frontier = new UrlFrontier({ defaultDelayMs: 1000 });
frontier.add('https://news.site/');           // high priority
frontier.add('https://shop.site/deep/a/b/c');  // low priority, different host
frontier.add('https://news.site/article');     // same host as first
console.log(frontier.next()); // a news.site URL (ready now)
console.log(frontier.next()); // the shop.site URL (different host, also ready)
console.log(frontier.next()); // null — news.site is cooling down, shop already served
```

Notice the payoff: because each host has exactly one back queue and one heap entry, two URLs of the same host can **never** be handed out within the delay window. Shard this whole object by `hash(host)` across machines and each shard enforces politeness with zero cross-machine chatter.

### Deep dive D — Crawler traps, huge files, freshness, coordination, DNS

**(a) Crawler traps.** Some sites generate *infinite* unique URLs: a calendar with a "next month" link forever, or URLs carrying a fresh `?sessionid=` on every visit, or deliberately hostile mazes. Defenses, layered:
- **Max depth** — stop following links beyond N hops from a seed.
- **Max URL length** — trap URLs often grow monstrously long.
- **Per-domain page cap** — never fetch more than X pages from one host.
- **Near-duplicate detection** (SimHash) — a trap emitting the "same" page under new URLs gets pruned because the content fingerprint repeats.

**(b) Non-HTML and huge files.** Check `Content-Type` and `Content-Length` *before* downloading the whole body (the fetcher above does both). A 5 GB "page" or a video stream should be skipped, not buffered into memory — a classic way to crash a worker.

**(c) Freshness / re-crawl scheduling.** Store each page's `lastCrawled` time and an estimated **change rate** learned from history (did the content hash change between crawls?). Prioritise re-crawling pages that change often *and* matter (news, popular pages). This is **adaptive re-crawl** — the alternative, re-crawling everything on a fixed schedule, wastes almost your entire budget on pages that never change.

**(d) Distributed coordination.** Workers share the frontier and dedup state without stepping on each other because the frontier is **partitioned by host** — worker A owns `a.com`, worker B owns `b.com`, so their back queues never overlap and politeness stays local. The Bloom filter can be sharded the same way (each shard owns its hosts' URLs) or run as a shared service. When a worker **dies**, its host partitions are reassigned via consistent hashing (only that worker's hosts move), and the new owner rebuilds those back queues from the durable frontier.

**(e) DNS as a bottleneck.** Every fetch needs the host's IP. Resolving *billions* of distinct hostnames through the OS resolver is slow and can overwhelm DNS servers. **Cache DNS aggressively** — a per-crawler DNS cache with a sensible TTL turns most lookups into instant memory hits (recall **[54 — DNS](./54-dns-deep-dive.md)** for how resolution works). Without this cache, DNS latency alone caps your throughput well below the 400 pages/s target.

---

## Bottlenecks and failure modes

| Bottleneck / failure | Why it hurts | Mitigation |
|---|---|---|
| **DNS resolution** | Billions of lookups; each fetch blocks on one | Aggressive DNS cache with TTL; batch/pre-resolve |
| **Politeness caps throughput** | Per-host delay limits pages/s from any one site | Crawl *many* hosts in parallel; the limit is per-host, not global |
| **Frontier grows unbounded** | Every page adds more links than you remove | Priority + dedup + depth/URL caps; drop low-value URLs; bound queue sizes |
| **Worker crash loses in-flight URLs** | A URL popped but not yet stored vanishes | Make the frontier **durable** (persist/log; ack only after store); reassign host partitions |
| **Crawler traps** | Infinite URL generation exhausts the frontier | Depth/length/per-domain caps + near-dup detection |
| **Hot-host skew** | One giant site (e.g. a forum) dominates one shard | Split very large hosts across sub-partitions; per-domain page caps |
| **Duplicate content floods storage** | Mirrors and session-IDs store the same page many times | Content hash + SimHash dedup before storing/indexing |
| **Slow/malicious servers** | A 60 s hang ties up a worker | Aggressive timeouts; size caps; move on |

**The one-line summary:** a web crawler is a distributed BFS whose entire difficulty is doing the traversal *politely* (per-host rate limiting via host-partitioned frontiers), *without repeating work* (Bloom-filter URL dedup + content/SimHash dedup), and *without dying* (trap defenses, size caps, durable frontier, DNS caching). Nail those four and you have designed a real crawler.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [67 — Message Queues](./67-message-queues.md) | The frontier and the crawler→indexer handoff ride on durable queues; understand them first |
| **Next** | [79 — Search Systems](./79-search-systems.md) | The crawler's whole purpose is often to feed a search index — see what happens to the text next |
| **Related** | [78 — Blob Storage](./78-blob-storage.md) | Where the petabytes of raw crawled pages actually live |
| **Related** | [74 — Consistent Hashing](./74-consistent-hashing.md) | How the frontier is partitioned by host so politeness stays local and worker failures are cheap |
| **Related** | [70 — Rate Limiting](./70-rate-limiting.md) | Politeness *is* per-host rate limiting; honouring 429/`Retry-After` reuses these ideas |
