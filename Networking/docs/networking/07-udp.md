# 07 — UDP

> **Phase 2 — Topic 2 of 8**
> Prerequisites: `01-how-the-internet-works.md`, `03-tcp-ip-model.md`, `05-ports.md`

---

## ELI5 — The Simple Analogy

Imagine two ways to send a message to a friend.

**Way 1 — a phone call (this is TCP):**
1. You dial. It rings. Your friend picks up and says "Hello?" You say "Hi, it's me." (**the handshake**).
2. Now you're connected. As you talk, your friend nods and says "mm-hm, go on" so you know they heard each word (**acknowledgments**).
3. If they miss a sentence, they say "sorry, what?" and you repeat it (**retransmission**).
4. When you're done, you both say goodbye and hang up (**teardown**).

**Way 2 — shouting a message across a crowded room (this is UDP):**
1. You just yell it. No dialing, no "hello", no waiting.
2. You don't know if they heard you. You don't wait for a nod.
3. If a truck drives by and drowns out a word, that word is gone forever. You never find out.
4. There's no goodbye. You just stop shouting.

Shouting sounds worse — until you realize it's **instant** and **cheap**. If you're a scoreboard announcing "the score is now 5-3" ten times a second, who cares if one shout gets lost? The next one is coming in 100 milliseconds anyway, and it carries the newest score. Waiting for a nod after every shout would only slow you down.

That's UDP. **Fire the message and forget about it.** No call setup, no confirmations, no memory of who you talked to. Fast, tiny, and it does not care whether the message arrives.

---

## Where This Lives in the Network Stack

UDP is a **Layer 4 (Transport)** protocol, sitting in the exact same slot as TCP. It rides on top of IP (Layer 3) and carries application data (Layer 7) inside it.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (DNS, DHCP, NTP, VoIP, QUIC, statsd)   │  ← your app hands data down
│  Layer 4 — Transport     (TCP  ← reliable │  UDP ← YOU ARE HERE) │  ← ports live here
│  Layer 3 — Network       (IP — routing, addresses)              │  ← carries the datagram
│  Layer 2 — Data Link     (Ethernet, Wi-Fi — MAC addresses)      │
│  Layer 1 — Physical      (cables, fiber, radio waves)           │
└─────────────────────────────────────────────────────────────────┘
```

TCP and UDP are **siblings**, not a stack. When an IP packet arrives, the IP header has a `Protocol` field: value **6** means "the payload is TCP", value **17** means "the payload is UDP". The OS reads that number and hands the payload to the right code path.

```
                 IP packet arrives
                        │
              read IP header "Protocol" field
                        │
          ┌─────────────┴─────────────┐
     Protocol = 6                Protocol = 17
          │                            │
        TCP                          UDP
   (reliable stream)          (fire-and-forget datagram)
