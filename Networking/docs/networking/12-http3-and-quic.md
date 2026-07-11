# 12 — HTTP/3 and QUIC

> **Phase 2 — Topic 7 of 8**
> Prerequisites: `01-how-the-internet-works.md`, `07-udp.md`, `11-http2.md`

---

## ELI5 — The Simple Analogy

Imagine a moving company delivering your furniture in a single big truck (**one TCP connection**). To fit everything, the company splits your stuff across many boxes: kitchen, bedroom, office (**HTTP/2 streams**). The rule of the truck is strict: boxes come off the ramp **in the exact order they were loaded**. If the bedroom box at the front gets stuck in the door, *every other box behind it waits* — even though the kitchen box could have come out fine. Your kitchen delivery is blocked by a bedroom problem that has nothing to do with it. That's **head-of-line blocking**.

HTTP/3 hires a smarter crew. Instead of one strict ramp, each room gets **its own independent ramp** (**QUIC streams**). If the bedroom box jams, the crew just keeps unloading the kitchen and office from their own ramps. Only the bedroom waits — and only until *its* box is freed. One stuck box no longer freezes the whole delivery.

And there's a second trick. The old crew showed up, exchanged paperwork to prove who they were (**TCP handshake**), *then* did a separate security check (**TLS handshake**) — two trips before any furniture moved. The new crew does the paperwork and the security check **in the same first visit**, and if they've delivered to your house before, they can start unloading **the moment they pull up** (**0-RTT**).

That smarter crew is **QUIC**. The furniture is **HTTP/3**.

---

## Where This Lives in the Network Stack

HTTP/3's whole reason for existing is that it swaps out the transport layer. Here is the exact comparison — the single most important diagram in this topic:

```
        HTTP/2 (the old stack)              HTTP/3 (the new stack)
   ┌─────────────────────────────┐    ┌─────────────────────────────┐
L7 │  HTTP/2 (binary framing)    │    │  HTTP/3 (binary framing)    │  ← same *idea*, different wire
   ├─────────────────────────────┤    ├─────────────────────────────┤
L6 │  TLS 1.2 / 1.3 (separate)   │    │       (TLS folded in)       │  ← no separate TLS layer
   ├─────────────────────────────┤    ├─────────────────────────────┤
L4 │  TCP (in-order byte stream) │    │  QUIC (streams + TLS 1.3)   │  ← THE BIG SWAP
   │   └─ kernel, port 443       │    │   └─ userspace, over UDP    │
   ├─────────────────────────────┤    ├─────────────────────────────┤
   │                             │    │  UDP (connectionless)       │  ← QUIC rides on UDP
   │                             │    │   └─ kernel, port 443/udp   │
   ├─────────────────────────────┤    ├─────────────────────────────┤
L3 │  IP (routing)               │    │  IP (routing)               │  ← unchanged
   └─────────────────────────────┘    └─────────────────────────────┘
```

Read that carefully:

- **HTTP/2** = HTTP over **TLS** over **TCP** over IP. TCP lives in the OS **kernel**; TLS is a distinct layer on top.
- **HTTP/3** = HTTP over **QUIC** over **UDP** over IP. QUIC lives in **userspace** (in the library/browser/server), it *absorbs TLS 1.3 inside itself*, and it uses UDP only as a dumb envelope to get bytes across the internet.

QUIC is best thought of as a **new Layer 4 transport** that just happens to be implemented *on top of* UDP instead of *beside* TCP. UDP is there because routers and NATs already know how to forward it — QUIC did not need the internet's hardware to change.

---

## What Is This?

**QUIC** is a general-purpose, connection-oriented, encrypted transport protocol. It was originally built at Google (2012–2013, "gQUIC") and then standardized by the IETF:

| Spec | RFC | What it defines |
|------|-----|-----------------|
| QUIC transport | RFC 9000 | Connections, streams, packets, flow/congestion control |
| QUIC + TLS | RFC 9001 | How TLS 1.3 is embedded to secure QUIC |
| QUIC loss detection | RFC 9002 | Acknowledgements, retransmission, congestion control |
| **HTTP/3** | RFC 9114 | How to carry HTTP semantics over QUIC |
| QPACK | RFC 9204 | Header compression for HTTP/3 (HPACK's HOL-safe cousin) |

The key mental split:

- **QUIC** is the *transport* — the reliable, encrypted, multiplexed pipe. It is not HTTP-specific; you could run other protocols on it.
- **HTTP/3** is *one application* that uses that pipe. It keeps HTTP's request/response semantics (methods, status codes, headers — same as HTTP/1.1 and HTTP/2) but maps each request/response onto a QUIC stream.

QUIC gives you four things TCP+TLS could not give together:

1. **Per-stream reliability** — independent streams, no cross-stream head-of-line blocking.
2. **Combined transport + crypto handshake** — 1-RTT setup for new connections, **0-RTT** for resumed ones.
3. **Connection migration** — a connection survives your IP address changing (Wi-Fi → cellular).
4. **Near-total encryption** — almost the entire packet, including most transport headers, is encrypted and authenticated.

---

## Why Does It Matter for a Backend Developer?

You mostly won't hand-write QUIC packets. But you *will*:

- **Decide whether to turn HTTP/3 on** at your CDN, load balancer, or edge — and be able to justify it.
- **Explain why a mobile app feels snappier on HTTP/3**: fewer round trips to first byte, and connections that don't die when the user walks out of Wi-Fi range.
- **Debug "why is HTTP/3 not being used?"** — the answer is almost always one of: UDP blocked, no `Alt-Svc` header, or the client silently fell back to HTTP/2. You need to recognize all three.
- **Understand the head-of-line blocking story end-to-end.** In an interview, "HTTP/2 fixed HOL blocking" is a *half-truth trap*. HTTP/2 fixed it at the **application** layer but TCP reintroduced it at the **transport** layer. HTTP/3 is what actually finishes the job. Knowing exactly where the blocking lives is the difference between a shallow and a deep answer.
- **Reason about lossy networks.** On a clean fiber link HTTP/3's win is modest. On a 2% packet-loss cellular link it can be dramatic, because one lost packet no longer stalls everything.
- **Not get burned by 0-RTT.** 0-RTT data is *replayable*. If you naively let a 0-RTT request run a non-idempotent operation (charge a card, POST an order), an attacker can replay it. That's an application-design concern that lands on your desk.

---

## The Packet/Protocol Anatomy

### Encapsulation: HTTP/3 on the wire

```
┌────────────────────────────────────────────────────────────────┐
│ IP PACKET (Layer 3)                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ UDP DATAGRAM (Layer 4 envelope)  src/dst port 443        │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ QUIC PACKET  (the real transport — mostly encrypted)│  │  │
│  │  │  ┌─────────────────────────────────────────────┐   │  │  │
│  │  │  │ QUIC FRAMES (inside the encrypted payload)  │   │  │  │
│  │  │  │   STREAM frame → carries HTTP/3 bytes       │   │  │  │
│  │  │  │   ACK frame, CRYPTO frame, etc.             │   │  │  │
│  │  │  │     ┌──────────────────────────────────┐    │   │  │  │
│  │  │  │     │ HTTP/3 frame  (HEADERS / DATA)   │    │   │  │  │
│  │  │  │     │   :method: GET                   │    │   │  │  │
│  │  │  │     │   :path: /users                  │    │   │  │  │
│  │  │  │     └──────────────────────────────────┘    │   │  │  │
│  │  │  └─────────────────────────────────────────────┘   │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘

Note: unlike TCP, there is NO "QUIC header" a router can read past the
bare minimum. The connection ID and payload are encrypted/authenticated.
A middlebox sees "some UDP on port 443" and almost nothing else.
```

### QUIC packet headers: long vs short

QUIC has two header formats. The **long header** is used during the handshake (before keys are fully established). The **short header** is used for everything after — the vast majority of packets.

```
LONG HEADER  (handshake: Initial, Handshake, 0-RTT packets)
┌───┬───┬───────┬─────────────────────────────────────────────┐
│ 1 │ 1 │ TT XXXX│  Header Form=1, Fixed=1, Type, type bits    │  1 byte
├───┴───┴───────┴─────────────────────────────────────────────┤
│ Version (32 bits)  e.g. 0x00000001 = QUIC v1                 │  ← plaintext
├──────────────────────────────────────────────────────────────┤
│ Dest Conn ID Length │ Destination Connection ID (0..20 bytes)│
├──────────────────────────────────────────────────────────────┤
│ Src  Conn ID Length │ Source Connection ID (0..20 bytes)     │
├──────────────────────────────────────────────────────────────┤
│ Packet Number (encrypted)  │  Protected Payload (encrypted)  │
└──────────────────────────────────────────────────────────────┘

SHORT HEADER  (1-RTT: normal data packets — most traffic)
┌───┬───┬─┬──┬─┬────┬───────────────────────────────────────────┐
│ 0 │ 1 │S│RR│K│ PP │  Form=0, Fixed=1, Spin, Reserved, KeyPhase │  1 byte
├───┴───┴─┴──┴─┴────┴───────────────────────────────────────────┤
│ Destination Connection ID  (length known from the connection) │  ← routing key
├──────────────────────────────────────────────────────────────┤
│ Packet Number (encrypted)                                     │
├──────────────────────────────────────────────────────────────┤
│ Protected Payload — QUIC frames — (encrypted + authenticated) │
└──────────────────────────────────────────────────────────────┘
```

What matters for you:

- The **Version** and **Connection ID** are the only meaningful cleartext, and even the Connection ID is opaque (it's a token, not an address).
- The **Packet Number** is encrypted (header protection). TCP's sequence numbers are in the clear; QUIC's are not.
- Everything of substance — the frames, the stream data, the HTTP headers — lives in the **encrypted, authenticated payload**. This is why middleboxes can't fiddle with QUIC the way they mangle TCP, and it's a deliberate anti-ossification design choice.

### QUIC frames and streams

Inside the encrypted payload, a QUIC packet carries one or more **frames**. The important ones:

```
CRYPTO frame  → carries the TLS 1.3 handshake messages
STREAM frame  → carries application data for ONE specific stream
ACK frame     → acknowledges received packets (per-packet, not per-byte)
MAX_DATA / MAX_STREAM_DATA → flow control windows
NEW_CONNECTION_ID → hands the peer spare connection IDs (for migration)
PING / PADDING / CONNECTION_CLOSE → keepalive, MTU probing, shutdown
```

A **STREAM frame** carries `(stream_id, offset, length, data)`. Each stream is an independent, ordered byte sequence. Crucially, **each stream has its own delivery guarantee**:

```
QUIC connection (1 connection ID, many streams):

Stream 0 ──[A0][A1][A2]──────►  request /style.css     ← delivered independently
Stream 4 ──[B0][ ✗ ][B2]─────►  request /app.js        ← B1 lost: only THIS stream waits
Stream 8 ──[C0][C1][C2]──────►  request /logo.png      ← unaffected, keeps flowing

Compare TCP (HTTP/2): all three share ONE ordered byte stream.
Lose one segment and the kernel holds back EVERYTHING after it,
even bytes for streams that arrived perfectly fine.
```

Stream IDs encode two things in their low bits: who initiated it (client vs server) and whether it's bidirectional or unidirectional. HTTP/3 requests use client-initiated bidirectional streams.

---

## How It Works — Step by Step

### The head-of-line blocking fix (the core idea)

First understand the problem precisely, because "HTTP/2 fixed HOL blocking" is only half true.

```
HTTP/1.1:  one request per connection at a time (or pipelining, which
           reintroduces HOL blocking). Fixed by opening 6 connections.

HTTP/2:    many streams multiplexed on ONE TCP connection.
           ✔ Fixes APPLICATION-layer HOL blocking (no more waiting your
             turn behind a slow response).
           ✗ But TCP guarantees a single in-order byte stream. If ONE
             TCP segment is lost, the kernel refuses to deliver ANY later
             bytes to the app until it's retransmitted — even bytes that
             belong to completely different streams.
           → TRANSPORT-layer HOL blocking. Unfixable in TCP, because
             in-order delivery is baked into the kernel and cannot be
             turned off per-stream.

HTTP/3:    each stream is independently reliable INSIDE QUIC (userspace).
           A lost packet only stalls the stream(s) whose bytes it carried.
           Other streams are delivered to the app immediately.
           → No transport-layer HOL blocking across streams.
```

The reason TCP can't be fixed: TCP delivers *one* ordered stream of bytes. The kernel has no idea your bytes belong to 100 different HTTP requests — it just sees byte 5000 is missing, so it holds bytes 5001+ hostage. QUIC moved reliability **up into userspace**, where the library *does* know which bytes belong to which stream, so it can deliver each stream on its own schedule.

### Connection setup — 1-RTT

TCP+TLS 1.3 needs a TCP handshake (1 RTT) *and then* a TLS handshake (1 RTT) = **2 round trips** before the first byte of HTTP. QUIC folds them together:

```
NEW CONNECTION (1-RTT):

Client                                                     Server
  │  Initial packet: QUIC params + TLS ClientHello           │
  │─────────────────────────────────────────────────────────►│   RTT 1
  │  Initial + Handshake: TLS ServerHello, cert, Finished     │
  │◄─────────────────────────────────────────────────────────│
  │  Handshake Finished  + (already can send) HTTP/3 request  │
  │─────────────────────────────────────────────────────────►│
  │  HTTP/3 response                                          │
  │◄─────────────────────────────────────────────────────────│

Transport AND encryption established in ONE round trip.
Compare TCP+TLS1.3 = 2 round trips before the first request.
```

### Connection setup — 0-RTT (resumption)

If the client has connected before, the server gave it a resumption ticket. On the next visit the client can send **application data in its very first packet**, before the handshake finishes:

```
RESUMED CONNECTION (0-RTT):

Client                                                     Server
  │  Initial: ClientHello (with session ticket)              │
  │  + 0-RTT packet: HTTP/3 GET /users  ◄── data in packet 1 │
  │─────────────────────────────────────────────────────────►│
  │  ServerHello, Finished, + HTTP/3 response                │
  │◄─────────────────────────────────────────────────────────│

Zero round trips of "dead time" before the request. The GET rode
along with the handshake's first flight.
```

**The 0-RTT caveat (memorize this):** 0-RTT data has **no replay protection**. An attacker who captures a 0-RTT packet can resend it, and the server may process it again. Therefore 0-RTT must only carry **idempotent, safe** requests (typically `GET`/`HEAD`). Never let a 0-RTT request perform a state-changing, non-idempotent action (transfer money, place an order). Servers commonly reject or defer 0-RTT for non-idempotent methods for exactly this reason.

### Connection migration

A TCP connection *is* its 4-tuple: `(src IP, src port, dst IP, dst port)`. Change any of those — say your phone switches from Wi-Fi to LTE and your source IP changes — and the TCP connection is dead. You re-do the whole handshake.

QUIC identifies a connection by an opaque **Connection ID**, not the 4-tuple:

```
Phone on Wi-Fi:   src 192.168.1.20:51000  ─┐
                                            ├─► Connection ID: 0x83f2ab...
Walks outside,                              │        (same connection!)
switches to LTE:  src 10.44.7.9:52990    ─┘

Server sees packets arriving from a NEW IP but carrying the SAME
Connection ID, validates the new path, and keeps the connection alive.
No re-handshake. No re-download. The in-flight video keeps streaming.
```

This is enormous for mobile. On TCP, every network change is a stall + full reconnect. On QUIC, it's a seamless path migration.

### Advertising HTTP/3 — the Alt-Svc dance

A browser can't just guess a server speaks HTTP/3. There's a bootstrap problem: the first connection is typically HTTP/2 over TCP. The server then advertises HTTP/3 with an **`Alt-Svc`** (Alternative Services) response header:

```
Step 1: Browser connects over HTTP/2 (TCP+TLS), sends GET /
Step 2: Server responds, and includes:

        alt-svc: h3=":443"; ma=86400

        "I also speak h3 (HTTP/3) on UDP port 443. This info is
         good for ma=86400 seconds (24h)."

Step 3: Browser caches that. On the NEXT request (and future visits
        within 24h) it races/attempts HTTP/3 over QUIC/UDP directly.
Step 4: If UDP works → HTTP/3. If UDP is blocked/times out → it stays
        on HTTP/2. This graceful fallback is why you rarely "break"
        anything by turning h3 on.
```

Newer HTTP/3-capable clients can also skip the bootstrap by reading an **HTTPS/SVCB DNS record** that advertises `h3` directly, saving the first TCP round-trip entirely.

---

## Exact Syntax Breakdown

### curl --http3 -v

```
curl  --http3   -v      https://cloudflare-quic.com/
│     │          │        │
│     │          │        └── URL (HTTP/3 always implies HTTPS/TLS)
│     │          └── verbose: show connection + protocol negotiation
│     └── force HTTP/3 over QUIC (fails if the build/site can't do it)
└── the transfer tool
```

Two related flags — know the difference:

| Flag | Behavior |
|------|----------|
| `--http3` | Try HTTP/3; on some curl builds this attempts h3 and can fall back to h2. |
| `--http3-only` | Use HTTP/3 or **fail** — no fallback. Best for *proving* h3 actually worked. |

**Prerequisite:** curl must be *built* with an HTTP/3 backend (`ngtcp2+nghttp3`, `quiche`, or `msh3`). Check with:

```
curl --version
# Look for "HTTP3" in the Features line:
# Features: alt-svc AsynchDNS HSTS HTTP2 HTTP3 HTTPS-proxy IDN IPv6 ...
#                                          ^^^^^  ← must be present
```

If `HTTP3` is not listed, `curl --http3` returns: `curl: option --http3: the installed libcurl version doesn't support this`. On macOS the stock `/usr/bin/curl` usually lacks it; install one that has it (e.g. `brew install curl` with an HTTP/3-enabled formula) or use a container image.

### Checking for the Alt-Svc header

```
curl -sI https://www.cloudflare.com/ | grep -i alt-svc
│    ││   │                            │
│    ││   │                            └── filter for the alt-svc line (case-insensitive)
│    ││   └── the URL (over h1/h2 first — that's how alt-svc is discovered)
│    │└── -I = HEAD request (headers only)
│    └── -s = silent (no progress meter)
└── curl
```

Expected:
```
alt-svc: h3=":443"; ma=86400
```

No `alt-svc: h3=...` line → the origin isn't advertising HTTP/3, so browsers will never try it.

---

## Example 1 — Basic

**Goal:** connect to a known HTTP/3 endpoint and prove you're on h3.

```bash
# 1. Confirm your curl even supports HTTP/3
curl --version | grep -o HTTP3
# → prints "HTTP3" if supported. Prints nothing if not.

# 2. Force HTTP/3 with no fallback, verbose
curl --http3-only -v https://cloudflare-quic.com/ 2>&1 | head -40
```

Annotated output (the lines that matter):

```
*   Trying 104.18.26.128:443...
* Connected to cloudflare-quic.com (104.18.26.128) port 443
* using HTTP/3                              ← ✔ QUIC/HTTP3 in use (NOT "HTTP/2")
* [HTTP/3] [0] OPENED stream for https://cloudflare-quic.com/
* h3 [:method: GET]                         ← HTTP/3 pseudo-headers (like HTTP/2)
* h3 [:path: /]
* h3 [:scheme: https]
* h3 [:authority: cloudflare-quic.com]
> GET / HTTP/3                              ← request line says HTTP/3
> Host: cloudflare-quic.com
>
< HTTP/3 200                               ← ✔ response is HTTP/3
< server: cloudflare
< alt-svc: h3=":443"; ma=86400
```

The two dead giveaways: `using HTTP/3` during setup, and `HTTP/3 200` on the response. Compare to Topic 01's `curl -v` where you saw `HTTP/2 200` — same request semantics, entirely different transport underneath.

Now prove the fallback path exists:

```bash
# Discover h3 the way a browser does — via alt-svc — then let curl reuse it.
# --alt-svc uses a cache file to remember the h3 advert between runs.
curl -sI --alt-svc altsvc-cache.txt https://www.cloudflare.com/ | grep -i alt-svc
# → alt-svc: h3=":443"; ma=86400

# Second call: curl reads the cache and upgrades to h3 on its own.
curl -s --alt-svc altsvc-cache.txt -o /dev/null -w "%{http_version}\n" https://www.cloudflare.com/
# → 3       (means HTTP/3; "2" would mean it stayed on HTTP/2)
```

---

## Example 2 — Production Scenario

**The situation:** You run a mobile-heavy app (food delivery). Backend API is behind Cloudflare. Users on flaky cellular complain the app "freezes for a second" while scrolling and that switching from Wi-Fi to cellular in the elevator drops their live order-tracking connection. Product asks: *"Should we turn on HTTP/3, and how do we prove it helped?"*

**Step 1 — Check what protocol you're serving today.**

```bash
curl -s -o /dev/null -w "protocol: HTTP/%{http_version}\n" https://api.yourapp.com/health
# → protocol: HTTP/2      ← currently HTTP/2 over TCP
```

**Step 2 — Enable HTTP/3 at the edge.** On Cloudflare this is a toggle (Network → HTTP/3). On other stacks it's a server-side config. For nginx (1.25+, `http_v3_module` built in):

```nginx
server {
    # QUIC/HTTP3 listens on UDP 443; keep TCP 443 for h1/h2 fallback
    listen 443 quic reuseport;      # HTTP/3 over QUIC (UDP)
    listen 443 ssl;                 # HTTP/1.1 + HTTP/2 over TCP (fallback)

    http2 on;
    ssl_certificate     /etc/ssl/api.crt;
    ssl_certificate_key /etc/ssl/api.key;
    ssl_protocols TLSv1.3;          # QUIC REQUIRES TLS 1.3 — no 1.2

    # Tell clients "I also speak h3" so browsers upgrade next request
    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

**Do not forget the firewall.** HTTP/3 is UDP. If your security group / firewall only opens TCP 443, HTTP/3 silently fails and every client falls back to h2 — you'll think it's "on" but nobody uses it.

```bash
# AWS security group: you MUST allow inbound UDP 443, not just TCP 443
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol udp --port 443 --cidr 0.0.0.0/0
```

**Step 3 — Prove HTTP/3 is actually being used** (not silently falling back):

```bash
# From a client that supports h3:
curl --http3-only -s -o /dev/null -w "%{http_version} via %{scheme}\n" \
  https://api.yourapp.com/health
# → 3 via https      ← success. If this HANGS or errors, UDP 443 is blocked.

# Confirm the advert is present for browsers:
curl -sI https://api.yourapp.com/health | grep -i alt-svc
# → alt-svc: h3=":443"; ma=86400
```

**Step 4 — Measure the win on a lossy link.** The point of HTTP/3 here isn't raw speed on a clean network; it's resilience to loss and network changes. Simulate 2% packet loss and compare connection-setup + multi-asset load:

```bash
# macOS: simulate loss with the packet filter (or use Linux `tc netem`)
# Linux: sudo tc qdisc add dev eth0 root netem loss 2%

# Time a fresh connection over h2 vs h3 under loss:
curl --http2 -s -o /dev/null -w "h2 total: %{time_total}s\n" https://api.yourapp.com/feed
curl --http3-only -s -o /dev/null -w "h3 total: %{time_total}s\n" https://api.yourapp.com/feed
```

**Typical result under loss + many parallel assets:**

```
h2 total: 1.84s     ← one lost TCP segment stalled ALL multiplexed streams
h3 total: 0.91s     ← lost QUIC packet only stalled its own stream
```

**Step 5 — The connection-migration payoff.** This one you observe in the app, not curl: on HTTP/3 the live order-tracking connection **survives** the Wi-Fi→cellular handoff (same Connection ID, path migration), so the "elevator freeze" disappears. On HTTP/2 that same handoff killed the TCP connection and forced a full reconnect + re-auth.

**The measured decision:** turn h3 on at the edge (fallback protects you), open UDP 443, verify with `--http3-only`, and expect the biggest gains on **mobile / lossy / roaming** traffic — not on your fast office Wi-Fi.

### Diagnosing "why isn't HTTP/3 being used?"

A decision tree you'll actually use:

```
Browser/curl won't use HTTP/3?
│
├─ Is there an `alt-svc: h3=...` response header?
│     NO → origin/CDN isn't advertising h3. Add Alt-Svc / enable h3. (root cause)
│     YES ↓
│
├─ Does `curl --http3-only URL` succeed?
│     NO / HANGS → UDP 443 is blocked or throttled (firewall, security
│                  group, corporate network, or an ISP that drops UDP).
│                  Fix the firewall to allow inbound UDP 443.
│     YES ↓
│
├─ Does curl even have HTTP/3 support? (`curl --version | grep HTTP3`)
│     NO → your client can't do h3; that's a client build issue, not server.
│     YES ↓
│
└─ It's working — the client just chose h2 for the FIRST request
      (alt-svc is discovered on request 1, used from request 2 onward).
      This is expected, not a bug.
```

---

## Common Mistakes

### Mistake 1: Thinking HTTP/3 is "HTTP/2 with some tweaks"

```
WRONG: "HTTP/3 is HTTP/2 plus a few optimizations."
RIGHT: HTTP/3 keeps HTTP *semantics* (methods, status codes, headers)
       but runs on a COMPLETELY DIFFERENT TRANSPORT.
       HTTP/2 = HTTP over TLS over TCP.
       HTTP/3 = HTTP over QUIC over UDP.
       It's a transport swap, not a feature bump. Everything about
       reliability, ordering, handshakes, and multiplexing is different.
```

The tell in an interview: if you say "HTTP/3 improved HPACK and server push," you've missed the entire point. The headline is *the move off TCP*.

### Mistake 2: Assuming UDP is always available

```
WRONG: "It's the internet, of course UDP 443 works."
RIGHT: Plenty of corporate networks, hotel Wi-Fi, and some ISPs
       block or aggressively rate-limit UDP (especially non-DNS UDP).
       QUIC rides UDP. If UDP 443 is dropped, HTTP/3 cannot establish
       and the client falls back to HTTP/2.
```

Consequence: you can enable h3 perfectly on the server and *still* see near-zero h3 traffic if your users are behind UDP-hostile networks. Always open **UDP** 443 on your own firewalls/security groups, and never assume the client side is unblocked.

### Mistake 3: Forgetting the h2/h3 fallback path exists (and testing wrong)

```
WRONG: `curl --http3 URL` returned a 200, so "h3 is confirmed working."
       (On some builds --http3 silently falls back to h2 on failure —
        you may have just tested h2 and not noticed.)
RIGHT: Use `--http3-only` to PROVE h3, because it refuses to fall back.
       And check `%{http_version}` == 3, not just a 200 status.
```

Corollary: the fallback is also a *feature*. It's why turning h3 on rarely breaks anything — but it's exactly what hides a broken h3 setup. "It still works" is not "h3 is being used."

### Mistake 4: Confusing QUIC with plain UDP

```
WRONG: "QUIC is just UDP, so it's unreliable / unordered / insecure."
RIGHT: QUIC is a RELIABLE, ORDERED (per stream), ENCRYPTED, connection-
       oriented transport. It uses UDP only as a delivery envelope.
       QUIC itself adds:
         - acknowledgements + retransmission (reliability)
         - per-stream in-order delivery
         - flow control + congestion control
         - TLS 1.3 encryption + authentication, built in
       "QUIC = UDP" is like saying "TCP = IP." UDP is the layer below.
```

### Mistake 5: Letting 0-RTT run non-idempotent requests

```
WRONG: Enable 0-RTT and let it carry any request, including
       POST /orders or POST /charge.
RIGHT: 0-RTT data is REPLAYABLE — an attacker can capture and resend it.
       Only allow 0-RTT for safe, idempotent requests (GET/HEAD).
       Reject or defer non-idempotent methods until the handshake
       completes (1-RTT). Most servers do this by default; don't undo it.
```

### Mistake 6: Enabling h3 but only opening TCP 443 on the firewall

```
WRONG: Security group allows TCP 443. h3 "enabled" on the server.
       Result: h3 handshake packets (UDP 443) are dropped at the edge.
       Every client falls back to h2. You see 0% h3 and blame the CDN.
RIGHT: HTTP/3 needs inbound UDP 443 open in EVERY firewall/security
       group/NACL in the path. TCP 443 stays open for the h1/h2 fallback.
```

---

## Hands-On Proof

```bash
# 1. Does your curl support HTTP/3 at all?
curl --version | grep -o HTTP3            # prints "HTTP3" or nothing

# 2. Force HTTP/3 and confirm the negotiated protocol
curl --http3-only -s -o /dev/null -w "negotiated: HTTP/%{http_version}\n" \
  https://cloudflare-quic.com/            # → negotiated: HTTP/3

# 3. See the QUIC/h3 setup in verbose mode
curl --http3-only -v https://cloudflare-quic.com/ 2>&1 | grep -Ei "http/3|quic|alt-svc"

# 4. Find who advertises h3 (the Alt-Svc header)
curl -sI https://www.google.com/     | grep -i alt-svc
curl -sI https://www.cloudflare.com/ | grep -i alt-svc
curl -sI https://www.facebook.com/   | grep -i alt-svc

# 5. Compare protocols on the same site: h2 vs h3
curl --http2      -s -o /dev/null -w "forced h2 → HTTP/%{http_version}\n" https://cloudflare-quic.com/
curl --http3-only -s -o /dev/null -w "forced h3 → HTTP/%{http_version}\n" https://cloudflare-quic.com/

# 6. Watch QUIC on the wire — it's UDP on port 443, not TCP
#    (run this in one terminal, then curl --http3 in another)
sudo tcpdump -n -i any 'udp port 443'     # you'll see UDP 443 = QUIC traffic
#    vs classic HTTPS which would be:
sudo tcpdump -n -i any 'tcp port 443'     # HTTP/1.1 and HTTP/2 live here

# 7. Prove the DNS-based discovery path (HTTPS/SVCB record advertising h3)
dig +short HTTPS cloudflare.com           # look for "alpn=... h3 ..." in the record
```

If you don't have an h3-capable curl handy, the quickest browser proof: open a site's page, DevTools → Network → right-click the columns → enable **Protocol**. Look for `h3` in that column. Chrome's `chrome://net-internals/#quic` also lists live QUIC sessions.

---

## Practice Exercises

### Exercise 1 — Easy: Spot HTTP/3 in the wild

```bash
# Check three big sites for HTTP/3 advertisement and usage.
for host in www.google.com www.cloudflare.com www.facebook.com; do
  echo "== $host =="
  curl -sI "https://$host/" | grep -i alt-svc
  curl --http3-only -s -o /dev/null -w "  used: HTTP/%{http_version}\n" "https://$host/" 2>/dev/null \
    || echo "  h3 attempt failed (no h3 support in curl, or UDP blocked)"
done

# Answer:
# 1. Which sites advertise h3 in alt-svc? What is the `ma=` value (cache lifetime)?
# 2. For which sites did the forced-h3 request return http_version = 3?
# 3. If alt-svc says h3 but --http3-only fails, what are the TWO likely causes?
```

### Exercise 2 — Medium: h2 vs h3 head-to-head

```bash
# Same endpoint, forced to each protocol. Run each 5 times.
URL="https://cloudflare-quic.com/"
echo "--- HTTP/2 ---"
for i in 1 2 3 4 5; do
  curl --http2 -s -o /dev/null -w "h2 run $i  TCP+TLS: %{time_appconnect}s  total: %{time_total}s\n" "$URL"
done
echo "--- HTTP/3 ---"
for i in 1 2 3 4 5; do
  curl --http3-only -s -o /dev/null -w "h3 run $i  handshake: %{time_appconnect}s  total: %{time_total}s\n" "$URL"
done

# Answer:
# 1. Is h3's connection-setup time (time_appconnect) lower than h2's? Why would it be?
#    (Hint: TCP handshake + TLS handshake vs one combined QUIC handshake.)
# 2. On a fast, clean network, is the TOTAL time dramatically different? Why or why not?
# 3. Where would you EXPECT h3 to pull ahead the most — and how would you simulate it?
```

### Exercise 3 — Hard (Production Simulation): Diagnose "HTTP/3 isn't being used"

```
Scenario: You enabled HTTP/3 on your origin (nginx) and at your CDN a week
ago. Analytics show <1% of requests are HTTP/3. Everyone is on HTTP/2.
Walk the full diagnosis and state the fix at each branch.

Step 1: Is the origin advertising h3?
  curl -sI https://api.yourapp.com/ | grep -i alt-svc
  → If EMPTY: what's misconfigured, and what's the fix?

Step 2: Is h3 actually reachable (UDP path)?
  curl --http3-only -v https://api.yourapp.com/ 2>&1 | grep -i "http/3\|refused\|timed out"
  → If it HANGS/times out: which layer is dropping packets, and which
    exact firewall/security-group rule is missing? (protocol + port + direction)

Step 3: Is the CDN terminating h3 but talking h2 to the origin?
  → Does that even matter for the CLIENT's experience? Explain.

Step 4: Client-side reality check.
  → Your users are on a corporate VPN that blocks non-DNS UDP. Even with
    everything server-side perfect, what protocol will they use, and why
    is that CORRECT (not a bug)?

Deliverable: a 4-line root-cause summary naming the single most likely
culprit for "<1% h3" and the one change that fixes it.
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **Draw the two stacks:** what sits under HTTP/2 vs under HTTP/3, layer by layer, down to IP? Which layer is the actual swap, and which protocol does QUIC ride on top of?

2. **HTTP/2 "fixed head-of-line blocking." Explain why that's only half true,** and identify exactly *which layer* still had HOL blocking under HTTP/2 and *why it was unfixable in TCP*.

3. **Why can QUIC deliver one stream while another stream is still waiting for a lost packet, but TCP cannot?** (Point to *where* reliability/ordering is implemented in each.)

4. **A new HTTP/3 connection costs 1 RTT before the first request; TCP+TLS 1.3 costs 2. Where did the extra round trip go?** Then: what is 0-RTT, and what's the one security rule you must follow with it?

5. **Your phone switches from Wi-Fi to cellular mid-download. Why does the TCP connection die but the QUIC connection survive?** Name the field QUIC uses to identify the connection instead of the 4-tuple.

6. **You enabled HTTP/3 but see almost no h3 traffic. List the top three root causes** and the one command that distinguishes "UDP is blocked" from "origin isn't advertising h3."

7. **Why does a middlebox (or your ISP) find QUIC harder to inspect and tamper with than TCP?** (What is encrypted in a QUIC packet that is *not* encrypted in a TCP segment?)

---

## Quick Reference Card

| Concept | HTTP/2 | HTTP/3 |
|---------|--------|--------|
| Transport | TCP | QUIC (over UDP) |
| Where transport lives | Kernel | Userspace (library) |
| Encryption | TLS as a separate layer | TLS 1.3 built into QUIC |
| Port | 443/tcp | 443/udp |
| New-connection setup | 2 RTT (TCP + TLS) | 1 RTT (combined) |
| Resumed setup | 1 RTT | 0 RTT (idempotent only) |
| HOL blocking (across streams) | Yes (transport-level, in TCP) | No (per-stream reliability) |
| Survives IP change | No (4-tuple bound) | Yes (Connection ID) |
| Header compression | HPACK | QPACK |
| Discovered via | — | `Alt-Svc: h3` header / HTTPS DNS record |
| Fallback | — | Falls back to h2 if UDP blocked |

**Where HTTP/3 stands today (measured):**
```
Browsers:  Chrome, Edge, Firefox, Safari — all support HTTP/3.
Big infra: Google, Cloudflare, Meta/Facebook, Fastly, Akamai serve h3.
CDNs/LBs:  Cloudflare (toggle), AWS CloudFront, Google Cloud LB — h3 available.
Servers:   nginx 1.25+ (built-in), Caddy (default on), LiteSpeed, H2O.
Clients:   curl needs an HTTP/3-enabled build (ngtcp2 / quiche / msh3).
Adoption:  a large and growing share of web traffic (~a third of sites,
           heavily concentrated behind the big CDNs). Still often opt-in
           at the origin; universal in the browser.
```

**Downsides / caveats:**
```
- UDP sometimes blocked or throttled (corporate nets, some ISPs) → falls back to h2.
- More CPU per packet: crypto + reliability run in userspace, not offloaded to the NIC/kernel.
- Middlebox / firewall visibility is reduced (mostly-encrypted headers).
- Newer/less-battle-tested tooling for deep debugging than TCP.
- 0-RTT is replayable → restrict to idempotent requests.
```

**Key commands:**
```bash
curl --version | grep HTTP3                         # is h3 supported in this build?
curl --http3-only -v URL                            # PROVE h3 (no fallback)
curl -s -o /dev/null -w "%{http_version}\n" URL     # 2 = HTTP/2, 3 = HTTP/3
curl -sI URL | grep -i alt-svc                      # is h3 advertised?
sudo tcpdump -n 'udp port 443'                      # watch QUIC on the wire
dig +short HTTPS example.com                         # h3 advertised in DNS?
```

**One-line mnemonics:**
```
HTTP/3 = HTTP over QUIC over UDP over IP.
QUIC   = reliable + encrypted + multiplexed transport that happens to ride UDP.
TCP HOL fix lives in USERSPACE, per stream — that's the whole point.
No alt-svc → no h3. UDP blocked → no h3. Always verify with --http3-only.
```

---

## When Would I Use This at Work?

### Scenario 1: "Should we enable HTTP/3 on our CDN?"
Almost always yes — the fallback to h2 means low risk — but set expectations correctly. The win is largest for **mobile, lossy, and roaming** users (per-stream reliability + connection migration), and smallest for users on fast, clean, wired networks. Enable it at the edge, open **UDP 443**, and verify real usage with `--http3-only` rather than assuming the toggle worked.

### Scenario 2: "Mobile users lose their live connection in elevators/subways"
This is the connection-migration pitch. On TCP, every Wi-Fi↔cellular handoff kills the connection (the 4-tuple changed) and forces a reconnect + re-auth. On QUIC/HTTP/3 the Connection ID keeps the session alive across the IP change. For long-lived mobile connections (live tracking, chat, streaming), that's a concrete UX fix, not a micro-optimization.

### Scenario 3: "HTTP/3 is enabled but analytics show almost no h3 traffic"
Run the decision tree: check for the `Alt-Svc: h3` header (no advert → no h3), then `curl --http3-only` (hang/timeout → UDP 443 blocked in a firewall/security group), then remember that clients use h2 on the *first* request and only upgrade afterward. Nine times out of ten it's a missing **inbound UDP 443** rule or a missing `Alt-Svc` header.

### Scenario 4: "Interview: how does HTTP/3 fix head-of-line blocking?"
Answer at full depth: HTTP/1.1 had application-layer HOL blocking; HTTP/2 fixed *that* by multiplexing streams over one TCP connection, but TCP's single in-order byte stream reintroduced HOL blocking at the **transport** layer — one lost segment stalls every stream, and it's unfixable in TCP because in-order delivery is enforced in the kernel with no per-stream awareness. HTTP/3 moves reliability into userspace QUIC, where each stream is independently reliable, so a lost packet stalls only its own stream. That single paragraph signals real understanding.

---

## Connected Topics

**Study before this:**
- `07-udp.md` — QUIC rides on UDP; understand connectionless delivery and why UDP was chosen as the envelope.
- `11-http2.md` — the multiplexing, binary framing, and HPACK that HTTP/3 keeps (as QPACK) — and the TCP HOL blocking that HTTP/3 exists to fix.
- `09-tls-ssl-in-depth.md` — QUIC embeds TLS 1.3; the handshake you learned there is folded into QUIC's setup.
- `08-tcp-in-depth.md` — to appreciate what QUIC replaces: the 3-way handshake, in-order delivery, and congestion control that QUIC reimplements in userspace.

**Study next:**
- `13-websockets.md` — the other way to escape request/response; contrast full-duplex framing with QUIC streams.
- `30-cdns.md` — where HTTP/3 is actually enabled and terminated in practice; the edge is where you flip h3 on and measure it.
- `37-grpc-and-protocol-buffers.md` — gRPC rides HTTP/2 today; understanding h3's transport helps you reason about where gRPC-over-h3 is heading.

**This topic completes the HTTP transport evolution:** HTTP/1.1 (text, one-at-a-time) → HTTP/2 (multiplexed over TCP) → HTTP/3 (multiplexed over QUIC/UDP, no transport HOL blocking, migratable connections).

---

*Doc saved: `/docs/networking/12-http3-and-quic.md`*
