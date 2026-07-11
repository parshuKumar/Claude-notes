# 22 — dig and DNS Debugging

> **Phase 4 — Topic 2 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `06-dns-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine you're a private investigator trying to find someone's home address. You don't just accept the first phone book you're handed — you want to know *who* told you that address, *how sure* they are, and *how old* the information is.

- **`nslookup`** is like asking a friend "hey, where does Alice live?" — they give you an answer, but you don't know where they got it.
- **`dig`** is the professional PI. It gives you the address **and** the paper trail: which office answered, how confident they were, how long the answer is valid for, and how long the whole lookup took.
- **`dig +trace`** is you personally driving to City Hall, then the district office, then the neighborhood registry, asking each one "who handles this street?" until you knock on the actual door — watching the *entire* chain of referrals with your own eyes.

DNS is the internet's phone book. `dig` is the tool that lets you read every page of it out loud, question the source, and catch it when it's lying or out of date.

---

## Where This Lives in the Network Stack

`dig` is a **client tool** that speaks the **DNS protocol** — an Application Layer (Layer 7) protocol that rides on **UDP (or TCP) port 53** at Layer 4.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (DNS protocol) ◄── dig speaks this      │
│  Layer 6 — Presentation  (—)                                    │
│  Layer 5 — Session       (—)                                    │
│  Layer 4 — Transport     (UDP/53 by default, TCP/53 fallback)   │  ← dig +tcp forces this
│  Layer 3 — Network       (IP — packet to your resolver)         │
│  Layer 2 — Data Link     (Ethernet/Wi-Fi)                       │
│  Layer 1 — Physical      (cable/radio)                          │
└─────────────────────────────────────────────────────────────────┘
```

`dig` sits at the very top. It crafts a DNS **query message**, hands it to the OS to send over UDP/53, and parses the DNS **response message** that comes back. Everything you see in `dig` output is a human-readable rendering of the bytes in that response packet. Topic 06 taught you the *DNS system*; this topic teaches you the *tool that lets you interrogate it*.

---

## What Is This?

**`dig`** (Domain Information Groper) is the standard command-line DNS query tool, shipped as part of BIND's `dnsutils`/`bind-utils` package. It sends a single DNS query to a resolver and prints the **complete, unabridged response** — every section, every flag, every TTL.

Unlike your application's `getaddrinfo()` (which just hands your code an IP and hides everything else), `dig` shows you:

- **Which resolver** answered and whether it was authoritative
- **The status code** (`NOERROR`, `NXDOMAIN`, `SERVFAIL`, …)
- **Every record** in the answer, with its **TTL counting down**
- **The delegation chain** (with `+trace`), hop by hop from the root
- **The exact query time** in milliseconds

There are three common DNS CLI tools:

| Tool | Verbosity | Scriptable | Backend-dev verdict |
|------|-----------|-----------|--------------------|
| `dig` | Full — every section & flag | Yes (`+short`) | **Preferred.** The debugging standard. |
| `host` | One line per record | Somewhat | Fine for a quick "what's the IP" |
| `nslookup` | Medium, but hides flags/TTL detail | Awkward | Legacy; interactive mode confuses; avoid for debugging |

`dig` is preferred because it shows you the **truth of the wire**: the raw response, not a friendly summary. When DNS is broken, the friendly summary is exactly what's lying to you.

---

## Why Does It Matter for a Backend Developer?

DNS is the **first thing that happens** on every outbound request your service makes, and it's the layer that fails in the most confusing ways. You need `dig` when:

- **A deploy "isn't live" for some users.** DNS propagation and TTL caching mean the same hostname can resolve differently depending on which resolver you ask. `dig @8.8.8.8` vs `dig @your-resolver` tells you instantly.
- **You changed a DNS record and nothing happened.** Did the authoritative server actually get updated? `dig @authoritative-ns` bypasses every cache and shows you the source of truth.
- **Your service throws `SERVFAIL` or `ENOTFOUND` intermittently.** These are *different* failures with different fixes. `dig` tells you which one and why.
- **A `CNAME` chain is slow or wrong.** `dig` shows the full chain; your app just sees the final IP.
- **You're debugging a slow API** (Topic 27). DNS resolution is one of the timing buckets. `dig`'s `Query time` isolates it.
- **You need to verify records you can't see in a dashboard:** `TXT` for SPF/DKIM/domain-verification, `MX` for mail, `CAA` for which CA may issue certs, `NS` for delegation.

If you can't read `dig` output fluently, DNS incidents will cost you hours. If you can, they cost you two commands.

---

## The Packet/Protocol Anatomy

