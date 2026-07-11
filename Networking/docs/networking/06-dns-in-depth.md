# 06 — DNS in Depth

> **Phase 2 — Topic 1 of 8**
> Prerequisites: `01-how-the-internet-works.md`, `04-ip-addresses.md`, `05-ports.md`

---

## ELI5 — The Simple Analogy

Imagine you want to call your friend Alice, but you only know her name — not her phone number. What do you do? You look her up in a phone book.

DNS (Domain Name System) is the internet's phone book. It turns names humans remember (`google.com`) into numbers machines need (`142.250.80.46`).

But it's not one giant book. It's more like a chain of librarians:

1. You ask your **local librarian** (your recursive resolver): "What's the number for `api.myapp.com`?"
2. The librarian doesn't know, so they ask the **head librarian of the whole world** (a root server): "Where do I find `.com` names?"
3. Head librarian says: "Go to the `.com` desk over there."
4. Librarian goes to the **`.com` desk**: "Where do I find `myapp.com`?"
5. `.com` desk says: "Ask `myapp.com`'s own private secretary" (the authoritative nameserver).
6. That secretary says: "`api.myapp.com` is `142.250.80.46`."
7. Your librarian hands you the number **and writes it on a sticky note** (caches it) so next time they answer instantly — until the sticky note expires (TTL).

The genius of DNS: no single phone book has to hold all 350+ million domains. The work is split across a tree of servers, and caching means you almost never have to walk the whole chain twice.

---

## Where This Lives in the Network Stack

DNS is an **Application-Layer (Layer 7)** protocol — the same layer as HTTP. But it's special: almost every other protocol depends on it *first*. Before your HTTP request can go anywhere, DNS has to turn the hostname into an IP.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, WebSocket, ▶ DNS ◀, SMTP)       │  ← DNS lives here
│  Layer 6 — Presentation  (TLS/SSL — DoT & DoH wrap DNS here)    │
│  Layer 5 — Session       (session management)                   │
│  Layer 4 — Transport     (▶ UDP (default) ◀, TCP (fallback))   │  ← DNS rides on port 53
│  Layer 3 — Network       (IP — the thing DNS is trying to find) │
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                      │
│  Layer 1 — Physical      (cables, fiber, radio)                 │
└─────────────────────────────────────────────────────────────────┘
```

The important thing to internalise: **DNS is a Layer 7 protocol whose entire job is to give you a Layer 3 address.** It's the bridge between human-readable names and the IP-based routing everything below depends on. It rides on **UDP port 53** by default and falls back to **TCP port 53** for large answers.

---

## What Is This?

DNS is a **globally distributed, hierarchical, cached database** that maps domain names to records (usually IP addresses, but many other things too).

Three properties make it work at planetary scale:

- **Hierarchical** — the namespace is a tree. Reading a domain right-to-left tells you the delegation path:
  ```
  api.myapp.com.
  │   │     │  └── "." = the root (usually invisible/implied)
  │   │     └────── ".com" = Top-Level Domain (TLD)
  │   └──────────── "myapp" = the second-level domain (the zone you own)
  └──────────────── "api" = a subdomain / hostname within your zone
  ```
- **Distributed** — no server holds the whole database. Root servers know where TLDs live. TLD servers know where each domain's authoritative servers live. Authoritative servers hold the actual records. Responsibility is *delegated* down the tree.
- **Cached** — every answer carries a TTL (Time To Live). Resolvers, operating systems, and browsers cache answers so the full chain is walked rarely. This is what makes DNS fast despite being distributed across the globe.

The pieces of the machine:

| Piece | Role | Example |
|-------|------|---------|
| **Stub resolver** | Tiny client baked into your OS/app; asks one question, expects a full answer | `getaddrinfo()`, Node's `dns` module |
| **Recursive resolver** | Does the legwork: walks the chain on your behalf, caches results | `8.8.8.8` (Google), `1.1.1.1` (Cloudflare), your ISP's resolver |
| **Root nameserver** | Top of the tree; points to TLD servers | 13 root "letters" a.root-servers.net → m.root-servers.net |
| **TLD nameserver** | Authoritative for a TLD; points to each domain's nameservers | Verisign runs `.com` |
| **Authoritative nameserver** | Holds the real records for a zone; the source of truth | `ns-1234.awsdns-56.org` (Route 53) |

---

## Why Does It Matter for a Backend Developer?

DNS is invisible when it works and catastrophic when it doesn't. You need to understand it because:

- **DNS changes are NOT instant.** You update an A record during a deploy, but users keep hitting the old server for minutes or hours because of TTL caching. This causes real outages. (See Example 2.)
- **DNS is a top cause of production incidents.** The 2016 Dyn attack took down Twitter, GitHub, Netflix, and Reddit — not by attacking those sites, but by attacking their DNS provider. If DNS is down, your perfectly healthy servers are unreachable.
- **You configure DNS constantly** — pointing a domain at a load balancer, setting up email (MX + SPF/DKIM TXT records), proving domain ownership for TLS certs (CAA, TXT), running blue/green deploys via DNS.
- **DNS latency is part of your API latency.** The first request to a cold hostname pays a full resolution walk. In a microservice mesh doing thousands of internal calls, resolver caching behaviour directly affects tail latency.
- **The CNAME-at-apex problem** bites nearly every developer once: you can't put a `CNAME` on `myapp.com` itself (only on subdomains), so you need ALIAS/ANAME records or a provider trick.
- **Load tests lie if you ignore resolver caching.** Your load generator resolves once and hammers a single IP, missing the DNS round-robin / GeoDNS behaviour real users get.

If you can read `dig` output and reason about TTL, you can diagnose a whole class of "the site is down but the servers are fine" incidents in minutes instead of hours.

---

## The Packet/Protocol Anatomy

A DNS message (query and reply share the same format) has a **12-byte header** followed by four variable sections. It travels inside a UDP datagram on port 53.

```
┌──────────────────────────────────────────────────────────────┐
│ ETHERNET / IP / UDP headers  (dst port 53)                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ DNS MESSAGE                                             │  │
│  │  ┌──────────────── HEADER (12 bytes) ────────────────┐  │  │
│  │  │ ID (16 bits)          │ transaction id, echoed back│  │  │
│  │  │ Flags (16 bits)       │ QR OPCODE AA TC RD RA RCODE│  │  │
│  │  │ QDCOUNT (16)          │ # of questions   (usually 1)│  │  │
│  │  │ ANCOUNT (16)          │ # of answer records         │  │  │
│  │  │ NSCOUNT (16)          │ # of authority records      │  │  │
│  │  │ ARCOUNT (16)          │ # of additional records     │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────── QUESTION SECTION ─────────────────┐   │  │
│  │  │ QNAME  = api.myapp.com   (labels, length-prefixed) │   │  │
│  │  │ QTYPE  = A (1) / AAAA (28) / MX (15) / ...          │   │  │
│  │  │ QCLASS = IN (1, "Internet")                        │   │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────── ANSWER SECTION ───────────────────┐   │  │
│  │  │ NAME  TYPE  CLASS  TTL  RDLENGTH  RDATA            │   │  │
│  │  │ api.myapp.com  A  IN  300  4  142.250.80.46         │   │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │  ┌──── AUTHORITY SECTION ──── (which NS is authoritative)│  │
│  │  ┌──── ADDITIONAL SECTION ─── (glue records, EDNS0 opt) │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### The flag bits (the part you actually read in `dig`)

