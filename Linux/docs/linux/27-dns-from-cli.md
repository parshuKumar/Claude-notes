# 27 — DNS from the Command Line

## ELI5 — The Simple Analogy

You want to mail a letter to your grandma, but you only know her name — not her street address.

- **The sticky note on your fridge** is `/etc/hosts`. It has a handful of addresses you wrote down yourself. You check it **first, always**. If Grandma is on the fridge note, you never phone anybody — even if the note is *wrong*.
- **The rule "check the fridge, then phone the directory"** is `/etc/nsswitch.conf`. It's a written household policy about *what to consult, in what order*. Change that line and you change everything downstream.
- **The phone number of the directory service** is `/etc/resolv.conf`. It doesn't contain answers. It contains the number you call to *get* answers.
- **The librarian you call** is your **recursive resolver**. She doesn't know Grandma's address either. But she knows *who to ask*.
- **Her research** is recursion: she calls the national registry ("who handles surnames ending in `.com`?"), which tells her which regional office, which finally tells her the actual street address.
- **Her notebook** is the **cache**. She writes the answer down, and the next 500 people who ask get an instant reply without any phone calls.
- **The expiry date she writes next to it** is the **TTL**. "Good for 24 hours."
- **"DNS propagation"** is a lie. Nobody pushes anything. It's just every librarian's notebook entry expiring, one by one, on its own schedule. **That is why you lower the TTL BEFORE a migration, not after.**

---

## Where This Lives in the Linux Stack

```
Hardware (NIC)
  └── KERNEL
       │   • Routes the UDP packet to port 53 (topic 26)
       │   • That is ALL the kernel does. DNS is NOT a kernel feature.
       │     There is no dns() syscall. The kernel doesn't know what a hostname is.
       │
       └── System Calls: socket(AF_INET, SOCK_DGRAM) / sendto() / recvfrom()
            │              open("/etc/hosts") / open("/etc/resolv.conf")
            │
            └── C Standard Library (glibc)  ◀◀◀ THIS TOPIC LIVES HERE
                 │   getaddrinfo()  ← THE function. Every "connect to a hostname" goes here.
                 │   • reads /etc/nsswitch.conf   → decides the ORDER
                 │   • reads /etc/hosts           → the local override
                 │   • reads /etc/resolv.conf     → which server to ask
                 │   • sends the UDP query itself
                 │
                 ├── systemd-resolved (a userspace stub at 127.0.0.53 — caches, forwards)
                 │
                 └── Programs
                      ├── Shell tools: dig, host, nslookup, getent
                      └── Your Node app: fetch('https://api.shipfast.io/v1/orders')
                                          └── dns.lookup() → getaddrinfo()
```

**The single biggest realization:** DNS is a **userspace library concern**, not a kernel one. That's why `/etc/hosts` can override the entire internet, why containers can have completely different resolution than their host, and why two tools on the same machine can give you two different answers.

---

## What Is This?

DNS turns a **name** (`api.shipfast.io`) into an **address** (`203.0.113.10`) so the kernel has something to route (topic 26). It is a distributed, hierarchical, aggressively-cached database, queried over UDP port 53 (falling back to TCP for large answers).

On Linux, "resolving a name" is not one step. It's a **chain** of local files, local daemons, and remote servers — and almost every DNS bug you will ever hit is caused by not knowing which link of that chain answered.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| The `/etc/hosts` → nsswitch → resolv.conf chain | `dig` will say `NXDOMAIN` while `curl` works fine, and you will not be able to explain it |
| `dig` vs `getent` | You'll debug with a tool that resolves *differently than your app does*, and chase a ghost |
| TTL | Your DNS cutover will strand users on the old server for 24 hours, mid-migration |
| `127.0.0.53` | You'll see it in `resolv.conf`, panic, and "fix" it by hand — where it gets overwritten in 10 minutes |
| CNAME-at-apex | You'll try to point `shipfast.io` (not `www.`) at an ELB hostname and your registrar will reject it |
| ndots:5 in Kubernetes | Every external API call from your pods fires 4+ wasted DNS queries. Your DNS bill and your p99 both suffer. |
| Docker's `127.0.0.11` | Your containers won't be able to find each other by name on the default `bridge` network, and you'll blame Docker |
| `dns.lookup()` vs `dns.resolve()` | You will not understand why a slow DNS server can **stall your entire Node event loop's threadpool** |

---

## The Physical Reality