Every `dig` invocation is one DNS **request message** out and one DNS **response message** back. Both share the same structure (RFC 1035): a fixed 12-byte header followed by four variable-length sections.

```
┌──────────────────────────────────────────────────────────────┐
│ DNS MESSAGE                                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ HEADER (12 bytes, fixed)                               │  │
│  │   ID          (16 bits) — matches request to response  │  │
│  │   Flags       (16 bits) — QR, Opcode, AA, TC, RD, RA,  │  │
│  │                            AD, CD, RCODE  ◄─ the status │  │
│  │   QDCOUNT / ANCOUNT / NSCOUNT / ARCOUNT (4×16 bits)    │  │
│  │                          — # of records in each section │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ QUESTION SECTION   (what you asked: name, type, class) │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ ANSWER SECTION     (the records that answer it)        │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ AUTHORITY SECTION  (which nameservers are authoritative)│  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ ADDITIONAL SECTION (bonus records: glue A/AAAA, OPT)   │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

The **flags** in that 12-byte header are what `dig` renders on the `;; flags:` line. Memorize these six:

```
QR — Query/Response.  0 = this is a question, 1 = this is an answer.
AA — Authoritative Answer. Set when the responder OWNS this zone
     (an authoritative nameserver), NOT a cache/resolver.
TC — TrunCated. The UDP answer was too big (>512B / EDNS limit);
     retry over TCP.  dig auto-retries with +tcp.
RD — Recursion Desired. YOU set this: "please do the full lookup for me."
RA — Recursion Available. The server sets this: "yes, I do recursion."
AD — Authenticated Data. DNSSEC: the resolver validated the signatures.
```

A DNS **resource record (RR)** — every line in the ANSWER/AUTHORITY/ADDITIONAL sections — always has the same five fields:

```
example.com.        117        IN        A        172.66.147.243
    │                │          │         │              │
   NAME            TTL        CLASS      TYPE          RDATA
 (the owner)   (seconds left) (always   (A, MX,     (the value:
                              IN today)  NS, TXT…)   IP, host, text…)
```

That five-column shape — **NAME / TTL / CLASS / TYPE / RDATA** — is the single most important thing to internalize. Once you see it, every `dig` answer reads the same way.

---

## How It Works — Step by Step

Let's read a **complete, real `dig` output** top to bottom. This is actual output, annotated line by line:

```
$ dig example.com

; <<>> DiG 9.10.6 <<>> example.com                    ← (1) version + the args you passed
;; global options: +cmd                                ← (2) default options in effect
;; Got answer:                                          ← (3) a response came back at all
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39396   ← (4) THE STATUS LINE
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1  ← (5) flags + counts

;; OPT PSEUDOSECTION:                                   ← (6) EDNS metadata (not a real record)
; EDNS: version: 0, flags:; udp: 1280

;; QUESTION SECTION:                                    ← (7) echoes exactly what you asked
;example.com.			IN	A

;; ANSWER SECTION:                                      ← (8) the records that answer it
example.com.		117	IN	A	172.66.147.243
example.com.		117	IN	A	104.20.23.154

;; Query time: 1468 msec                                ← (9) how long the lookup took
;; SERVER: 2409:40d0:2424:24b8::42#53(...)              ← (10) which resolver answered, on port 53
;; WHEN: Sun Jul 12 00:23:41 IST 2026                   ← (11) timestamp
;; MSG SIZE  rcvd: 72                                    ← (12) bytes in the response packet
```

### (1)–(2) The header banner
`; <<>> DiG 9.10.6 <<>>` echoes your dig version and the arguments. Lines starting with `;` are **comments** (ignored by parsers) — that's why `+short` strips them all. `+cmd` just means "print the command echo."

### (4) The status line — read this FIRST, every time
```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39396
```
`status` is the RCODE. This one word tells you the outcome:

| status | Meaning | What to do |
|--------|---------|-----------|
| `NOERROR` | Query succeeded. (May still have **0 answers** — see below.) | Check the ANSWER count. |
| `NXDOMAIN` | The name **does not exist**. Authoritatively. | Typo? Record never created? Wrong zone? |
| `SERVFAIL` | The resolver **failed** to answer. | Broken authoritative server, DNSSEC failure, timeout. |
| `REFUSED` | The server won't answer you. | You queried a server that isn't allowed to serve you. |

### (5) The flags line — the state of the answer
```
;; flags: qr rd ra ad;   QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
         │  │  │  │       └── the four section counts (QDCOUNT/ANCOUNT/NSCOUNT/ARCOUNT)
         │  │  │  └── ad: DNSSEC-validated
         │  │  └── ra: recursion available (this is a recursive resolver)
         │  └── rd: recursion desired (you asked it to do the full walk)
         └── qr: this is a response