```

Everything above Layer 4 that you thought was "just the internet" — DNS lookups, NTP time sync, video calls, HTTP/3 — is UDP underneath.

---

## What Is This?

**UDP (User Datagram Protocol)** is the internet's minimal transport protocol. Its entire job is to add just enough information to a raw IP packet so that data can be delivered to the correct **application** (via a port number) with an optional integrity check — and nothing more.

UDP is defined in **RFC 768 (1980)** — a specification so short it fits on three pages. That brevity is the whole point.

A single unit of UDP data is called a **datagram**. A datagram is self-contained: it has a source port, a destination port, a length, a checksum, and a payload. It is not part of a "stream" and it has no relationship to the datagram before or after it. Each one is an independent shout.

What UDP gives you:
- **Multiplexing** — port numbers so many apps can share one IP address.
- **Optional integrity** — a checksum to detect corrupted data (which is then *dropped*, not fixed).

What UDP deliberately does **NOT** give you:
- No connection setup (no handshake).
- No acknowledgments — the sender never learns if the datagram arrived.
- No retransmission — a lost datagram stays lost.
- No ordering — datagram #2 may arrive before datagram #1.
- No de-duplication — a datagram may arrive twice.
- No flow control — a fast sender can overwhelm a slow receiver.
- No congestion control — UDP does not slow down when the network is jammed.

If you want any of those guarantees, **your application code provides them.** UDP hands you the raw pipe and steps out of the way.

---

## Why Does It Matter for a Backend Developer?

You might think "I use HTTP, HTTP is TCP, so I never touch UDP." Wrong. UDP is under your feet constantly:

- **Every DNS lookup** your service makes (resolving a database hostname, an external API, a Kafka broker) is UDP on port 53. When DNS is slow, you're debugging UDP behavior.
- **Metrics and observability.** StatsD, DogStatsD (Datadog), and many tracing agents ship metrics over UDP. If your app emits 50,000 metrics/second and some vanish, that's UDP dropping datagrams — by design.
- **Time sync (NTP)** keeps your servers' clocks aligned. Clock skew breaks JWT expiry, TLS validation, and distributed transaction ordering. NTP is UDP.
- **HTTP/3 / QUIC** — the newest version of HTTP runs on UDP, not TCP. When you enable HTTP/3 on your CDN or load balancer, you are opting into a UDP-based transport (Topic 12).
- **Service discovery** (mDNS, some Consul/gossip protocols) and **VPNs** (WireGuard, most OpenVPN configs) run over UDP.
- **Real-time features** — if you ever build voice/video, live game state, or telemetry, TCP's "wait and retransmit" is a liability and UDP is the correct choice.

Knowing UDP means you can answer: *"Why did we lose 3% of our metrics during the traffic spike?"* (UDP send buffer overflowed), *"Why is my DNS resolver randomly slow?"* (a UDP query got dropped and the client waited for a 5-second timeout before retrying), and *"Should this new telemetry firehose use TCP or UDP?"* (do you need every event, or the freshest events?).

---

## The Packet/Protocol Anatomy

Here is the entire UDP header. **8 bytes. Four fields. That's it.**

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│          Source Port          │       Destination Port        │  ← bytes 0-3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│            Length             │           Checksum            │  ← bytes 4-7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│                                                               │
│                       Data (payload)                          │  ← your bytes
│                                                               │
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Size | What it holds |
|-------|------|---------------|
| **Source Port** | 16 bits (0–65535) | Which port the datagram came FROM. Optional — can be 0 if no reply expected. |
| **Destination Port** | 16 bits (0–65535) | Which port/app it's going TO. This is how the OS knows to hand it to your listener. |
| **Length** | 16 bits | Total length of header + data, in bytes. Minimum 8 (header only, empty payload). |
| **Checksum** | 16 bits | Error-detection over header + data + a pseudo-header from IP. Optional in IPv4, mandatory in IPv6. |

### Contrast with the TCP header

This is the punchline. Here's TCP's header next to UDP's:

```
UDP HEADER — 8 bytes total                TCP HEADER — 20 bytes MINIMUM (up to 60)
┌────────────┬────────────┐               ┌────────────┬────────────┐
│ Src Port   │ Dst Port   │               │ Src Port   │ Dst Port   │
├────────────┴────────────┤               ├────────────┴────────────┤
│ Length     │ Checksum   │               │   Sequence Number       │  ← ordering
└────────────┴────────────┘               ├─────────────────────────┤
     that's the whole thing               │ Acknowledgment Number   │  ← reliability
                                          ├──────┬──────────────────┤
                                          │ Data │ Flags │ Window   │  ← flow control
                                          │ Off. │ SYN.. │          │
                                          ├──────┴──────────────────┤
                                          │ Checksum   │ Urgent Ptr │
                                          ├─────────────────────────┤
                                          │ Options (0–40 bytes)    │  ← MSS, SACK, timestamps
                                          └─────────────────────────┘
```

Every extra field in TCP exists to provide a guarantee UDP doesn't:
- **Sequence Number** → reordering + duplicate detection. *(UDP: none)*
- **Acknowledgment Number** → "I got it" confirmations + retransmission. *(UDP: none)*
- **Window** → flow control, so a fast sender doesn't drown a slow receiver. *(UDP: none)*
- **Flags (SYN/ACK/FIN)** → connection setup and teardown. *(UDP: none)*

UDP's header is smaller because UDP does less. Fewer bytes on the wire, and — far more importantly — **fewer round trips** before your data can flow.

### What "connectionless" really means

This is the single most important idea in this topic. "Connectionless" is not vague marketing — it has a precise meaning:

```
TCP (connection-oriented):                UDP (connectionless):

  1. SYN         →                          (no handshake at all)
  2.      ←  SYN-ACK                         sendto() — datagram leaves immediately
  3. ACK         →                          done.
     [1 full round-trip BEFORE any data]
  4. now send data →                        There is NO connection object,
  5.      ← ACK (got it)                     NO shared state on either end,
  6. resend if no ACK                        NO memory of "who am I talking to".
  7. FIN handshake to close                  Each datagram is independent.
```

With UDP there is no "connection" living anywhere. The kernel does not remember that you sent a datagram to `10.0.0.5:9125` a moment ago. It does not track sequence numbers, timers, or acknowledgments for that peer. Send a datagram and every trace of the exchange is gone. That's why it's "fire-and-forget": you fire, and there is literally nothing left to remember.

---

## How It Works — Step by Step

Let's trace a single UDP datagram from a Node.js app to a metrics server. Compare this to the multi-step TCP dance in Topic 01 — it's dramatically shorter.

```
YOU CALL: socket.send("api.latency:42|ms", 8125, "metrics.internal")
```

### Step 1 — Resolve the destination (if it's a hostname)
```
"metrics.internal" is a name, not an IP. The OS does a DNS lookup.
(That DNS lookup is ITSELF a UDP datagram to port 53 — UDP all the way down.)