Three files. Learn them cold.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ /etc/nsswitch.conf     ← THE ROUTER. Decides WHICH source, in WHAT order. │
├──────────────────────────────────────────────────────────────────────────┤
│  hosts:  files mdns4_minimal [NOTFOUND=return] dns                       │
│          │     │                                │                         │
│          │     │                                └── "dns" = do a real     │
│          │     │                                    DNS query using       │
│          │     │                                    /etc/resolv.conf      │
│          │     └── mDNS for *.local (Avahi/Bonjour). Not your problem.    │
│          └── "files" = /etc/hosts.  ★ CHECKED FIRST. ALWAYS. ★            │
│                                                                           │
│  Other values you'll see:                                                 │
│    myhostname  → resolves your own hostname + "gateway" without DNS       │
│    resolve     → ask systemd-resolved over D-Bus instead of UDP           │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ /etc/hosts             ← THE OVERRIDE. Beats the entire internet.         │
├──────────────────────────────────────────────────────────────────────────┤
│  127.0.0.1     localhost                                                  │
│  ::1           localhost ip6-localhost ip6-loopback                       │
│  10.0.0.15     web-01 web-01.internal                                     │
│  203.0.113.99  api.shipfast.io       ← an entry here BEATS real DNS.      │
│  │             │                        No query is ever sent.            │
│  │             └── one or more names (first is canonical, rest are aliases)│
│  └── the IP                                                               │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ /etc/resolv.conf       ← WHO TO ASK. Usually a SYMLINK. Usually GENERATED.│
├──────────────────────────────────────────────────────────────────────────┤
│  nameserver 127.0.0.53          ← up to 3 are used; tried IN ORDER        │
│  options edns0 trust-ad ndots:1                                           │
│  search ec2.internal            ← suffixes appended to short names        │
└──────────────────────────────────────────────────────────────────────────┘
```

```bash
# PROVE IT: resolv.conf is not a real file
ls -l /etc/resolv.conf
# lrwxrwxrwx 1 root root 39 Mar  3 09:12 /etc/resolv.conf ->
#                              ../run/systemd/resolve/stub-resolv.conf
#                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                              /run is a TMPFS. It is REBUILT on every boot.
#                              Anything you type into it is gone. (topic 03: symlinks)
```

### resolv.conf directives, annotated

| Directive | Meaning | Default |
|---|---|---|
| `nameserver <ip>` | A resolver to query. **Max 3 are used.** Tried in order, top first. | — |
| `search a.com b.com` | Suffixes to append to *short* names (see `ndots`) | the local domain |
| `domain x.com` | Legacy single-entry `search`. Ignored if `search` is present. | — |
| `options ndots:N` | Names with **< N dots** get the search list tried **FIRST** | `1` |
| `options timeout:N` | Seconds to wait per nameserver before trying the next | `5` |
| `options attempts:N` | Times to loop over the whole nameserver list | `2` |
| `options rotate` | Round-robin the nameservers instead of always hammering #1 | off |
| `options single-request-reopen` | Fixes a real glibc bug where the parallel A + AAAA queries confuse some broken routers/firewalls and one answer is dropped → **2-second stalls or 5-second timeouts on every lookup** | off |

**Do the math on the defaults:** `timeout:5` × `attempts:2` × 3 nameservers = **30 seconds** of hanging before your app gives up on a name. That is longer than most HTTP client timeouts, and it is why a single dead resolver can look exactly like "the API is down."

---

## How It Works — Step by Step

### The full resolution path — `fetch('https://api.shipfast.io/orders')`

```
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 1. NODE                                                                    │
 │    fetch()/http.request()/net.connect() → dns.lookup('api.shipfast.io')    │
 │    dns.lookup is NOT a DNS client. It calls libuv, which calls...          │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 2. glibc getaddrinfo("api.shipfast.io", "443", ...)                        │
 │    ★ This is a BLOCKING C call. libuv runs it on a THREADPOOL thread.      │
 │      Default pool size: 4. Four slow lookups = the pool is FULL =          │
 │      fs.readFile and crypto and zlib all queue behind DNS.                 │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 3. read /etc/nsswitch.conf   →   "hosts: files dns"                        │
 │    "OK. Try files. Then dns."                                              │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 4. "files"  →  open("/etc/hosts"), scan it linearly                        │
 │                                                                            │
 │    HIT?  → RETURN IMMEDIATELY. ★ NO DNS QUERY IS EVER SENT. ★              │
 │            No TTL. No cache. No network. This is why /etc/hosts wins.      │
 │    MISS? → fall through to "dns"                                           │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 5. "dns"  →  read /etc/resolv.conf                                         │
 │              nameserver 127.0.0.53                                         │
 │              options ndots:1                                               │
 │                                                                            │
 │    ndots check: "api.shipfast.io" has 2 dots. 2 >= ndots(1)?  YES.         │
 │    → query it AS-IS first, don't try the search domains.                   │
 │    (If it had FEWER dots than ndots, the search list is tried FIRST —      │
 │     this is the Kubernetes trap, below.)                                   │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 6. socket(AF_INET, SOCK_DGRAM) → sendto(127.0.0.53:53)                     │
 │    Two queries fired in PARALLEL: one for A, one for AAAA.                 │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 7. systemd-resolved (the STUB at 127.0.0.53 — a userspace process on THIS  │
 │    machine, not a remote server)                                           │
 │      • Checks its own in-memory cache. HIT → return in ~0 ms. DONE.        │
 │      • MISS → forwards to the REAL upstream resolver from DHCP/config      │
 │        (e.g. 10.0.0.2, 1.1.1.1)                                            │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 8. THE RECURSIVE RESOLVER (your ISP's / 1.1.1.1 / your VPC's)              │
 │    Cache miss → it walks the hierarchy FOR you:                            │
 │                                                                            │
 │    ┌─ "." ROOT servers (13 named sets, a.root-servers.net …)               │
 │    │    Q: where is shipfast.io?                                           │
 │    │    A: I don't know, but the .io TLD servers do → NS + glue            │
 │    ▼                                                                       │
 │    ┌─ ".io" TLD servers                                                    │
 │    │    Q: where is shipfast.io?                                           │
 │    │    A: I don't know, but ns1.dnsprovider.com is AUTHORITATIVE → NS     │
 │    ▼                                                                       │
 │    ┌─ ns1.dnsprovider.com — the AUTHORITATIVE server. It OWNS the zone.    │
 │         Q: A record for api.shipfast.io?                                   │
 │         A: 203.0.113.10, TTL 300.   ← flag `aa` = authoritative answer     │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 9. CACHED ON THE WAY BACK — at EVERY layer, each honouring the TTL:        │
 │      • the recursive resolver          (for TTL seconds)                   │
 │      • systemd-resolved on your box    (for TTL seconds)                   │
 │      • your app?  ★ NODE DOES NOT CACHE DNS. AT ALL. BY DEFAULT. ★         │
 │        Every single outbound request re-calls getaddrinfo(). This          │
 │        surprises everyone. Use an HTTP agent with keep-alive, or a         │
 │        caching lookup (e.g. cacheable-lookup), if it hurts.                │
 └───────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ 10. 203.0.113.10 returns to Node → connect() → route lookup (topic 26)     │
 └───────────────────────────────────────────────────────────────────────────┘
```

### `dns.lookup()` vs `dns.resolve()` — a Node-specific trap

| | `dns.lookup()` | `dns.resolve4()` / `dns.resolve()` |
|---|---|---|
| Used by | `http`, `https`, `fetch`, `net.connect`, **everything by default** | only when you call it explicitly |
| Under the hood | glibc `getaddrinfo()` | **c-ares** — Node's own DNS client |
| Reads `/etc/hosts`? | ✅ **YES** | ❌ **NO** |
| Reads `/etc/nsswitch.conf`? | ✅ YES | ❌ NO |
| Blocks? | ✅ Runs on the **libuv threadpool** (4 threads). Can starve `fs`/`crypto`/`zlib`. | ❌ Fully async on the event loop |
| Sees Docker container names? | ✅ (via the container's resolver) | ✅ (it reads resolv.conf, so yes) |

**The consequence:** a Docker container name or an `/etc/hosts` entry resolves fine with `dns.lookup()` (so your `fetch` works) but returns `ENOTFOUND` from `dns.resolve()`. And `dig` behaves like `dns.resolve()` — pure DNS, ignores `/etc/hosts` entirely. **This is the single most confusing thing in Linux DNS, and it has a one-command fix (`getent`, below).**

---

## The Tools — and the One That Actually Tells the Truth

```
┌──────────┬─────────────────┬───────────┬──────────────────────────────────┐
│ Tool     │ What it consults│ /etc/hosts│ Use it when                       │
├──────────┼─────────────────┼───────────┼──────────────────────────────────┤
│ dig      │ DNS servers ONLY│    ❌ NO  │ Investigating DNS itself          │
│ host     │ DNS servers ONLY│    ❌ NO  │ A quick answer                    │
│ nslookup │ DNS servers ONLY│    ❌ NO  │ You're on a box with nothing else │
│ getent   │ ★ nsswitch.conf │    ✅ YES │ ★ "What will MY APP actually get?"│
│ ping     │ nsswitch (libc) │    ✅ YES │ (accidental, but it does)         │
│ curl     │ nsswitch (libc) │    ✅ YES │ (accidental, but it does)         │
└──────────┴─────────────────┴───────────┴──────────────────────────────────┘
```

**`getent hosts` is the only tool that resolves the way your Node app resolves.** It goes through `nsswitch.conf` and `getaddrinfo()` — the exact same code path as `dns.lookup()`. This single fact resolves the two most maddening DNS situations:

```
"dig says NXDOMAIN but curl works!"
   → There's an /etc/hosts entry (or you're in a container using Docker's
     embedded resolver in a way dig is bypassing). dig doesn't read /etc/hosts.
     getent hosts <name> will show you the truth.

"dig works but my app says ENOTFOUND!"
   → Your app is using dns.resolve() (which ignores /etc/hosts), or nsswitch
     is misconfigured, or you're inside a container with different resolv.conf
     than the host you ran dig on.
```

```bash
getent hosts api.shipfast.io
# 203.0.113.10    api.shipfast.io