```
**Notice `aa` is absent.** This answer came from a *cache/resolver*, not the authoritative server. When you query an authoritative server directly you'll see `aa` appear — that's your proof you hit the source of truth.

### (7) QUESTION SECTION
```
;example.com.			IN	A
```
An exact echo of what you asked: name `example.com.` (note the trailing dot — the fully-qualified root), class `IN` (Internet), type `A`. Always confirm this matches what you *meant* to ask — a wrong search-domain suffix shows up here.

### (8) ANSWER SECTION — the payload
```
example.com.		117	IN	A	172.66.147.243
example.com.		117	IN	A	104.20.23.154
```
Two `A` records (this host is behind Cloudflare, round-robin). Read left to right: **NAME=**`example.com.`, **TTL=**`117` seconds, **CLASS=**`IN`, **TYPE=**`A`, **RDATA=**the IP. That `117` is a **countdown** — run it again in 10 seconds and it'll say `107`. That's your window into caching (see the "Comparing Resolvers" section).

### (3)/(6)/(9)–(12) The footer & pseudo-section
- **`;; Got answer:`** — a response arrived (contrast with `;; connection timed out; no servers could be reached`).
- **OPT PSEUDOSECTION** — EDNS(0) metadata. `udp: 1280` is the max UDP payload the resolver advertised. Not a real DNS record.
- **`Query time: 1468 msec`** — end-to-end time for *this* query. A cached answer is `0–5 msec`; a cold recursive lookup can be hundreds of ms.
- **`SERVER: …#53`** — the resolver that answered, and the port. If this isn't the resolver you expected, that's your bug.
- **`WHEN`** — wall-clock timestamp.
- **`MSG SIZE rcvd: 72`** — response size in bytes. Watch this near 512 (UDP truncation threshold).