```
 Bit  Field    Meaning
 ───  ──────   ───────────────────────────────────────────────
  QR    Query/Response      0 = query, 1 = response
  OPCODE (4 bits)           0 = standard query
  AA    Authoritative Answer 1 = this reply came from the authoritative server
  TC    TrunCated           1 = answer too big for UDP → retry over TCP
  RD    Recursion Desired   1 = "please do the full walk for me" (stub → resolver)
  RA    Recursion Available 1 = "yes, I do recursion" (resolver → stub)
  RCODE (4 bits)            response code: see table below
```

### RCODE — the DNS status code

| RCODE | Name | Meaning |
|-------|------|---------|
| 0 | NOERROR | Success (but check ANCOUNT — 0 answers = name exists, no such record type) |
| 1 | FORMERR | Malformed query |
| 2 | SERVFAIL | Resolver broke, upstream timeout, or DNSSEC validation failed |
| 3 | NXDOMAIN | The name does not exist at all |
| 5 | REFUSED | Server refuses to answer (policy, not authoritative) |

### UDP-first, TCP-fallback (the 512-byte rule)

DNS uses **UDP** because a query + answer is tiny and connectionless is fast — no handshake, one packet out, one packet back. But UDP has a classic limit:

```
Answer fits in 512 bytes?
   YES → send over UDP, done. (most queries)
   NO  → server sets the TC (truncated) flag and sends what fits.
         Client sees TC=1 → re-asks the SAME query over TCP port 53.

EDNS0 (OPT record in the ADDITIONAL section) lets a client advertise a
larger UDP buffer (e.g. 4096 bytes), avoiding TCP fallback for medium answers.

ALWAYS TCP:
   • Zone transfers (AXFR/IXFR) — copying a whole zone between nameservers
   • Any answer still too big after EDNS0
```

This is why firewalls must allow **both UDP/53 and TCP/53**. A common misconfiguration is allowing only UDP/53 — everything works until one big answer (many records, DNSSEC signatures) needs TCP, and then that one lookup mysteriously fails.

---

## How It Works — Step by Step

Let's resolve `api.myapp.com` from a completely cold cache. This is the full chain. Watch who asks whom.

### The recursive vs iterative distinction (read this first)

```
RECURSIVE query:  "Give me the final answer. Do all the work yourself."
                  Your stub resolver → recursive resolver.  ONE question, ONE final answer.

ITERATIVE query:  "Give me the answer OR tell me who to ask next."
                  Recursive resolver → root/TLD/authoritative.  A chain of referrals.
```

Your laptop is lazy — it sends **one recursive** query and expects a finished answer. The recursive resolver does the hard work by sending a series of **iterative** queries down the tree.

### The full walk