getent ahostsv4 api.shipfast.io          # IPv4 only, all addresses, with socktypes
# 203.0.113.10    STREAM api.shipfast.io
# 203.0.113.10    DGRAM
# 203.0.113.10    RAW

getent ahosts api.shipfast.io            # both families, in the order libc will TRY them
```

**Rule:** when your app can't resolve something, run `getent`, not `dig`. When you want to know if *DNS itself* is broken, run `dig`.

---

## Record Types

| Type | Holds | Example |
|---|---|---|
| **A** | An IPv4 address | `api.shipfast.io. 300 IN A 203.0.113.10` |
| **AAAA** | An IPv6 address | `api.shipfast.io. 300 IN AAAA 2001:db8::10` |
| **CNAME** | An **alias** to another *name* | `www.shipfast.io. 300 IN CNAME shipfast.io.` |
| **MX** | Mail exchanger + priority (**lower = higher priority**) | `shipfast.io. 3600 IN MX 10 mx1.mailhost.net.` |
| **TXT** | Free text — used for SPF, DKIM, DMARC, domain verification | `shipfast.io. 3600 IN TXT "v=spf1 include:_spf.mailhost.net ~all"` |
| **NS** | Which servers are authoritative for this zone | `shipfast.io. 172800 IN NS ns1.dnsprovider.com.` |
| **SOA** | Start of Authority — the zone's metadata | see below |
| **SRV** | service + port (used by Kubernetes, SIP, XMPP) | `_https._tcp.shipfast.io. 300 IN SRV 10 5 443 api.shipfast.io.` |
| **PTR** | Reverse: IP → name. Lives under `in-addr.arpa`. | `10.113.0.203.in-addr.arpa. IN PTR api.shipfast.io.` |
| **CAA** | Which CAs may issue certs for this domain | `shipfast.io. IN CAA 0 issue "letsencrypt.org"` |

### The CNAME rule that will bite you

**RFC 1034: a CNAME cannot coexist with any other record at the same name.**

The zone apex (`shipfast.io`, with no subdomain) is *required* to have `SOA` and `NS` records. Therefore:

```
❌ ILLEGAL — you cannot do this. Ever.
   shipfast.io.  IN  CNAME  my-lb-1234.eu-west-1.elb.amazonaws.com.
   shipfast.io.  IN  SOA    ...        ← conflict! CNAME can't share a name.
   shipfast.io.  IN  NS     ...        ← conflict!

✅ LEGAL — a subdomain has no SOA/NS, so it may be a CNAME:
   www.shipfast.io.  IN  CNAME  my-lb-1234.eu-west-1.elb.amazonaws.com.
```

So how does everyone point their apex at a load balancer whose IP changes? **Provider-side hacks with different names for the same trick:**

| Provider | Feature |
|---|---|
| Route 53 | `ALIAS` record |
| Cloudflare | **CNAME flattening** (you type a CNAME, they serve an A) |
| DNSimple / others | `ALIAS` / `ANAME` |

They all do the same thing: the *authoritative server* resolves the target itself and hands back a plain **A record** at the apex. It's not a real DNS record type — it's a server-side trick. Every backend dev hits this the first time they point a bare domain at an ELB.

### SOA, and why "minimum" isn't what it sounds like

```
shipfast.io.  3600  IN  SOA  ns1.dnsprovider.com. hostmaster.shipfast.io. (
                     2026071201   ; SERIAL  — bump on every change; secondaries
                                  ;           compare this to know to re-sync
                     7200         ; REFRESH — how often secondaries check the primary
                     3600         ; RETRY   — if that check fails, retry after this
                     1209600      ; EXPIRE  — secondaries stop answering after 14d
                                  ;           with no contact from the primary
                     300 )        ; MINIMUM — ★ NOT a minimum TTL. Since RFC 2308 this
                                  ;   is the NEGATIVE CACHING TTL: how long an NXDOMAIN
                                  ;   is remembered. Set it high and a typo'd record
                                  ;   you just FIXED still returns NXDOMAIN for hours.
```

**If you just created a record and the world still says NXDOMAIN — the negative cache is why.**

---

## TTL — The Thing That Controls Your Migration

```
api.shipfast.io.   300   IN   A   203.0.113.10
                   ^^^
                   "Any resolver may cache this for 300 seconds."
```

TTL is a **contract with every cache on earth**. It is not advisory to *you* — it is binding on *them*. When you change a record, resolvers that already cached the old answer **will keep serving it until their copy expires**. Nothing you do can force them to drop it.

### The DNS cutover runbook

```
       T-48h                    T-0                     T+1h              T+24h
         │                       │                       │                  │
 ┌───────▼────────┐    ┌─────────▼────────┐    ┌─────────▼──────┐  ┌────────▼───────┐
 │ LOWER the TTL  │    │ CHANGE the record│    │ Old server is  │  │ RAISE the TTL  │
 │ 3600s → 60s    │    │ 203.0.113.10     │    │ still getting  │  │ 60s → 3600s    │
 │                │    │   → 198.51.100.7 │    │ ~0 traffic.    │  │                │
 │ Now WAIT for   │    │                  │    │ Verify with    │  │ (low TTL = more │
 │ the OLD TTL    │    │ Caches hold it   │    │ access logs,   │  │  queries = more │
 │ (3600s) to     │    │ for at most 60s  │    │ not with dig.  │  │  cost + latency)│
 │ fully expire.  │    │ now. ✅          │    │                │  │                │
 │ ★ THIS WAIT IS │    │                  │    │ THEN shut it   │  │                │
 │   THE WHOLE    │    │                  │    │ down.          │  │                │
 │   POINT ★      │    │                  │    │                │  │                │
 └────────────────┘    └──────────────────┘    └────────────────┘  └────────────────┘

 ❌ Skip the T-48h step and: caches hold the OLD IP for up to 3600s (or longer —
    some resolvers clamp/ignore short TTLs). Your users hit a decommissioned box.
    You cannot fix it. You can only wait.

 ⚠️  KEEP THE OLD SERVER ALIVE for at least the old TTL after the cutover. Always.
```

---

## `dig` — Read It Section by Section

```bash
dig api.shipfast.io
```

```
; <<>> DiG 9.18.18-0ubuntu0.22.04.2-Ubuntu <<>> api.shipfast.io
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51828
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;api.shipfast.io.               IN      A

;; ANSWER SECTION:
api.shipfast.io.        287     IN      A       203.0.113.10