Result: metrics.internal = 10.0.4.20
```

### Step 2 — The OS builds ONE datagram. No handshake.
```
There is NO SYN. NO connection setup. NO waiting.
The kernel wraps your 17 bytes of payload:

┌─────────────────────────────────────────────────┐
│ IP header  │ Protocol = 17 (UDP)                 │
│            │ src 10.0.1.7  dst 10.0.4.20         │
├─────────────────────────────────────────────────┤
│ UDP header │ src port 51000 (ephemeral, chosen)  │
│            │ dst port 8125                        │
│            │ length 25  (8 header + 17 data)      │
│            │ checksum 0x4a2f                       │
├─────────────────────────────────────────────────┤
│ Payload    │ "api.latency:42|ms"                 │
└─────────────────────────────────────────────────┘
```

### Step 3 — The datagram is handed to IP and shipped
```
The datagram leaves the NIC. The kernel's send() call returns IMMEDIATELY —
it does not wait for anything. As far as your app is concerned, "sent" means
"handed to the kernel", NOT "arrived at the destination".

Hop 1: your host      → top-of-rack switch
Hop 2: switch         → data-center router
Hop 3: router         → metrics host's NIC

If ANY hop is congested and drops the datagram, it is gone. Silently.
No error comes back to your app. send() already returned success.
```

### Step 4 — The receiver's OS delivers it (or doesn't)
```
The metrics server has a socket bound to UDP port 8125.
Its OS receives the datagram and puts it in the socket's RECEIVE BUFFER.

  If the buffer has room → the app's recvfrom() will read it.
  If the buffer is FULL  → the OS DROPS the datagram. Silently.
                           (This is the #1 cause of "lost metrics".)
```

### Step 5 — The app reads one whole datagram
```
The receiving app calls recvfrom(). It gets back:
  - the payload "api.latency:42|ms" as ONE complete message
  - the sender's IP and port (10.0.1.7:51000)

KEY: it gets exactly ONE datagram, with its BOUNDARY preserved.
If the sender sent three datagrams, the receiver does three reads —
never one big blob. (TCP would merge them into a byte stream; UDP does not.)
```

### Step 6 — There is no Step 6
```
No ACK is sent back. No teardown. No FIN.
The sender never learns the outcome. The exchange is over.

Total round trips before data flowed: ZERO.
(TCP would have spent 1 RTT on the handshake before sending a single byte.)
```

### The whole picture, side by side

```
   TCP: send "hello"                        UDP: send "hello"
   ─────────────────                        ─────────────────
   SYN            →   ┐                      sendto("hello") →   ┐  datagram leaves
        ←     SYN-ACK │ 1 RTT wasted                             │  in ~0 time
   ACK            →   ┘  before data                            done.
   "hello"        →
        ←         ACK  (confirmed)           (no confirmation, ever)
   FIN            →   ┐
        ←         ACK │ teardown
        ←         FIN │
   ACK            →   ┘
```

---

## Exact Syntax Breakdown

### The socket system calls (what the OS actually exposes)

TCP and UDP use *different* socket calls. This is where "connectionless" shows up in code:

```
TCP server flow:                          UDP server flow:
  socket()   create socket                  socket()   create socket
  bind()     claim a port                    bind()     claim a port
  listen()   mark as accepting               recvfrom() read a datagram + sender addr
  accept()   ← BLOCKS for a connection        sendto()  reply to that addr (optional)
  read()/write()  stream I/O                 (no accept, no listen, no connect)
```

Notice: **UDP has no `accept()` and no `listen()`.** There is no connection to accept. You `bind()` to a port and immediately start reading datagrams with `recvfrom()`, which also tells you *who* sent each one.

```
recvfrom(sockfd, buffer, len, flags, &src_addr, &addrlen)
   │        │      │       │    │       │          │
   │        │      │       │    │       │          └── OS fills in: length of src_addr
   │        │      │       │    │       └── OS fills in: WHO sent this datagram
   │        │      │       │    └── options (usually 0)
   │        │      │       └── max bytes to read (size of ONE datagram)
   │        │      └── where to put the payload
   │        └── the bound UDP socket
   └── returns: number of bytes in this ONE datagram
```

```
sendto(sockfd, data, len, flags, &dest_addr, addrlen)
   │      │     │    │    │        │           │
   │      │     │    │    │        │           └── size of dest_addr struct
   │      │     │    │    │        └── WHERE to send it (IP + port) — given EVERY call
   │      │     │    │    └── options (usually 0)
   │      │     │    └── number of bytes to send (becomes ONE datagram)
   │      │     └── your payload
   │      └── the UDP socket
   └── returns: bytes handed to kernel (NOT bytes delivered!)
```

The destination is passed on **every single `sendto()`** because there's no stored connection to remember it.

### Node.js `dgram` module — the API surface

Node.js exposes UDP through the built-in `dgram` module (no npm install needed):

```javascript
const dgram = require('dgram');

dgram.createSocket('udp4')   // create a UDP/IPv4 socket ('udp6' for IPv6)
  │
  ├── .bind(port[, addr])              // claim a port to RECEIVE on (server)
  ├── .send(msg, port, host[, cb])     // fire a datagram (sender) — host+port EVERY time
  ├── .on('message', (msg, rinfo)=>{}) // fired per received datagram; rinfo = sender info
  ├── .on('listening', ()=>{})         // fired once bind() succeeds
  ├── .on('error', (err)=>{})          // fired on socket errors
  ├── .setBroadcast(true)              // allow sending to broadcast address
  ├── .addMembership(multicastAddr)    // join a multicast group
  └── .close()                          // release the socket (there's no "connection" to tear down)
```

Key detail: `.send()` takes the `host` and `port` on every call, exactly like `sendto()`. And the `'message'` event's `rinfo` object (`{ address, port, family, size }`) is exactly what `recvfrom()` fills in.

---

## Example 1 — Basic

A minimal UDP echo server and client in Node.js. Notice there is **no `.listen()`, no `.accept()`, no `.connect()`** — you bind and you're done.

**`udp-server.js`:**
```javascript
const dgram = require('dgram');
const server = dgram.createSocket('udp4');

// Fired ONCE per received datagram. Each datagram is a complete message.
server.on('message', (msg, rinfo) => {
  console.log(`Got "${msg}" from ${rinfo.address}:${rinfo.port} (${rinfo.size} bytes)`);
  // Reply by sending a datagram BACK to whoever sent this one.
  // We know their address only because 'message' handed us rinfo.
  server.send(`echo: ${msg}`, rinfo.port, rinfo.address);
});

server.on('listening', () => {
  const a = server.address();
  console.log(`UDP server listening on ${a.address}:${a.port}`);
});

server.bind(41234); // claim UDP port 41234. No accept() — datagrams just arrive.
```

**`udp-client.js`:**
```javascript
const dgram = require('dgram');
const client = dgram.createSocket('udp4');
const message = Buffer.from('hello udp');

// Fire and forget. host + port given right here, on the send call.
client.send(message, 41234, 'localhost', (err) => {
  if (err) console.error(err);
  else console.log('datagram sent (handed to kernel — NOT confirmed delivered)');
});

// If we want the echo back, listen for it:
client.on('message', (msg) => {
  console.log(`server replied: ${msg}`);
  client.close(); // nothing to tear down — just release the socket
});
```

**Run it:**
```bash
node udp-server.js &
node udp-client.js
```

**Output:**
```
UDP server listening on 0.0.0.0:41234        ← from the server
datagram sent (handed to kernel — NOT confirmed delivered)   ← from the client
Got "hello udp" from 127.0.0.1:52918 (9 bytes)   ← server received it
server replied: echo: hello udp                   ← client got the echo
```

The client's `52918` is an **ephemeral port** the OS picked automatically (Topic 05). We never bound the client to a port — for a sender, the OS assigns one so replies have somewhere to go.

**Prove it's connectionless with the command-line `nc` (netcat):**
```bash
# -u = UDP mode. Type a line, hit Enter — it's sent as a datagram.
nc -u localhost 41234
hello from netcat
echo: hello from netcat      ← the server's reply appears

# There was no "Connected to..." message. nc didn't establish anything.
# It just started firing datagrams at the port.
```

---

## Example 2 — Production Scenario

**The situation:** Your Node.js services emit application metrics (request latency, error counts, queue depth) to a **StatsD** collector over UDP port 8125. StatsD aggregates them and forwards to Datadog/Graphite. This works beautifully for months. Then, during a Black Friday traffic spike, your dashboards show latency graphs going **flat and jagged** — data points are missing. No errors in your app logs. No exceptions. Metrics are just... vanishing.

**Why UDP was chosen here (and why that was correct):**
```
StatsD over UDP is deliberate. The reasoning:

  1. Metrics are HIGH VOLUME — tens of thousands per second per host.
  2. Metrics are DISPOSABLE — one lost latency sample out of 50,000 is
     statistically invisible. The p99 barely moves.
  3. Metrics must NOT slow down the app. With TCP, if the metrics
     collector were slow or down, TCP back-pressure (flow control) could
     BLOCK your request handler waiting to write. Your API would slow down
     because your METRICS pipeline is slow. That's insane.
  4. With UDP, send() returns instantly and NEVER blocks on the collector.
     If the collector is overwhelmed, datagrams drop. Your app is untouched.

Trade-off accepted: "We'd rather lose some metrics than slow down the app."
```

So the dropped datagrams during the spike are not a bug — they're the trade-off working as designed. But you still want to *detect and quantify* the drops so you can size the pipeline correctly.

**Step 1 — Confirm datagrams are being dropped at the OS level.**

On Linux, the kernel counts UDP receive errors (buffer overflows). Check `ss`:
```bash
ss -u -a -n
```
```
State    Recv-Q   Send-Q   Local Address:Port    Peer Address:Port
UNCONN   212992   0        0.0.0.0:8125          0.0.0.0:*
         ^^^^^^
         Recv-Q is PEGGED at the max buffer size (212992 bytes).
         The app can't drain the socket fast enough — the buffer is full.
         Every new datagram that arrives now gets DROPPED.
```

**Step 2 — Read the systemwide UDP drop counter.**

`netstat -su` (or `ss -u` on newer systems) exposes cumulative UDP stats:
```bash
netstat -su
```
```
Udp:
    48923011 packets received
    12 packets to unknown port received
    284517 packet receive errors        ← THIS. Datagrams dropped: buffer full.
    48401203 packets sent
    RcvbufErrors: 284517                 ← receive-buffer overflow count
    SndbufErrors: 0
    InErrors: 284517
```
```
The "packet receive errors" / "RcvbufErrors" number is your drop count.
Watch it over time to get a DROP RATE:

  netstat -su | grep "receive errors"    # sample it every few seconds
```

Sample it twice, 10 seconds apart, and subtract:
```bash
# Take two samples 10s apart and compute drops/sec
A=$(netstat -su | awk '/receive errors/ {print $1}'); sleep 10; \
B=$(netstat -su | awk '/receive errors/ {print $1}'); \
echo "dropped $(( (B - A) / 10 )) datagrams/sec"
```
```
dropped 4213 datagrams/sec     ← during the spike. That's your missing metrics.
```

**Step 3 — Understand the root cause.**

```
   your 200 Node.js pods                 StatsD collector (single host)
   each firing 300 metrics/sec           socket recv buffer = 208 KB
            │                                      │
            └──────── 60,000 datagrams/sec ───────►│  app reads ~55,000/sec
                                                    │
                                          buffer fills → OS drops the
                                          remaining ~5,000/sec silently
```

The sender side is fine (UDP `send()` never blocks). The **receiver** can't drain its socket buffer fast enough, so the kernel drops the overflow.

**Step 4 — The fixes (in order of preference):**

```
1. INCREASE the receive buffer (buys headroom for bursts):
   sysctl -w net.core.rmem_max=26214400
   sysctl -w net.core.rmem_default=26214400
   ...and have StatsD request a bigger SO_RCVBUF.

2. SCALE OUT the collector — run StatsD on multiple hosts, shard by
   metric name, or run a local StatsD agent on EACH app host (localhost
   UDP never leaves the box, near-zero loss).

3. SAMPLE at the source — send 1 in 10 datagrams and tell StatsD the
   sample rate: "api.hits:1|c|@0.1". 90% fewer datagrams, same aggregate.

4. ACCEPT the loss — for many metrics, 7% loss during a 20-minute spike
   is genuinely fine. Document it. Don't over-engineer.
```

**The lesson:** UDP drops are **silent and invisible to the application** — no exception, no log line. You detect them with OS counters (`netstat -su`, `ss -u`), never from your app. And the correct fix is often *not* "switch to TCP" — it's "size the pipeline for the burst, or accept the loss." Choosing UDP was a deliberate trade of *guaranteed delivery* for *guaranteed non-blocking speed*.

---

## Common Mistakes

### Mistake 1: Assuming UDP guarantees delivery
```javascript
// WRONG mental model:
client.send(criticalOrder, 9000, 'orders.internal');
// "Great, the order was delivered."   ← NO. It was handed to the kernel.
```
**Reality:** `send()` returning success means "the kernel accepted the bytes", not "the peer received them". The datagram can be dropped by any router, by a full receive buffer, or vanish entirely — and **you get no error**. Never send data you *must not lose* over bare UDP without building your own acknowledgment/retry on top. For critical data, use TCP (Topic 08) or a message queue with delivery guarantees.

---

### Mistake 2: Forgetting datagrams can be reordered or duplicated
```javascript
// WRONG: assuming datagrams arrive in send order
socket.send('step:1', port, host);
socket.send('step:2', port, host);
socket.send('step:3', port, host);
// Receiver might see: 1, 3, 2   ... or  1, 2, 2, 3  (duplicate!) ... or  1, 3
```
**Reality:** IP can route datagrams over different paths, so #2 can overtake #1. Routers can also duplicate a datagram (rare, but real). UDP has **no sequence numbers**, so it neither detects nor fixes this. **Fix:** if order or de-duplication matters, put your *own* sequence number in the payload and handle it in the app:
```javascript
socket.send(JSON.stringify({ seq: 1, data: 'step one' }), port, host);
// receiver tracks the highest seq seen, discards duplicates & stale ones
```

---

### Mistake 3: Sending payloads larger than the path MTU (causing IP fragmentation)
```javascript
// WRONG: one giant 60 KB UDP datagram
const huge = Buffer.alloc(60000);
socket.send(huge, port, host);
```
**Reality:** A typical Ethernet path MTU is **1500 bytes**. A UDP payload above ~**1472 bytes** (1500 − 20 IP − 8 UDP) forces **IP fragmentation** — the datagram is split into multiple IP fragments. Here's the trap: with UDP, **if even ONE fragment is lost, the ENTIRE datagram is discarded** (there's no per-fragment retransmission). Fragmentation also gets blocked by many firewalls, and a fragmented datagram is more likely to be lost overall. **Fix:** keep UDP payloads small — **under ~1400 bytes** is a safe rule of thumb for internet paths. If you need to send more, chunk it in the application and reassemble, or use TCP/QUIC which handle segmentation properly.

---

### Mistake 4: Expecting a "connection" to exist
```javascript
// WRONG: treating UDP like TCP
socket.on('connect', () => { ... });   // never fires meaningfully for a listener
socket.on('close', handlePeerDisconnect); // there's no peer "disconnect" event
// "Is my UDP peer still connected?"  ← there was never a connection to lose.
```
**Reality:** There is no connection object, no "connected" state, and no way to be *notified* that the other side went away — because nothing was ever established. If a UDP peer crashes, you find out only because its datagrams stop arriving (which is indistinguishable from "the network dropped them" or "it's just quiet"). **Fix:** if you need liveness, build an application-level **heartbeat/keepalive** — have each side send a small "I'm alive" datagram every N seconds and treat silence past a timeout as "gone."

---

### Mistake 5: Reading a datagram into a buffer that's too small
```javascript
// In lower-level languages / raw sockets: recvfrom(sock, buf, 512, ...)
// If the datagram is 900 bytes and your buffer is 512...
```
**Reality:** Unlike TCP (a byte stream you can read in any-sized chunks), a UDP datagram is an **atomic message**. If your read buffer is smaller than the datagram, the **excess is silently truncated and lost** — you do not get the rest on the next read. **Fix:** always size your receive buffer to the largest datagram you could receive (65507 bytes is the theoretical max UDP payload; in practice size it to your protocol's max message). In Node's `dgram`, the `'message'` event always hands you the *complete* datagram, so this bites you mainly in C/Go/raw-socket code — but the mental model matters everywhere.

---

## Hands-On Proof

```bash
# 1. See UDP sockets your machine has open (LISTEN-equivalent for UDP is "UNCONN")
ss -u -a -n            # Linux: -u UDP, -a all, -n numeric
lsof -nP -iUDP         # macOS/Linux: which processes own which UDP ports

# 2. Watch a DNS lookup happen over UDP port 53 with tcpdump
sudo tcpdump -n -i any udp port 53      # start this, then in another shell:
dig example.com                          # you'll see the UDP query + response fly by
# Note: ONE datagram out (query), ONE datagram back (answer). No handshake.

# 3. Fire a raw UDP datagram by hand with netcat (no connection is established)
nc -u 8.8.8.8 53                         # just... starts. No "Connected" line.
# (Ctrl-C to quit. You can't easily type a valid DNS query by hand, but note
#  there was NO handshake — nc went straight to "ready to send".)

# 4. Read systemwide UDP statistics — receives, sends, and DROP counters
netstat -su                              # Linux/macOS: look for "receive errors"
#   Udp:
#       packets received
#       packet receive errors   ← dropped due to full buffers
#       packets sent

# 5. Compare header sizes on the wire — capture and inspect
sudo tcpdump -n -v -i any udp port 53 -c 1   # -v shows the 8-byte UDP header fields:
#   ...  IP (... proto UDP (17), length 56) 10.0.0.1.51000 > 8.8.8.8.53: ...
#                       ^^^^^^^^^^^^^^^^          ^^^^^^^^^^   ^^^^^^^^^
#                   protocol 17 = UDP          src port      dst port 53

# 6. Prove send() doesn't confirm delivery: send to a port with NOTHING listening
echo "into the void" | nc -u -w1 127.0.0.1 59999
# It "succeeds" — no error — even though nobody is on port 59999.
# (You MIGHT get an ICMP "port unreachable" back, but the send itself never failed.)

# 7. See NTP (UDP 123) time sync traffic
sudo tcpdump -n -i any udp port 123
```

---

## Practice Exercises

### Exercise 1 — Easy: Watch UDP with your own eyes
```bash
# DNS is the easiest UDP protocol to observe. In terminal 1:
sudo tcpdump -n -i any udp port 53

# In terminal 2, trigger a fresh lookup:
dig +short github.com

# Answer these from the tcpdump output:
# 1. How many datagrams did you see for one DNS lookup? (query + response)
# 2. What was the source port of your query? Is it ephemeral (49152–65535)?
# 3. What was the destination port? (should be 53)
# 4. Was there ANY handshake before the query datagram? (compare to TCP's SYN/SYN-ACK/ACK)
```

### Exercise 2 — Medium: Build a UDP ping and measure loss
```javascript
// Write a UDP client that sends 100 datagrams, each with a sequence number,
// and a server that echoes them back. Count how many replies you get.
//
// server.js:
const dgram = require('dgram');
const s = dgram.createSocket('udp4');
s.on('message', (m, r) => s.send(m, r.port, r.address)); // echo it straight back
s.bind(41234);
//
// client.js — YOU FILL IN:
// - send 100 datagrams: Buffer.from(String(i)) for i in 0..99
// - listen for 'message' replies, count how many come back
// - after 2 seconds, print: "sent 100, received N, lost 100-N"
//
// Questions:
// 1. On localhost, do you lose any? (Should be ~0 — no real network.)
// 2. Now flood it: send 100000 datagrams as fast as possible in a tight loop.
//    Do you start losing replies? Check `netstat -su` for "receive errors".
// 3. WHY does flooding cause loss even on localhost? (Hint: receive buffer.)
```

### Exercise 3 — Hard (Production Simulation): Add reliability on top of UDP
```javascript
// UDP gives you nothing. Build a MINIMAL reliable layer on top — this is
// literally what TCP and QUIC do internally, in miniature.
//
// Requirements:
// - Sender sends each message with a sequence number: {seq, payload}
// - Receiver sends back an ACK datagram: {ack: seq}
// - Sender keeps a map of unacked messages. If no ACK arrives within 200ms,
//   RETRANSMIT that message. Give up after 5 retries.
// - Receiver de-duplicates: if it sees a seq it already processed, it
//   re-sends the ACK but does NOT re-process the payload.
//
// Test it by artificially dropping datagrams:
//   in the receiver, `if (Math.random() < 0.3) return;` // drop 30% on purpose
//
// Questions:
// 1. Does your data still get through with 30% loss? (It should — retries win.)
// 2. What happens to LATENCY when loss is high? (Retries add 200ms each.)
// 3. You just reinvented a piece of TCP. Which fields in the TCP header
//    correspond to your {seq} and {ack}? (See the header diagram above.)
// 4. What did you NOT implement that TCP has? (ordering across many messages,
//    flow control, congestion control...) — this is why "just use TCP" is
//    usually the right call unless you have a specific reason not to.
```

---

## Mental Model Checkpoint

Answer these from memory. If you can't, re-read the relevant section.

1. **The UDP header is 8 bytes with four fields. Name all four.** (And which one is optional in IPv4 but mandatory in IPv6?)

2. **"Connectionless" — list at least four specific guarantees UDP does NOT provide** that TCP does. (Think about what each TCP header field was for.)

3. **Your Node.js app calls `socket.send()` and the callback fires with no error. What has actually been guaranteed at that moment?** (Careful — the answer is much weaker than "delivered".)

4. **You're designing a metrics pipeline emitting 80,000 samples/sec. Argue for UDP over TCP in two sentences.** (What's the failure mode you're specifically avoiding by NOT using TCP?)

5. **A colleague sends 4 KB JSON blobs over UDP and complains they "randomly disappear" more than small messages do. What's happening at the IP layer, and what's the fix?**

6. **Name four real protocols/systems that run on UDP and, for each, one sentence on why UDP fits.** (Hint: DNS, NTP, VoIP, QUIC/HTTP-3, statsd, DHCP...)

7. **UDP has no `accept()` and no `listen()`. What does a UDP "server" do instead to start receiving, and how does it learn who sent each datagram?**

---

## Quick Reference Card

### TCP vs UDP — the comparison that matters

| Property | TCP | UDP |
|----------|-----|-----|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless (fire-and-forget) |
| **Reliability** | Guaranteed delivery (ACK + retransmit) | None — datagrams can be lost silently |
| **Ordering** | Guaranteed in-order | None — can arrive out of order |
| **Duplicates** | Detected & removed | Possible — app must de-dup |
| **Header size** | 20 bytes min (up to 60) | **8 bytes, fixed** |
| **Speed / latency** | Slower — 1 RTT handshake before data | Faster — data flows immediately, 0 RTT setup |
| **Flow control** | Yes (sliding window) | No |
| **Congestion control** | Yes (backs off when network is jammed) | No (app's responsibility) |
| **Head-of-line blocking** | Yes (a lost segment stalls the stream) | No (each datagram independent) |
| **Data boundaries** | Byte stream (no message boundaries) | **Datagram boundaries preserved** |
| **Broadcast / multicast** | No (unicast only) | **Yes** |
| **Connection state in kernel** | Yes (per connection) | None |
| **Typical use** | HTTP/1-2, SSH, DB connections, file transfer | DNS, DHCP, NTP, VoIP, gaming, streaming, QUIC, metrics |

### When to choose UDP over TCP

```
CHOOSE UDP when...                          STICK WITH TCP when...
─────────────────                           ──────────────────────
• Speed/latency > reliability               • You cannot lose data (payments,
• Data is disposable or self-refreshing       orders, records)
  (metrics, sensor readings, game state)    • You need in-order delivery
• You need broadcast or multicast           • You want the OS to handle retries,
• You'll build your OWN reliability           ordering, flow & congestion control
  (QUIC, game netcode)                       • You're doing request/response over
• Tiny request/response, one datagram          a long-lived connection (most APIs)
  each way (DNS)                             • You don't want to reinvent TCP
```

### UDP-based protocols you'll actually meet

```
Port 53    DNS      — name resolution (falls back to TCP for large responses)
Port 67/68 DHCP     — automatic IP address assignment
Port 123   NTP      — clock synchronization
Port 161   SNMP     — network device monitoring
Port 8125  StatsD   — metrics collection
Port 443   QUIC/HTTP-3 — reliable transport BUILT ON TOP of UDP (Topic 12)
RTP/SRTP   VoIP & video calls — real-time media
WireGuard  modern VPN tunnels
```

### Cheat block — commands & facts

```bash
# Inspect / debug UDP
ss -u -a -n                    # list UDP sockets (UNCONN = "listening")
lsof -nP -iUDP                 # which process owns which UDP port
netstat -su                    # UDP stats incl. "receive errors" (= drops)
sudo tcpdump -n udp port 53    # watch DNS (UDP) traffic live
nc -u HOST PORT                # fire raw UDP datagrams by hand

# Facts to remember
UDP header       = 8 bytes  (src port, dst port, length, checksum)
TCP header       = 20+ bytes
Max UDP payload  = 65507 bytes (theoretical); keep < ~1400 to avoid fragmentation
IP Protocol num  = 17 (UDP), 6 (TCP)
Handshake cost   = 0 RTT (UDP) vs 1 RTT (TCP)
Delivery         = fire-and-forget; send() != delivered
```

---

## When Would I Use This at Work?

### Scenario 1: "We're losing 5% of our application metrics under load"
You now know this is almost certainly UDP receive-buffer overflow, not a bug in your app. You'd confirm with `netstat -su` (watch "receive errors" climb) and `ss -u` (Recv-Q pegged at max). The fix is to raise `net.core.rmem_max`, run a local StatsD agent per host, or sample at the source — **not** to switch to TCP, which would risk back-pressuring your request handlers. This is the trade-off from Example 2, live.

### Scenario 2: "Should our new IoT telemetry firehose use TCP or UDP?"
The deciding question is: *do we need every reading, or the freshest reading?* Ten thousand sensors reporting temperature every second → losing an occasional reading is invisible, and UDP's non-blocking speed keeps the sensors cheap and simple → **UDP**. But billing events from those same sensors ("customer used 4.2 kWh") → cannot lose a single one → **TCP or a durable queue**. You'll often run *both*: UDP for the high-volume disposable stream, TCP for the low-volume critical events.

### Scenario 3: "Our DNS resolution is intermittently slow — random 5-second stalls"
DNS is UDP. A UDP query datagram (or its response) got dropped on the network, and there's no retransmission — so the resolver client just **waits for its timeout** (often 5 seconds) before retrying. Those exact-multiple-of-5s stalls are the fingerprint. You'd look at packet loss on the path to your resolver, consider a closer/local caching resolver, and tune the resolver timeout. Knowing DNS rides on lossy UDP is what makes this diagnosable.

### Scenario 4: "Marketing wants us to enable HTTP/3 — what actually changes?"
HTTP/3 runs on **QUIC, which is built on UDP** (Topic 12). Enabling it means your load balancer/CDN now listens on **UDP** 443 in addition to TCP 443. You'd verify UDP 443 is open through every firewall and security group (teams often forget UDP rules), and understand that QUIC re-implements reliability, ordering, and congestion control in userspace *on top of* UDP — getting TCP's guarantees without TCP's head-of-line blocking. Your understanding of "UDP gives nothing, the app builds reliability on top" is exactly what QUIC is.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — IP packets, encapsulation, and why the network drops packets
- `03-tcp-ip-model.md` — where the Transport layer sits
- `05-ports.md` — port numbers, ephemeral ports (UDP multiplexes with these)

**Study next (in order):**
- `08-tcp-in-depth.md` — **the reliable counterpart.** Everything UDP refuses to do — handshake, sequence numbers, ACKs, retransmission, flow & congestion control — TCP does. Read it right after this to see the trade-off from the other side.
- `06-dns-in-depth.md` — the most important UDP protocol you use daily; see the fallback-to-TCP behavior for large responses
- `12-http3-and-quic.md` — **QUIC builds reliability, ordering, and encryption directly on top of UDP**, getting TCP-like guarantees without head-of-line blocking. This topic is the foundation that makes HTTP/3 make sense.

**Debugging tie-ins:**
- `23-tcpdump-and-wireshark.md` — capture and read UDP datagrams on the wire
- `24-netstat-and-ss.md` — read UDP socket tables and the drop counters from Example 2

---

*Doc saved: `/docs/networking/07-udp.md`*