```
   ┌──────────────┐
   │ Your machine │  "What is api.myapp.com?"  (RD=1, recursive)
   │ stub resolver│──────────────────────────────┐
   └──────────────┘                               │
        ▲                                          ▼
        │ final answer                    ┌──────────────────┐
        │ 142.250.80.46 (TTL 300)         │ RECURSIVE RESOLVER│  (e.g. 8.8.8.8)
        └─────────────────────────────────│  does iterative   │
                                          │  queries below    │
                                          └──────────────────┘
   iterative queries ↓ (RD ignored — each server answers OR refers)

   ① → ROOT server (one of 13 clusters, a–m.root-servers.net)
        Q: "api.myapp.com A?"
        A: "I don't know, but .com is handled by these TLD servers:
            a.gtld-servers.net ... (referral, AA=0)"

   ② → .com TLD server (run by Verisign)
        Q: "api.myapp.com A?"
        A: "I don't know the address, but myapp.com is delegated to:
            ns-1234.awsdns-56.org, ns-567.awsdns-11.co.uk (referral + glue)"

   ③ → myapp.com AUTHORITATIVE server (e.g. AWS Route 53)
        Q: "api.myapp.com A?"
        A: "api.myapp.com. 300 IN A 142.250.80.46   (AA=1, the real answer!)"

   Resolver caches the answer for TTL (300s) and returns it to your machine.
```

### The caching layers (why the walk almost never happens twice)

```
Request for api.myapp.com travels UP this cache stack, stopping at the first hit:

┌─────────────────────────────────────────────────────────────┐
│ 1. Browser DNS cache        ~60s, in-memory (chrome://net-internals) │
│ 2. OS / stub resolver cache  systemd-resolved, nscd, mDNSResponder   │
│ 3. Recursive resolver cache  8.8.8.8 shares one hot cache for millions│
│ 4. (miss) → full iterative walk: root → TLD → authoritative          │
└─────────────────────────────────────────────────────────────┘
Each hop that CACHES the answer means the layers below it never see the query.
Because 8.8.8.8 serves millions of users, popular names are almost always
already cached there — your "cold" lookup is warm for someone else.
```

### TTL and negative caching

- **TTL** is set by the zone owner *per record*. It says "you may cache this answer for N seconds." When it expires, the next lookup re-queries.
- **Negative caching** — even a "this name does not exist" (`NXDOMAIN`) answer is cached, so you don't hammer the authoritative server for a name that isn't there. The duration is the **`minimum` field of the zone's SOA record** (per RFC 2308), capped by the SOA record's own TTL. This is why fixing a typo'd DNS record can still show `NXDOMAIN` for a while afterwards — the *negative* answer is cached too.

### Low TTL vs high TTL — the fundamental trade-off

```
LOW TTL  (e.g. 60s)                 HIGH TTL (e.g. 86400s = 1 day)
─────────────────                    ────────────────────────────
+ Changes propagate fast             + Fewer queries → less load, lower cost
+ Great for failover/blue-green      + Faster average lookups (more cache hits)
- More queries hit authoritative     - Changes take up to TTL to propagate
- More resolver load & $ (Route 53    - A bad deploy "sticks" — users pinned
  charges per query)                    to the old IP for the whole TTL

Standard playbook: LOWER the TTL (e.g. to 60s) a day BEFORE a planned DNS
migration, do the switch, confirm, then raise it back up.
```

---

## Exact Syntax Breakdown

### The `dig` command

```
dig  @8.8.8.8   api.myapp.com   A   +short
│    │          │               │   │
│    │          │               │   └── output option (terse: just the answer)
│    │          │               └────── record TYPE to ask for (default: A)
│    │          └────────────────────── the NAME to look up
│    └───────────────────────────────── @server: which resolver to ask (optional)
└────────────────────────────────────── "domain information groper"
```

### Reading a full `dig` answer, line by line

```
$ dig api.github.com

; <<>> DiG 9.10.6 <<>> api.github.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24096   ← RCODE=NOERROR, transaction id
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
     │    │  │  │
     │    │  │  └── ra = Recursion Available (resolver does recursion)
     │    │  └───── rd = Recursion Desired   (we asked for the full walk)
     │    └──────── qr = this is a Response
     └───────────── (no "aa" = this came from a CACHE, not the authoritative server)

;; QUESTION SECTION:
;api.github.com.                IN      A          ← what we asked

;; ANSWER SECTION:
api.github.com.         21      IN      A       140.82.112.6
     │                  │       │       │       │
     │                  │       │       │       └── RDATA: the IP
     │                  │       │       └────────── TYPE
     │                  │       └────────────────── CLASS (IN = Internet)
     │                  └────────────────────────── TTL remaining (21s — counting DOWN in cache!)
     └───────────────────────────────────────────── NAME

;; Query time: 15 msec              ← how long THIS lookup took
;; SERVER: 8.8.8.8#53(8.8.8.8)      ← which resolver answered, on port 53
;; WHEN: Sat Jul 12 10:04:11 UTC 2026
;; MSG SIZE  rcvd: 71               ← answer size in bytes (well under 512 → UDP was fine)
```

Two tells to memorise:
- **No `aa` flag + a TTL that decreases each time you run `dig`** → you're reading a cached answer. Add `@ns-authoritative-server` to bypass the cache and see the source of truth (it'll have `aa` and the full, non-decreasing TTL).
- **`status: NXDOMAIN`** → the name genuinely doesn't exist. **`status: NOERROR` with `ANSWER: 0`** → the name exists but has no record of *that type* (e.g. no AAAA).

### `dig +trace` — watch the whole chain yourself

```
dig +trace api.myapp.com
│         │
│         └── follow the delegation from the root, printing every referral
└── instead of asking your resolver for the final answer, dig itself walks:
    root servers → .com TLD servers → authoritative servers, showing each step.
```