;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Sun Jul 12 09:31:44 UTC 2026
;; MSG SIZE  rcvd: 60
```

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51828
                                      │
     ┌────────────────────────────────┘
     │  status is the FIRST thing you read:
     │    NOERROR   → the query succeeded. (An empty ANSWER + NOERROR means
     │                "the NAME exists, but not with THAT record type." Common
     │                when you dig AAAA on an IPv4-only host. NOT an error.)
     │    NXDOMAIN  → ★ THE NAME DOES NOT EXIST AT ALL. A typo, or not published.
     │    SERVFAIL  → the resolver TRIED and FAILED. Broken DNSSEC, dead
     │                authoritative server, or the resolver itself is sick.
     │                ★ NXDOMAIN = "no such name." SERVFAIL = "I'm broken."
     │                 These have COMPLETELY different fixes. Never confuse them.
     │    REFUSED   → the server won't answer you (not a resolver for you, or ACL'd)

;; flags: qr rd ra
            │  │  │
            │  │  └── ra = Recursion Available (this server WILL do the work for you)
            │  └── rd = Recursion Desired (you asked it to)
            └── qr = this is a Response
            aa      = ★ AUTHORITATIVE ANSWER — you are talking to the server that
                      OWNS this zone. If `aa` is ABSENT, you got a CACHED answer
                      from a middleman, and it may be STALE.

;; QUESTION SECTION:   what you asked. Sanity-check it — did the search domain get
;;                     appended? Is it the name you thought?

;; ANSWER SECTION:
api.shipfast.io.        287     IN      A       203.0.113.10
│                       │       │       │       │
│                       │       │       │       └── the data (RDATA)
│                       │       │       └── record type
│                       │       └── class (always IN — "Internet")
│                       └── ★ TTL, in seconds. And here's the trick:
│                           this is the REMAINING time in the CACHE, counting DOWN.
│                           Run dig twice: 287 … 281 … → you are hitting a CACHE.
│                           A TTL that stays PINNED at its full value (e.g. always
│                           300) → you're hitting the AUTHORITATIVE server.
└── the name (note the trailing dot — the root)

;; AUTHORITY SECTION:  the NS records for the zone (who you should ask). Present when
;;                     you get a referral or an NXDOMAIN (with the SOA, showing the
;;                     negative-cache TTL).
;; ADDITIONAL SECTION: bonus records the server volunteered — usually the A records
;;                     ("glue") for the nameservers in AUTHORITY, saving you a lookup.

;; Query time: 0 msec      ← 0-2 ms = a CACHE HIT (local!). 30-200 ms = a real query.
;; SERVER: 127.0.0.53#53   ← ★ WHICH RESOLVER ACTUALLY ANSWERED YOU. This one line
;;                            tells you whether you're reading your local systemd
;;                            stub's cache or the real world. Read it. Every time.
;; MSG SIZE rcvd: 60       ← >512 bytes historically forced a TCP retry
```

### The four `dig` invocations that solve real problems

```
dig @8.8.8.8 api.shipfast.io
    │
    └── ★ BYPASS YOUR ENTIRE LOCAL STACK. Skip systemd-resolved, skip your ISP,
        skip your corporate resolver, skip your VPC resolver. Ask a known-good
        public resolver directly.

        THE DECISION TREE:
          dig api.shipfast.io        → fails
          dig @8.8.8.8 api.shipfast.io → WORKS   → your LOCAL resolver is broken/stale.
          dig @8.8.8.8 api.shipfast.io → FAILS   → DNS is genuinely broken. Not you.

        This is the highest-value 10 seconds in DNS debugging.
```

```
dig +trace api.shipfast.io
    │
    └── ★ DO THE RECURSION YOURSELF. dig starts at the ROOT servers and walks the
        delegation chain down, printing each hop, using NO cache at any step.

        Root (.)  →  .io TLD  →  ns1.dnsprovider.com  →  the answer

        THE definitive tool for: "I created the record at my registrar an hour ago
        and it works for me but not for anyone else."  +trace shows you the
        delegation as the WORLD sees it, with zero caching. If +trace fails at the
        TLD step, your NS records at the registrar are wrong. If +trace succeeds
        but a plain dig fails, it's a cache.
```

```
dig +short api.shipfast.io          # 203.0.113.10        ← scriptable, just the data
dig +noall +answer api.shipfast.io  # the ANSWER section only, WITH the TTL. The
                                    # best "give me the facts" default for humans.
dig -x 203.0.113.10                 # REVERSE lookup (dig builds the .in-addr.arpa
                                    # name for you)
dig MX shipfast.io                  # any type: A AAAA MX TXT NS SOA CNAME SRV CAA
dig ANY shipfast.io                 # ⚠️ mostly USELESS now — RFC 8482, most servers
                                    #   answer ANY with a stub. Don't rely on it.
dig @ns1.dnsprovider.com shipfast.io   # ★ ask the AUTHORITATIVE server directly.
                                    #   Compare its answer to @8.8.8.8's answer.
                                    #   Different? → it's a cache, and you just wait.
                                    #   Same? → it's real, and it's you.
dig +tcp / dig +notcp               # force transport
dig +nsid / dig +stats              # which anycast node answered
```

---

## Exact Syntax Breakdown

```
dig @8.8.8.8 api.shipfast.io A +noall +answer +trace
│   │        │               │ │            │
│   │        │               │ │            └── walk the delegation from the root
│   │        │               │ └── show ONLY the ANSWER section
│   │        │               │      (+noall turns everything off, +answer turns one on)
│   │        │               └── record TYPE (default: A)
│   │        └── the NAME to query
│   └── @SERVER — which resolver to ask. OMIT and it uses /etc/resolv.conf.
└── dig = "Domain Information Groper" (from BIND's dnsutils / bind9-dnsutils package)
```

```
getent hosts api.shipfast.io
│      │     │
│      │     └── the name to resolve
│      └── the NSSWITCH DATABASE to query.
│          hosts     → resolve a name (follows nsswitch: files, then dns)
│          ahostsv4  → IPv4 only, all addrs
│          ahostsv6  → IPv6 only
│          ahosts    → both, in the order libc will actually TRY them
│          (also: passwd, group, services — topic 06)
└── getent = "get entries" — from glibc's NSS. ★ Resolves EXACTLY like your app does.
```

```
resolvectl query api.shipfast.io
│          │     │
│          │     └── the name
│          └── query | status | flush-caches | statistics | dns | domain
└── resolvectl — the CORRECT tool when systemd-resolved is running.
    (`systemd-resolve` is the old name for the same binary. Same thing.)

resolvectl status              # ★ which resolver is used PER INTERFACE. The VPN's
                               #   DNS vs your LAN's DNS. Shows the truth that
                               #   /etc/resolv.conf (127.0.0.53) hides from you.
resolvectl flush-caches        # ★ THE way to clear a stale record on Ubuntu.
                               #   NOT `/etc/init.d/networking restart`.
                               #   NOT rebooting. This.
resolvectl statistics          # cache hit rate — is your resolver even helping?
```

```
tcpdump -i any -nn port 53
                       │
                       └── watch every DNS query leave the box, live.
                           The nuclear option for "is my app even ASKING?"
                           Answers "is it DNS?" in five seconds, definitively.
```

---

## systemd-resolved and the 127.0.0.53 Mystery

You `cat /etc/resolv.conf` on a fresh Ubuntu box and see:

```
nameserver 127.0.0.53
options edns0 trust-ad
search .
```

**"My nameserver is my own machine?! Who broke this?"** Nobody. This is correct.

```
   Your app  ──getaddrinfo()──▶  127.0.0.53:53
                                       │
                                 ┌─────▼──────────────────────────┐
                                 │ systemd-resolved               │
                                 │  (a normal userspace process!) │
                                 │                                │
                                 │  • an in-memory DNS CACHE      │
                                 │  • PER-INTERFACE resolvers     │
                                 │    (VPN DNS for VPN names,     │
                                 │     LAN DNS for LAN names)     │
                                 │  • DNSSEC validation           │
                                 │  • DNS-over-TLS                │
                                 └─────┬──────────────────────────┘
                                       │  forwards on a cache MISS
                                       ▼
                              the REAL upstream (10.0.0.2, 1.1.1.1 …)
                              — learned from DHCP or netplan
```

