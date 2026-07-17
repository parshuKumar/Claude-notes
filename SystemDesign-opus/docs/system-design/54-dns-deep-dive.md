# 54 — DNS — Domain Name System Deep Dive
## Category: HLD Components

---

## What is this?

DNS (Domain Name System) is the internet's phone book. Humans remember names like `netflix.com`; machines can only talk to numbers like `54.237.226.164`. DNS is the global, distributed lookup system that turns one into the other.

Think of it like calling a friend. You don't memorise their 10-digit number — you type "Mom" into your contacts app, and the phone silently looks up the digits. DNS is that contacts app, except it's shared by the entire planet, it's split across thousands of servers run by different organisations, and nobody owns the whole thing.

---

## Why does it matter?

**If you don't understand DNS, three things will bite you in real work:**

1. **Deploys and migrations go wrong.** You move your app to a new server, change the DNS record, and half your users still hit the old (now dead) box for hours. You'll blame "propagation" — and you'll be wrong about why.
2. **You'll reach for DNS as a load balancer and get burned.** DNS looks like a free global load balancer. It is — a bad one. It has no idea whether your server is alive.
3. **You'll debug the wrong layer.** "The site is down" is a DNS problem surprisingly often. `curl` works from one machine and fails on another; the difference is a stale cache 3 hops away.

**Interview angle:** DNS shows up in almost every HLD interview as the *first hop*. When the interviewer says "walk me through what happens when a user types `example.com` into the browser," the first 60 seconds of your answer is DNS. It's also the entry point to CDNs, geo-routing, and multi-region failover — you cannot discuss those without DNS.

**Real-work angle:** every cutover, every blue-green deploy, every "point the domain at Cloudflare" ticket, every SPF/DKIM email-deliverability bug is DNS.

---

## The core idea — explained simply

### The Library Card Catalogue Analogy

Imagine a colossal library with no single index. Instead, the lookup is delegated in layers.

You walk to the front desk and say: *"I need the book `www.example.com`."*

The librarian at the front desk (your **recursive resolver**) doesn't know where it is. But she knows how to find out — and she's willing to do all the walking for you. That's the deal: **you ask once, she does the legwork.**

She walks to the **Master Index** at the centre of the library (the **root nameserver**). The Master Index doesn't store book locations. It only says: *"Books ending in `.com` are in the East Wing. Go ask the East Wing desk."*

She walks to the **East Wing desk** (the **`.com` TLD nameserver**). The East Wing desk doesn't store book contents either. It says: *"Anything belonging to `example` is on Shelf 12, managed by their own private librarian."*

She walks to **Shelf 12's private librarian** (the **authoritative nameserver** for `example.com`). *This* librarian actually owns the data. She says: *"`www.example.com` is at position `93.184.216.34`."*

The front-desk librarian walks back to you and hands you the answer. She also **writes it on a sticky note** and keeps it at her desk for the next hour, so the next person who asks gets it instantly. That sticky note is the **cache**, and the hour is the **TTL**.

### Mapping the analogy back

| In the analogy | In DNS | What it actually does |
|---|---|---|
| You, at the front desk | The stub resolver on your OS / browser | Asks **one** question, waits for **one** final answer |
| The front-desk librarian | **Recursive resolver** (ISP, 8.8.8.8, 1.1.1.1) | Does the multi-step legwork on your behalf; caches results |
| The Master Index | **Root nameserver** (`.`) | Knows only which server handles each TLD |
| East Wing desk | **TLD nameserver** (`.com`, `.org`, `.io`) | Knows only which nameserver is authoritative for each domain |
| Shelf 12's private librarian | **Authoritative nameserver** (Route 53, Cloudflare) | Owns the real records. The source of truth |
| Sticky note on the desk | **Cache** | Avoids repeating the whole walk |
| "Keep the note for 1 hour" | **TTL** (time-to-live) | How long a cached answer stays valid |

**The single most important sentence in this doc:** *you* make **one recursive** query. The **resolver** makes several **iterative** queries. Get this straight and the rest of DNS falls into place.

---

## Key concepts inside this topic

### 1. The full resolution walkthrough — step by step

You type `www.example.com` into Chrome. Here is every single thing that happens, in order.

**Step 0 — Browser cache.** Chrome keeps its own tiny in-process DNS cache (you can see it at `chrome://net-internals/#dns`). If `www.example.com` is in there and not expired, we are done in ~0 ms. No network traffic at all.

**Step 1 — OS cache (stub resolver).** Your operating system keeps a cache too (`systemd-resolved` on Linux, the DNS Client service on Windows, `mDNSResponder`/`discoveryd` on macOS). Hit? Done in ~0 ms.

**Step 2 — `/etc/hosts`.** A plain text file on your machine that hard-codes name→IP mappings. It **wins over everything on the network**. This is why `127.0.0.1 myapp.local` works with no DNS server anywhere. It's also why "DNS is broken on my laptop only" is sometimes a stale line in `/etc/hosts` that a colleague added six months ago.