`+trace` is the single best teaching tool in DNS — it turns the abstract "recursive walk" into concrete output you can read (see Hands-On Proof).

---

## Example 1 — Basic

**Goal:** look up a hostname and read every field.

```bash
$ dig example.com A +noall +answer
example.com.  3585  IN  A  93.184.215.14
```

Decoded: the name `example.com.` has an **A** record (IPv4 address) of `93.184.215.14`, cacheable for **3585 more seconds** (~1 hour), in the **IN** (Internet) class.

Now the IPv6 equivalent:

```bash
$ dig example.com AAAA +short
2606:2800:21f:cb07:6820:80da:af6b:8b2c
```

`AAAA` ("quad-A") is the IPv6 version of `A`. A dual-stack host has both. Your OS's stub resolver typically asks for **both A and AAAA** and picks per Happy Eyeballs (RFC 8305).

See what the resolver used and how long it took:

```bash
$ dig example.com | grep -E "Query time|SERVER"
;; Query time: 24 msec
;; SERVER: 192.168.1.1#53(192.168.1.1)   ← my home router forwarded to the ISP
```

Run it twice in a row and watch the TTL drop and the query time collapse:

```bash
$ dig example.com +noall +answer ; sleep 2 ; dig example.com +noall +answer
example.com.  3585  IN  A  93.184.215.14     ← first: TTL 3585, ~24ms
example.com.  3583  IN  A  93.184.215.14     ← second: TTL 3583, ~1ms (cache hit!)
```

The TTL went **down by 2** (the 2 seconds we slept), and the second query was near-instant. That's the resolver cache in action — proof, in two lines, that DNS caches and counts down.

---

## Example 2 — Production Scenario: "We deployed 20 minutes ago but half our users still hit the OLD server"

**The situation:** You run a Node.js API behind `api.myapp.com`. You're migrating from an old EC2 box (`54.10.10.10`) to a new load balancer (`54.20.20.20`). At 14:00 you updated the A record in Route 53. It's now 14:20. New requests mostly work — but ~40% of users are still hitting the *old* server, which you've started shutting down. Errors are spiking. What happened?

**Step 1 — Confirm the record actually changed at the source (authoritative):**

```bash
# Ask the AUTHORITATIVE nameserver directly — bypass all caches
$ dig @ns-1234.awsdns-56.org api.myapp.com A +noall +answer +ttlid
api.myapp.com.  3600  IN  A  54.20.20.20
                │
                └── TTL is 3600 = ONE HOUR. There's your problem.
```

The change is live at the source, and it points to the new IP. Good. But the TTL is **3600 seconds**.

**Step 2 — Confirm what caches are still serving:**

```bash
# Ask a few public resolvers to see what's cached out in the world
$ dig @8.8.8.8 api.myapp.com +short      # Google
54.20.20.20                               # new (this resolver already re-fetched)

$ dig @1.1.1.1 api.myapp.com +short      # Cloudflare
54.10.10.10                               # STILL OLD — cached before 14:00, TTL not expired yet
```

**Root cause:** You changed the record at 14:00, but resolvers that had fetched it *before* 14:00 cached the old IP with a **3600s TTL**. Those caches won't re-query until up to an hour after they last fetched. Any user behind a resolver that cached `54.10.10.10` at, say, 13:55 will keep hitting the old box until ~14:55. **DNS changes are not instant — they are bounded by the TTL that was in effect when the old record was cached.**

**Why lowering the TTL *now* doesn't help:** the resolvers already hold the old record with the *old* 3600s TTL. Your new low TTL only applies to answers fetched *after* the change.

**The fix (immediate):** Do **not** shut the old server down. Keep `54.10.10.10` serving until the old TTL fully drains — max ~1 hour from the change. Ideally have the old box proxy to the new one during the drain window.

**The fix (so this never happens again — the standard playbook):**

```
T-24h : Lower the TTL to 60s.  dig confirms authoritative shows TTL 60.
        Now WAIT at least the OLD TTL (3600s) so every cache re-fetches
        and picks up the new low TTL of 60.
T-0   : Make the actual A-record change (old→new IP).
        Now the whole world's caches expire within ~60s.
T+2min: Verify @8.8.8.8, @1.1.1.1, @208.67.222.222 all return the new IP.
T+... : Keep old server alive for a few minutes as a safety net, then decommission.
T+1h  : Raise the TTL back up (e.g. 300–3600s) to reduce query load/cost.
```

**The one-liner that would have caught it in code review:** `dig api.myapp.com +noall +answer` — if the TTL column reads `3600`, you know any cutover will take up to an hour to fully propagate. Deep tooling for exactly this kind of diagnosis lives in `22-dig-and-dns-debugging.md` (Phase 4).

---

## Common Mistakes

### Mistake 1: "DNS changes take effect immediately"

**Wrong belief:** "I updated the A record, so everyone hits the new server now."

**Correction:** Everyone hits the new server *within the TTL that was in effect when they last cached the old record* — which can be seconds, hours, or (for some ISPs that ignore your TTL) longer. Propagation time ≈ old TTL, not zero. Always check the TTL **before** a cutover and lower it in advance. There is no "flush the whole internet's DNS cache" button.

---