`127.0.0.53` is a **stub listener**. It exists so that `/etc/resolv.conf` — a file with a dumb, 30-year-old format that can't express "use the VPN's DNS for `*.corp` and the LAN's DNS for everything else" — can stay simple, while the real logic lives in a daemon that *can* express that.

**The consequence:** `/etc/resolv.conf` no longer tells you which server actually answers. `dig` will always report `SERVER: 127.0.0.53`. To see the real upstream:

```bash
resolvectl status
# Global
#        Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
# resolv.conf mode: stub
#
# Link 2 (ens3)
#     Current Scopes: DNS
#          Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS
# Current DNS Server: 10.0.0.2                 ← ★ THE REAL ONE
#        DNS Servers: 10.0.0.2 1.1.1.1
#         DNS Domain: eu-west-1.compute.internal
#
# Link 5 (wg0)
#     Current Scopes: DNS
#          Protocols: +DefaultRoute ...
# Current DNS Server: 10.99.0.1                ← the VPN's DNS, for VPN names only
#         DNS Domain: ~corp.internal           ← "~" = a ROUTING-ONLY domain

resolvectl flush-caches    # clear the cache. This is the "fix" for a stale record.
```

**Who overwrites `/etc/resolv.conf`, in practice:**

| Writer | Where it points |
|---|---|
| systemd-resolved | `/run/systemd/resolve/stub-resolv.conf` (127.0.0.53) or `resolv.conf` (real servers) |
| NetworkManager | writes `/etc/resolv.conf` directly, or via `resolvconf` |
| the DHCP client | rewrites it on every lease renewal |
| cloud-init | rewrites it on first boot |
| Docker | **generates a fresh one inside every container** |
| `resolvconf` package | merges from many sources |

**Editing `/etc/resolv.conf` by hand is a mistake in every one of these cases.** It works, briefly, and then it is silently reverted — usually at 3 a.m. during a DHCP renewal.

---

## The `ndots` Trap (Kubernetes)

A pod's `/etc/resolv.conf`:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`ndots:5` means: **if the name has fewer than 5 dots, try every search domain FIRST, and only try the name as-typed LAST.**

Your app calls `fetch('https://api.stripe.com/v1/charges')`. `api.stripe.com` has **2 dots**. 2 < 5. So:

```
  QUERY 1: api.stripe.com.default.svc.cluster.local   → NXDOMAIN   ✗
  QUERY 2: api.stripe.com.svc.cluster.local           → NXDOMAIN   ✗
  QUERY 3: api.stripe.com.cluster.local               → NXDOMAIN   ✗
  QUERY 4: api.stripe.com.                            → 203.0.113.5 ✅  ← finally

  ...and getaddrinfo fires A **and** AAAA for each → ★ 8 DNS QUERIES for ONE lookup.
```

Multiply by every outbound HTTP request from every pod (remember: **Node does not cache DNS**). You get:
- CoreDNS CPU pinned, and a cluster-wide DNS outage under load
- +5–20 ms added to the p99 of every external call
- Mysterious intermittent `EAI_AGAIN` when CoreDNS starts dropping queries

**Why does `ndots:5` exist at all?** So that `my-service` and `my-service.other-namespace` (0 and 1 dots) resolve inside the cluster without you writing the FQDN. It's a convenience that costs you on every *external* call.

**The fixes, in order of preference:**

```javascript
// 1. THE TRAILING DOT. A name ending in "." is fully qualified — the search list is
//    SKIPPED ENTIRELY. One query. Zero config change. Free.
fetch('https://api.stripe.com./v1/charges')
//                          ^  ← yes, really. It is valid, and it works.
//    (⚠️ but it can break TLS SNI / Host-header matching with some servers and
//     some HTTP clients — test it. This is the one gotcha.)
```

```yaml
# 2. Lower ndots for the pod that makes lots of external calls:
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"        # now api.stripe.com (2 dots) is tried AS-IS first
```

```
# 3. NodeLocal DNSCache — a per-node DNS cache. Solves the LOAD problem,
#    not the wasted-query problem.
# 4. Use an HTTP agent with keep-alive so you resolve once per connection,
#    not once per request. (Do this anyway. It's free performance.)
```

---

## Docker DNS

```bash
docker exec api cat /etc/resolv.conf
# nameserver 127.0.0.11
# options ndots:0
```

**`127.0.0.11`** is Docker's **embedded DNS server**, running inside the container's network namespace (topic 26). Docker DNATs traffic to it. It:
- resolves **container names** and **network aliases** on the same user-defined network
- forwards everything else to the host's real resolvers

```bash
# ★ THE CLASSIC. The DEFAULT bridge network does NOT do name resolution.
docker run -d --name db  postgres:16
docker run -it --rm      postgres:16 psql -h db
# psql: error: could not translate host name "db" to address:
#       Name or service not known                  ← ✗ on the default bridge

# ✅ A USER-DEFINED network DOES:
docker network create app-net
docker run -d --name db --network app-net postgres:16
docker run -it --rm --network app-net postgres:16 psql -h db -U postgres
# psql (16.2)  ← ✅ "db" resolved to the container's IP, by name

# docker-compose creates a user-defined network for you automatically, which is why
# `postgres://db:5432` "just works" in compose but not with plain `docker run`.
```

```bash
# Prove which resolver a container is using — and that dig can't see container names:
docker exec api getent hosts db      # 172.18.0.3   db     ← ✅ works (nsswitch → 127.0.0.11)
docker exec api dig +short db        # (empty or NXDOMAIN if dig bypasses/misconfigures)
# ★ getent tells you what your APP will get. Always trust getent over dig here.

# Static overrides without touching DNS:
docker run --add-host=api.shipfast.io:203.0.113.99 myimage
#   → writes a line into the container's /etc/hosts. Beats DNS. Great for
#     pinning to a staging IP without changing code.
```

---

## Example 1 — Basic

```bash
# Who am I asking?
cat /etc/resolv.conf
# nameserver 127.0.0.53      ← the systemd-resolved stub
resolvectl status | grep 'Current DNS'
# Current DNS Server: 10.0.0.2         ← the REAL upstream

# A basic lookup
dig +noall +answer github.com
# github.com.  60  IN  A  140.82.121.4

# Just the IP, for a script
dig +short github.com

# What does my APP see? (the one that matters)
getent hosts github.com

# Prove /etc/hosts wins over the entire internet
echo "203.0.113.99  github.com" | sudo tee -a /etc/hosts
getent hosts github.com
# 203.0.113.99   github.com        ← ✅ the fake IP. libc never asked DNS.
dig +short github.com
# 140.82.121.4                     ← ✗ dig IGNORES /etc/hosts. Different answers!
                                   #    THIS IS THE WHOLE LESSON.
sudo sed -i '/203.0.113.99/d' /etc/hosts     # undo it

# Watch the TTL count down — proving you're hitting a cache
dig +noall +answer github.com ; sleep 5 ; dig +noall +answer github.com
# github.com.  57  IN  A  140.82.121.4
# github.com.  52  IN  A  140.82.121.4      ← ★ it decremented. That's a CACHE.