### What the AUTHORITY and ADDITIONAL sections look like (from an NXDOMAIN)
```
;; ANSWER SECTION:                        ← (empty — 0 answers)

;; AUTHORITY SECTION:                     ← who is authoritative for the zone
com.  900  IN  SOA  a.gtld-servers.net. nstld.verisign-grs.com. 1783796042 1800 900 604800 900
```
When there's no answer, the **AUTHORITY** section often carries the zone's **SOA** record — telling you *which* server is the authority and (via the SOA's last field) the **negative-caching TTL** (how long "this doesn't exist" is remembered). The **ADDITIONAL** section carries "glue" — e.g., the A/AAAA of the nameservers named in AUTHORITY, so you don't need a second lookup.

---

## Exact Syntax Breakdown

### `dig @1.1.1.1 example.com MX +short`
```
dig   @1.1.1.1     example.com    MX     +short
│      │            │              │       │
│      │            │              │       └── OUTPUT MODIFIER: print only the RDATA,
│      │            │              │           strip all comments/sections (scriptable)
│      │            │              └── QUERY TYPE: A (default), AAAA, MX, TXT, NS, SOA,
│      │            │                  CNAME, CAA, PTR, SRV, ANY…
│      │            └── NAME: the domain to look up (trailing dot optional)
│      └── @SERVER: query THIS resolver instead of your system default
│          (@8.8.8.8 Google, @1.1.1.1 Cloudflare, @ns1.example.com an authoritative NS)
└── the dig command
```
The general grammar is: `dig [@server] [name] [type] [+options] [-flags]`. **Order is flexible** — dig figures out which token is the server (`@`), which is the type (a known RR type), and which is the name. `+options` are dotted "query/output modifiers"; `-flags` are short options like `-x` (reverse) and `-p` (port).

### `dig +trace example.com A`
```
dig   +trace        example.com   A
│      │             │             └── type to resolve at the end of the walk
│      │             └── name
│      └── +trace: DON'T ask your recursive resolver to do the work.
│          Instead, dig itself walks the delegation chain:
│          root  →  .com TLD  →  example.com authoritative  →  answer
│          printing every referral. (Implies +norecurse to each server.)
└── dig
```
`+trace` turns off recursion-desired and makes **dig** the resolver, so you *see* the hierarchy Topic 06 described instead of getting a single canned answer.

### A few more you'll type constantly
```
dig example.com +short              # just the IP(s), one per line
dig example.com +noall +answer      # ONLY the ANSWER section (clean, keeps TTLs)
dig -x 8.8.8.8                      # reverse lookup: IP → hostname (PTR)
dig example.com +norecurse          # ask WITHOUT recursion (probe a cache)
dig example.com +dnssec             # request DNSSEC records (RRSIG) too
dig example.com +tcp                # force TCP instead of UDP
dig example.com +stats              # include the timing/server footer (on by default)
```

---

## Example 1 — Basic

**Goal: look up a hostname's IP, then get just the value for a script.**

```bash
$ dig example.com A
```
```
; <<>> DiG 9.10.6 <<>> example.com A
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39396
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:
example.com.		117	IN	A	172.66.147.243
example.com.		117	IN	A	104.20.23.154

;; Query time: 1468 msec
;; SERVER: 2409:40d0:2424:24b8::42#53
;; MSG SIZE  rcvd: 72
```
Read it in the order that matters: **status `NOERROR`** (good), **`ANSWER: 2`** (two records), then the two IPs. Done.

**Now make it scriptable** — you rarely want all that in a shell pipeline:

```bash
$ dig example.com +short
172.66.147.243
104.20.23.154

$ dig example.com NS +short          # who runs this zone?
elliott.ns.cloudflare.com.
hera.ns.cloudflare.com.

$ dig -x 8.8.8.8 +short               # reverse: what hostname owns this IP?
dns.google.
```
`+short` prints only the RDATA — perfect for `IP=$(dig +short api.internal)`. Different record types answer different questions:

```bash
$ dig example.com MX +noall +answer     # mail servers
example.com.		26	IN	MX	0 .          # "0 ." = explicitly NO mail (RFC 7505 null MX)

$ dig example.com SOA +noall +answer    # zone's authority + timers
example.com.	1800	IN	SOA	elliott.ns.cloudflare.com. dns.cloudflare.com. 2407636105 10000 2400 604800 1800
#                                       primary-NS               admin-email(dns@)    serial     refresh retry expire minTTL
```
The SOA's seven fields, in order: **primary NS**, **admin email** (first dot = `@`), **serial** (bump this on every change), **refresh**, **retry**, **expire**, and **minimum TTL** (also the negative-cache TTL). When you change a record and it "won't update," check that the **serial** actually incremented.

---

## Example 2 — Production Scenario

**The incident:** You shipped a new API subdomain, `api-v2.myapp.com`, pointing at a fresh load balancer. QA in the office confirms it works. But **~30% of production users report `ENOTFOUND` / "cannot resolve host."** Support is escalating. Your job: is this a code bug, a DNS record bug, a delegation bug, or a caching problem?

### Step 1 — Reproduce it against YOUR resolver
```bash
$ dig api-v2.myapp.com +short
# (empty output)

$ dig api-v2.myapp.com
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 5181
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; AUTHORITY SECTION:
myapp.com.  1800  IN  SOA  ns1.myapp.com. hostmaster.myapp.com. 2024110801 7200 3600 1209600 1800
```
**`NXDOMAIN`** — your resolver believes this name genuinely doesn't exist. But QA says it works. So different resolvers disagree. This is a **propagation / caching** shape, not a total outage.

### Step 2 — Ask multiple resolvers and the authoritative server
```bash
$ dig @8.8.8.8 api-v2.myapp.com +short          # Google Public DNS
# (empty — NXDOMAIN)

$ dig @1.1.1.1 api-v2.myapp.com +short          # Cloudflare
203.0.113.42                                     # ← Cloudflare HAS it!

$ dig @ns1.myapp.com api-v2.myapp.com +noall +answer +stats   # THE SOURCE OF TRUTH
api-v2.myapp.com.	300	IN	A	203.0.113.42
;; flags: qr aa rd; ...                          # ← note: aa (authoritative!) is set
```
The record **exists** and is correct on the authoritative server (`aa` flag proves you hit the source). Cloudflare has it. **Google's `8.8.8.8` is still returning `NXDOMAIN`** — it cached the *negative* answer (the "doesn't exist" result) from *before* you created the record.

### Step 3 — Confirm the cause: negative caching + the SOA minimum TTL
Look back at Step 1's AUTHORITY section: the SOA's last field is **`1800`** — the **negative-cache TTL**. When Google queried this name *before* the record existed, it got `NXDOMAIN` and is allowed to remember "doesn't exist" for **1800 seconds (30 minutes)**. Users whose ISP resolver forwards to Google (or cached NXDOMAIN themselves) are stuck until that expires.

```bash
# Watch a positive answer's TTL count DOWN to confirm caching is at play:
$ dig @1.1.1.1 api-v2.myapp.com | grep -A1 ANSWER
api-v2.myapp.com.	287	IN	A	203.0.113.42
# ...wait 10s, run again:
api-v2.myapp.com.	277	IN	A	203.0.113.42     # TTL dropped 10 → this is a CACHED answer
```
A **countdown** TTL = you're reading a cache. A TTL that resets to the full value (here `300`) every time = you're hitting the authoritative server (or the cache just expired and refreshed).

### Step 4 — Root cause & fix
- **Root cause:** The DNS record was created *correctly*, but *earlier* application traffic (health checks, an eager frontend, or a support engineer's browser) queried the name **before** it existed, seeding a `NXDOMAIN` into public resolver caches. Those negative answers persist for the SOA minimum TTL (30 min).
- **Immediate fix:** Nothing to "push" — you cannot force-purge Google's cache. Communicate the ETA: *"resolves everywhere within 30 minutes (negative-cache TTL); already live on Cloudflare."*
- **Prevention next time:** **Create DNS records FIRST**, wait one negative-TTL window, *then* announce the endpoint. And set a **low TTL** (e.g., 60s) on records you're about to cut over, raising it back to 300–3600s once stable.

### The alternative failure to recognize: SERVFAIL, not NXDOMAIN
If Step 1 had shown `SERVFAIL` instead, the story is completely different — the name *may* exist, but a resolver **couldn't complete the lookup**. Common causes and the `dig` that proves each:

```bash
# Is the authoritative server itself broken/unreachable?
$ dig @ns1.myapp.com api-v2.myapp.com          # times out → the NS is down

# Is it a DNSSEC validation failure? Ask with checking DISABLED:
$ dig api-v2.myapp.com +cd                      # +cd = "don't validate DNSSEC"
# If +cd returns an answer but the normal query is SERVFAIL → DNSSEC is misconfigured
# (e.g., you rotated keys but the DS record at the parent is stale)
```
`SERVFAIL` = "the machinery broke" (dead NS, timeout, bad DNSSEC). `NXDOMAIN` = "the name is definitively absent." Never confuse them — they point at opposite ends of the system.

---

## Common Mistakes

### Mistake 1: Querying only your local cache and calling it "the answer"
```bash
# WRONG: you asked your laptop's cached resolver and declared the deploy done
$ dig api.myapp.com +short
203.0.113.10        # "great, it's live!" — but this is just YOUR cache
```
**Why it's wrong:** Your resolver may hold a stale answer (or a stale negative). It says nothing about what *other* users' resolvers, or the authoritative server, actually hold.
```bash
# RIGHT: triangulate — your resolver, two public ones, and the source of truth
$ dig api.myapp.com +short                        # your resolver
$ dig @8.8.8.8 api.myapp.com +short               # Google
$ dig @1.1.1.1 api.myapp.com +short               # Cloudflare
$ dig @ns1.myapp.com api.myapp.com +short         # authoritative (look for the aa flag)
```
If the authoritative server is right but the public ones lag, it's **propagation/TTL** — wait it out. If the authoritative server is *wrong*, it's a **record bug** — fix your DNS config.

---

### Mistake 2: Ignoring TTL when judging "propagation"
```bash
# You "fixed" a record, re-query, still see the old IP, and panic:
$ dig api.myapp.com +noall +answer
api.myapp.com.	3600	IN	A	203.0.113.10     # old IP, TTL 3600
```
**Why it's wrong:** That `3600` means resolvers may serve the old value for up to **an hour** after your change. Nothing is broken — you're watching the cache expire.
```bash
# Prove it: hit the AUTHORITATIVE server (bypasses all caching):
$ dig @ns1.myapp.com api.myapp.com +noall +answer
api.myapp.com.	60	IN	A	203.0.113.55        # NEW ip is live at the source
```
**The rule:** always **lower the TTL before a planned cutover** (e.g., to 60s a day ahead), migrate, then raise it back. "Propagation delay" is almost always just TTL you set too high.

---

### Mistake 3: Confusing NXDOMAIN with SERVFAIL (and with NOERROR-but-empty)
```
NXDOMAIN → the name does NOT exist. Fix: create the record / fix the typo / check the zone.
SERVFAIL → the resolver FAILED. Fix: check the authoritative NS, timeouts, DNSSEC.
NOERROR + ANSWER: 0 → the name EXISTS but has no record of THAT TYPE.
```
The third is the sneakiest:
```bash
$ dig myapp.com AAAA
;; ->>HEADER<<- ... status: NOERROR, ...; ANSWER: 0    # ← success, but zero answers!
```
`NOERROR` with `ANSWER: 0` is **not** a failure — `myapp.com` exists, it just has no `AAAA` (IPv6) record. Your IPv6-only client will still fail to connect, but the *DNS* is fine. Always read **both** the status **and** the answer count.

---

### Mistake 4: Forgetting the trailing dot / search-domain surprise
```bash
# On a host with `search corp.local` in /etc/resolv.conf:
$ dig api          # NO trailing dot, not an FQDN
;; QUESTION SECTION:
;api.corp.local.   IN   A       # ← resolver appended the search domain! Not what you meant.
```
**Why it's wrong:** A name without a trailing dot may be treated as *relative* and have a search suffix appended, so you're debugging the wrong name. Always check the **QUESTION SECTION** to see what actually got queried.
```bash
# RIGHT: use a fully-qualified name with the trailing dot to disable search suffixes:
$ dig api.myapp.com.        # the trailing dot = "this is absolute, don't append anything"
```

---

### Mistake 5: Not re-checking the authoritative server after a change
```bash
# You edited the record in your DNS provider's UI. Did it actually take effect?
# Asking a cache tells you nothing — it may still hold the old value.
$ dig @$(dig myapp.com NS +short | head -1) api.myapp.com +noall +answer
```
Always confirm the **source of truth** reflects your change (`aa` flag present, correct RDATA, and the **SOA serial incremented**) *before* you blame propagation. If the authoritative server doesn't have your change, no amount of waiting will help.

---

## Hands-On Proof

```bash
# 1. Read a full response top-to-bottom. Find the status, flags, and TTL.
dig example.com

# 2. The TTL countdown = proof of caching. Run twice, ~10s apart, watch it drop:
dig example.com +noall +answer ; sleep 10 ; dig example.com +noall +answer

# 3. Same name, three resolvers — do they agree? (propagation check)
dig @8.8.8.8 example.com +short
dig @1.1.1.1 example.com +short
dig @9.9.9.9 example.com +short

# 4. Go straight to the source of truth. Note the `aa` flag appears here:
dig @$(dig example.com NS +short | head -1) example.com

# 5. Watch the ENTIRE recursive walk: root → TLD → authoritative.
dig +trace example.com

# 6. NXDOMAIN vs SERVFAIL vs NOERROR-empty — see all three:
dig thisdomaindoesnotexist-zzzq12345.com   # NXDOMAIN
dig example.com AAAA                         # NOERROR, ANSWER: 0 (no IPv6 record)
dig @1.1.1.1 example.com +norecurse          # may SERVFAIL/empty if not cached there

# 7. Reverse lookup: which hostname owns an IP?
dig -x 1.1.1.1 +short          # one.one.one.one.
dig -x 8.8.8.8 +short          # dns.google.

# 8. Compare tools on the same query:
dig  example.com +short
host example.com
nslookup example.com
```

### Reading `dig +trace` — the full recursion, hop by hop
This is the money shot for understanding Topic 06. Real output, trimmed:

```
$ dig +trace example.com A

.  513983  IN  NS  a.root-servers.net.        ← HOP 1: the ROOT servers (the "." zone)
.  513983  IN  NS  b.root-servers.net.
...(13 root servers a–m)...
;; Received 525 bytes from 1.1.1.1#53 in 1351 ms   ← dig got the root list from your resolver

com.  172800  IN  NS  a.gtld-servers.net.     ← HOP 2: root REFERS dig to the .com TLD servers
com.  172800  IN  NS  b.gtld-servers.net.
...
com.  86400  IN  DS  19718 13 2 8ACBB0CD...    ← DNSSEC delegation signer for .com
;; Received 1171 bytes from 2001:503:ba3e::2:30#53(a.root-servers.net) in 1801 ms  ← asked a root server

example.com.  172800  IN  NS  hera.ns.cloudflare.com.     ← HOP 3: .com REFERS dig to the
example.com.  172800  IN  NS  elliott.ns.cloudflare.com.  ←        authoritative NS for example.com
;; Received 506 bytes from 192.48.79.30#53(j.gtld-servers.net) in 1361 ms  ← asked a .com server

example.com.  300  IN  A  104.20.23.154        ← HOP 4: the AUTHORITATIVE server gives the answer
example.com.  300  IN  A  172.66.147.243
;; Received 179 bytes from 2803:f800:50::6ca2:c0a2#53(hera.ns.cloudflare.com) in 1119 ms
```

Tie it back to Topic 06's diagram:
```
   dig  ──"who serves . ?"──►  (root list)
    │
    ├──"who serves com. ?"──►  ROOT server ──refers──►  .com TLD servers
    │
    ├──"who serves example.com. ?"──►  .com server ──refers──►  Cloudflare NS
    │
    └──"A record for example.com. ?"──►  Cloudflare NS ──answers──►  104.20.23.154
                                                                     172.66.147.243
```
Each `;; Received … from <server>` line is dig telling you **which machine it just talked to**. Follow those lines and you're literally watching the delegation chain resolve. If the walk **stops** at a hop — say `.com` refers you to a nameserver that then times out — you've pinpointed a **broken delegation**.

---

## Practice Exercises

### Exercise 1 — Easy: Read a response and identify every part
```bash
dig github.com
```
From the output, answer:
1. What is the **status** (RCODE)? How many records are in the ANSWER section?
2. List the **flags**. Is the `aa` flag present? What does its absence tell you about who answered?
3. What is the **TTL** on the answer? Run the command again 15 seconds later — did the TTL go **down**? What does that prove?
4. Which resolver answered (the `SERVER:` line), and how long did the query take (`Query time`)?

### Exercise 2 — Medium: Detect a propagation/caching difference
```bash
# Pick any domain you control, or use one of these public ones.
# Ask three different resolvers AND the authoritative server:
dig @8.8.8.8  wikipedia.org +short
dig @1.1.1.1  wikipedia.org +short
dig           wikipedia.org +short            # your default resolver
dig @$(dig wikipedia.org NS +short | head -1) wikipedia.org +noall +answer +stats
```
Answer:
1. Do all four return the **same** IP(s)? If they differ, is that a bug or normal (geo/anycast) behavior?
2. In the authoritative query, is the `aa` flag set? Why does it appear here but not in the `@8.8.8.8` query?
3. Compare the **TTL** from `@8.8.8.8` vs the authoritative server. Which is a countdown (cached) and which is the full/original value? How can you tell?

### Exercise 3 — Hard (Production Simulation): Diagnose a broken/misconfigured record
```bash
# Scenario: users report intermittent failures reaching `mail`-related features.
# Investigate the mail (MX) and sender-policy (TXT/SPF) config of a domain.

# 1. Find the mail servers and follow the chain to their IPs:
dig gmail.com MX +noall +answer
dig gmail.com MX +short | awk '{print $2}' | while read mx; do
  echo "== $mx =="; dig "$mx" A +short
done

# 2. Check the SPF record (a TXT record starting with "v=spf1"):
dig gmail.com TXT +short | grep spf1

# 3. Now simulate a FAILURE diagnosis. For a made-up subdomain:
dig mail.definitely-not-a-real-zone-xyz987.com

# Questions:
# a) In step 3, is the status NXDOMAIN or SERVFAIL? What is the practical difference
#    for your application's error handling (retry vs fail-fast)?
# b) An MX record's value has a PRIORITY number before the hostname (e.g. "10 mx.host.").
#    What does a lower number mean, and why would a domain list several?
# c) Your app is getting SERVFAIL for a domain that clearly exists in a browser.
#    Write the two dig commands you'd run to decide whether it's (i) a dead
#    authoritative nameserver or (ii) a DNSSEC validation failure.
#    (Hint: query the NS directly; and re-query with +cd.)
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the linked section.

1. **You run `dig api.myapp.com` and see `status: NOERROR` but `ANSWER: 0`.** Does the name exist? Will your app connect? What single word explains what's missing?

2. **What does the `aa` flag mean, and why is it usually *absent* when you query `8.8.8.8` but *present* when you query the domain's own nameserver?**

3. **A record's TTL reads `300` the first time and `287` twelve seconds later.** Which server are you almost certainly talking to — a cache or the authoritative one — and how do you know?

4. **State the difference between `NXDOMAIN` and `SERVFAIL` in one sentence each,** and give the first `dig` command you'd run for each.

5. **In `dig +trace` output, what do the `;; Received N bytes from <server>` lines tell you,** and how would you spot a *broken delegation* from them?

6. **You changed an A record 5 minutes ago. `dig @8.8.8.8` still shows the old IP, but `dig @your-authoritative-ns` shows the new one.** Is anything broken? What determines when Google will show the new value?

7. **Why is `dig` preferred over `nslookup` for debugging,** and what does `+short` give up in exchange for being scriptable?

---

## Quick Reference Card

### Response status codes (read this line FIRST)
| status | Means | Typical cause / fix |
|--------|-------|--------------------|
| `NOERROR` | Success | Check ANSWER count; 0 = no record of that *type* |
| `NXDOMAIN` | Name does not exist | Typo, record never created, wrong zone/search-domain |
| `SERVFAIL` | Resolver failed | Dead authoritative NS, timeout, DNSSEC misconfig |
| `REFUSED` | Server won't serve you | Not an open resolver / ACL blocks you |

### The flags on the `;; flags:` line
| Flag | Name | Who sets it | Meaning |
|------|------|-------------|---------|
| `qr` | Query/Response | server | It's a response |
| `aa` | Authoritative Answer | server | Answer came from the **zone owner** (source of truth) |
| `tc` | Truncated | server | UDP answer too big → retry over TCP |
| `rd` | Recursion Desired | **you** | "Do the full lookup for me" |
| `ra` | Recursion Available | server | Server offers recursion |
| `ad` | Authenticated Data | server | DNSSEC validated |

### The five fields of every record
```
NAME            TTL      CLASS   TYPE   RDATA
example.com.    300      IN      A      104.20.23.154
(owner)     (secs left) (IN)  (record)  (the value)
```

### Common record types
| Type | Answers the question | Example RDATA |
|------|---------------------|---------------|
| `A` | IPv4 address? | `104.20.23.154` |
| `AAAA` | IPv6 address? | `2606:4700::6810:...` |
| `CNAME` | Alias to what canonical name? | `github.com.` |
| `MX` | Mail servers? (priority + host) | `10 mx.host.` (`0 .` = no mail) |
| `NS` | Which nameservers run this zone? | `ns1.cloudflare.com.` |
| `SOA` | Zone authority + timers | `primary admin serial refresh retry expire minTTL` |
| `TXT` | Arbitrary text (SPF/DKIM/verify) | `"v=spf1 include:_spf... ~all"` |
| `CAA` | Which CAs may issue certs? | `0 issue "letsencrypt.org"` |
| `PTR` | Hostname for this IP? (reverse) | `dns.google.` |

### Cheat block — commands you'll actually type
```bash
dig example.com                       # full response, read status→flags→answer
dig example.com +short                # just the value(s) — scriptable
dig example.com +noall +answer        # only the ANSWER section, keeps TTLs
dig example.com MX                    # a specific record type (A/AAAA/MX/TXT/NS/SOA/CAA)
dig @8.8.8.8 example.com              # query a SPECIFIC resolver
dig @ns1.example.com example.com      # query the AUTHORITATIVE server (look for aa)
dig -x 8.8.8.8                        # reverse lookup (IP → PTR hostname)
dig +trace example.com               # watch root → TLD → authoritative
dig example.com +dnssec              # request RRSIG/DNSSEC records
dig example.com +cd                  # DISABLE DNSSEC validation (isolate DNSSEC bugs)
dig example.com +tcp                 # force TCP/53
dig example.com +norecurse           # probe a cache without triggering a full lookup

# Compare all resolvers at once:
for r in 8.8.8.8 1.1.1.1 9.9.9.9; do echo "$r:"; dig @$r example.com +short; done
```

---

## When Would I Use This at Work?

### Scenario 1: "The new endpoint works for us but not for customers"
Run `dig` against your resolver, two public resolvers, and the authoritative NS. If the authoritative server is correct but public resolvers lag, it's **TTL/propagation** — quote the SOA minimum TTL as the ETA. If a public resolver returns `NXDOMAIN` while the source is fine, it cached a **negative** answer — same wait. This is Example 2, and it's the single most common DNS incident.

### Scenario 2: "We rotated our DNS provider / changed nameservers and email broke"
`dig myapp.com NS` shows the delegated nameservers; `dig myapp.com MX @<new-ns>` confirms the mail records made the jump. `dig myapp.com TXT +short | grep spf1` verifies SPF survived. Missing an `MX` or `TXT` during a provider migration silently breaks mail deliverability — `dig` catches it in seconds.

### Scenario 3: "Intermittent SERVFAIL from a partner API"
`SERVFAIL` under load often means one of the partner's authoritative nameservers is flaky, or their DNSSEC is broken. `dig @each-of-their-NS partner.com` finds the dead one; `dig partner.com +cd` vs a normal query isolates DNSSEC. You can then hand the partner a precise bug report instead of "your DNS is weird."

### Scenario 4: Slow-API triage (feeds Topic 27)
When an API is slow, DNS is the first timing bucket. `dig`'s `Query time` tells you if resolution itself is the cost (cold cache, distant resolver, huge `+trace` chain). Pair it with `curl -w "%{time_namelookup}"` (Topic 21) to confirm whether to blame DNS or move on to TCP/TLS.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — where DNS sits in the end-to-end request
- `06-dns-in-depth.md` — the resolution chain, record types, TTL, and caching that `dig` lets you *observe*

**Study alongside / next:**
- `21-curl-mastery.md` — `curl`'s `%{time_namelookup}` timing and `--resolve` to pin an IP; the other half of request debugging
- `27-debugging-a-slow-api.md` — DNS is the first timing bucket; `dig` isolates it from TCP/TLS/TTFB

**Connects forward to:**
- `29-reverse-proxies.md` / `30-cdns.md` — CNAME chains and anycast make `dig` answers vary by location
- `36-service-to-service-communication.md` — internal DNS and service discovery are debugged with the exact same `dig` skills

---

*Doc saved: `/docs/networking/22-dig-and-dns-debugging.md`*