### Mistake 2: Putting a CNAME on the zone apex (`myapp.com` itself)

**Wrong belief:** "I'll just `CNAME myapp.com → my-lb.elb.amazonaws.com` like I do for `www`."

**Correction:** The DNS spec (RFC 1034) forbids a CNAME coexisting with any other record at the same name — and the apex **must** hold SOA and NS records. So a CNAME at the apex is illegal; most nameservers will refuse it or break email/other records.

```
✗ myapp.com.        CNAME  my-lb.elb.amazonaws.com.   ← ILLEGAL (apex has SOA+NS)
✓ www.myapp.com.    CNAME  my-lb.elb.amazonaws.com.   ← fine (subdomain, no other records)
```

**The workaround:** use your DNS provider's synthetic record — **ALIAS** (Route 53 calls it "Alias", others call it **ANAME**). It behaves like a CNAME but is resolved *server-side* by the provider, which returns actual A records at the apex. So the client just sees `myapp.com A 54.20.20.20`, spec-compliant, while your provider silently tracks the load balancer's changing IPs.

---

### Mistake 3: Forgetting the negative-cache TTL

**Wrong belief:** "I just added the missing record, so the `NXDOMAIN` error will clear instantly."

**Correction:** When a lookup returned `NXDOMAIN` (or `NOERROR`/0-answers) *before* you added the record, that **negative result was cached** for the duration of the zone's **SOA `minimum`** field. If your SOA minimum is 3600, clients may keep seeing "does not exist" for up to an hour after you create the record. When designing a zone, keep the SOA `minimum` modest (e.g. 300–900s) so mistakes recover quickly.

---

### Mistake 4: Hardcoding IP addresses instead of hostnames

**Wrong belief:** "DNS is slow and flaky — I'll just put the IP `54.10.10.10` directly in my config/connection string."

**Correction:** Cloud IPs change constantly — load balancers, RDS instances, and NAT gateways get new IPs on failover, scaling, or maintenance. Hardcode an IP and your app breaks silently the moment the provider moves the service. **Always connect by hostname.** DNS *is* the indirection layer that lets infrastructure move without you redeploying. The performance cost is a one-time resolution that's then cached and reused via connection pooling.

```javascript
// ✗ Brittle — breaks when RDS fails over to a standby with a new IP
const db = new Client({ host: "10.0.3.47" });

// ✓ Correct — RDS endpoint is a CNAME that repoints on failover automatically
const db = new Client({ host: "prod-db.abc123.us-east-1.rds.amazonaws.com" });
```

---

### Mistake 5: Ignoring resolver caching in load tests

**Wrong belief:** "My load test hits `api.myapp.com` 50,000 times, so I'm exercising DNS + GeoDNS + round-robin just like real users."

**Correction:** Your load generator resolves the hostname **once**, caches it, and then hammers that single IP for the whole run. You never exercise DNS round-robin, GeoDNS routing, or the resolution path real users hit — and you completely miss DNS as a latency/failure source. Real users come through thousands of different resolvers with independent caches. If DNS behaviour matters, either force re-resolution per iteration or test the resolution step explicitly (`dig` in a loop) as its own metric.

---

### Mistake 6: Allowing only UDP/53 through the firewall

**Wrong belief:** "DNS is UDP, so I only need to open UDP port 53."

**Correction:** Large answers (many records, DNSSEC signatures, big TXT records) exceed 512 bytes, set the **TC** flag, and force a **retry over TCP/53**. Zone transfers (AXFR) are always TCP. Block TCP/53 and those specific lookups fail intermittently and bafflingly while small lookups work fine. Open **both UDP/53 and TCP/53**.

---

## Every Record Type You Actually Need

| Type | Maps NAME → | Example RDATA | When you use it |
|------|-------------|---------------|-----------------|
| **A** | IPv4 address | `93.184.215.14` | Point a hostname at a server |
| **AAAA** | IPv6 address | `2606:2800:21f::c` | IPv6 endpoint (dual-stack) |
| **CNAME** | another name (alias) | `www → myapp.com.` | Alias a subdomain to another name |
| **MX** | mail server + priority | `10 mail.myapp.com.` | Where to deliver email for the domain |
| **TXT** | arbitrary text | `"v=spf1 include:_spf.google.com ~all"` | SPF, DKIM, DMARC, domain verification |
| **NS** | authoritative nameserver | `ns-1234.awsdns-56.org.` | Delegate a zone to nameservers |
| **SOA** | zone metadata | (see below) | One per zone; "start of authority" |
| **PTR** | IP → name (reverse) | `6.112.82.140.in-addr.arpa → host` | Reverse DNS; email server reputation |
| **SRV** | service host+port | `0 5 5060 sip.myapp.com.` | Service discovery (SIP, XMPP, LDAP, k8s) |
| **CAA** | which CAs may issue certs | `0 issue "letsencrypt.org"` | Restrict who can issue TLS certs for you |

### SOA — the record that defines a zone

Every zone has exactly one SOA ("Start Of Authority") record. Its fields:

```
myapp.com.  IN  SOA  ns-1234.awsdns-56.org.  admin.myapp.com. (
        2026071201   ; SERIAL   version number; BUMP on every change so secondaries re-sync
        7200         ; REFRESH  secondary checks primary this often (7200s = 2h)
        900          ; RETRY    if refresh fails, retry after this (900s = 15m)
        1209600      ; EXPIRE   secondary stops answering if it can't reach primary (14 days)
        300 )        ; MINIMUM  ← negative-cache TTL (how long NXDOMAIN is cached)
        │                        (in modern DNS this field = negative caching TTL, RFC 2308)
        └── first two lines: primary NS, and admin email (first "." → "@": admin@myapp.com)
```

**The SERIAL matters operationally:** secondaries pull the zone only when the primary's serial is higher than theirs. Forget to bump it after editing the zone file and your secondaries silently serve stale data. Common convention: `YYYYMMDDnn`.

### TXT: SPF, DKIM, DMARC, and domain verification

TXT records hold free-form strings and carry most of email authentication:

```
# SPF — which servers are allowed to send mail as @myapp.com
myapp.com.            TXT  "v=spf1 include:_spf.google.com include:amazonses.com ~all"

# DKIM — public key used to verify the signature on outgoing mail
sel1._domainkey.myapp.com.  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCSq..."

# DMARC — policy for what to do when SPF/DKIM fail
_dmarc.myapp.com.     TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@myapp.com"

# Domain ownership verification (Google, AWS ACM, etc.)
myapp.com.            TXT  "google-site-verification=abc123..."
```

### PTR — reverse DNS (IP → name)

A → forward (name to IP). PTR → reverse (IP to name), looked up under the special `in-addr.arpa` (IPv4) or `ip6.arpa` (IPv6) tree with the IP reversed:

```bash
$ dig -x 140.82.112.6 +short           # -x = reverse lookup
lb-140-82-112-6-iad.github.com.
# Under the hood this queries: 6.112.82.140.in-addr.arpa  PTR ?
```