# Other record types
dig +short MX  gmail.com
dig +short TXT github.com
dig +short NS  github.com
dig -x 8.8.8.8              # dns.google.
```

---

## Example 2 — Production Scenario

**03:40.** Your Node API is throwing, in a burst, then stopping, then bursting again:

```
Error: getaddrinfo EAI_AGAIN db.internal.shipfast.io
    at GetAddrInfoReqWrap.onlookup [as oncomplete] (node:dns:107:26)
```

`EAI_AGAIN` is a **temporary** failure from `getaddrinfo` — "I asked and got no usable answer." (Contrast `ENOTFOUND` = `EAI_NONAME` = **NXDOMAIN**, the name genuinely doesn't exist.) So the name probably *does* exist and something is *timing out*. Triage.

```bash
# ── 1. What does the APP see? (getent, not dig — same code path as dns.lookup)
time getent hosts db.internal.shipfast.io
# (hangs...)
#
# real    0m10.021s      ← ★ TEN SECONDS. timeout:5 × attempts:2. It's TIMING OUT,
# (no output)               not being told "no". That's a RESOLVER problem, not a
#                           record problem.

# ── 2. Is DNS itself broken, or is MY resolver broken? Bypass everything.
dig @8.8.8.8 +short db.internal.shipfast.io
# (empty)          ← well, it's an INTERNAL name. A public resolver can't see it. Fine.

dig @10.0.0.2 +short db.internal.shipfast.io      # the actual VPC resolver
# ;; communications error to 10.0.0.2#53: timed out
# ;; communications error to 10.0.0.2#53: timed out
#                  ← ★★★ THERE IT IS. The upstream resolver is not answering.
#                    Not "wrong answer." NO answer. The record is irrelevant.

# ── 3. Confirm it's the resolver and not us
dig @8.8.8.8 +short github.com
# 140.82.121.4     ← ✅ DNS to the outside world works fine. Our network is fine.
#                     It is specifically 10.0.0.2 that is dead or overloaded.

# ── 4. Is it dead, or is it OVERLOADED? (intermittent = overloaded)
for i in $(seq 1 20); do
  dig @10.0.0.2 +short +time=2 +tries=1 db.internal.shipfast.io || echo TIMEOUT
done | sort | uniq -c
#   14 TIMEOUT
#    6 10.0.0.30
#                  ← ★ 30% success. It's ALIVE but DROPPING QUERIES. Overloaded.
#                    That explains the BURSTS of errors, not a steady failure.

# ── 5. Who is hammering it? Are WE?
sudo tcpdump -i any -nn port 53 -c 200 2>/dev/null | awk '{print $NF}' | sort \
  | uniq -c | sort -rn | head -5
#  148 A? db.internal.shipfast.io. (44)
#   12 A? api.stripe.com. (35)
#                  ★ 148 queries for the SAME name in the time it took to catch 200
#                    packets. Node does NOT cache DNS, and our HTTP client has NO
#                    keep-alive → we re-resolve on EVERY request. WE are the load.

# ── 6. IMMEDIATE MITIGATION (buy time — put the answer in /etc/hosts so libc
#       never sends a query at all)
resolvectl query db.internal.shipfast.io    # get the IP from any source you trust
echo "10.0.0.30  db.internal.shipfast.io" | sudo tee -a /etc/hosts
getent hosts db.internal.shipfast.io
# 10.0.0.30   db.internal.shipfast.io       ← ✅ instant. Zero DNS traffic.
systemctl restart shipfast-api              # (topic 22)
# → errors stop. ⚠️ Now you OWE yourself a task to remove this before the DB IP
#   ever changes, or you have created a time bomb. Write the ticket NOW.

# ── 7. REAL FIXES (in the postmortem)
#   a) Add keep-alive to the HTTP/DB client → 1 lookup per connection, not per request
#   b) Add a DNS cache in Node (cacheable-lookup) → honour the TTL
#   c) Add a local caching resolver on the box (systemd-resolved is already there —
#      make sure the app actually goes through it)
#   d) Harden resolv.conf so ONE slow server can't cost 10 seconds:
#         options timeout:1 attempts:2 rotate
#      → worst case 2s instead of 10s, and `rotate` spreads load across all
#        nameservers instead of always beating on #1.
#   e) Scale the VPC resolver / add a second nameserver
```

**The chain of reasoning:** `EAI_AGAIN` (not `ENOTFOUND`) → timeout, not NXDOMAIN → `getent` reproduced it with a 10s hang → `dig @<upstream>` proved the resolver, not the record, was at fault → a loop proved it was *intermittent* (overload, not death) → `tcpdump` proved *we* were the load → `/etc/hosts` stopped the bleeding in 30 seconds.

---

## Common Mistakes

### Mistake 1 — Editing `/etc/resolv.conf` by hand

**Wrong:**
```bash
sudo vim /etc/resolv.conf     # change nameserver to 1.1.1.1
# ...works! ...for about ten minutes.
```

**Root cause:** It's a symlink to a file in `/run` (a tmpfs), owned by `systemd-resolved`, NetworkManager, the DHCP client, or cloud-init. Any of them will regenerate it — on the next DHCP lease renewal, on the next `netplan apply`, on the next boot. Your edit vanishes and the outage returns at the worst possible moment, with no obvious cause.

**Diagnose:** `ls -l /etc/resolv.conf`. If it's a symlink into `/run`, you must not edit it.

**Fix — set DNS at the source that OWNS it:**

```yaml
# Ubuntu / netplan:  /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false            # ← stop DHCP from stomping on our choice
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
        search: [internal.shipfast.io]