**Step 3 — Ask the recursive resolver.** Now we actually leave the machine. Your OS sends **one UDP packet on port 53** to whatever resolver is configured (from DHCP: usually your ISP's; or you set it manually to Google's `8.8.8.8` or Cloudflare's `1.1.1.1`). The question is: *"Give me the A record for `www.example.com`. I'm not going to help. Come back with the final answer."* That flag — "please do the work for me" — is the **RD (Recursion Desired)** bit. **This is the ONE recursive query.**

**Step 4 — Resolver checks its own cache.** The resolver serves millions of users. `www.example.com` is probably already cached. If so, it answers in ~5-20 ms and we stop here. **This is the common case — the full walk below happens rarely.** On a cold cache, it continues.

**Step 5 — Resolver asks a root nameserver (ITERATIVE query #1).** There are 13 logical root server addresses (`a.root-servers.net` … `m.root-servers.net`), each actually hundreds of physical machines behind anycast. The resolver has these baked in as a "root hints" file — this is the one thing DNS has to know without asking. The root does **not** return the IP of `www.example.com`. It returns a **referral**: *"I don't know, but the `.com` nameservers are at these addresses."*

**Step 6 — Resolver asks the `.com` TLD nameserver (ITERATIVE query #2).** Run by Verisign. Again, no final answer — another **referral**: *"`example.com` is delegated to `ns1.exampledns.com` and `ns2.exampledns.com`."* These are the **NS records**.

**Step 7 — Resolver asks the authoritative nameserver (ITERATIVE query #3).** This is your DNS provider — Route 53, Cloudflare, whatever. It **owns** the zone file for `example.com`. It answers **authoritatively**: `www.example.com. 300 IN A 93.184.216.34`. The `300` is the TTL in seconds.

**Step 8 — Back down the chain, caching at every level.** The resolver caches the answer for 300 s and hands it to your OS. Your OS caches it. Your browser caches it. Now Chrome opens a TCP connection to `93.184.216.34:443`, and the *actual* request finally begins.

**Recursive vs iterative, stated plainly:**

| | Who does it | How many queries | "Come back with the answer, or a referral?" |
|---|---|---|---|
| **Recursive** | You → resolver | **1** | "Don't come back until you have the final answer." |
| **Iterative** | Resolver → root → TLD → authoritative | **3+** | "Give me the answer, or tell me who to ask next." |

Root, TLD, and authoritative servers **refuse to do recursion**. They will never do the walk for you. If they did, anyone could use a root server as a free resolver and the internet would melt.

**Cost of a cold lookup:** roughly 20 ms (root) + 30 ms (TLD) + 40 ms (authoritative) ≈ **90-150 ms**. Warm cache: **0-20 ms**. That difference is why caching is the whole game.

### 2. Record types

A domain's records live in a **zone file** on the authoritative server. Each record has a **name**, a **type**, a **TTL**, and a **value**.

| Type | Maps | What it's for | Example |
|---|---|---|---|
| **A** | name → IPv4 | The workhorse. Points a hostname at a v4 address. | `api.example.com. 300 IN A 93.184.216.34` |
| **AAAA** | name → IPv6 | Same, for IPv6. ("Quad-A" — 4× bigger than A.) | `api.example.com. 300 IN AAAA 2606:2800:220:1::` |
| **CNAME** | name → another **name** | Alias. "Whatever `www` is, go look up `example.com` instead." | `www.example.com. 300 IN CNAME example.com.` |
| **MX** | domain → mail server | Where to deliver email for this domain. Has a **priority** (lower = preferred). | `example.com. 3600 IN MX 10 mail.example.com.` |
| **TXT** | name → arbitrary text | Domain verification (Google/AWS "prove you own this"), **SPF**, **DKIM**, **DMARC** — the anti-spam trio. | `example.com. IN TXT "v=spf1 include:_spf.google.com ~all"` |
| **NS** | zone → nameserver | **Delegation.** "The authoritative servers for this zone are…" This is what the TLD returns in step 6. | `example.com. 172800 IN NS ns1.exampledns.com.` |
| **SOA** | zone → metadata | "Start of Authority." One per zone. Primary NS, admin email, serial number, refresh/retry/expire timers, and the **negative caching TTL** (how long "this name does not exist" is cached). | `example.com. IN SOA ns1... admin... 2024011501 7200 3600 1209600 300` |
| **PTR** | IP → name | **Reverse DNS.** The opposite direction. Mail servers check it: "this connection claims to be from mail.example.com — does the IP agree?" A missing PTR is a top cause of email landing in spam. | `34.216.184.93.in-addr.arpa. IN PTR example.com.` |
| **SRV** | service → host + **port** | Locates a service *including its port*. Used by SIP, XMPP, Kubernetes internal DNS, Minecraft. | `_sip._tcp.example.com. IN SRV 10 60 5060 sip.example.com.` |
| **CAA** | domain → allowed CAs | "Only Let's Encrypt may issue certs for me." Stops a rogue CA from minting a valid cert for your domain. | `example.com. IN CAA 0 issue "letsencrypt.org"` |

#### Why you can't CNAME the root domain

This trips up **everyone**, so understand the mechanism, not just the rule.

The RFC says: **if a name has a CNAME, it can have NO other records.** A CNAME means "this name is entirely an alias — ignore everything else and go look up the target."

Now, the root of your zone (`example.com`, with no subdomain — called the **apex** or **naked domain**) is *required* to have an **SOA** record and **NS** records. That's what makes it a zone at all.

So: apex must have SOA + NS. CNAME forbids any other record. **Contradiction.** Therefore `example.com. IN CNAME anything` is illegal. But `www.example.com. IN CNAME …` is perfectly fine — `www` is not the apex and has no SOA.

This is a real problem, because you often *want* `example.com` to point at a load balancer whose IP changes (`my-lb-1234.elb.amazonaws.com`). Solutions:

- **ALIAS / ANAME / CNAME-flattening** — a *provider-side* hack. Route 53 calls it an **Alias record**; Cloudflare calls it **CNAME flattening**. The provider resolves the target itself and serves the resulting **A record** to the world. To the client it looks like a normal A record. Fully legal, because no CNAME ever leaves the server. It only works if your DNS provider supports it (Route 53, Cloudflare, DNSimple do; a raw BIND server does not).
- **Redirect** — just point the apex at a tiny server that 301s to `www.example.com`.

### 3. TTL and the caching trade-off

TTL is a number of **seconds**, attached to every record, that says: *"you may cache this answer for this long."* It is the single knob that trades **agility** against **cost and speed**.

| TTL | Failover speed | Query volume hitting your NS | Typical use |
|---|---|---|---|
| **60 s** | Fast — traffic moves within a minute | High — every client re-asks every minute | During a migration; health-checked failover records |
| **300 s (5 min)** | Good | Moderate | Sensible default for app records that might move |
| **3600 s (1 hr)** | Slow — an hour of stale traffic | Low | Stable web records |
| **86400 s (24 hr)** | Terrible — a full day of stale traffic | Tiny | NS records, MX records, things that never change |

Work the numbers. Say 1,000,000 users refresh your page hourly.

- **TTL = 3600 s:** each user's resolver caches for the full hour → roughly **1 lookup per resolver per hour**. With, say, 5,000 distinct resolvers worldwide, that's ~5,000 queries/hour ≈ **1.4 QPS** on your nameservers. Nothing.
- **TTL = 60 s:** each resolver must re-ask 60× more often → ~300,000 queries/hour ≈ **83 QPS**. Still fine for a managed provider, but you're paying per query (Route 53 charges ~$0.40 per million standard queries), and you've made yourself dependent on your DNS provider staying up minute-to-minute.

**The pre-migration trick — memorise this, it's a real interview answer:**

```
Day 0 (24h+ BEFORE the cutover):
  Change TTL from 3600 → 60. Change NOTHING else.
  Now WAIT at least the OLD TTL (1 hour) — ideally a full day.
  Why: caches out there still hold the record with the OLD 3600s TTL.
       You must wait for those to expire before the new 60s TTL is
       what everyone is honouring.

Day 1 (the cutover):
  Change the A record to the new IP.
  Worst-case staleness is now 60 seconds, not 1 hour.
  Watch traffic drain from old server → new server.

Day 2 (after it's stable):
  Raise TTL back to 3600 to cut query cost and latency.
```

The subtle part everyone misses: **lowering the TTL doesn't take effect immediately either.** Resolvers that already cached the record at TTL 3600 will hold it for up to an hour regardless. You are always waiting out the *previous* TTL.

### 4. "DNS propagation delay" is a misnomer

You'll hear "we changed the record, now we're waiting for it to propagate — up to 48 hours." This is wrong, and the wrongness matters.

**Nothing propagates.** There is no push. Your authoritative server does not broadcast the change to anyone. The moment you save the record, your authoritative nameserver is serving the new answer to anyone who asks it, immediately, worldwide.

What actually happens: thousands of resolvers around the world are holding a **cached copy** of the *old* answer, each with its own countdown clock. They will not ask you again until their clock hits zero. So:

> **You are not waiting for something to spread. You are waiting for cached copies to EXPIRE.**

This reframing gives you real predictive power:

- **Maximum stale window ≈ the TTL that was in effect when the record was cached.** Not 48 hours — unless your TTL was 48 hours.
- Different users switch over at **different times**, because each resolver's clock started when *it* first cached. This is why "it works for me but not for my boss."
- **You can't force it.** You can flush *your own* caches (`ipconfig /flushdns`, `sudo dscacheutil -flushcache`, `sudo systemd-resolve --flush-caches`) but you cannot flush Comcast's resolver in Ohio.
- The "48 hours" folklore comes from **NS record** changes at the registrar (changing DNS *providers*), where TLD-level NS records legitimately have 24-48 h TTLs. For a routine A-record change, it's just your TTL.
- Some resolvers are badly behaved and ignore short TTLs, enforcing a floor (often 30-60 s, occasionally much worse). Plan for a long tail.

Use a checker like `dig @8.8.8.8 example.com` and `dig @1.1.1.1 example.com` to see what different resolvers currently believe. If they disagree, one of them is just holding an older cache entry.

### 5. DNS-based load balancing and geo-routing

Because the authoritative server chooses what to answer, it can answer *differently* per query. That makes DNS a crude but globally-distributed traffic director.

**Round-robin DNS** — return multiple A records for the same name, rotating the order:

```
api.example.com. 60 IN A 10.0.0.1
api.example.com. 60 IN A 10.0.0.2
api.example.com. 60 IN A 10.0.0.3
```
The client typically tries the first one. Rotating the order roughly spreads load. It's free and needs zero infrastructure.

**GeoDNS** — answer based on where the *query came from*. A resolver in Frankfurt gets the Frankfurt IP; one in São Paulo gets the São Paulo IP. This is how a CDN sends you to a nearby edge.

**Latency-based routing** — the provider maintains a live map of measured latency between network regions, and returns whichever of your endpoints is *fastest* from the user's region (which is not always the geographically closest — network paths are weird).

**Weighted records** — "send 95% of queries to `v1`, 5% to `v2`." A canary deploy, implemented purely in DNS.

#### Now the weaknesses. These are the interview follow-up.

1. **Clients and resolvers cache aggressively and ignore you.** You set TTL=60; a resolver enforces a 5-minute floor; a Java JVM with the old default caches DNS **forever** for the process lifetime; a browser holds its own copy. Your beautifully-weighted 95/5 split is being applied only to *cache misses*, not to actual traffic. **DNS controls where new lookups go, not where existing connections go.**
2. **DNS has no health awareness by itself.** A plain A record is a static string. If that server catches fire, DNS will keep cheerfully handing out its IP to everyone who asks, forever. There is no feedback loop. (Managed providers bolt health checks on top — see §7 — but vanilla DNS has none.)
3. **No session affinity.** DNS returns an IP. It has no concept of "this user was on server 3, keep them there." Anything stateful must be handled above DNS (sticky sessions at an L7 load balancer, or shared session state in Redis).
4. **Granularity is the resolver, not the user.** GeoDNS sees the *resolver's* IP, not the user's. If a user in Mumbai uses a resolver in Virginia, they get sent to Virginia. (The EDNS Client Subnet extension partially fixes this by passing a truncated client subnet along — but not every resolver sends it, and privacy-focused ones like 1.1.1.1 deliberately don't.)

**So the standard architecture is: use DNS for coarse, global, regional steering; use a real load balancer for fine-grained, health-aware, per-request steering within a region.**

### 6. Anycast

**Unicast:** one IP = one machine. Packets to `93.184.216.34` go to exactly one box in one datacentre.

**Anycast:** one IP is **announced from many locations at once** via BGP (the internet's routing protocol). The network itself decides which one you reach — normally the topologically nearest. `1.1.1.1` is announced from 300+ cities; the `1.1.1.1` you reach in Delhi is a completely different physical machine from the one someone reaches in Toronto. Same IP. No DNS trickery involved — this happens at the **routing** layer, below DNS.

Why it's brilliant here:

- **Latency:** you always land on a nearby instance.
- **DDoS absorption:** an attack from Brazil is absorbed by the Brazilian sites; it never touches the rest.
- **Failover for free:** if a site goes down, it withdraws its BGP announcement and routing reconverges on the next-nearest site. No cache to expire, no TTL to wait for. **This is anycast's superpower over DNS-based failover.**

**How it's used:** all 13 root nameservers are anycast (that's how 13 addresses serve the planet). Google's `8.8.8.8` and Cloudflare's `1.1.1.1` are anycast. CDNs anycast their edge IPs. The trade-off: it's poor for long-lived stateful TCP connections (a routing change mid-connection can land your packets on a different machine that knows nothing about your session), so it's ideal for short, stateless UDP exchanges — which is exactly what DNS is.

### 7. DNS in practice, and security

**AWS Route 53.** Authoritative DNS with routing policies: `Simple`, `Weighted`, `Latency`, `Geolocation`, `Failover`, `Multivalue`. Its **Alias record** solves the apex-CNAME problem and is free to query when pointing at AWS resources. Route 53 **health checks** poll your endpoint (e.g. `GET /health` every 30 s from multiple regions); if it fails N times, Route 53 **stops returning that record** — this is health-checked DNS failover, the missing feedback loop from §5.2. Note the failover is still bounded below by your TTL.

**Cloudflare.** Anycast authoritative DNS, consistently among the fastest measured. Provides **CNAME flattening** at the apex. In proxied ("orange cloud") mode it doesn't return *your* IP at all — it returns a **Cloudflare anycast IP**, and Cloudflare proxies to your origin. Your real server IP is hidden, DDoS is absorbed at the edge, and you can move your origin without touching public DNS at all.

**Security, briefly:**

- **DNS spoofing / cache poisoning.** DNS is UDP and (classically) unauthenticated. An attacker races the real authoritative server, forging a reply that says `yourbank.com → attacker-ip`. If the resolver accepts it, it caches the lie and serves it to every user behind that resolver until the TTL expires. The **Kaminsky attack** (2008) made this dramatically easier; the emergency mitigation was **source-port randomisation** (the attacker must now guess a 16-bit query ID *and* a 16-bit source port).
- **DNSSEC.** Cryptographically **signs** DNS records, so a resolver can verify the answer really came from the zone owner and wasn't tampered with. Trust chains from the root down (root signs `.com`'s key, `.com` signs `example.com`'s key). Note: DNSSEC provides **authenticity and integrity, not confidentiality** — the query is still in plaintext. Adoption is mediocre because it's operationally fiddly (a botched key rollover takes your domain off the internet).
- **DNS over HTTPS (DoH) / DNS over TLS (DoT).** Solves the *other* half: **encrypts** the query between you and your resolver, so your ISP or the coffee-shop Wi-Fi can't read or modify which sites you're visiting. DoH runs on port 443 and looks like normal HTTPS. Firefox and Chrome ship it. Complementary to DNSSEC, not a replacement: **DoH hides the query in transit; DNSSEC proves the answer is genuine.**

### 8. DNS in Node.js — `dns.lookup` vs `dns.resolve`

This is a genuine production footgun, and it's a great thing to know in an interview.

```js
import dns from 'node:dns';
import { promises as dnsPromises } from 'node:dns';

// ── dns.lookup() ─────────────────────────────────────────────────
// Calls the OPERATING SYSTEM's resolver (getaddrinfo). It honours
// /etc/hosts, /etc/nsswitch.conf, mDNS, the OS cache — everything a
// normal program would see. This is what http.request(), fetch(),
// and every npm client library use under the hood.
//
// THE TRAP: getaddrinfo is a BLOCKING C call. libuv cannot do it
// async, so it runs it on the libuv THREADPOOL — which has only
// 4 threads by default (UV_THREADPOOL_SIZE). The same pool is used
// by fs, crypto.pbkdf2, and zlib.
// A slow DNS server can therefore stall your file reads and hashing.
const addr = await dnsPromises.lookup('example.com', { family: 4 });
console.log(addr); // { address: '93.184.216.34', family: 4 }

// ── dns.resolve4() ───────────────────────────────────────────────
// Performs a REAL DNS QUERY over the network (via c-ares), talking
// directly to a resolver. It IGNORES /etc/hosts and the OS cache.
// It is genuinely async — no threadpool involved.
// Use it when you want DNS truth, not "what the OS would do".
const ips = await dnsPromises.resolve4('example.com');
console.log(ips); // [ '93.184.216.34' ]

// resolve4 with TTLs — lookup() can never give you this.
dns.resolve4('example.com', { ttl: true }, (err, records) => {
  if (err) throw err;
  console.log(records); // [ { address: '93.184.216.34', ttl: 300 } ]
});
```

| | `dns.lookup()` | `dns.resolve*()` |
|---|---|---|
| Mechanism | OS resolver (`getaddrinfo`) | Real DNS query (c-ares) |
| Honours `/etc/hosts` | **Yes** | **No** |
| Honours OS cache | Yes | No |
| Async? | Fake — blocks a **libuv threadpool** thread | Truly async, no threadpool |
| Can return TTL | No | **Yes** (`{ ttl: true }`) |
| Can query MX/TXT/NS | No | Yes (`resolveMx`, `resolveTxt`, `resolveNs`) |
| Used by `fetch`/`http` | **Yes** | No |

**A practical DNS cache to avoid stalling the threadpool.** Node does **not** cache DNS results itself — every outbound HTTP request re-resolves. Under load that hammers the threadpool.

```js
import { promises as dnsPromises } from 'node:dns';

/**
 * Caches A-record lookups and respects the record's own TTL.
 * Why this exists: Node caches nothing, so a high-QPS service
 * issues one getaddrinfo per outbound request and saturates the
 * 4-thread libuv pool. In production, use the `cacheable-lookup`
 * package — this is here to show the mechanism.
 */
class DnsCache {
  #cache = new Map(); // hostname -> { addresses, expiresAt }

  async resolve(hostname) {
    const hit = this.#cache.get(hostname);
    if (hit && hit.expiresAt > Date.now()) return hit.addresses;

    // ttl: true is why we use resolve4 and not lookup — the TTL is
    // the server's own instruction about how long we may cache.
    const records = await dnsPromises.resolve4(hostname, { ttl: true });
    if (records.length === 0) throw new Error(`No A record for ${hostname}`);

    // Honour the SHORTEST ttl in the set; clamp so a 1s TTL from a
    // misconfigured zone can't turn the cache off entirely.
    const ttl = Math.max(30, Math.min(...records.map((r) => r.ttl)));

    const addresses = records.map((r) => r.address);
    this.#cache.set(hostname, { addresses, expiresAt: Date.now() + ttl * 1000 });
    return addresses;
  }

  /** Poor man's round-robin across the returned A records. */
  async pickOne(hostname) {
    const addresses = await this.resolve(hostname);
    return addresses[Math.floor(Math.random() * addresses.length)];
  }
}

// Inspect other record types — handy for debugging mail and verification.
async function inspectDomain(domain) {
  const safe = (p) => p.catch(() => []); // NXDOMAIN for a type is normal
  const [mx, txt, ns] = await Promise.all([
    safe(dnsPromises.resolveMx(domain)),
    safe(dnsPromises.resolveTxt(domain)),
    safe(dnsPromises.resolveNs(domain)),
  ]);
  return {
    mail: mx.sort((a, b) => a.priority - b.priority).map((r) => `${r.priority} ${r.exchange}`),
    spf: txt.flat().filter((t) => t.startsWith('v=spf1')),
    nameservers: ns,
  };
}

const cache = new DnsCache();
console.log(await cache.pickOne('example.com'));
console.log(await inspectDomain('example.com'));
```

---

## Visual / Diagram description

### Diagram 1 — The full resolution chain

```
                      ┌──────────────────────────────┐
                      │   YOU: browser types         │
                      │   www.example.com            │
                      └──────────────┬───────────────┘
                                     │
        ┌────────────────────────────▼───────────────────────────┐
        │  LOCAL CACHES (checked in this order, all ~0 ms)       │
        │  ┌───────────┐   ┌───────────┐   ┌─────────────────┐   │
        │  │ 1 Browser │──▶│ 2 OS      │──▶│ 3 /etc/hosts    │   │
        │  │   cache   │   │   cache   │   │   (beats all!)  │   │
        │  └───────────┘   └───────────┘   └─────────────────┘   │
        └────────────────────────────┬───────────────────────────┘
                                     │  MISS
                        ┌────────────▼────────────┐
                        │  4  ONE RECURSIVE QUERY │
                        │     (UDP :53, RD=1)     │
                        └────────────┬────────────┘
                                     ▼
   ┌─────────────────────────────────────────────────────────────┐
   │        RECURSIVE RESOLVER   (ISP / 8.8.8.8 / 1.1.1.1)       │
   │        ┌──────────────────────────────────────┐             │
   │        │ 5  Own cache?  HIT → return in ~10ms │             │
   │        └──────────────────────────────────────┘             │
   │                        │ MISS → do the walk                 │
   └───┬─────────────────┬───────────────────┬────────────────▲──┘
       │ iterative #1    │ iterative #2      │ iterative #3   │
       │                 │                   │                │
       ▼                 ▼                   ▼                │
 ┌───────────┐    ┌─────────────┐    ┌────────────────────┐   │
 │  6 ROOT   │    │  7 TLD      │    │  8 AUTHORITATIVE   │   │
 │    "."    │    │   ".com"    │    │  ns1.example.com   │   │
 │           │    │  (Verisign) │    │  (Route53/CF)      │   │
 ├───────────┤    ├─────────────┤    ├────────────────────┤   │
 │ "ask .com │    │ "ask ns1.   │    │ "A 93.184.216.34,  │───┘
 │  at x.x"  │───▶│  example"   │───▶│  TTL 300"          │
 │ (referral)│    │ (referral)  │    │  ★ REAL ANSWER ★   │
 └───────────┘    └─────────────┘    └────────────────────┘
   ~20 ms            ~30 ms                 ~40 ms

 Answer travels BACK DOWN, and is CACHED at every level:
   resolver (300s) → OS (300s) → browser (300s) → app
 Cold total ≈ 90-150 ms. Warm total ≈ 0-15 ms.
```

**What it shows.** The left-to-right walk is what makes DNS *hierarchical*: nobody holds the whole map. Root knows only TLDs; TLD knows only delegations; only the authoritative server knows the real value. The vertical split matters too — **everything above the "ONE RECURSIVE QUERY" line is your machine doing nothing but cache checks; everything below it is the resolver doing all the work.** Notice the three iterative arrows all originate from the resolver, never from you.

### Diagram 2 — Anycast: one IP, many machines

```
       Users in Delhi          Users in Toronto        Users in Berlin
             │                        │                       │
             │  "connect 1.1.1.1"     │ "connect 1.1.1.1"     │ "connect 1.1.1.1"
             ▼                        ▼                       ▼
     ┌───────────────────────────────────────────────────────────────┐
     │            BGP  —  the internet's routing layer                │
     │   "1.1.1.1 is announced from Delhi, Toronto, Berlin, +300"    │
     │            Each router picks its SHORTEST path.                │
     └───┬──────────────────────┬──────────────────────────┬─────────┘
         ▼                      ▼                          ▼
   ┌───────────┐         ┌────────────┐            ┌────────────┐
   │  POP:DEL  │         │  POP:YYZ   │            │  POP:BER   │
   │ 1.1.1.1   │         │  1.1.1.1   │            │  1.1.1.1   │
   │  ~4 ms    │         │   ~6 ms    │            │   ~3 ms    │
   └───────────┘         └────────────┘            └────────────┘
        SAME IP.  DIFFERENT PHYSICAL BOX.  No DNS involved.

   If POP:BER dies → it withdraws its BGP announcement →
   Berlin traffic reroutes to Frankfurt in SECONDS.
   No TTL. No cache to expire. THIS is why anycast beats
   DNS-failover for speed.
```

**What it shows.** Anycast solves failover *below* DNS, at the routing layer, so caching cannot defeat it. DNS-based failover is always hostage to the TTL; anycast is not. Draw this next to Diagram 1 on a whiteboard and you've explained why root servers and CDN edges use anycast rather than clever DNS.

---

## Real world examples

### Cloudflare — DNS as the front door of the whole product

When you put a domain "behind" Cloudflare in proxied mode, Cloudflare's authoritative DNS does **not** return your origin server's IP. It returns a **Cloudflare anycast IP** (from ranges like `104.16.0.0/12`). The user's TCP connection therefore terminates at the nearest Cloudflare edge PoP, which then proxies to your origin. Three consequences fall out of that one design choice: your origin IP is never public (DDoS can't target it directly), TLS terminates at the edge (so caching and WAF are possible), and you can **move your origin server without changing any public DNS record at all** — you just update the origin address inside Cloudflare. Cloudflare also implements **CNAME flattening**, so an apex `example.com` can point at `my-app.herokudns.com` despite the RFC prohibition.

### AWS Route 53 — health-checked, latency-based DNS failover

Route 53 is authoritative DNS with a control plane bolted on. A common production pattern: two ALBs, one in `us-east-1` and one in `eu-west-1`, both fronted by **latency-based routing records** for `api.example.com` with a **60 s TTL**. Route 53 **health checkers** in multiple regions poll each ALB's `/health` endpoint. If `us-east-1` starts failing, Route 53 removes that record from its answers, and within roughly one TTL (plus the health-check detection window, typically ~30-90 s) every new lookup gets the EU address. Route 53's **Alias** records also solve the apex problem natively, and Alias queries to AWS targets are free. Note that Route 53's authoritative servers are themselves anycast — you cannot build a highly-available system on a non-HA DNS layer.

### The root nameservers — 13 names, hundreds of sites, one anycast trick

There are exactly **13 root server addresses** (`a.` through `m.root-servers.net`) — a limit inherited from the old 512-byte UDP DNS packet size. Yet the root serves the entire planet. The resolution: each of those 13 addresses is **anycast** from many physical sites (the `l.root-servers.net` operated by ICANN alone runs from 100+ locations). They are run by 12 independent organisations (Verisign, ICANN, NASA, universities, RIPE NCC, and others) — deliberately, so no single entity can take the internet's naming layer down. They answer *only* referrals: they never resolve anything for you, they just point at the TLD.

---

## Trade-offs

### Low TTL vs high TTL

| | Low TTL (60 s) | High TTL (24 h) |
|---|---|---|
| Failover / migration speed | Fast — minutes | Slow — up to a day of stale traffic |
| Query load on nameservers | High (and you pay per query) | Very low |
| Average lookup latency for users | Higher — more cache misses | Lower — mostly cache hits |
| Resilience if your DNS provider goes down | Poor — caches expire fast, users break quickly | Good — cached answers keep working for hours |
| Best for | Records that might move; failover records | NS, MX, and records that never change |

### DNS-based load balancing vs a real load balancer

| | DNS LB (round-robin / GeoDNS) | L4/L7 load balancer (ALB, NGINX) |
|---|---|---|
| Cost | Nearly free | Runs 24/7, costs money |
| Scope | **Global** — steers across regions | **Regional** — steers within one region |
| Health awareness | None by itself (managed providers bolt it on) | Native, per-request, sub-second |
| Reaction time | Bounded by TTL + client cache | Immediate |
| Session affinity | None | Yes (sticky sessions) |
| Granularity | Per *resolver*, per lookup | Per *request* |
| Can it be defeated by caching? | **Yes, constantly** | No |

### Anycast vs DNS failover

| | Anycast | DNS failover |
|---|---|---|
| Where it operates | Routing layer (BGP) | Naming layer |
| Failover speed | Seconds (BGP reconvergence) | TTL + health-check detection |
| Defeated by client caching? | **No** | Yes |
| Setup cost | High — needs your own AS + BGP peering, or a provider | Trivial — a checkbox |
| Good for long-lived TCP | Weak (route flaps can break sessions) | Fine |

**The sweet spot:** use DNS for **coarse global steering** (which region), with a **60-300 s TTL** on anything that might move and a **24 h TTL** on things that never do. Put a **real health-aware load balancer** behind it for per-request decisions inside a region. Never rely on DNS alone to take a dead server out of rotation — it will happily hand out a corpse's IP for as long as the TTL allows.

**Rule of thumb:** *before* any cutover, drop the TTL to 60 s a full day ahead; *after* it settles, raise it back.

---

## Common interview questions on this topic

### Q1: "What happens when you type `google.com` into a browser and hit Enter?"

**Hint:** The classic. The DNS half of the answer: browser cache → OS cache → `/etc/hosts` → **one recursive query** to the resolver → resolver checks its cache → if cold, **three iterative queries**: root (returns `.com` referral) → TLD (returns NS for google.com) → authoritative (returns the A record) → answer cached at every layer on the way back. Then: TCP handshake → TLS handshake → HTTP request. Explicitly say the words *recursive* and *iterative* and say **who** does each — that's the discriminator between a junior and a senior answer.

### Q2: "You changed your A record 20 minutes ago. Some users still hit the old server. Why, and what can you do?"

**Hint:** Because **nothing propagates** — resolvers around the world are holding cached copies, each with its own countdown that started when *it* first cached the record. The stale window is bounded by the TTL that was in force *at cache time*, not the new TTL. You cannot flush other people's resolvers. What you *can* do: keep the old server running as a "gravestone" until the old TTL has fully elapsed (never shut it off at cutover time), and next time, lower the TTL to 60 s a day before the change. Bonus: mention that some resolvers enforce a TTL floor and ignore short TTLs entirely, so plan for a long tail.

### Q3: "Why can't you put a CNAME on the root domain, and what do you do instead?"

**Hint:** The RFC says a name with a CNAME may have **no other records**. But the zone apex is *required* to have SOA and NS records. Contradiction → illegal. Instead use a provider-side **ALIAS/ANAME record** (Route 53 Alias) or **CNAME flattening** (Cloudflare): the provider resolves the target itself and serves a plain **A record** to the client, so no CNAME ever leaves the server. Or, redirect the apex to `www`.

### Q4: "Can I just use round-robin DNS as my load balancer?"

**Hint:** You can, but name the three failures: (1) **no health awareness** — DNS will keep returning a dead server's IP until you manually remove it; (2) **caching defeats your weights** — resolvers and clients (notably older JVM defaults) hold answers far longer than your TTL says, so your 95/5 split applies only to cache misses; (3) **no session affinity and no per-request control**. It also load-balances per *resolver*, not per user. Use it for coarse geographic steering; put a real health-checked load balancer underneath.

### Q5: "How would you build DNS failover across two regions?"

**Hint:** Authoritative DNS (Route 53) with **failover or latency-based routing records**, TTL at 60 s, plus **health checks** polling `/health` on each region's load balancer from multiple vantage points. On failure the record is withdrawn from answers; recovery time ≈ health-check detection (~30-90 s) + TTL (60 s) + client cache slop. Then state the honest limitation: this is **minutes, not seconds**, and clients with sticky caches lag further. If you need seconds, you need **anycast** — failover at the BGP layer, where no cache can defeat you.

---

## Practice exercise

### "The 3 a.m. Migration Plan"

You run `api.shopfast.com`, currently a single EC2 box at `54.10.20.30` in `us-east-1`, with an A record whose **TTL is 86400 (24 hours)**. You must move it to a new load balancer in `us-east-1` **and** add a warm standby in `eu-west-1`. Downtime budget: **zero**.

**Produce, in a single markdown file:**

1. **A dated cutover runbook.** Every DNS step, with the *exact* wait time between each and the reason for that wait. You must explain why you cannot just change the record now, and why lowering the TTL today does not take effect today.
2. **The zone file** (as text) for `shopfast.com` after the migration. Include: the apex `A`/ALIAS, `www` as a CNAME, `api` with **two latency-based records**, an `MX`, a `TXT` SPF record, and the `NS` records. Give each record a TTL and **justify each TTL in a comment**.
3. **The "gravestone" question, answered with a number.** How long must the old `54.10.20.30` box stay alive after you flip the record? Show the arithmetic.
4. **A failover analysis.** With a 60 s TTL and a health check that fires every 30 s and needs 3 consecutive failures, compute the **worst-case time** from "us-east-1 dies" to "the last user is served from eu-west-1". Then write one paragraph on why this number cannot be driven below ~1 minute with DNS alone, and what you'd use instead.
5. **Run the `DnsCache` class** from §8 against a real domain and paste the output, including the observed TTL. Then explain in two sentences why `dns.resolve4` gives you a TTL and `dns.lookup` cannot.

**Deliverable:** one file, `dns-migration-plan.md`, plus your `dns-cache.js`. Timebox: 30-40 minutes.

---

## Quick reference cheat sheet

- **DNS = phone book.** Names for humans, IPs for machines. Hierarchical and distributed — nobody holds the whole map.
- **One recursive query, several iterative queries.** *You* ask the resolver once. The *resolver* walks root → TLD → authoritative. Root/TLD/authoritative never do recursion for you.
- **The chain:** browser cache → OS cache → `/etc/hosts` → resolver cache → root → TLD → authoritative. Cached on the way back at every layer.
- **`/etc/hosts` beats the entire internet.** First place to look when one machine misbehaves.
- **Referral vs answer:** root and TLD return *referrals* ("ask them"). Only the **authoritative** server returns the real record.
- **Record types:** A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), TXT (SPF/verification), NS (delegation), SOA (zone metadata + negative-cache TTL), PTR (reverse), SRV (host+port), CAA (which CA may issue certs).
- **You cannot CNAME the apex** — a CNAME forbids other records, but the apex must have SOA + NS. Use **ALIAS** (Route 53) or **CNAME flattening** (Cloudflare).
- **TTL is the one knob:** low = fast failover, high query load; high = cheap and fast, slow to move traffic.
- **The pre-migration trick:** drop TTL to 60 s a **full day before** the cutover, wait out the old TTL, cut over, then raise it back.
- **"Propagation" is a myth.** Nothing is pushed. You are waiting for cached copies to **expire**. Bounded by the TTL that was in force *when it was cached*.
- **Keep the old server alive** for at least the old TTL after a cutover. Do not shut it off at flip time.
- **DNS load balancing is coarse and blind:** no health awareness, no session affinity, defeated by caching, per-resolver not per-user. Great for regional steering, useless as your only load balancer.
- **Anycast** = one IP announced from many places via BGP; the network routes you to the nearest. Failover in seconds, immune to DNS caching. Used by root servers, `1.1.1.1`, `8.8.8.8`, and every CDN.
- **Security:** cache poisoning is the attack; **DNSSEC** signs answers (authenticity, *not* privacy); **DoH/DoT** encrypts the query in transit (privacy, *not* authenticity). They are complementary.
- **Node:** `dns.lookup()` = OS resolver, honours `/etc/hosts`, **blocks a libuv threadpool thread**, used by `fetch`/`http`. `dns.resolve4()` = real DNS query, truly async, can return TTLs. Node caches **nothing** — add `cacheable-lookup` under load.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [53 — Load Balancers](./55-load-balancing.md) — DNS steers you *to* a region; the load balancer steers you *within* it. Read them as a pair. |
| **Next** | [55 — CDN — Content Delivery Networks](./60-cdn.md) — CDNs are built on GeoDNS and anycast; this doc is the mechanism, that doc is the product. |
| **Related** | [56 — Caching Strategies](./59-caching-in-depth.md) — TTL, cache invalidation, and stale reads are the same problems DNS has, one layer up. |
| **Related** | [57 — Reverse Proxy and API Gateway](./57-reverse-proxy.md) — the health-aware, per-request layer that DNS deliberately is not. |
