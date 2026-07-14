# 10 — Networking Fundamentals for System Design
## Category: HLD Fundamentals

---

## What is this?

Networking is **how one computer sends bytes to another computer** — and every single box-and-arrow in every system design diagram you will ever draw is, physically, a network connection. The arrows are not free. They cost time, they can fail, and they have rules.

Think of the network like the **postal system**. An IP address is the street address of a building. A port number is the apartment number inside that building. TCP is registered mail (the postman waits for a signature, retries if nobody's home, and delivers letters in the order you sent them). UDP is dropping a postcard in a mailbox (fast, cheap, and you'll never know if it arrived). This doc teaches you enough of that postal system to reason about the arrows.

---

## Why does it matter?

Because **the network is where your latency comes from and where your system breaks.**

- Your API is "slow" and you spend a week profiling your Node code — but the real cost was 4 sequential HTTPS connections, each paying a fresh TLS handshake, because you never enabled keep-alive.
- You build a real-time dashboard by polling `GET /updates` every second and wonder why 50,000 users melt your server. WebSockets would have used one connection each instead of 50,000 new TCP connections *per second*.
- Your service works in staging and dies in production because the load balancer closes idle connections after 60s and your connection pool doesn't know.

**In interviews:** "Why did you choose WebSockets over polling?" "Walk me through the TCP handshake." "Would you use TCP or UDP for a video call?" These are gimme points — and getting them wrong is disproportionately damaging, because it signals you've never looked below the `fetch()` call.

**At work:** You will configure keep-alive, connection pool sizes, timeouts, and TLS. You will read a flame graph and see time spent in `connect`. You need this.

---

## The core idea — explained simply

### The Postal System Analogy

You want to send a 400-page manuscript from Mumbai to a publisher in Berlin.

**You cannot mail 400 pages in one envelope.** So you split it into 40 envelopes of 10 pages each. You number them: "1 of 40", "2 of 40", etc. Each envelope has:
- The destination **building address** (IP address)
- The destination **department inside the building** (port)
- A **sequence number** so the publisher can reassemble them in order
- Your return address (source IP + port), so they can reply

The postal system doesn't care what's inside. It just routes envelopes hop by hop — Mumbai sorting office → Delhi → Frankfurt → Berlin. Different envelopes may take **different routes** and arrive **out of order**. Envelope 17 might get lost entirely.

Now, two service tiers:

**Registered mail (TCP):** Before you send anything, you phone the publisher: "Can I send you a manuscript?" — "Yes, go ahead" — "Great, sending now." (That's the **3-way handshake.**) Then for each envelope, the publisher mails back a receipt ("got #17"). If you don't get a receipt for #17 within a timeout, **you send #17 again.** The publisher holds #18 and #19 in a drawer until #17 arrives, then hands the whole run to the editor **in order**. If the postal system looks congested, you slow down your sending rate. (That's **congestion control**.)

**Postcards (UDP):** You just drop 40 postcards in the box and walk away. Fast. No phone call, no receipts, no retries, no reordering. Some arrive, some don't, and they arrive in whatever order they arrive.

### Mapping the analogy back

| Analogy | Technical concept |
|---|---|
| Building street address | **IP address** (e.g. `142.250.183.14`) |
| Department / apartment number | **Port** (e.g. `443`) |
| (address + apartment) pair on both ends | **Socket** — a connection is uniquely identified by the 4-tuple (srcIP, srcPort, dstIP, dstPort) |
| One envelope | One **packet** / segment |
| Splitting the manuscript into envelopes | **Segmentation** — data is chopped to fit the MTU (~1500 bytes on Ethernet) |
| Numbering the envelopes | **Sequence numbers** |
| The "can I send?" phone call | **TCP 3-way handshake** (SYN → SYN-ACK → ACK) |
| Mailed-back receipts | **ACKs** |
| Resending a lost envelope | **Retransmission** — this is what makes TCP *reliable* |
| Holding #18 until #17 arrives | **In-order delivery** (and the source of *head-of-line blocking*) |
| Slowing down when the post is jammed | **Congestion control** (slow start, AIMD) |
| Postcards | **UDP** |
| Sealing the envelope so only the publisher can open it | **TLS** (encryption) |

---

## Key concepts inside this topic

### 1. The layers (just enough to be useful)

The OSI model has 7 layers. In practice you only ever talk about four of them. Here's the honest version:

```
┌───────────────────────────────────────────────────────────────────────┐
│ L7  APPLICATION   HTTP, WebSocket, gRPC, DNS, SMTP                     │
│                   "What do the bytes MEAN?"                            │
│                   → You live here. Express, fetch, axios.              │
├───────────────────────────────────────────────────────────────────────┤
│ L4  TRANSPORT     TCP, UDP        (+ TLS sits between L4 and L7)       │
│                   "Which app on the machine? Reliable or not?"         │
│                   → Ports live here. Node's `net` and `dgram` modules. │
├───────────────────────────────────────────────────────────────────────┤
│ L3  NETWORK       IP, ICMP, routing                                    │
│                   "Which MACHINE, anywhere on Earth?"                  │
│                   → IP addresses live here. `ping`, `traceroute`.      │
├───────────────────────────────────────────────────────────────────────┤
│ L2/L1 LINK/PHYS   Ethernet, WiFi, MAC addresses, fiber, copper         │
│                   "Get this frame to the next hop."                    │
└───────────────────────────────────────────────────────────────────────┘
```

The only two facts you must internalize:

- **A Layer 4 load balancer** looks only at IP + port. It's fast and dumb — it can't see URLs or cookies. **A Layer 7 load balancer** parses the actual HTTP request and can route `/api/*` to one pool and `/images/*` to another. That distinction shows up in *every* system design interview. (Covered in depth in [55 — Load Balancing].)
- **Each layer wraps the one above it**, adding a header. Your 100-byte JSON body becomes ~100 + 20 (TCP hdr) + 20 (IP hdr) + 14 (Ethernet) ≈ 154 bytes on the wire. This is why "just send more small requests" is not free.

### 2. IP addresses, ports, and sockets

**IP address** — identifies a *machine* (well, a network interface).
- IPv4: `142.250.183.14` — four bytes, 32 bits, ~4.3 billion possible. We ran out.
- IPv6: `2404:6800:4009:82b::200e` — 128 bits. Effectively infinite.
- **Private ranges** (`10.x.x.x`, `172.16–31.x.x`, `192.168.x.x`) are not routable on the public internet. Your VPC uses these. `127.0.0.1` is *this machine* (loopback).

**Port** — identifies a *process* on that machine. 16 bits → **0 to 65535**.
- 0–1023: well-known. `80` HTTP, `443` HTTPS, `22` SSH, `53` DNS, `5432` Postgres, `6379` Redis, `27017` MongoDB, `9092` Kafka.
- 1024–49151: registered. Your Node app on `3000` or `8080` lives here.
- 49152–65535: **ephemeral** — the OS hands these out to *outgoing* connections.

That last point matters more than it looks. **A single machine can only make ~28,000 outbound connections to the same destination IP:port**, because it runs out of ephemeral ports. That's a real ceiling that real teams hit.

**Socket** — the endpoint of a connection. A TCP connection is uniquely identified by a **4-tuple**:

```
(source IP, source port, destination IP, destination port)

  Your laptop                             Google's server
  192.168.1.5 : 51234   ─────────────▶   142.250.183.14 : 443
       │                                        │
       └── ephemeral, chosen by your OS         └── well-known, HTTPS
```

This is why one server on port 443 can serve a million clients: the destination (IP:443) is the same for all of them, but each client contributes a **different source IP + source port**, so every 4-tuple is unique.

### 3. TCP — the reliable, ordered, connection-oriented one

**The 3-way handshake.** Before a single byte of your data moves, three packets fly:

```
  CLIENT                                            SERVER
     │                                                 │
     │  1. SYN         seq=x                           │
     │  "I want to talk. My seq starts at x."          │
     │────────────────────────────────────────────────▶│
     │                                                 │
     │  2. SYN-ACK     seq=y, ack=x+1                  │
     │  "OK. My seq starts at y. I got your x."        │
     │◀────────────────────────────────────────────────│
     │                                                 │
     │  3. ACK         ack=y+1                         │
     │  "Got it. Connection established."              │
     │────────────────────────────────────────────────▶│
     │                                                 │
     │  ═══════ NOW you may send data ═══════          │
     │  4. HTTP GET /index.html                        │
     │────────────────────────────────────────────────▶│
```

**Cost: 1 full round trip (1 RTT) before any data flows.** If the RTT to the server is 100 ms, you burned 100 ms doing *nothing useful*. This single fact is why keep-alive, connection pooling, and CDNs exist.

TCP's four guarantees, in one line each:

| Property | What it means | How TCP does it |
|---|---|---|
| **Connection-oriented** | Both sides agree to talk before data flows | The 3-way handshake |
| **Reliable** | Lost data is re-sent; you never silently lose bytes | ACKs + retransmission timers |
| **Ordered** | Bytes arrive in the order sent | Sequence numbers; receiver buffers out-of-order segments |
| **Flow + congestion controlled** | Doesn't drown the receiver or the network | Receive window (flow), slow-start + AIMD (congestion) |

**Congestion control** is worth 30 more seconds. TCP starts *slow* (a small "congestion window" — a few packets), then **doubles** it each RTT (slow start), until it detects loss. On loss, it cuts the window (roughly in half — "additive increase, multiplicative decrease"). Consequence: **a brand-new TCP connection is slow for its first few round trips.** A connection that has been running a while is fast. That is another reason to *reuse* connections.

**The dark side: head-of-line blocking.** Because TCP guarantees order, if packet #17 is lost, packets #18–#40 sit in the kernel buffer **unusable** until #17 is retransmitted and arrives. Your application literally cannot see data that has already physically arrived. That single sentence explains why HTTP/3 abandoned TCP for QUIC-over-UDP.

### 4. UDP — the fire-and-forget one

UDP is TCP with everything removed. **No handshake. No ACKs. No retransmission. No ordering. No congestion control.** An 8-byte header (vs TCP's 20+) with source port, dest port, length, checksum. That's it.

```
  CLIENT                                            SERVER
     │  datagram ────────────────────────────────────▶│   (arrived)
     │  datagram ──────────────────✗ (lost forever)   │
     │  datagram ────────────────────────────────────▶│   (arrived, out of order)
     │
     │  No connection. No confirmation. Zero RTT setup cost.
```

**Pick UDP when late data is worse than lost data.** In a video call, a 200 ms-old audio packet is *useless* — retransmitting it would only make you fall further behind. Better to drop it and play the next one. That's the whole argument.

| Use TCP when… | Use UDP when… |
|---|---|
| Every byte must arrive (HTTP, DB queries, file transfer, payments) | Losing a few packets is fine (live video/voice, game state, telemetry) |
| Order matters (a JSON body is useless with a hole in it) | You can tolerate reordering / handle it yourself |
| You can afford 1 RTT of setup | Setup cost must be zero (**DNS** — a query and a reply, why pay a handshake?) |
| The connection is long-lived | Fire-and-forget, one-shot messages (StatsD/metrics, syslog) |
| — | You want to build your *own* reliability on top (**QUIC/HTTP3, WebRTC** do exactly this) |

The modern twist: **QUIC** (the transport under HTTP/3) runs on UDP and then rebuilds reliability, ordering, and congestion control *in user space* — but per-stream, so a lost packet in stream A doesn't block stream B. It gets TCP's guarantees without TCP's head-of-line blocking.

### 5. HTTP/1.1 vs HTTP/2, and the TLS handshake

**HTTP/1.1** — text-based, one request in flight per connection.
- To load a page with 30 assets, browsers open **~6 parallel TCP connections per domain** and queue the rest. That's the entire reason "domain sharding" and "sprite sheets" were ever a thing.
- **Head-of-line blocking at the application layer:** request #2 on a connection cannot be answered until request #1 is done.

**HTTP/2** — binary, **multiplexed** over ONE TCP connection.
- Many concurrent **streams** on a single connection, each with its own ID. Responses can interleave and come back out of order.
- **HPACK header compression** — HTTP headers are hugely repetitive (same cookies, same user-agent on every request). HPACK keeps a shared table and sends 2 bytes instead of 800.
- **Server push** (largely deprecated in practice — browsers removed it).
- Catch: it still runs over **one TCP connection**, so a single lost packet stalls *all* streams — TCP-level head-of-line blocking is back. HTTP/3 (QUIC/UDP) fixes exactly this.

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Format | Text | Binary frames | Binary frames |
| Concurrency | ~6 TCP conns/domain | 1 conn, N multiplexed streams | 1 QUIC conn, N independent streams |
| Header compression | None (gzip body only) | HPACK | QPACK |
| Transport | TCP | TCP | **UDP** (QUIC) |
| HOL blocking | App-level + TCP-level | TCP-level only | **None** |
| Setup RTTs (with TLS) | 1 (TCP) + 2 (TLS1.2) | 1 + 1 (TLS1.3) | **0–1** (QUIC merges them) |

**HTTPS = HTTP + TLS.** The TLS 1.3 handshake, after TCP is already up:

```
  CLIENT                                              SERVER
     │  [TCP 3-way handshake already done: 1 RTT]        │
     │                                                    │
     │  ClientHello                                       │
     │  - TLS version, cipher suites I support            │
     │  - a random number                                 │
     │  - my key-share (guessed key exchange params)      │
     │───────────────────────────────────────────────────▶│
     │                                                    │
     │  ServerHello + Certificate + Finished              │
     │  - chosen cipher                                   │
     │  - server's key-share                              │
     │  - CERTIFICATE (proves it's really google.com,     │
     │    signed by a CA your OS trusts)                  │
     │◀───────────────────────────────────────────────────│
     │                                                    │
     │  [both sides now derive the SAME symmetric key     │
     │   via Diffie-Hellman — the key itself never        │
     │   travels over the wire]                           │
     │                                                    │
     │  Finished + encrypted application data             │
     │───────────────────────────────────────────────────▶│
```

**TLS 1.3 = 1 RTT.** TLS 1.2 needed 2. So a fresh HTTPS request to a server 100 ms away costs: **100 ms (TCP) + 100 ms (TLS 1.3) + 100 ms (request/response) = 300 ms before a single byte of HTML renders.** Reuse that connection and the next request costs 100 ms. That 3× difference is the entire business case for keep-alive.

Two extras worth naming: **session resumption** (0-RTT on reconnect, at the cost of replay risk) and **TLS termination at the load balancer** (decrypt once at the edge, speak plain HTTP inside your trusted VPC — see [57 — Reverse Proxy]).

### 6. WebSockets — the persistent two-way pipe

HTTP is **request/response**: the client asks, the server answers, done. The server **cannot** initiate. For a chat app, that's a disaster. Your options historically:

| Technique | How it works | Cost |
|---|---|---|
| **Short polling** | `GET /messages` every 2s | Wasteful. 50k users = 25k req/s of mostly-empty responses. |
| **Long polling** | Server holds the request open until it has news | Better, but still one HTTP request per message, plus reconnect churn. |
| **SSE** (Server-Sent Events) | One long-lived HTTP response, server streams events | Great, but **one-directional** (server → client only). |
| **WebSocket** | Upgrade the HTTP connection to a raw, full-duplex TCP pipe | **Bidirectional**, low overhead (~2–6 byte frame header). The right answer for chat. |

The upgrade handshake is just an HTTP request with special headers:

```
CLIENT →  GET /chat HTTP/1.1
          Host: example.com
          Upgrade: websocket
          Connection: Upgrade
          Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
          Sec-WebSocket-Version: 13

SERVER →  HTTP/1.1 101 Switching Protocols        ← the magic status code
          Upgrade: websocket
          Connection: Upgrade
          Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

          ══ from here on, it's NOT HTTP. It's a raw duplex frame stream. ══
          Either side can send, any time, until someone closes it.
```

**The system-design consequence:** WebSockets are **stateful**. A connected user is *bound to one specific server*. That breaks the beautiful stateless-app-server model:
- Your load balancer needs sticky sessions (or consistent hashing).
- If server 7 dies, its 40,000 users must all reconnect — a **thundering herd**.
- To send a message from a user on server 3 to a user on server 9, you need a **pub/sub backplane** (Redis pub/sub, Kafka) between servers.
- Each open connection costs memory (~10–50 KB), so one Node box realistically holds tens of thousands, not millions.

### 7. Keep-alive and connection reuse

We showed a fresh HTTPS request costs ~3 RTTs of pure setup. **Keep-alive means: don't close the TCP connection after the response — reuse it for the next request.**

HTTP/1.1 has keep-alive **on by default** (`Connection: keep-alive`). But — and this bites people constantly — **Node's default HTTP client historically did NOT reuse connections.** Every `http.request()` opened a fresh socket. If your service calls a downstream API 500 times/second, you were paying 500 handshakes/second and burning ephemeral ports.

```js
import http from 'node:http';
import https from 'node:https';

// A pooled agent. Without this, every outbound request pays a fresh
// TCP (+TLS) handshake — often 100-300ms of pure waste per call.
const agent = new https.Agent({
  keepAlive: true,
  keepAliveMsecs: 30_000,   // TCP keep-alive probe interval
  maxSockets: 50,           // max concurrent sockets PER HOST
  maxFreeSockets: 10,       // idle sockets to retain for reuse
  timeout: 60_000,
});

// Node 18+: undici (the fetch impl) already pools by default.
// But for the classic http/https modules you MUST pass the agent.
const res = await new Promise((resolve, reject) => {
  https.get('https://api.example.com/v1/users', { agent }, resolve).on('error', reject);
});
```

**The trap to know:** if your load balancer or upstream closes idle connections after, say, 60 s, but your pool's timeout is 120 s, your pool will hand out a socket the peer has *already* closed → sporadic `ECONNRESET`. **Always set your client's idle timeout LOWER than the server's.** This is a real, common, maddening production bug.

### 8. Bandwidth vs latency vs RTT — stop confusing them

| Term | Definition | Unit | Analogy |
|---|---|---|---|
| **Bandwidth** | How much data fits through the pipe per second | Mbps / Gbps | The **width** of the highway |
| **Latency** | Time for one bit to travel from A to B | ms | The **length** of the highway |
| **RTT** | Round-trip time — A→B→A | ms | Driving there *and back* |
| **Throughput** | Data you *actually* achieve (≤ bandwidth) | Mbps | Cars/hour you actually move, given traffic |

**Latency is bounded by physics.** Light in fiber travels ~200,000 km/s. New York → London is ~5,600 km, so the *theoretical minimum* one-way is 28 ms, RTT 56 ms. Real-world with routers and non-straight cables: **~70–90 ms.** You cannot buy your way out of this. No amount of money makes a packet outrun light. **This is why CDNs exist** — the only fix is to *move the data closer.*

**Bandwidth you can buy. Latency you cannot.** Upgrading from 100 Mbps to 1 Gbps does *nothing* for a 50 ms RTT.

And the killer consequence — the **bandwidth-delay product**:

```
Max throughput on one TCP connection ≈ TCP_window / RTT

With a 64 KB window and 100 ms RTT:
    65,536 bytes / 0.1 s = 655,360 B/s ≈ 5.2 Mbps

You have a 1 Gbps link. You are getting 5 Mbps.
The pipe isn't full — you're just waiting for ACKs.
```

That's why long-distance bulk transfer uses **parallel connections** or **window scaling**, and why "our gigabit link is slow" is almost always a latency problem, not a bandwidth problem.

---

## Visual / Diagram description

### Diagram 1: What actually happens on the wire for one HTTPS request

```
 Browser                                                      Server
 (Mumbai)                                                    (Virginia)
    │                        RTT ≈ 200ms                         │
    │                                                            │
    ├──── SYN ──────────────────────────────────────────────────▶│  ┐
    │◀─────────────────────────────────────────── SYN-ACK ───────┤  │ TCP
    ├──── ACK ──────────────────────────────────────────────────▶│  ┘ 200ms
    │                                                            │
    ├──── ClientHello (ciphers, key-share) ─────────────────────▶│  ┐
    │◀────── ServerHello + Certificate + Finished ───────────────┤  │ TLS 1.3
    ├──── Finished ─────────────────────────────────────────────▶│  ┘ 200ms
    │                                                            │
    ├──── GET /api/users  (encrypted) ──────────────────────────▶│  ┐
    │                                             [server thinks]│  │ REQ
    │                                             [DB query 20ms]│  │ 220ms
    │◀────── 200 OK  {"users":[...]}  (encrypted) ───────────────┤  ┘
    │                                                            │
    │   TOTAL: ~620ms for the FIRST request                      │
    │                                                            │
    ├──── GET /api/posts  (SAME connection, keep-alive) ────────▶│  ┐
    │◀────── 200 OK ─────────────────────────────────────────────┤  ┘ 220ms
    │                                                            │
    │   SECOND request: ~220ms.  2.8x faster — for free.         │
```

The point of this diagram: **most of your first request is not your code.** Before Express even sees the request, 400 ms is gone to handshakes. Redraw this on a whiteboard whenever someone proposes "just make another HTTP call."

### Diagram 2: Where each protocol lives in a real architecture

```
                             ┌──────────────┐
                             │   BROWSER    │
                             └──┬────────┬──┘
                  HTTPS (TCP443)│        │ WSS (TCP 443, upgraded)
                                │        │  persistent, bidirectional
                                ▼        ▼
                        ┌────────────────────────┐
                        │   LOAD BALANCER (L7)   │  ← terminates TLS here
                        │   Nginx / ALB          │  ← sticky sessions for WS
                        └───┬────────────────┬───┘
              plain HTTP/1.1│                │ plain WS
              (inside VPC,  │                │
               already      ▼                ▼
               trusted)  ┌──────┐        ┌──────┐
                         │ API  │        │  WS  │
                         │ Node │        │ Node │
                         └──┬───┘        └──┬───┘
                            │               │
              ┌─────────────┼───────────────┤
              │             │               │
   TCP:5432   ▼   TCP:6379  ▼    TCP:9092   ▼
        ┌──────────┐  ┌──────────┐   ┌──────────┐
        │ Postgres │  │  Redis   │   │  Kafka   │
        └──────────┘  └────┬─────┘   └──────────┘
                           │
                           └── pub/sub BACKPLANE: lets WS-node-1
                               push a message to a user connected
                               to WS-node-7.  (Required because
                               WebSockets are STATEFUL.)

   Meanwhile, sideways:
        DNS lookups     → UDP:53   (no handshake, one shot)
        Metrics/StatsD  → UDP:8125 (lossy is fine, never block the app)
```

Every arrow is a socket. Every arrow has a protocol, a port, a timeout, and a failure mode. When you draw an HLD diagram, this is what the arrows *are*.

---

## Real world examples

### 1. Zoom / WebRTC — UDP because late is worse than lost

Real-time voice and video use **UDP** (via WebRTC/RTP). If an audio packet from 300 ms ago is lost, retransmitting it is *actively harmful* — by the time it arrives, that moment of speech has passed, and holding the buffer to wait for it would add delay to *everything after*. So the codec simply conceals the gap (interpolates) and moves on. You hear a tiny artifact instead of a growing lag. A TCP-based video call would degrade into an ever-increasing delay under packet loss — which is exactly what early TCP-based conferencing tools felt like.

### 2. WhatsApp — long-lived connections, few servers

WhatsApp famously ran ~2 million concurrent connections **per server** (on FreeBSD/Erlang, heavily tuned) — because they held a **persistent connection** per client rather than polling. A phone that polled every 5 seconds would generate constant TCP handshakes, drain battery, and multiply server load. A single held socket costs a bit of memory and almost no CPU while idle. This is the entire architectural argument for WebSockets/persistent connections in a messaging system — and it's why [99 — Design WhatsApp] leans on it.

### 3. Google — pushing the handshake cost to zero

Google invented **SPDY** (→ HTTP/2) and **QUIC** (→ HTTP/3) for one reason: **RTTs are the enemy.** With HTTP/1.1 + TLS 1.2, a first page load paid 1 RTT (TCP) + 2 RTTs (TLS) = 3 RTTs before a byte of HTML. On a mobile network with 150 ms RTT, that's 450 ms of *nothing*. QUIC merges the transport and crypto handshakes into **1 RTT — and 0 RTT on reconnect** to a previously-seen server. They then deployed it globally, because for a search engine, hundreds of milliseconds of first-byte latency is measurable revenue.

---

## Trade-offs

| Choice | You gain | You give up |
|---|---|---|
| **TCP** | Reliability, ordering, congestion control — you write less code | 1 RTT setup, head-of-line blocking, slow-start ramp |
| **UDP** | Zero setup, no HOL blocking, minimal overhead | You must handle loss, ordering, and congestion **yourself** (or accept them) |
| **HTTP/1.1** | Universally supported, dead simple to debug (`curl -v`, plain text) | 6-connection limit, no header compression, app-level HOL blocking |
| **HTTP/2** | One connection, multiplexed, compressed headers | Harder to debug (binary), still TCP-HOL-blocked, needs TLS in practice |
| **WebSocket** | True bidirectional push, tiny per-message overhead | **Stateful servers** → sticky sessions, pub/sub backplane, painful reconnect storms |
| **SSE** | Dead simple, works over plain HTTP, auto-reconnect built in | One-directional only; ~6 connection limit on HTTP/1.1 |
| **Long polling** | Works absolutely everywhere, no infra changes | Wasteful, high latency variance, reconnect churn |
| **Keep-alive on** | Saves 2–3 RTTs per request; massive p99 win | Sockets held open (memory + FD limits); idle-timeout mismatch → `ECONNRESET` |
| **TLS termination at LB** | Decrypt once; app servers are cheap and simple | Traffic inside your VPC is plaintext (fine if the VPC is trusted; not fine for zero-trust) |

**Rule of thumb:**
- **Request/response, correctness matters** → HTTP over TCP. Turn keep-alive **on**. This is 95% of what you build.
- **Server needs to push, and the client also talks back** → WebSocket. Budget for the stateful-server tax.
- **Server pushes, client never replies** (live scores, notifications, log tail) → **SSE**. It's simpler than WebSocket and people forget it exists.
- **Late data is worthless** (voice, video, game position, metrics) → **UDP**.

---

## Common interview questions on this topic

### Q1: "Walk me through what TCP does that UDP doesn't."
**Hint:** Four things: (1) **connection setup** — 3-way handshake, SYN/SYN-ACK/ACK, costing 1 RTT; (2) **reliability** — ACKs and retransmission of lost segments; (3) **ordering** — sequence numbers, receiver buffers out-of-order data; (4) **flow + congestion control** — receive window and slow-start/AIMD so you don't drown the receiver or the network. Then land the punchline: those guarantees cost you 1 RTT of setup and introduce **head-of-line blocking** — which is precisely why HTTP/3 went back to UDP and rebuilt reliability per-stream.

### Q2: "TCP or UDP for a multiplayer game? For a payment API?"
**Hint:** Game → **UDP**. Player positions are superseded every 50 ms; a retransmitted position from 200 ms ago is garbage, and TCP would stall *all* newer positions behind it. Send state snapshots, tolerate loss. Payment → **TCP** (over TLS), and beyond that, **idempotency keys** — because even TCP can't tell you whether the server processed a request when the connection drops mid-flight. Reliability at L4 ≠ reliability at L7.

### Q3: "Why use WebSockets instead of just polling every second?"
**Hint:** Polling with 50,000 users at 1 Hz = **50,000 requests/second**, each potentially a fresh TCP+TLS handshake, and 99% of responses are `[]`. WebSockets = 50,000 *idle* sockets, ~0 CPU, and messages delivered the instant they exist (no up-to-1s delay). Then show the cost you accept: WebSocket servers are **stateful**, so you need sticky routing, a Redis/Kafka pub-sub backplane to route between server instances, and a reconnect strategy for when a node dies.

### Q4: "What does keep-alive do and why does it matter so much?"
**Hint:** It reuses one TCP connection for multiple HTTP requests. Without it, every request pays TCP handshake (1 RTT) + TLS handshake (1–2 RTTs). At 100 ms RTT that's 200–300 ms of pure overhead **per request**. With keep-alive it's ~0 after the first. Bonus point that gets you hired: mention that the client's idle timeout must be **shorter** than the server/LB's, or you'll hand out sockets the peer already closed → intermittent `ECONNRESET`.

### Q5: "A user in Sydney says our API (hosted in Virginia) is slow. Bandwidth is fine. Why?"
**Hint:** **Latency, not bandwidth.** Sydney↔Virginia RTT is ~200–230 ms — that's the speed of light in fiber plus routing, and it's unbuyable. If the client makes 5 sequential API calls, that's 1+ second of pure network. Fixes: (a) **CDN / edge PoP** to terminate TLS close to the user (kills the handshake RTTs); (b) **collapse the 5 calls into 1** (batch endpoint / GraphQL / BFF); (c) **read replica or full region in ap-southeast-2**; (d) HTTP/2 or /3 so the calls at least go in parallel over one connection. Note (b) is often the biggest single win and costs nothing in infra.

---

## Practice exercise

### Build the Three Servers

Create one Node project and write **three separate servers**, then measure them. ~30–40 minutes.

**Part 1 — A raw TCP echo server (`net`).**

```js
// tcp-server.js
import net from 'node:net';

const server = net.createServer((socket) => {
  // `socket` IS the connection — the 4-tuple made concrete.
  console.log(`connected: ${socket.remoteAddress}:${socket.remotePort}`);

  // TCP is a BYTE STREAM, not a message stream. There is no
  // guarantee that one write() on the client = one 'data' event here.
  // Framing is YOUR problem. This is the #1 thing people get wrong.
  socket.on('data', (chunk) => {
    console.log(`recv ${chunk.length} bytes`);
    socket.write(chunk); // echo it straight back
  });

  socket.on('end', () => console.log('client disconnected'));
  socket.on('error', (err) => console.error('socket error:', err.message));
});

server.listen(4000, () => console.log('TCP echo server on :4000'));
```

Run it, then connect with `nc localhost 4000` (or `telnet`). Type things. Watch the bytes echo.
**Then do this:** send two rapid messages and observe that they may arrive in a **single** `data` event. Now you understand why every real TCP protocol needs framing (a length prefix or a delimiter).

**Part 2 — An HTTP server, and prove HTTP is just text over TCP.**

```js
// http-server.js
import http from 'node:http';

const server = http.createServer((req, res) => {
  res.writeHead(200, {
    'Content-Type': 'application/json',
    'Connection': 'keep-alive',   // reuse this TCP connection
  });
  res.end(JSON.stringify({ path: req.url, method: req.method }));
});

// Node closes idle keep-alive sockets after this. Keep it BELOW your LB's
// idle timeout, or the LB hands you sockets we've already closed.
server.keepAliveTimeout = 30_000;
server.headersTimeout = 35_000;   // must exceed keepAliveTimeout

server.listen(4001, () => console.log('HTTP server on :4001'));
```

**Now the killer demo:** point your **Part 1 raw TCP server** at port 4000 and run `curl http://localhost:4000/hello`. Look at what your TCP server prints. You'll see the literal HTTP request text — `GET /hello HTTP/1.1\r\nHost: ...`. **HTTP is just a text convention on top of TCP.** Internalize that.

**Part 3 — A WebSocket server.** (`npm i ws`)

```js
// ws-server.js
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 4002 });
const clients = new Set();

wss.on('connection', (ws, req) => {
  clients.add(ws);
  console.log(`ws connected. total=${clients.size}`);

  ws.on('message', (raw) => {
    // Broadcast to everyone — the thing HTTP fundamentally cannot do,
    // because the server has no way to initiate a request/response.
    for (const c of clients) {
      if (c !== ws && c.readyState === c.OPEN) c.send(raw.toString());
    }
  });

  // A dead TCP socket can look "open" for MINUTES. Heartbeat or leak.
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });
  ws.on('close', () => { clients.delete(ws); });
});

setInterval(() => {
  for (const ws of clients) {
    if (!ws.isAlive) { ws.terminate(); clients.delete(ws); continue; }
    ws.isAlive = false;
    ws.ping();
  }
}, 30_000);

console.log('WebSocket server on :4002');
```

**Part 4 — Measure and write up.**
1. Run `curl -w "dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" -o /dev/null -s https://www.google.com` — **twice in a row**. Note how much the second run saves.
2. Write 5 sentences: which part of the timing was TCP? Which was TLS? Which was the server actually working? Where would a CDN help, and where would it not?

---

## Quick reference cheat sheet

- **IP = machine. Port = process on that machine. Socket = the 4-tuple** (srcIP, srcPort, dstIP, dstPort) that uniquely names a connection.
- **Ports:** 80 HTTP, 443 HTTPS, 22 SSH, 53 DNS, 5432 Postgres, 6379 Redis, 27017 Mongo, 9092 Kafka. Ephemeral = 49152–65535.
- **TCP = connection-oriented, reliable, ordered, congestion-controlled.** Costs **1 RTT** to set up (SYN → SYN-ACK → ACK).
- **UDP = fire and forget.** No handshake, no ACKs, no order, no congestion control. Use it when **late data is worse than lost data**.
- **TLS 1.3 = 1 extra RTT** (TLS 1.2 = 2). So a cold HTTPS request ≈ **3 RTTs before your code even runs**.
- **Keep-alive kills those RTTs.** Always pool connections. Set your client idle timeout **below** the server's, or you'll see random `ECONNRESET`.
- **HTTP/1.1:** ~6 TCP connections per domain, one request in flight each. **HTTP/2:** one connection, N multiplexed streams, HPACK headers. **HTTP/3:** QUIC over UDP, no TCP head-of-line blocking.
- **Head-of-line blocking:** TCP guarantees order, so one lost packet stalls every byte behind it — even bytes that already arrived. This is why HTTP/3 left TCP.
- **WebSocket** = HTTP request with `Upgrade: websocket` → **`101 Switching Protocols`** → raw full-duplex pipe. Bidirectional.
- **WebSockets make servers stateful.** Budget for sticky sessions, a Redis/Kafka pub-sub backplane, and reconnect storms.
- **SSE is the forgotten middle ground** — server→client streaming over plain HTTP, auto-reconnect, dead simple. Use it when the client never needs to talk back.
- **Bandwidth ≠ latency.** Bandwidth = pipe width (buyable). Latency = pipe length (physics — **not** buyable). RTT = there and back.
- **Speed of light in fiber ≈ 200,000 km/s.** NY↔London RTT ≈ 70–90 ms, minimum. The only fix is to **move the data closer** — that's what a CDN is.
- **Bandwidth-delay product:** single-connection throughput ≈ TCP window / RTT. A 64 KB window at 100 ms RTT caps you at ~5 Mbps *on a gigabit link*.
- **L4 load balancer** sees IP:port only (fast, dumb). **L7** parses HTTP (can route by URL, header, cookie).

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [09 — CAP Theorem](./09-cap-theorem.md) — the "P" is a network partition; now you know what the network actually is |
| **Next** | [11 — How the Web Works End to End](./11-how-the-web-works.md) — assembles every protocol here into the full journey of one page load |
| **Related** | [82 — Protocols: HTTP/2, HTTP/3, WebSockets, SSE, Long Polling](./82-content-negotiation-and-protocols.md) — the deep dive on choosing a real-time protocol |
| **Related** | [55 — Load Balancing](./55-load-balancing.md) — where L4 vs L7 stops being trivia and becomes a design decision |
| **Related** | [12 — Numbers Every Engineer Should Know](./12-numbers-every-engineer-should-know.md) — the latency numbers that make these RTTs concrete |