```
```bash
sudo netplan try        # auto-reverts in 120s if you don't confirm (topic 26)
```
```ini
# Or, systemd-resolved directly: /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1 8.8.8.8
Domains=internal.shipfast.io
```
```bash
sudo systemctl restart systemd-resolved && resolvectl status
```

**Prevent:** Never `vim /etc/resolv.conf`. Find the generator, configure the generator.

---

### Mistake 2 — Debugging your app's resolution with `dig`

**Wrong:** "`dig api` says NXDOMAIN, so the container name isn't registered — Docker's DNS is broken."

**Root cause:** `dig` speaks **only DNS**. It ignores `/etc/nsswitch.conf` and it ignores `/etc/hosts`. Your app calls `getaddrinfo()`, which reads **both**. They are literally different code paths. They *can and will* disagree, and both are telling the truth about their own path.

```bash
# ❌ dig api.shipfast.io          → NXDOMAIN   ("DNS doesn't know this name")
# ✅ getent hosts api.shipfast.io → 10.0.0.30  ("but /etc/hosts does, and so
#                                                will your app")
```

**Fix:** Use `getent hosts <name>` to answer "**what will my app get?**" Use `dig` to answer "**what does DNS say?**" They are different questions. Ask the right one.

**Prevent:** Make `getent hosts` your reflex, and `dig` your second command.

---

### Mistake 3 — Cutting over DNS without lowering the TTL first

**Wrong:** Change the A record at 2 p.m. on migration day. Watch `dig +short` from your laptop, see the new IP, declare victory, and shut down the old server.

**Root cause:** The old record had `TTL 3600`. Every resolver on earth that looked it up in the last hour has it cached and **will keep serving the old IP for up to another hour** — some resolvers even clamp TTLs to a minimum of their own. Your laptop's resolver happened to have a cold cache. Everyone else's didn't. You just took down production for the users you can't see.

**Diagnose (before you cut over):**
```bash
dig +noall +answer api.shipfast.io          # what's the CURRENT TTL?
# api.shipfast.io.  3600  IN  A  203.0.113.10       ← one hour of blindness ahead
```

**Fix:** Lower the TTL to 60s. **Then wait for the OLD TTL to fully elapse** (3600s) so that every cache in the world has re-fetched the record *and learned the new short TTL*. Only then change the IP. Keep the old server running for at least the old TTL after the switch. Raise the TTL back a day later.

**Prevent:** Put "lower TTL to 60" in the migration runbook as a step **48 hours before** the maintenance window. And verify the cutover against the **authoritative** server, not against your laptop:
```bash
dig @ns1.dnsprovider.com +noall +answer api.shipfast.io   # the source of truth
dig @8.8.8.8            +noall +answer api.shipfast.io   # what the world sees
# When these agree, propagation is done. That's the definition.
```

---

### Mistake 4 — Trying to CNAME the apex domain

**Wrong:**
```
shipfast.io.  CNAME  my-lb-1234.eu-west-1.elb.amazonaws.com.
# → the DNS provider rejects it, or (worse) accepts it and silently breaks
#   your MX records so all your email vanishes.
```

**Root cause:** RFC 1034. A CNAME cannot coexist with any other record at the same name. The apex **must** have `SOA` and `NS`. Therefore the apex can never be a CNAME. If a provider "lets" you, it's a non-standard hack that will break the coexisting `MX`/`TXT` records.

**Fix:** Use your provider's apex-alias feature — Route 53 `ALIAS`, Cloudflare CNAME flattening, DNSimple `ALIAS`. They resolve the target server-side and return a plain `A`. Or, if the LB has a stable IP, just use an `A` record.

**Prevent:** Remember the rule: **`www.` can be a CNAME. The bare domain cannot.**

---

### Mistake 5 — "Just flush the DNS cache" (from the wrong layer)

**Wrong:** `sudo systemctl restart networking` / reboot the server / `sudo /etc/init.d/nscd restart` on a box that isn't running nscd.

**Root cause:** There are **four or five independent caches** in the path, and restarting the wrong one does nothing:

```
  1. Your Node app          → doesn't cache by default (unless you added one)
  2. systemd-resolved       → resolvectl flush-caches      ← usually this one
  3. nscd (legacy)          → sudo nscd -i hosts
  4. Docker's 127.0.0.11    → restart the container
  5. Your upstream resolver → ★ YOU CANNOT FLUSH IT. Only the TTL can.
  6. The world's resolvers  → ★ definitely cannot flush. Only the TTL.
```

**Fix:** Identify which cache is stale first, with the TTL trick:
```bash
dig +noall +answer api.shipfast.io ; sleep 3 ; dig +noall +answer api.shipfast.io
# TTL is COUNTING DOWN (287 → 284)  → a CACHE is serving you. Flush it locally.
# TTL is PINNED at full (300 → 300) → you're hitting the AUTHORITATIVE server.
#                                      The record itself is what's wrong. Fix the record.
```

**Prevent:** Internalize that **you cannot flush a cache you don't own.** "DNS propagation" is not a process you can accelerate — it's TTL expiry, and the only lever you ever had was setting a low TTL *in advance*.

---

## Hands-On Proof

```bash
# PROVE IT: resolv.conf is a generated symlink into a tmpfs
ls -l /etc/resolv.conf
findmnt /run | head -2               # tmpfs — it does not survive a reboot

# PROVE IT: /etc/hosts beats the entire internet
echo "127.0.0.1 www.google.com" | sudo tee -a /etc/hosts
curl -sS -m3 -o /dev/null -w '%{remote_ip}\n' http://www.google.com
# 127.0.0.1        ← ✅ curl went to LOOPBACK. libc never sent a DNS query.
getent hosts www.google.com          # 127.0.0.1
dig +short www.google.com            # the REAL IPs — dig ignored /etc/hosts entirely
sudo sed -i '/127.0.0.1 www.google.com/d' /etc/hosts

# PROVE IT: nsswitch.conf is the router
grep '^hosts:' /etc/nsswitch.conf
# hosts: files mdns4_minimal [NOTFOUND=return] dns
#        ^^^^^ ← "files" comes FIRST. That is the entire reason /etc/hosts wins.

# PROVE IT: getaddrinfo() reads these files. Watch the syscalls. (topic 01!)
strace -f -e trace=openat,connect,sendto getent hosts github.com 2>&1 \
  | grep -E 'nsswitch|hosts|resolv|connect'
# openat(AT_FDCWD, "/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 3
# openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 3          ← files, first
# openat(AT_FDCWD, "/etc/resolv.conf", O_RDONLY|O_CLOEXEC) = 4    ← then dns
# connect(4, {sa_family=AF_INET, sin_port=htons(53),
#             sin_addr=inet_addr("127.0.0.53")}, 16) = 0          ← the UDP query!
# ★ You just watched the entire chain from this doc, in syscalls. Nothing is magic.

# PROVE IT: dig does NOT read these files
strace -e trace=openat dig +short github.com 2>&1 | grep -c '/etc/hosts'
# 0                    ← it never even OPENS it.

# PROVE IT: the TTL counts down in a cache and is pinned at the authority
dig +noall +answer github.com; sleep 4; dig +noall +answer github.com

# PROVE IT: two containers, two different DNS worlds
docker network create proofnet
docker run -d --name db --network proofnet postgres:16
docker run --rm --network proofnet busybox getent hosts db     # ✅ resolves
docker run --rm                     busybox getent hosts db     # ✗ default bridge: nothing
docker rm -f db && docker network rm proofnet

# PROVE IT: which resolver ACTUALLY answered
dig github.com | grep SERVER
# ;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)      ← the local stub, not the internet
resolvectl status | grep 'Current DNS Server'      # the real upstream behind it
```

---

## Practice Exercises

### Exercise 1 — Easy

In the terminal:
1. Show your `/etc/resolv.conf`. Is it a symlink? To what? Who generates it?
2. Show the `hosts:` line from `/etc/nsswitch.conf`. Which source is consulted first?
3. `dig +noall +answer github.com`. What is the TTL? Run it again 5 seconds later — did the TTL change? What does that tell you?
4. `dig github.com | grep SERVER`. Which resolver answered? Is that a real DNS server or a local stub?
5. Get github.com's MX records, TXT records, and NS records. Reverse-lookup `1.1.1.1`.

### Exercise 2 — Medium

Prove, with commands, that `dig` and `getent` resolve differently — and explain **why** at the level of files and syscalls.

```bash
# 1. Add "203.0.113.77  test.example.com" to /etc/hosts
# 2. Run BOTH:  getent hosts test.example.com   and   dig +short test.example.com
#    Show that they disagree. Explain which one your Node app will agree with, and why.
# 3. Prove it with Node itself:
node -e "
  const dns = require('dns');
  dns.lookup('test.example.com',   (e,a) => console.log('lookup :', e?e.code:a));
  dns.resolve4('test.example.com', (e,a) => console.log('resolve:', e?e.code:a));