Mail servers rely on this heavily: if your sending IP has no matching PTR record (or it doesn't match the HELO name), many receivers mark your mail as spam. PTR is controlled by whoever owns the IP block (your cloud provider), not by you in your own zone.

### CNAME chains — correct but costly

A CNAME can point to another name that is itself a CNAME. Every hop is a separate resolution the resolver must chase:

```
www.myapp.com.   CNAME  myapp.map.cdn.com.
myapp.map.cdn.com.  CNAME  edge-42.cdn.com.
edge-42.cdn.com.    A      151.101.1.55
```

Each link is a lookup. A deep CNAME chain on a cold cache adds real latency to the first request and is a subtle cause of slow TTFB — keep chains short and prefer ALIAS/flattening at the apex.

---

## Modern DNS: Encrypted Transport (DoT & DoH)

Classic DNS on port 53 is **plaintext** — anyone on the path (your ISP, a coffee-shop Wi-Fi operator) can see every hostname you look up and can even forge answers. Two protocols fix this by wrapping DNS in encryption:

```
                    Port   Transport            Looks like...
DNS (classic)        53    UDP/TCP, plaintext   obviously DNS (easy to block/log/spoof)
DoT (DNS over TLS)   853   TLS                  encrypted DNS on a dedicated port
DoH (DNS over HTTPS) 443   HTTPS                encrypted DNS blended in with web traffic
```

- **DoT (RFC 7858)** — DNS inside a TLS session on **port 853**. Encrypted, but on a distinct port, so a network operator can still *see that you're doing DNS* (and choose to block 853), just not *what* you asked.
- **DoH (RFC 8484)** — DNS queries sent as HTTPS requests to a URL like `https://cloudflare-dns.com/dns-query`, on **port 443**. Because it's indistinguishable from ordinary web traffic, it's very hard to block or surveil. This is what Firefox/Chrome use for their built-in "secure DNS."

**Backend implications:** DoH/DoT protect user privacy but complicate network-level control. If your corporate network relies on DNS filtering (blocking malware domains via a controlled resolver), a browser doing DoH to `1.1.1.1` bypasses it entirely. Enterprises often push a policy or a "canary domain" (`use-application-dns.net`) to disable browser DoH. For your own services, encrypted DNS matters most for privacy-sensitive clients; server-to-server internal DNS usually stays on plaintext 53 within a trusted VPC.

---

## Hands-On Proof

```bash
# 1. Basic forward lookup — just the answer
dig example.com +short
#   → 93.184.215.14

# 2. Full answer with all the sections and flags (read status, flags, TTL)
dig github.com

# 3. WATCH THE ENTIRE CHAIN: root → TLD → authoritative
dig +trace www.github.com
#   Reads top-down: first a list of root servers, then the .com referral,
#   then github.com's authoritative NS, then the final A record. This is the
#   iterative walk your resolver normally hides from you.

# 4. Ask a specific record type
dig google.com MX +short          # mail servers
dig myapp.com TXT +short          # SPF/DKIM/verification strings
dig google.com NS +short          # authoritative nameservers
dig google.com SOA +noall +answer # the zone's SOA (see serial/minimum)

# 5. Reverse DNS (IP → name)
dig -x 8.8.8.8 +short             # → dns.google.

# 6. Bypass your cache: ask the authoritative server directly
#    (note the 'aa' flag appears and the TTL is the full, non-decreasing value)
dig @ns1.google.com google.com +noall +answer

# 7. Prove caching: run twice, watch TTL count DOWN and query time drop
dig example.com +noall +answer +stats | grep -E "IN\s+A|Query time"
sleep 3
dig example.com +noall +answer +stats | grep -E "IN\s+A|Query time"

# 8. Compare what different resolvers currently have cached
dig @8.8.8.8   github.com +short   # Google
dig @1.1.1.1   github.com +short   # Cloudflare
dig @9.9.9.9   github.com +short   # Quad9

# 9. See a truncated answer force TCP (large TXT / DNSSEC)
dig cloudflare.com TXT +noall +comments   # look for "flags: ... tc" then retry
dig cloudflare.com TXT +tcp               # force TCP explicitly

# 10. nslookup equivalent (works on Windows too)
nslookup github.com
nslookup -type=MX google.com
```

What to notice: in `+trace` output, each block ends by handing you the nameservers for the next level down — that *is* delegation. In step 6 vs step 2, the authoritative answer carries the `aa` flag and a stable TTL; your cached answer does not and counts down.

---

## Practice Exercises

### Exercise 1 — Easy: Read a record and its TTL

```bash
# Run these:
dig cloudflare.com A +noall +answer
dig cloudflare.com AAAA +noall +answer
dig cloudflare.com MX +short
dig cloudflare.com NS +short

# Answer from the output:
# 1. What is cloudflare.com's IPv4 address, and what's its TTL?
# 2. Does it have an IPv6 (AAAA) record? What is it?
# 3. How many mail servers (MX) does it list?
# 4. Which nameservers are authoritative for the domain?
# 5. Run the A-record command 3 times with a few seconds between.
#    Does the TTL go up or down? What does that tell you about where the answer came from?
```

### Exercise 2 — Medium: Watch the full resolution chain and identify each level

```bash
# Trace the whole delegation path
dig +trace api.github.com

# Answer:
# 1. How many root servers appear at the top? (letters a–m)
# 2. Which company/registry runs the .com TLD servers? (hint: look at the referral names)
# 3. What are api.github.com's authoritative nameservers?
# 4. Which line in the output shows AA=1 (the authoritative final answer)?
# 5. Now compare timing:
dig api.github.com +noall +stats | grep "Query time"   # via your resolver (cached)
#    Why is this SO much faster than +trace? (Which caches did it hit?)
```

### Exercise 3 — Hard (Production Simulation): Plan and verify a zero-downtime DNS cutover

```
Scenario: api.myapp.com currently → 54.10.10.10 (old), TTL 3600.
You must move it to 54.20.20.20 (new load balancer) with ZERO users
hitting the old box after cutover. You control Route 53.

Tasks:
# 1. Before touching anything, capture the current state:
dig api.myapp.com +noall +answer            # note the TTL

# 2. Write out the exact timeline (with times/waits) you'd follow.
#    Account for the CURRENT 3600s TTL still cached in the wild.
#    (Hint: lowering TTL only helps AFTER the old TTL has drained.)

# 3. After the change, write the commands to VERIFY every major public
#    resolver has picked up the new IP before you decommission the old box:
dig @8.8.8.8  api.myapp.com +short
dig @1.1.1.1  api.myapp.com +short
dig @9.9.9.9  api.myapp.com +short
dig @ns-XXXX.awsdns-YY.org api.myapp.com +short   # authoritative = source of truth

# Questions:
# a) Why does lowering the TTL to 60s at T-0 NOT make the cutover instant?
# b) What is the MAXIMUM time you must keep the old server alive after
#    the change, assuming the old cached TTL was 3600?
# c) The apex 'myapp.com' also needs to point at the new LB. Why can't you
#    just add a CNAME there, and what do you use instead?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the linked section.

1. **A stub resolver sends a recursive query; the recursive resolver sends iterative queries. Explain the difference in one sentence each, and say who asks whom.**

2. **Walk the full resolution chain for a cold lookup of `blog.myapp.com`: name every server type in order and say what each one returns (an answer vs a referral).**

3. **You changed an A record 10 minutes ago but some users still hit the old IP. What single field explains this, where do you read it, and roughly how long until it fully clears?**

4. **DNS runs on UDP port 53. Name two situations that force it onto TCP port 53, and what flag/limit triggers the fallback.**

5. **Why can't you put a CNAME on `myapp.com` itself? What record type do you use instead, and what does it actually return to the client?**

6. **In `dig` output, you see `status: NOERROR` but `ANSWER: 0`. What does that mean — and how is it different from `NXDOMAIN`? Which SOA field governs how long the "no such name" answer is cached?**

7. **You're setting up email for `myapp.com`. Name the record types involved and what each one does (delivery target, sender authorisation, signature key, reverse lookup).**

---

## Quick Reference Card

| Concept | What it is | Key detail |
|---------|-----------|-----------|
| Stub resolver | Client in your OS/app | Sends ONE recursive query (RD=1) |
| Recursive resolver | Does the walk for you | 8.8.8.8, 1.1.1.1; caches heavily |
| Root servers | Top of the tree | 13 clusters, a–m.root-servers.net; point to TLDs |
| TLD server | Authoritative for `.com` etc. | Run by registries (Verisign for `.com`) |
| Authoritative NS | Holds the real records | Source of truth; answer has `aa` flag |
| TTL | Cache lifetime of a record | Set per record; propagation ≈ old TTL |
| Negative caching | Caching of "doesn't exist" | Governed by SOA `minimum` (RFC 2308) |
| Port | Where DNS listens | UDP/53 default, TCP/53 fallback (>512B, AXFR) |
| DoT / DoH | Encrypted DNS | DoT = port 853 (TLS), DoH = port 443 (HTTPS) |

**Record types cheat block:**
```
A      name → IPv4                     AAAA   name → IPv6
CNAME  name → another name (alias)     MX     mail server + priority
NS     delegate zone to nameservers    SOA    zone metadata (1 per zone)
TXT    SPF / DKIM / DMARC / verify      PTR    IP → name (reverse, in-addr.arpa)
SRV    service host+port discovery      CAA    which CAs may issue TLS certs
ALIAS/ANAME = CNAME-like at the APEX (provider resolves it server-side)
```

**SOA fields:** `SERIAL` (bump on every change) · `REFRESH` · `RETRY` · `EXPIRE` · `MINIMUM` (= negative-cache TTL)

**dig cheat sheet:**
```bash
dig NAME                     # full answer with all sections + flags
dig NAME +short              # just the answer data
dig NAME TYPE                # ask a specific record (A/AAAA/MX/TXT/NS/SOA/CAA)
dig NAME +noall +answer      # only the ANSWER section
dig +trace NAME              # walk root → TLD → authoritative yourself
dig @8.8.8.8 NAME            # ask a specific resolver (compare caches)
dig @<authoritative> NAME    # bypass caches; look for the 'aa' flag
dig -x IP                    # reverse lookup (PTR)
dig NAME +tcp                # force TCP/53
```

**dig flags to read:** `qr` response · `aa` authoritative (no `aa` = cached) · `rd` recursion desired · `ra` recursion available · `tc` truncated (retry over TCP)

**RCODE:** `NOERROR` (0, success) · `NXDOMAIN` (3, name doesn't exist) · `SERVFAIL` (2, resolver/upstream/DNSSEC broke) · `REFUSED` (5)

---

## When Would I Use This at Work?

### Scenario 1: "The site is down!" — but the servers are healthy

Your monitoring says all app servers are green, yet users can't reach the site. First move: `dig yoursite.com`. If it's `SERVFAIL` or times out, the problem is DNS, not your app — check your DNS provider's status, your NS records, and whether a recent zone edit broke something (bad serial, deleted record). This is the 2016 Dyn outage in miniature: healthy servers, unreachable because DNS was down.

### Scenario 2: Blue/green or failover deploy via DNS

You cut traffic from the blue environment to green by repointing a record. Before the deploy you *pre-lower the TTL* (to 60s), wait out the old TTL, switch, then verify with `dig @8.8.8.8` / `@1.1.1.1` that public resolvers show the new target before decommissioning the old one. Understanding TTL is the difference between a clean cutover and a 40%-of-users-on-the-dead-box incident (Example 2).

### Scenario 3: Setting up email for a new domain and mail lands in spam

You add MX records but mail still goes to spam. You now know to check the full stack: `MX` (delivery target), `TXT`/SPF (which servers may send as you), DKIM (signature key), DMARC (failure policy), and the sending IP's `PTR` (reverse DNS matching the HELO). Missing SPF or PTR is the usual culprit — `dig myapp.com TXT` and `dig -x <your-mail-IP>` diagnose it in seconds.

### Scenario 4: Issuing a TLS certificate and it fails or gets blocked

Let's Encrypt validates domain control via a DNS-01 challenge (a `TXT` record it tells you to add) and checks your `CAA` record to see if it's even *allowed* to issue. If your CAA record only lists `digicert.com`, Let's Encrypt refuses. `dig myapp.com CAA` tells you immediately whether your chosen CA is permitted.

### Scenario 5: Internal microservice can't find another service

`ENOTFOUND service-b` in your logs means the name didn't resolve. In Kubernetes that's usually a wrong service name/namespace (`service-b.namespace.svc.cluster.local`) or CoreDNS misconfig. You'd `dig` the internal name from inside a pod to see whether it's a DNS problem or a connectivity problem — the same tools, pointed at your cluster's resolver.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — DNS is Step 1 of the request lifecycle sketched there
- `04-ip-addresses.md` — DNS's whole job is to hand you an IP; know what an A vs AAAA record contains
- `05-ports.md` — DNS lives on port 53 (and DoT 853, DoH 443)

**Study next (in order):**
- `07-udp.md` — DNS's default transport; understand *why* connectionless UDP fits DNS and where it breaks (the 512-byte / TCP-fallback rule)
- `08-tcp-in-depth.md` — the TCP fallback for large answers and zone transfers
- `09-tls-ssl-in-depth.md` — DoT/DoH wrap DNS in TLS; CAA records gate certificate issuance

**Deep tooling companion:**
- `22-dig-and-dns-debugging.md` (Phase 4) — the hands-on debugging deep dive: reading `dig` output exhaustively, tracing resolution, and diagnosing every common DNS failure mode. This topic gives you the model; that one gives you the drills.

**Connects forward to:**
- `30-cdns.md` — GeoDNS and CNAME-to-edge are how CDNs route you to the nearest node
- `28-load-balancers.md` — the ALIAS/CNAME target of your apex is usually a load balancer
- `36-service-to-service-communication.md` — internal DNS and service discovery in a mesh

---

*Doc saved: `/docs/networking/06-dns-in-depth.md`*
