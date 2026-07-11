# 08 — TCP in Depth

> **Phase 2 — Topic 3 of 8**
> Prerequisites: `01-how-the-internet-works.md`, `05-ports.md`, `07-udp.md`

---

## ELI5 — The Simple Analogy

Imagine you and a friend want to have a long, important phone conversation, but you're using a terrible walkie-talkie system that:
- Sometimes drops words entirely
- Sometimes delivers your sentences out of order
- Sometimes echoes a word twice

You'd invent rules to survive this:

1. **"Are you there? — Yes, I'm here. — Great, I hear you too."** (You confirm the line is open *before* saying anything important. That's the **3-way handshake**.)
2. **You number every sentence.** "Sentence 1... sentence 2..." So your friend can reassemble them in order and notice if #3 never arrived. (That's **sequence numbers**.)
3. **Your friend says "got up to sentence 5"** after every batch. If you never hear that, you repeat the missing part. (That's **acknowledgments** and **retransmission**.)
4. **Your friend says "slow down, I can only write 3 sentences at a time."** (That's **flow control**.)
5. **You both notice when the line gets crackly and automatically talk slower**, then speed back up. (That's **congestion control**.)
6. **When done, you say a polite goodbye on both ends** — "I'm done." "Ok." "I'm done too." "Ok, bye." (That's the **4-way teardown**.)

TCP is *exactly* this set of rules, built on top of IP (the unreliable walkie-talkie). It turns a lossy, out-of-order, dumb packet network into a **reliable, ordered stream of bytes**. That's the entire job of TCP.

---

## Where This Lives in the Network Stack

TCP is a **Layer 4 (Transport)** protocol. It sits on top of IP (Layer 3) and carries application protocols (Layer 7) like HTTP, TLS, and SSH.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, WebSocket, SMTP, SSH, Redis)    │  ← rides inside TCP's byte stream
│  Layer 6 — Presentation  (TLS/SSL)                              │  ← runs ON TOP of TCP (Topic 09)
│  Layer 5 — Session       (connection lifetime)                  │
│  Layer 4 — Transport ══► TCP ◄══ UDP  (ports live here)         │  ← YOU ARE HERE
│  Layer 3 — Network       (IP — best-effort, unreliable)         │  ← TCP builds reliability on top of this
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                      │
│  Layer 1 — Physical      (fiber, copper, radio)                 │
└─────────────────────────────────────────────────────────────────┘
```

The key mental jump: **IP below TCP is unreliable and knows nothing about connections.** IP just fires packets and forgets them. TCP is the layer that adds handshakes, numbering, acknowledgments, retransmission, and ordering. Everything TCP does is software running in the operating system kernel of the two endpoints — routers in the middle never touch it (they only read Layer 3).

Compare with **UDP** (Topic 07): same layer, but UDP adds *nothing* — no handshake, no ordering, no retransmission. TCP is the "reliable stream" transport; UDP is the "fire and forget datagram" transport.

---

## What Is This?

**TCP (Transmission Control Protocol)** is a connection-oriented, reliable, ordered, byte-stream transport protocol defined originally in RFC 793 (1981), updated by RFC 9293 (2022).

Its guarantees, all built on top of unreliable IP:

| Guarantee | How TCP delivers it |
|-----------|---------------------|
| **Connection-oriented** | A 3-way handshake establishes shared state before any data flows |
| **Reliable** | Every byte is acknowledged; lost bytes are retransmitted |
| **Ordered** | Sequence numbers let the receiver reassemble bytes in the exact order sent |
| **Byte stream** | The app sees a continuous stream of bytes, *not* messages/packets |
| **Flow controlled** | The receiver tells the sender how much it can accept (rwnd) |
| **Congestion controlled** | The sender backs off when the *network* is congested (cwnd) |
| **Full-duplex** | Data flows independently in both directions on one connection |

The single most important and most misunderstood point: **TCP is a byte stream, not a message stream.** When you `write()` "HELLO" then "WORLD", the other side might `read()` "HELLOWORLD" in one chunk, or "HEL" then "LOWORLD". TCP preserves *byte order*, never *message boundaries*. If you need messages, you must add framing yourself (length prefixes, delimiters, etc.).

---

## Why Does It Matter for a Backend Developer?

TCP is the substrate under almost everything you build. HTTP/1.1, HTTP/2, TLS, Postgres, MySQL, Redis, Kafka, gRPC, SSH — all run on TCP. You don't write TCP code directly, but its behavior determines whether your service is fast, slow, or falls over.

You need TCP to:
- **Read connection tables** (`ss`, `netstat`) and understand what `TIME_WAIT`, `CLOSE_WAIT`, `ESTABLISHED` mean — the difference between "normal" and "you have a socket leak bug."
- **Diagnose latency**: the handshake costs 1 full round-trip *before your app sends a single byte*. On a Tokyo↔Virginia link that's ~150ms of pure overhead per new connection. This is why connection pooling and keep-alive exist.
- **Understand mysterious 40ms stalls** caused by Nagle's algorithm fighting delayed ACK.
- **Fix "too many open files" / port exhaustion** caused by thousands of sockets stuck in `TIME_WAIT` on a busy server.
- **Debug data corruption bugs** that are actually *framing* bugs — you assumed one `write` = one `read`, and TCP's byte-stream nature bit you.
- **Reason about throughput**: on high-latency links, the receive window and congestion window — not your bandwidth — cap your speed (the bandwidth-delay product).

If you can't explain the difference between `TIME_WAIT` and `CLOSE_WAIT` in an incident, you can't debug the most common production socket problem there is.

---

## The Packet/Protocol Anatomy

A TCP segment = **TCP header (20–60 bytes) + payload (your data)**. The header is where all the magic lives.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |  ← which app/socket (Topic 05)
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |  ← byte offset of this segment's data
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |  ← next byte I expect FROM you
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |Rsrvd|C|E|U|A|P|R|S|F|                                 |
| Offset|     |W|C|R|C|S|S|Y|I|            Window               |  ← flags + receive window (rwnd)
|(hdr len)    |R|E|G|K|H|T|N|N|                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |  ← error check + (rarely used) URG
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (0–40 bytes)       |    Padding    |  ← MSS, window scale, SACK, timestamps
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data ...                          |  ← your bytes (HTTP, TLS, etc.)
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Field-by-field, the parts you actually care about:**

| Field | Size | What it does |
|-------|------|--------------|
| Source / Dest Port | 16 bits each | Identifies the sending and receiving socket. `(src IP, src port, dst IP, dst port)` = the 4-tuple that uniquely names a connection |
| Sequence Number | 32 bits | The byte offset (in the stream) of the *first* data byte in this segment. On SYN, it's the ISN |
| Acknowledgment Number | 32 bits | The *next* byte the sender expects to receive. "I have everything up to ack−1." Valid only when ACK flag is set |
| Data Offset | 4 bits | Header length in 32-bit words. Min 5 (=20 bytes), max 15 (=60 bytes). Tells the receiver where options end and data begins |
| Window | 16 bits | The **receive window (rwnd)**: how many more bytes I can buffer right now. Flow control. Max 65,535 unless window scaling is on |
| Checksum | 16 bits | Covers header + data + a pseudo-header of the IP addresses. Corrupt segments are dropped |

**The flag bits (the control plane of TCP):**

| Flag | Name | Meaning |
|------|------|---------|
| `SYN` | Synchronize | Start a connection; "here is my initial sequence number" |
| `ACK` | Acknowledge | The Acknowledgment Number field is valid (set on nearly every segment after the first SYN) |
| `FIN` | Finish | "I have no more data to send." Graceful close |
| `RST` | Reset | "This connection is broken / refused — tear it down NOW." No graceful goodbye |
| `PSH` | Push | "Deliver this to the app immediately, don't wait to buffer more" |
| `URG` | Urgent | Urgent Pointer is valid (essentially obsolete; avoid) |

**Key options (in the SYN, both sides advertise capabilities):**

| Option | Purpose |
|--------|---------|
| **MSS** (Maximum Segment Size) | Largest payload I'll accept per segment. Typically **1460** on Ethernet (1500 MTU − 20 IP − 20 TCP) |
| **Window Scaling** (wscale) | Multiplies the 16-bit window by 2^N (N up to 14), allowing windows up to ~1 GB. Essential for high-speed/high-latency links |
| **SACK** (Selective ACK) | Lets the receiver say "I got bytes 1000–2000 AND 3000–4000, just resend 2000–3000" instead of only cumulative ACKs |
| **Timestamps** | Enables accurate round-trip time measurement (RTT) and protects against wrapped sequence numbers (PAWS) |

---

## How It Works — Step by Step

### Part 1 — The 3-Way Handshake (opening the connection)

Before any data flows, both sides must agree on starting sequence numbers and confirm each direction works. This takes three segments.

```
  Client (active open)                              Server (passive open, LISTEN)
       │                                                   │
   CLOSED                                              LISTEN
       │                                                   │
       │──── ① SYN  seq=x  (ISN_c = 2841750034) ─────────►│
       │       "Let's talk. My byte counter starts at x."  │
  SYN_SENT                                            SYN_RECV
       │                                                   │
       │◄─── ② SYN, ACK  seq=y (ISN_s=1170928129)          │
       │            ack=x+1  ─────────────────────────────│
       │  "OK. My counter starts at y. I got your SYN,     │
       │   next byte I expect from you is x+1."            │
       │                                                   │
       │──── ③ ACK  seq=x+1  ack=y+1 ─────────────────────►│
       │  "Got your SYN. Next byte I expect from you       │
       │   is y+1. Let's go."                              │
  ESTABLISHED                                        ESTABLISHED
       │                                                   │
       │════════ data can now flow both ways ════════════│
```

**Why three, and why sequence numbers matter:**
- **The SYN consumes one sequence number.** That's why the server ACKs `x+1` even though no data byte was sent — the SYN itself is counted.
- **ISN (Initial Sequence Number) is randomized** (RFC 6528), not started at 0. This prevents an attacker from guessing sequence numbers to inject forged segments into your connection, and stops stale segments from an old connection being mistaken for the new one.
- **Both directions get synchronized.** Client learns the server can receive (SYN-ACK proves it) and the server learns the client can receive (the final ACK proves it). After 3 messages, both sides *know that the other side knows* the channel works.

**Cost:** the handshake is **1 full round-trip (RTT)** before your app sends a single byte of data. This is the "connection setup tax."

### Part 2 — Sequence & ACK Numbers: turning IP into a reliable stream

Every byte in the stream has an implicit number. The sender labels each segment with the sequence number of its first byte; the receiver replies with a **cumulative ACK** = the next byte it expects.

```
Client sends 300 bytes in 3 segments (seq numbers relative to ISN for readability):

  seq=1,   100 bytes  ──►     ◄── ack=101   "got everything through byte 100"
  seq=101, 100 bytes  ──►     ◄── ack=201   "got everything through byte 200"
  seq=201, 100 bytes  ──►     ◄── ack=301   "got everything through byte 300"

Cumulative ACK: ack=301 means "I have ALL bytes up to 300." A single ACK
can acknowledge multiple segments at once.
```

**What happens on loss:**

```
  seq=1,   100 bytes  ──►     ◄── ack=101
  seq=101, 100 bytes  ──X (LOST in the network)
  seq=201, 100 bytes  ──►     ◄── ack=101   ← DUPLICATE ACK! "still waiting for byte 101"
  seq=301, 100 bytes  ──►     ◄── ack=101   ← duplicate #2
  seq=401, 100 bytes  ──►     ◄── ack=101   ← duplicate #3

  → 3 duplicate ACKs = FAST RETRANSMIT trigger
  seq=101, 100 bytes  ──► (resent immediately, no waiting)   ◄── ack=501
                        "now I have everything through 500"
```

Two recovery mechanisms:

1. **RTO (Retransmission Timeout)** — the fallback. When a segment is sent, a timer starts (dynamically computed from measured RTT, roughly `SRTT + 4×RTTVAR`). If no ACK arrives before it fires, the segment is resent and the timeout is **doubled** (exponential backoff). Slow but always works.
2. **Fast Retransmit** — the fast path. If the sender receives **3 duplicate ACKs** for the same byte, it assumes that one segment was lost (later segments arrived, since they generated the duplicate ACKs) and retransmits *immediately*, without waiting for the RTO timer. Much faster than waiting hundreds of milliseconds for a timeout.

**SACK** improves this further: instead of only "I'm stuck at 101," the receiver can say "I have 101 missing but I *do* have 201–500," so the sender resends only the true gap, not everything after it.

### Part 3 — Flow Control: don't overwhelm the *receiver*

The receiver has a finite buffer. It advertises how much free space it has in the **Window** field of every ACK — this is the **receive window (rwnd)**. The sender must never have more unacknowledged data "in flight" than the receiver's advertised window.

```
Receiver buffer = 8 KB, app is reading slowly:

  ◄── ack=1001, win=8000    "send me up to 8000 more bytes"
  Sender sends 8000 bytes... app hasn't read them yet...
  ◄── ack=9001, win=2000    "buffer filling up, only 2000 room left"
  Sender sends 2000 bytes... app STILL hasn't read...
  ◄── ack=11001, win=0      "STOP. Zero window. Buffer full."

  Sender pauses. Later the app reads and drains the buffer:
  ◄── ack=11001, win=8000   "window update — go again"  (unsolicited ACK)
```

**Zero window** is normal back-pressure, not an error — it means the receiving application is slower than the sender. To avoid deadlock if the "window update" ACK is lost, the sender periodically sends a 1-byte **zero-window probe** to re-check. A connection stuck with `Send-Q` full and a peer window of 0 for a long time usually means the *remote application stopped calling read()*.

### Part 4 — Congestion Control: don't overwhelm the *network*

Flow control protects the receiver. **Congestion control** protects the network between them — the shared routers and links that TCP can't see directly. The sender maintains a second, private limit: the **congestion window (cwnd)**. At any moment:

```
data in flight ≤ min(rwnd, cwnd)
                    │      │
          receiver's limit  network's limit
```

TCP can't measure congestion directly, so it *infers* it: **packet loss = the network is congested.** The classic algorithm (TCP Reno):

```
cwnd
 │                                    ╱╲  ← packet loss: halve cwnd
 │                        ╱╲        ╱    (multiplicative decrease)
 │              ╱╲      ╱    ╲    ╱
 │  slow    ╱      ╲  ╱        ╲╱
 │  start ╱  ← congestion avoidance: +1 MSS per RTT
 │      ╱    (additive increase)
 │   ╱  ← exponential growth (double cwnd each RTT)
 │╱
 └──────────────────────────────────────────────► time
   ssthresh reached → switch from slow start to avoidance
```

1. **Slow Start** — begin tiny (cwnd ≈ 10 MSS today), **double** cwnd every RTT (exponential ramp-up) until reaching a threshold `ssthresh` or hitting loss. "Slow" is a misnomer — it's an exponential *start*.
2. **Congestion Avoidance (AIMD)** — past `ssthresh`, grow linearly: **+1 MSS per RTT** (additive increase). Probe gently for more bandwidth.
3. **On loss** — **halve** cwnd (multiplicative decrease). This Additive-Increase/Multiplicative-Decrease (**AIMD**) sawtooth is what keeps the internet fair and stable.
4. **Fast Recovery** — after a fast retransmit (3 dup ACKs, a mild loss signal), don't crash all the way back to slow start; halve cwnd and continue in avoidance. A full timeout (RTO) is treated as more severe and *does* reset to slow start.

**Algorithms you'll hear named:**

| Algorithm | Idea | Where used |
|-----------|------|------------|
| **Reno / NewReno** | Classic loss-based AIMD (above) | Historical baseline |
| **CUBIC** | Loss-based but grows via a cubic function — much better on high-bandwidth, high-latency links | **Linux default** since 2.6.19 |
| **BBR** | Models actual bottleneck **B**andwidth and **R**ound-trip time instead of treating loss as the only signal; great on lossy/wireless paths | Google (YouTube), Cloudflare |

### Part 5 — The 4-Way Teardown (closing the connection)

TCP connections are full-duplex, so each direction is closed independently. It takes four segments because each side sends its own FIN and gets its own ACK.

```
  Client (active close)                              Server
       │                                                │
  ESTABLISHED                                     ESTABLISHED
       │──── ① FIN, ACK ───────────────────────────────►│
       │    "I'm done sending."                          │
  FIN_WAIT_1                                        CLOSE_WAIT
       │◄─── ② ACK ─────────────────────────────────────│
       │    "OK, I heard you're done."                   │
  FIN_WAIT_2                                             │  ← server app can still send!
       │                          (server finishes work) │
       │◄─── ③ FIN, ACK ────────────────────────────────│
       │    "Now I'm done too."                    LAST_ACK
  TIME_WAIT                                             │
       │──── ④ ACK ────────────────────────────────────►│
       │    "OK, goodbye."                            CLOSED
       │
       │  ... waits 2×MSL (Linux: 60s) ...
       │
   CLOSED
```

**Why `TIME_WAIT` and why 2×MSL:** The side that sends the *last ACK* (the active closer, usually the client) enters `TIME_WAIT` and lingers for **2 × Maximum Segment Lifetime**. Two reasons:
1. **If the final ACK is lost**, the peer resends its FIN — the lingering side must still be around to re-ACK it. Otherwise the peer gets stuck in `LAST_ACK` and eventually RSTs.
2. **Prevent stale segments** from this connection (which might still be wandering the network) from being mistaken for a *new* connection reusing the same 4-tuple. 2×MSL guarantees any old packet has died first.

RFC says MSL = 2 min, so 2×MSL = 4 min. **Linux hardcodes `TIME_WAIT` to 60 seconds** (`TCP_TIMEWAIT_LEN`).

### Part 6 — The TCP State Machine

```
                              ┌──────────┐
        passive open ────────►│  LISTEN  │◄──────── server bind()+listen()
                              └────┬─────┘
                     recv SYN /    │
                     send SYN,ACK  │                ┌───────────┐
                              ┌────▼─────┐  send SYN │           │
                              │ SYN_RECV │◄──────────│ SYN_SENT  │◄─ client connect()
                              └────┬─────┘           └─────┬─────┘
                     recv ACK      │      recv SYN,ACK /   │
                                   │      send ACK         │
                                   ▼◄──────────────────────┘
                            ┌─────────────┐
                            │ ESTABLISHED │  ← data transfer happens here
                            └──────┬──────┘
                  ┌────────────────┴─────────────────┐
     app calls    │                                  │  recv FIN /
     close() /    │                                  │  send ACK
     send FIN     ▼                                  ▼
           ┌────────────┐                     ┌────────────┐
           │ FIN_WAIT_1 │                     │ CLOSE_WAIT │ ← app must call close()!
           └─────┬──────┘                     └─────┬──────┘
                 │ recv ACK                          │ app close() / send FIN
                 ▼                                   ▼
           ┌────────────┐                     ┌────────────┐
           │ FIN_WAIT_2 │                     │  LAST_ACK  │
           └─────┬──────┘                     └─────┬──────┘
                 │ recv FIN / send ACK               │ recv ACK
                 ▼                                   ▼
           ┌────────────┐                       ┌────────┐
           │ TIME_WAIT  │──── 2×MSL timeout ───►│ CLOSED │
           └────────────┘                       └────────┘
```

The two states that matter most in production:
- **`TIME_WAIT`** — the *active closer* (usually the client). **Normal and expected.** Thousands of these on a busy client/proxy is not automatically a bug.
- **`CLOSE_WAIT`** — the peer sent FIN, TCP ACKed it, but **your application has not yet called `close()`**. A pile of `CLOSE_WAIT` is almost always an **application bug: a socket/file-descriptor leak.**

### Part 7 — Nagle's Algorithm, Delayed ACK, and the 40ms stall

Two independent latency optimizations that **interact badly**:

- **Nagle's algorithm** (sender side): don't send a small segment while there's already unacknowledged data in flight — buffer small writes and coalesce them, to avoid flooding the network with tiny 1-byte packets. Waits for an ACK before sending the next small chunk.
- **Delayed ACK** (receiver side): don't ACK immediately; wait up to ~40–200ms hoping to piggyback the ACK on a data segment (or to ACK two segments at once).

Now watch them deadlock: the sender (Nagle) is waiting for an ACK before sending its small final chunk. The receiver (delayed ACK) is waiting for more data before sending that ACK. Neither moves until the delayed-ACK timer expires — a mysterious **~40ms latency spike** on small request/response workloads.

**The fix:** set **`TCP_NODELAY`** to disable Nagle on latency-sensitive sockets (every RPC/DB/HTTP client library does this by default). Now small writes go out immediately.

---

## Exact Syntax Breakdown

Reading a TCP handshake in `tcpdump` output — the single most useful skill for TCP debugging.

```
14:22:01.100 IP 10.0.0.5.54231 > 93.184.216.34.443: Flags [S], seq 2841750034, win 64240, options [mss 1460,sackOK,TS val 9 ecr 0,nop,wscale 7], length 0
             │        │              │               │        │              │           │
             │        │              │               │        │              │           └─ options advertised
             │        │              │               │        │              └─ window (before scaling)
             │        │              │               │        └─ seq = client's ISN
             │        │              │               └─ Flags [S] = SYN
             │        │              └─ destination: server IP . port 443
             │        └─ source: client IP . ephemeral port 54231
             └─ Layer 3 protocol
```

**tcpdump flag notation** (this trips everyone up):

| Notation | Flags set | Meaning |
|----------|-----------|---------|
| `[S]` | SYN | Handshake segment 1 |
| `[S.]` | SYN, ACK | Handshake segment 2 (the `.` = ACK) |
| `[.]` | ACK only | Bare acknowledgment (segment 3, or any pure ACK) |
| `[P.]` | PSH, ACK | Data segment carrying a payload |
| `[F.]` | FIN, ACK | Graceful close |
| `[R]` / `[R.]` | RST / RST,ACK | Connection reset/refused |

**`ss` — the modern connection inspector** (replaces `netstat`; full deep dive in Topic 24):

```
ss  -t   -a   -n   -p
│   │    │    │    │
│   │    │    │    └─ -p = show the process (PID/program) owning each socket
│   │    │    └─ -n = numeric (don't resolve ports to names, e.g. 443 not "https")
│   │    └─ -a = all sockets (including LISTEN and non-ESTABLISHED)
│   └─ -t = TCP only
└─ socket statistics
```

Filter by state directly:
```bash
ss -tan state time-wait          # only TIME_WAIT sockets
ss -tan state close-wait         # only CLOSE_WAIT sockets (the bug hunt)
ss -tan state established        # active connections
```

---

## Example 1 — Basic: Watch a Real 3-Way Handshake and Teardown

Capture the handshake to a real server. Run the `tcpdump` in one terminal, the `curl` in another.

```bash
# Terminal 1 — capture just the handshake + teardown packets
sudo tcpdump -i any -n 'tcp and host example.com and port 443' -c 12

# Terminal 2 — make one HTTPS request
curl -s https://example.com > /dev/null
```

**Annotated output:**

```
# ── THE 3-WAY HANDSHAKE ──────────────────────────────────────────
14:22:01.100 IP 10.0.0.5.54231 > 93.184.216.34.443: Flags [S],  seq 2841750034,               win 64240, options [mss 1460,sackOK,TS...,wscale 7], length 0
14:22:01.119 IP 93.184.216.34.443 > 10.0.0.5.54231: Flags [S.], seq 1170928129, ack 2841750035, win 65535, options [mss 1460,sackOK,TS...,wscale 9], length 0
14:22:01.119 IP 10.0.0.5.54231 > 93.184.216.34.443: Flags [.],                  ack 1170928130, win 502,   length 0
#            ▲ 19ms between SYN and SYN-ACK = 1 RTT = the handshake tax

# ── DATA (TLS handshake + HTTP, all inside TCP) ──────────────────
14:22:01.120 IP 10.0.0.5.54231 > 93.184.216.34.443: Flags [P.], seq 1:518, ack 1, win 502, length 517   # ClientHello
14:22:01.139 IP 93.184.216.34.443 > 10.0.0.5.54231: Flags [.],  ack 518,   win 507, length 0            # ACK of our data

# ── THE 4-WAY TEARDOWN ───────────────────────────────────────────
14:22:01.205 IP 10.0.0.5.54231 > 93.184.216.34.443: Flags [F.], seq 1200, ack 4500, win 502, length 0   # our FIN
14:22:01.224 IP 93.184.216.34.443 > 10.0.0.5.54231: Flags [F.], seq 4500, ack 1201, win 507, length 0   # their FIN+ACK
14:22:01.224 IP 10.0.0.5.54231 > 93.184.216.34.443: Flags [.],            ack 4501, win 502, length 0   # final ACK → TIME_WAIT
```

**What to notice:**
- `[S]` → `[S.]` → `[.]` is the handshake. The 19ms gap between the first two is one RTT — pure latency before any real work.
- The server's SYN-ACK `ack 2841750035` = our ISN + 1 (the SYN consumed one sequence number).
- After the request, note the local socket sits in `TIME_WAIT`:

```bash
ss -tan | grep 54231
# TIME-WAIT   0   0   10.0.0.5:54231   93.184.216.34:443
```

That socket will linger ~60 seconds. Perfectly normal for the side that closed first.

---

## Example 2 — Production Scenario: "The server is dying — thousands of stuck sockets"

**The situation:** Your Node.js API behind an Nginx reverse proxy starts throwing `EADDRNOTAVAIL` and `connect: cannot assign requested address` under load. New outbound connections to your Postgres database and to an upstream API intermittently fail. The box has plenty of CPU and RAM. This is a **socket/port problem**, not a resource problem.

**Step 1 — Get the state histogram. This one command tells you almost everything:**

```bash
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
```
```
  28453 TIME-WAIT      ← the smoking gun
   1204 ESTAB
     87 LISTEN
     41 CLOSE-WAIT     ← a second, different problem
```

**Diagnosis A — 28,453 `TIME_WAIT` sockets (port exhaustion):**

Your app opens a **fresh** TCP connection for every outbound request (no pooling, no keep-alive) and closes it. Being the active closer, each one lands in `TIME_WAIT` for 60s. The ephemeral port range is only ~28,000 ports (`32768–60999` by default). You are *literally running out of source ports* to open new connections.

```bash
# confirm the ephemeral range and count
cat /proc/sys/net/ipv4/ip_local_port_range      # 32768   60999   (~28k ports)
ss -tan state time-wait | grep -c ':5432'         # how many are to Postgres:5432
```

**The fix — in priority order:**

```
1. CONNECTION POOLING / KEEP-ALIVE  ← the real fix (Topic 39)
   Reuse a small pool of long-lived connections instead of open→use→close.
   - Postgres:   use a pool (pg-pool) or PgBouncer in front of the DB
   - HTTP calls: enable keep-alive Agent ({ keepAlive: true })
   → connections stay ESTABLISHED and get reused; almost nothing enters TIME_WAIT

2. Let the SERVER not be the closer where possible
   Use HTTP keep-alive so the *client* reuses the connection.

3. tcp_tw_reuse (initiator side, safe with timestamps):
   sysctl -w net.ipv4.tcp_tw_reuse=1
   → lets the kernel safely reuse a TIME_WAIT socket for a NEW outbound
     connection. Helps outbound port exhaustion.

4. Widen the ephemeral range (a band-aid, not a fix):
   sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

> ⚠️ Do **not** reach for the old `net.ipv4.tcp_tw_recycle` — it was **removed from Linux in 4.12** because it broke clients behind NAT. And `SO_REUSEADDR` is about letting a *server* re-`bind()` its listening port immediately after restart while old connections drain — it does **not** reduce TIME_WAIT counts.

**Diagnosis B — 41 `CLOSE_WAIT` sockets (an actual application bug):**

`CLOSE_WAIT` means: the peer sent FIN, your kernel ACKed it, but **your application never called `close()`** on that socket. These do *not* time out on their own — they sit forever, leaking file descriptors until you hit `EMFILE: too many open files`.

```bash
# Which process is leaking, and to whom?
ss -tanp state close-wait
```
```
State       Recv-Q Send-Q  Local Address:Port   Peer Address:Port  Process
CLOSE-WAIT  1      0       10.0.0.5:8080        10.0.2.9:51234     users:(("node",pid=3812,fd=23))
CLOSE-WAIT  1      0       10.0.0.5:8080        10.0.2.9:51240     users:(("node",pid=3812,fd=24))
...
```

Note `Recv-Q=1`: there's unread data plus the FIN sitting in the buffer that the app never drained. The fix is **in your code, not in sysctl**: find the code path that opens a socket/response/DB handle and fails to close it on the error branch. Classic culprits: a missing `finally { conn.release() }`, an unhandled exception before `res.end()`, or an HTTP client whose response body is never consumed.

**The lesson:** `TIME_WAIT` buildup → a *design* issue (open/close churn), fix with pooling/keep-alive. `CLOSE_WAIT` buildup → a *bug* in your code (you forgot to close). They look similar in `ss` but have completely different causes and fixes.

---

## Common Mistakes

### Mistake 1: Confusing `TIME_WAIT` (normal) with `CLOSE_WAIT` (a bug)
```
TIME_WAIT   = "I closed the connection and I'm politely waiting 2×MSL."
              Appears on the ACTIVE CLOSER (usually the client/proxy).
              Thousands of them = high connection churn. Fix: pooling/keep-alive.
              These clean themselves up. Not inherently a bug.

CLOSE_WAIT  = "The peer said goodbye (FIN) and I acknowledged it,
              but MY APP NEVER CALLED close()."
              Appears when your code leaks sockets. These NEVER go away
              on their own → file-descriptor leak → 'too many open files'.
              This is ALWAYS an application bug.
```
Mixing these up sends you tuning kernel `sysctl` knobs when the real problem is a missing `close()` in your code (or vice versa).

---

### Mistake 2: Assuming TCP preserves message boundaries
```javascript
// ❌ WRONG — assumes one write = one read
socket.write('{"cmd":"ping"}');     // message 1
socket.write('{"cmd":"pong"}');     // message 2

socket.on('data', (chunk) => {
  const msg = JSON.parse(chunk);    // 💥 might receive:
  //   '{"cmd":"ping"}{"cmd":"pong"}'  (both at once), OR
  //   '{"cmd":"pi'                     (a partial message)
});
```
**TCP is a byte stream, not a message stream.** The kernel is free to coalesce your two writes into one segment, or split one write across two segments. You **must add framing yourself**: length-prefix each message (send a 4-byte length, then that many bytes), use a delimiter (newline-delimited JSON), or use a protocol that already frames (HTTP with `Content-Length`, WebSocket frames, gRPC). This bug hides in dev (small, fast, local) and explodes in production.

---

### Mistake 3: Thinking a successful handshake means the app is healthy
```bash
# The TCP connection succeeds instantly...
nc -vz db.internal 5432
# Connection to db.internal 5432 port [tcp/postgresql] succeeded!
```
A completed 3-way handshake only proves **the kernel is listening on that port**. It says *nothing* about whether the application accept()ed the socket, whether Postgres is out of connections, whether it's deadlocked, or whether it'll ever answer your query. Your load balancer's TCP health check can be green while every real request times out. **Health-check at the application layer** (an HTTP `/health` endpoint, a real `SELECT 1`), not just the TCP layer.

---

### Mistake 4: Ignoring the cost of the handshake RTT (opening a new connection per request)
```
Every brand-new TCP connection costs:
  TCP handshake:  1 RTT   (before ANY data)
  + TLS handshake: 1 RTT  (TLS 1.3) — Topic 09

On a 150ms Tokyo↔Virginia link, that's 300ms of pure setup latency
BEFORE your first byte of HTTP is sent. Do that for 1,000 requests and
you've spent 5 minutes just shaking hands.
```
**Fix:** reuse connections. HTTP keep-alive, a database connection pool, an HTTP client with a persistent agent. The handshake tax is paid *once* per pooled connection, then amortized across thousands of requests.

---

### Mistake 5: Blaming the network for a mysterious 40ms latency
```
Symptom: small request/response RPCs randomly take ~40ms instead of ~1ms.
Cause:   Nagle's algorithm (sender buffering small writes) deadlocked with
         the peer's delayed ACK (receiver waiting to piggyback the ACK).
Fix:     enable TCP_NODELAY on the socket. (Most modern client libs already do.)
```
People spend days blaming DNS, the load balancer, or GC pauses for a stall that is actually two well-meaning TCP optimizations fighting each other.

---

## Hands-On Proof

```bash
# 1. Watch a live 3-way handshake and teardown (run the curl in another shell)
sudo tcpdump -i any -n 'tcp port 443 and host example.com' -c 12
#   Look for: [S]  [S.]  [.]  ... [F.]  [F.]  [.]

# 2. See your own TIME_WAIT sockets after a request
curl -s https://example.com > /dev/null
ss -tan state time-wait | head

# 3. Get the connection-state histogram of your whole machine
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# 4. Find CLOSE_WAIT sockets and WHICH process owns them (the leak hunt)
ss -tanp state close-wait

# 5. Inspect the negotiated MSS / window scaling / SACK on live connections
ss -tie | head          # -i shows internal TCP info: rtt, cwnd, mss, retransmits

# 6. See the ephemeral port range you can exhaust
cat /proc/sys/net/ipv4/ip_local_port_range

# 7. Watch congestion-control state and retransmits on a real transfer
ss -tie dst example.com
#   Fields to spot: cwnd:<n>  rtt:<ms>/<var>  mss:<bytes>  retrans:<a/b>

# 8. Which congestion-control algorithm is your kernel using?
sysctl net.ipv4.tcp_congestion_control      # usually: cubic
sysctl net.ipv4.tcp_available_congestion_control
```

macOS notes: use `netstat -an -p tcp` for the state list (no `ss`), and `sudo tcpdump -i en0 ...` on your Wi-Fi interface.

---

## Practice Exercises

### Exercise 1 — Easy: Capture and label a handshake
```bash
# In terminal 1:
sudo tcpdump -i any -n 'tcp and host example.com and port 443' -c 6
# In terminal 2:
curl -s https://example.com > /dev/null

# Answer from YOUR output:
# 1. Which line is the SYN?  The SYN-ACK?  The final ACK? (match [S] [S.] [.])
# 2. What is the client's ISN? What ack number does the server reply with,
#    and why is it ISN+1 rather than ISN?
# 3. What is the time gap (ms) between the SYN and SYN-ACK? That IS your RTT.
# 4. What MSS and wscale did each side advertise in its options?
```

### Exercise 2 — Medium: Prove TCP has no message boundaries
```bash
# Terminal 1 — a raw TCP listener that prints exactly what each read() returns:
nc -l 9000 | xxd

# Terminal 2 — fire several tiny writes back-to-back:
printf 'AAA'; sleep 0.01; printf 'BBB'; sleep 0.01; printf 'CCC' | nc localhost 9000

# Better: write a 5-line script that does socket.write('AAA') then write('BBB')
# three times with no delay, and log each 'data' event on the receiver.

# Answer:
# 1. Did the receiver get 'AAA', 'BBB', 'CCC' as three separate reads,
#    or did some get coalesced into 'AAABBB'?
# 2. Now add a 100ms sleep between writes. Does the coalescing change? Why?
# 3. Design a framing scheme (length-prefix OR newline-delimited) that would
#    make the receiver correctly recover the 3 original messages every time.
```

### Exercise 3 — Hard (Production Simulation): Diagnose and fix TIME_WAIT exhaustion
```bash
# Scenario: a script hammers a local server, opening a NEW connection each time.

# Step 1 — start a throwaway server:
while true; do printf 'HTTP/1.1 200 OK\r\nContent-Length: 2\r\nConnection: close\r\n\r\nhi' | nc -l 8080; done &

# Step 2 — generate churn: 2000 short-lived connections (each closes → TIME_WAIT):
for i in $(seq 1 2000); do curl -s http://localhost:8080/ > /dev/null; done

# Step 3 — measure the damage:
ss -tan state time-wait | wc -l
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# Questions:
# 1. Roughly how many sockets are in TIME_WAIT? Which SIDE owns them
#    (the curl client or the nc server) and why? (Hint: 'Connection: close'
#    — who initiates the close?)
# 2. The ephemeral range is ~28k ports. At what request rate (conn/sec), given
#    a 60s TIME_WAIT, would you exhaust ports? (Compute: ports / 60s.)
# 3. Re-run Step 2 but with keep-alive:
#      curl -s http://localhost:8080/ ... with a persistent connection
#      (or ab -k -n 2000 -c 10 http://localhost:8080/).
#    How does the TIME_WAIT count change? Explain WHY in one sentence.
# 4. Name two production fixes and say which side each applies to.
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **Walk through the 3-way handshake segment by segment. Why does the server's SYN-ACK acknowledge `ISN_client + 1` and not `ISN_client`? Why are ISNs randomized instead of starting at 0?**

2. **A sender transmits 5 segments; segment #2 is lost. Describe exactly what the receiver's ACKs look like, and the two different mechanisms (one timer-based, one ACK-count-based) that trigger retransmission. Which is faster and why?**

3. **What is the difference between the receive window (rwnd) and the congestion window (cwnd)? Which one protects the receiver and which protects the network? What single expression bounds how much data can be in flight?**

4. **Explain AIMD. When cwnd grows exponentially vs linearly, and what happens to cwnd on a fast-retransmit loss vs on a full RTO timeout.**

5. **You run `ss -tan` and see 30,000 sockets in `TIME_WAIT` and 200 in `CLOSE_WAIT`. Which one is normal, which is a bug, which side of the connection owns each, and what's the fix for each?**

6. **Why does `TIME_WAIT` exist at all, and why specifically 2×MSL? Give both reasons. What does Linux actually set the duration to?**

7. **You send two `socket.write()` calls. The receiver's `read()` returns both messages glued together. Is this a TCP bug? What must you add, and name two real protocols that already solve it for you.**

---

## Quick Reference Card

**TCP connection states (the ones you'll actually see in `ss`/`netstat`):**

| State | Meaning | Whose side | Concern? |
|-------|---------|-----------|----------|
| `LISTEN` | Server waiting for connections | Server | Normal |
| `SYN_SENT` | Sent SYN, awaiting SYN-ACK | Client | Stuck here = firewall/unreachable |
| `SYN_RECV` | Got SYN, sent SYN-ACK, awaiting ACK | Server | Many = possible SYN flood |
| `ESTABLISHED` | Connection open, data flowing | Both | Normal |
| `FIN_WAIT_1/2` | We closed, awaiting peer's FIN | Active closer | Normal, transient |
| `CLOSE_WAIT` | Peer closed; **we haven't called close()** | Passive closer | **BUG — socket leak** |
| `LAST_ACK` | We closed after peer; awaiting final ACK | Passive closer | Normal, transient |
| `TIME_WAIT` | We closed first; waiting 2×MSL | Active closer | Normal; many = churn |
| `CLOSED` | No connection | — | — |

**TCP flags:**

| Flag | Meaning | tcpdump |
|------|---------|---------|
| SYN | Open connection, sync sequence numbers | `[S]` |
| SYN+ACK | Server accepts, syncs its own | `[S.]` |
| ACK | Acknowledge received bytes | `[.]` |
| FIN | Graceful "I'm done sending" | `[F.]` |
| RST | Abrupt reset / connection refused | `[R]` |
| PSH | Deliver buffered data to app now | `[P.]` |

**Numbers worth memorizing:**

```
TCP header:            20 bytes min, 60 bytes max (with options)
Typical MSS:           1460 bytes (1500 MTU − 20 IP − 20 TCP)
Window field:          16 bits → 65,535 bytes max (unless window scaling)
Handshake cost:        1 RTT before any data
Fast retransmit:       3 duplicate ACKs
TIME_WAIT (Linux):     60 seconds (TCP_TIMEWAIT_LEN)
Ephemeral port range:  32768–60999 (~28k ports) on Linux
Nagle/delayed-ACK stall: ~40ms — fix with TCP_NODELAY
Default cong. control:  CUBIC (Linux); alternatives: Reno, BBR
```

**Command cheat block:**
```bash
ss -tan                                  # all TCP sockets, numeric
ss -tan state time-wait | wc -l          # count TIME_WAIT
ss -tanp state close-wait                # find CLOSE_WAIT + owning process (leak hunt)
ss -tan | awk 'NR>1{print $1}' | sort | uniq -c | sort -rn   # state histogram
ss -tie dst HOST                         # live cwnd, rtt, mss, retransmits
sudo tcpdump -i any -n 'tcp port 443 and host HOST' -c 12    # capture handshake
sysctl net.ipv4.tcp_congestion_control   # which algorithm
sysctl -w net.ipv4.tcp_tw_reuse=1        # safely reuse TIME_WAIT for new outbound
```

---

## When Would I Use This at Work?

### Scenario 1: "Our API server hits 'too many open files' and falls over"
You run `ss -tanp state close-wait` and see hundreds of `CLOSE_WAIT` sockets all owned by your app process, all to your Redis host. That's a **socket leak** — a code path opens a Redis connection and never closes it on the error branch. No kernel tuning will help; you fix the missing `close()`/`release()` in a `finally` block. TCP knowledge told you *where* the bug is (your code, not the network) in 30 seconds.

### Scenario 2: "Postgres connections randomly fail with 'cannot assign requested address'"
`ss -tan state time-wait | wc -l` returns 27,000. You're **exhausting ephemeral ports** because the app opens and closes a fresh DB connection per request. The fix isn't a bigger box — it's **connection pooling** (PgBouncer / a client-side pool, Topic 39), which keeps a handful of connections `ESTABLISHED` and reused. TIME_WAIT plummets, the errors vanish.

### Scenario 3: "Cross-region gRPC calls have a mysterious fixed ~40ms floor"
Small requests never go faster than 40ms even though ping is 3ms. You recognize the **Nagle + delayed-ACK deadlock** and enable `TCP_NODELAY` on the channel. Latency drops to near-ping. Without TCP knowledge, this looks like an unsolvable "the network is just slow" ghost.

### Scenario 4: "The load balancer says the backend is healthy but users get timeouts"
The LB does a **TCP-level** health check — a successful handshake — but the app is deadlocked and never processes requests. You switch the health check to an **application-level** HTTP `/health` probe. TCP knowledge told you that a completed handshake proves only that the kernel is listening, not that the app is alive.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the big picture; where the handshake fits
- `05-ports.md` — ports and the 4-tuple that names a TCP connection
- `07-udp.md` — the connectionless contrast; understand TCP by what UDP *lacks*

**Study next / closely related:**
- `09-tls-ssl-in-depth.md` — **TLS runs on top of TCP.** The TLS handshake happens *after* the TCP handshake completes, adding another RTT (or reducing it with TLS 1.3 / 0-RTT)
- `24-netstat-and-ss.md` — the full tooling deep dive for reading every TCP state and connection table
- `10-http1.1-in-depth.md` — keep-alive is HTTP asking TCP to *not* tear the connection down, to amortize the handshake tax
- `39-database-connections-over-network.md` — connection pooling / PgBouncer, the production answer to TIME_WAIT churn and handshake cost
- `23-tcpdump-and-wireshark.md` — capture and decode the segments this topic describes, live on the wire

**This topic is load-bearing for all of Phase 2–6.** Nearly every protocol you'll study (HTTP/1.1, HTTP/2, TLS, gRPC, WebSockets, database wire protocols) is a passenger riding inside the reliable byte stream TCP provides.

---

*Doc saved: `/docs/networking/08-tcp-in-depth.md`*