"
#    → predict the two outputs BEFORE you run it. Then run it. Were you right?
# 4. strace `getent` and find the openat() of /etc/hosts. strace `dig` and show
#    that it never opens it.
# 5. Clean up /etc/hosts.
```

### Exercise 3 — Hard (Production Simulation)

**Simulate a DNS outage and a DNS cutover, end to end.**

```bash
# PART A — Break DNS deliberately, then diagnose it like an incident.
#   1. Back up /etc/resolv.conf's target. Point systemd-resolved (or resolv.conf,
#      in a container) at a BLACKHOLE:  nameserver 192.0.2.1   (TEST-NET-1: nothing
#      is there; packets go into the void → TIMEOUTS, not refusals).
#   2. `time getent hosts github.com` — how long does it hang? Explain the number
#      using timeout: and attempts: from resolv.conf. Do the arithmetic.
#   3. Prove the resolver (not DNS) is the problem, using `dig @8.8.8.8`.
#   4. Add `options timeout:1 attempts:1`. Time it again. Explain the new number.
#   5. Restore.

# PART B — Trace a real delegation with NO cache.
#   dig +trace api.github.com
#   → Identify: which root server answered? Which TLD servers? Which authoritative
#     servers? At which step does the `aa` flag first appear? Why not before?

# PART C — Plan a cutover for api.shipfast.io: 203.0.113.10 → 198.51.100.7.
#   Write the runbook as a numbered list with EXACT commands and EXACT wait times.
#   It must answer:
#     - What is the FIRST thing you do, and how long before the change?
#     - How long must you wait after lowering the TTL, and WHY that number?
#     - How do you VERIFY the cutover? (dig against WHAT, specifically?)
#     - When is it safe to shut down the old server? Justify the timing.
#     - What do you do LAST, and how long after?

# PART D — Reproduce the ndots problem.
#   In a container, set:  options ndots:5  and  search a.local b.local c.local
#   Run `tcpdump -i any -nn port 53` in another terminal and then
#   `getent hosts api.github.com`. COUNT the queries. Now try `api.github.com.`
#   (trailing dot) and count again. Report both numbers.
```

---

## Mental Model Checkpoint

1. **Trace `fetch('https://api.shipfast.io')` from Node all the way to a UDP packet.** Name every file consulted, in order, and the libc function at the center of it.
2. **Why can `dig` return NXDOMAIN for a name that `curl` reaches perfectly?** Which tool would you run to settle it in one command, and why does *that* one tell the truth?
3. **What is `127.0.0.53`, why is it in your `resolv.conf`, and what command shows you the resolver actually behind it?**
4. **You changed an A record 20 minutes ago. Some users get the new IP, some get the old.** Explain exactly why, whether you can force the stragglers to update, and what you should have done 48 hours ago.
5. **Why can `shipfast.io` not be a CNAME while `www.shipfast.io` can?** What do DNS providers do about it?
6. **`options ndots:5` and the name `api.stripe.com`. How many DNS queries fire, and why?** Give two fixes.
7. **`EAI_AGAIN` vs `ENOTFOUND`.** What does each one prove, and how does that change which command you run next?

---

## Quick Reference Card

| Command | What it does | Key flags |
|---|---|---|
| `dig NAME` | Full DNS query with all sections | `+short`, `+noall +answer`, `+trace`, `+tcp` |
| **`dig @8.8.8.8 NAME`** | **Bypass your local resolver — "is it DNS, or is it me?"** | any `@server` |
| **`dig +trace NAME`** | **Walk the delegation from the root. No caches.** | — |
| `dig -x IP` | Reverse lookup (PTR) | `+short` |
| `dig TYPE NAME` | Query a specific type | `A AAAA MX TXT NS SOA CNAME SRV CAA` |
| `dig @ns1.provider.com NAME` | Ask the **authoritative** server — the source of truth | — |
| **`getent hosts NAME`** | **Resolve exactly like your app does (follows nsswitch + /etc/hosts)** | `ahosts`, `ahostsv4`, `ahostsv6` |
| `host NAME` | Quick, readable lookup (DNS only) | `-t TYPE`, `-a` |
| `nslookup NAME` | Legacy; present everywhere | `-type=MX`, `server 8.8.8.8` |
| `resolvectl status` | **Which resolver is used per interface** (the truth behind 127.0.0.53) | — |
| `resolvectl query NAME` | Ask systemd-resolved directly | — |
| `resolvectl flush-caches` | **Clear the local DNS cache on Ubuntu** | — |
| `tcpdump -i any -nn port 53` | Watch DNS queries live | `-c N`, `-A` |
| `/etc/hosts` | Static overrides. **Beats DNS.** | — |
| `/etc/nsswitch.conf` | The `hosts:` line decides source order | — |
| `/etc/resolv.conf` | Which nameservers, `search`, `options` | **Usually auto-generated — don't edit** |

---

## When Would I Use This at Work?

### Scenario 1: Pinning to staging without touching code
QA needs to test the new payment service before it has a real DNS name. Instead of a code change and a redeploy, you add `--add-host=payments.shipfast.io:10.0.5.22` to the container (or a line in `/etc/hosts`). Because `getaddrinfo()` checks `files` before `dns`, the app hits the new box immediately, with zero configuration change and zero risk to prod. Then you remove it — and you know to remove it, because you understand it's an override, not a config.

### Scenario 2: The 4 a.m. "is it DNS?" page
Your API is throwing `EAI_AGAIN`. You run three commands in 20 seconds: `getent hosts <name>` (hangs for 10s → it's a timeout, not an NXDOMAIN), `dig @8.8.8.8 github.com` (works → the network and DNS-at-large are fine), `dig @<your-vpc-resolver> <name>` (times out → **there** is the fault). You've localized a production outage to one component before most people have finished reading the alert.

### Scenario 3: Migrating the API to a new load balancer
You lower the TTL to 60s **two days ahead**, wait out the old 3600s TTL, switch the record, verify by comparing `dig @ns1.provider.com` (authoritative) against `dig @8.8.8.8` (the world), watch the old server's access log drop to zero, keep it alive an extra hour anyway, then shut it down and raise the TTL back. Zero dropped requests — because you knew that "propagation" is just TTL expiry and planned around the one lever you actually had.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 28 — Ports and Sockets | You have an IP now. The next question is: *is anything listening on the port?* `ss -tulpn` is the other half of every connectivity failure. |
| **Builds on** | 26 — Network Interfaces | DNS gives you an IP; the routing table (topic 26) decides which interface it leaves by. And a container's DNS lives in its **netns** — that's why `127.0.0.11` works only inside. |
| **Builds on** | 01 — What Linux Actually Is | DNS is **userspace** (glibc), not the kernel. There is no `dns()` syscall. That is *why* `/etc/hosts` can override the whole internet. |
| **Builds on** | 03 — Everything Is a File | `/etc/resolv.conf` is a **symlink** into a tmpfs. Understanding symlinks is why you know not to edit it. |
| **Builds on** | 12 — Pipes and Redirection | `dig +short \| xargs` and `tcpdump \| awk \| sort \| uniq -c` are how you turn raw DNS output into an answer. |
| **Used by** | 29 — Firewalls | UDP/53 (and TCP/53 for big answers) must be allowed outbound, or *everything* breaks in a way that looks like a hundred different bugs. |
| **Used by** | 30 — curl and wget | `curl --resolve name:443:1.2.3.4` lets you test a server **before** you cut DNS over to it. It's `/etc/hosts` for a single command. |
| **Used by** | 33 — Node in Production | `dns.lookup()` on the libuv threadpool, keep-alive agents, and DNS caching are all real production-Node concerns you now understand from the bottom up. |
