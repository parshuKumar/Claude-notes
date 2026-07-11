# 23 — tcpdump and Wireshark

> **Phase 4 — Topic 3 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `08-tcp-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine every conversation in a busy office building travels through the mail room. Letters, packages, memos — everything passes through that one room before it goes anywhere.

Now imagine you install a security camera in the mail room that photographs every envelope: who sent it, who it's addressed to, how big it is, and what stamp it carries. You don't open sealed envelopes (those are encrypted), but you can see **everything on the outside** and the exact order things came through.

That camera is **packet capture**. `tcpdump` is the plain security camera with a text log. **Wireshark** is the same camera hooked up to a giant control room with search, replay, colored highlighting, and an assistant who reads each envelope and tells you "this one is a complaint letter" or "this delivery got sent twice."

When your app says "connection reset" or "timeout" and the logs tell you nothing, you go to the mail room and watch the actual envelopes. The packets never lie — the application logs might.

---

## Where This Lives in the Network Stack

Packet capture doesn't live *at* one layer — it **taps the wire below your application** and shows you every layer at once. It hooks into the kernel just above the network card (NIC), before the OS hands data up to your process.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (your Node.js app, nginx)             │  ← you normally only see THIS
│  Layer 6 — Presentation  (TLS record — ENCRYPTED payload)      │  ← capture sees the handshake, not plaintext
│  Layer 5 — Session                                             │
│  Layer 4 — Transport     (TCP/UDP — flags, seq, ack, ports)    │  ← capture reads all of this
│  Layer 3 — Network       (IP — src/dst addresses, TTL)         │  ← capture reads all of this
│  Layer 2 — Data Link     (Ethernet — MAC addresses)            │  ← capture reads all of this
│  Layer 1 — Physical                                            │
└─────────────────────────────────────────────────────────────────┘
        ▲
        │  libpcap + BPF tap sits HERE (kernel, at the NIC)
        │  copies every frame in/out of the interface
        ▼
  ┌──────────────┐
  │  tcpdump     │  reads the copied frames, prints them / writes .pcap
  │  Wireshark   │
  └──────────────┘
```

Because the tap is at the NIC/kernel — **below** your app and **below** TLS decryption — you see raw wire bytes. That's why you can watch a TCP handshake fail before your app ever gets an error, and also why you **cannot** read the body of an HTTPS request without extra keys: it's already encrypted by the time it hits the wire.

---

## What Is This?

**Packet capture** is the act of copying network frames as they pass through a network interface, so you can inspect them. Three pieces make it work:

- **libpcap** — the C library that captures packets on Unix (Npcap on Windows). Both tools use it.
- **BPF (Berkeley Packet Filter)** — a tiny filtering engine *in the kernel*. Your capture filter is compiled to BPF bytecode and runs in-kernel, so packets you don't want are dropped **before** they're ever copied to userspace. This is why filtering at capture time is cheap and fast.
- **The NIC in promiscuous mode** — normally a NIC ignores frames not addressed to it. Capture tools flip it into "promiscuous mode" to grab everything on the segment. (On a switched network you mostly still see only your own traffic plus broadcasts — a switch doesn't forward other hosts' unicast to your port.)

Two tools sit on top of libpcap:

| Tool | Interface | Strength |
|------|-----------|----------|
| **tcpdump** | CLI | Everywhere by default. Perfect on a headless server over SSH. Lightweight capture to `.pcap`. |
| **Wireshark** | GUI | Deep **dissectors** for 3000+ protocols, Follow Stream, expert warnings, graphs. The analysis powerhouse. |
| **tshark** | CLI | Wireshark's engine without the GUI — same dissectors and display-filter syntax, scriptable. |

**The standard workflow:** capture on the server with `tcpdump -w cap.pcap` (no GUI on a prod box), copy the file down, open it in Wireshark on your laptop to analyze.

**Two hard truths up front:**

1. **You almost always need root/sudo.** Reading raw frames and enabling promiscuous mode is a privileged operation (needs `CAP_NET_RAW` on Linux). `tcpdump` without sudo usually fails with "permission denied" or "no suitable device found."
2. **TLS payloads are encrypted.** You see the TCP/IP metadata (who, when, how much, which flags) and the *unencrypted parts* of the TLS handshake — the `ClientHello` including **SNI** (the hostname), the negotiated version and cipher. You do **not** see the HTTP request/response body. To decrypt in Wireshark you need either the server's private key (only for old non-forward-secret RSA key exchange) or the per-session keys via the **`SSLKEYLOGFILE`** environment variable (works with curl, Chrome, Firefox, Node).

---

## Why Does It Matter for a Backend Developer?

Application logs tell you what your code *thinks* happened. Packet capture tells you what *actually crossed the wire*. When those two disagree, the packets win — and that gap is exactly where the nastiest bugs live.

You reach for capture when:

- **"Connection reset by peer" with no stack trace.** Who sent the RST — your app, the load balancer, or the OS? The capture shows you the exact packet and direction.
- **Intermittent timeouts.** Is the SYN even leaving your host? Is a SYN-ACK coming back? A capture answers in one line.
- **"It works with curl but not from the app."** Diff the two captures — different TLS version, missing header, wrong SNI, IPv6 vs IPv4.
- **Retransmissions and slowness** that no APM tool explains — packet loss shows up as retransmits and duplicate ACKs that never reach your metrics.
- **Verifying claims.** "The firewall allows that port." "The service sent the response." Capture is the tie-breaker in cross-team arguments.

This is the tool of last resort *and* the tool of ground truth. Topic `24-netstat-and-ss.md` tells you the *state* of connections; tcpdump tells you the *packets* that produced those states. Topic `27-debugging-a-slow-api.md` uses both together.

---

## The Packet/Protocol Anatomy

To read capture output you need to know the fields tcpdump prints. Here's the TCP segment header — every field tcpdump surfaces comes from here or the IP header wrapping it.

```
TCP HEADER (the part tcpdump summarizes on each line)
┌───────────────────────────────────────────────────────────┐
│ Source Port (16)          │ Destination Port (16)          │  → "10.0.1.20.51514 > ...443"
│ Sequence Number (32)                                       │  → "seq 2841"
│ Acknowledgment Number (32)                                 │  → "ack 2842"
│ DataOff│Rsvd│ FLAGS (9 bits) │ Window Size (16)            │  → "Flags [S.]" and "win 65535"
│ Checksum (16)             │ Urgent Pointer (16)            │
│ Options (variable: MSS, SACK, timestamps, window scale)    │  → "options [mss 1460,sackOK,...]"
└───────────────────────────────────────────────────────────┘

THE FLAGS BYTE (offset 13 in the TCP header) — this is what tcp[tcpflags] reads:
   bit:   7    6    5    4    3    2    1    0
        [CWR][ECE][URG][ACK][PSH][RST][SYN][FIN]
                              8    4    2    1    ← value if set

   SYN = 2, ACK = 16, FIN = 1, RST = 4, PSH = 8
```

**How tcpdump renders flags** (`.` always means ACK is set):

| tcpdump shows | Meaning | TCP moment (ties to Topic 08) |
|---------------|---------|-------------------------------|
| `[S]`  | SYN | Client opens connection (1st handshake packet) |
| `[S.]` | SYN + ACK | Server accepts (2nd handshake packet) |
| `[.]`  | ACK only | 3rd handshake packet, or a plain acknowledgment |
| `[P.]` | PSH + ACK | Data being pushed to the app (a real payload segment) |
| `[F.]` | FIN + ACK | Graceful close, one side done sending |
| `[R]` / `[R.]` | RST / RST+ACK | Abrupt reset — "go away," connection killed |

One captured line, fully labeled:

```
14:23:45.123456 IP 10.0.1.20.51514 > 93.184.216.34.443: Flags [P.], seq 1:518, ack 1, win 501, length 517
│               │  │              │  │             │      │          │        │      │        └ payload bytes in THIS segment
│               │  │              │  │             │      │          │        │      └ receiver's advertised window
│               │  │              │  │             │      │          │        └ next byte expected from peer
│               │  │              │  │             │      │          └ this segment carries bytes 1..517 (relative seq)
│               │  │              │  │             │      └ PSH+ACK: data being delivered up to the app
│               │  │              │  │             └ destination port (443 = HTTPS)
│               │  │              │  └ destination IP
│               │  │              └ source port (ephemeral, chosen by client OS)
│               │  └ source IP
│               └ Layer 3 protocol
└ timestamp (HH:MM:SS.microseconds)
```

Note: seq/ack numbers are shown **relative** (starting at 1) by default — tcpdump remembers the initial sequence number per flow. Use `-S` to see the real absolute numbers.

---

## How It Works — Step by Step

What actually happens from "you run tcpdump" to "you read a bug":

### Step 1 — You attach to an interface
```
sudo tcpdump -i eth0
    │
    ▼
libpcap opens a capture handle on eth0, sets it promiscuous (optional),
and installs your (empty) BPF filter into the kernel.
```

### Step 2 — The kernel filters in-place with BPF
```
Every frame in/out of eth0
        │
        ▼
   ┌─────────────┐   filter says "tcp port 443 and host 1.2.3.4"?
   │ BPF engine  │──► NO  → dropped immediately, never copied (cheap!)
   │ (in kernel) │──► YES → copied into a ring buffer for tcpdump
   └─────────────┘
```
This is the key efficiency: a **capture filter** runs *before* the copy, so a busy server with a tight filter costs almost nothing. An unfiltered capture on a 10Gbps link, by contrast, can drown you.

### Step 3 — tcpdump reads, dissects, prints (or writes)
```
ring buffer → tcpdump → (a) print one line per packet to your terminal
                        (b) OR write raw bytes to cap.pcap with -w
```
With `-w`, tcpdump does **no** parsing — it dumps raw frames. That's faster and lossless; you parse later. Without `-w`, it decodes each packet to a text summary as it goes.

### Step 4 — You save, transfer, and analyze
```
server:  sudo tcpdump -i any -w /tmp/cap.pcap 'tcp port 5432'
              │
              ▼  (Ctrl-C to stop, or -c 1000 to auto-stop)
         /tmp/cap.pcap
              │
              ▼  scp cap.pcap laptop:~/
laptop:  wireshark cap.pcap   ← full GUI analysis: Follow Stream, filters, graphs
```

### Step 5 — Capture filter (BPF) vs display filter (Wireshark) — the crucial split
```
CAPTURE FILTER (BPF)                   DISPLAY FILTER (Wireshark/tshark -Y)
─────────────────────                  ───────────────────────────────────
Runs in kernel, at capture time        Runs in the app, on already-captured data
Decides what gets STORED               Decides what gets SHOWN
Cannot be undone (dropped = gone)      Change it freely, nothing is lost
Syntax:  tcp port 443 and host 1.2.3.4 Syntax:  tcp.port==443 && ip.addr==1.2.3.4
Coarse (protocol/addr/port/flags)      Rich (any field of any protocol)
```
**Different tools, different syntax, different moment.** Mixing them up is the #1 beginner mistake — `http.request` is a Wireshark display filter and is *invalid* as a tcpdump capture filter, and `tcp port 443` typed into Wireshark's display-filter bar turns it red.

---

## Exact Syntax Breakdown

### The capture command, dissected

```
sudo tcpdump -i any -nn 'tcp port 443 and host api.example.com' -w cap.pcap
│    │       │   │   │                                          │
│    │       │   │   └── BPF CAPTURE FILTER (quote it! shell would split it)
│    │       │   │       tcp        → only TCP segments
│    │       │   │       port 443   → src OR dst port is 443
│    │       │   │       and        → boolean AND
│    │       │   │       host X     → src OR dst IP matches X (resolved ONCE at start)
│    │       │   └── -nn: don't resolve IP→hostname AND don't resolve port→name
│    │       │            (443 stays "443", never becomes "https")
│    │       └── -i any: capture on ALL interfaces (Linux). Use eth0 for one NIC.
│    └── tcpdump
│        -w cap.pcap → write raw packets to file instead of printing
└── sudo: capture needs root / CAP_NET_RAW
```

> **Gotcha with `host api.example.com`:** tcpdump resolves that hostname to an IP **once**, when the filter compiles. If the name has multiple A records or the DNS answer changes, you may miss traffic. On a server, resolve it yourself and filter by IP for certainty.
> **`-i any` note:** works on Linux (uses a "cooked" pseudo-header, so no Ethernet MACs). Not supported the same way on macOS — pick a real interface like `en0` there.

### Common flags, one line each

```
-i <iface>   interface to capture on (eth0, en0, lo, any)
-n           don't resolve IPs to hostnames (no reverse DNS)
-nn          also don't resolve port numbers to service names
-c <count>   stop after N packets (great for a quick sample)
-w <file>    write raw packets to a .pcap file (for later analysis)
-r <file>    read and replay a .pcap file (apply filters to saved data)
-v -vv -vvv  increasing verbosity (TTL, options, checksums, IP id...)
-A           print packet payload as ASCII (readable for plaintext HTTP)
-X           print payload as hex + ASCII side by side
-s <snaplen> bytes captured per packet (0 or 262144 = whole packet)
-e           show Layer-2 (Ethernet) header incl. MAC addresses
-S           show absolute sequence numbers (not relative)
-tttt        human-readable timestamp with date
```

### A Wireshark display-filter example (post-capture, different syntax)

```
tcp.flags.reset == 1 && ip.addr == 93.184.216.34
│                │       │          │
│                │       │          └── either src OR dst is this IP
│                │       └── field: ip.addr (matches source or destination)
│                └── field: the RST flag bit; ==1 means "set"
└── field path: protocol.field, compared with ==, !=, >, <, contains, matches
```
Other display filters you'll use constantly: `http.request`, `http.response.code == 500`, `tcp.analysis.retransmission`, `tcp.analysis.zero_window`, `dns.flags.rcode != 0`, `tcp.port == 5432`.

---

## Example 1 — Basic

**Goal:** capture and read a single TCP three-way handshake to a website, and prove you understand the flags.

```bash
# Capture 6 packets of TCP traffic to example.com's IP, no name resolution.
# In one terminal:
sudo tcpdump -i any -nn -c 6 'tcp and host 93.184.216.34'

# In a second terminal, trigger the connection:
curl -sI https://example.com > /dev/null
```

**Annotated output:**

```
10:15:32.100000 IP 10.0.1.20.51514 > 93.184.216.34.443: Flags [S],  seq 2841,        win 65535, length 0
10:15:32.145000 IP 93.184.216.34.443 > 10.0.1.20.51514: Flags [S.], seq 7134, ack 2842, win 65535, length 0
10:15:32.145100 IP 10.0.1.20.51514 > 93.184.216.34.443: Flags [.],       ack 7135, win 2048,  length 0
      │                                                        │
   ↑ 45ms gap between SYN and SYN-ACK = ~45ms RTT to the server (one round trip)
                                                          └ this is the 3-way handshake completing
```

Read it top to bottom:

```
Packet 1  [S]   client → server   "I want to talk, my seq starts at 2841"     (SYN)
Packet 2  [S.]  server → client   "OK, my seq is 7134, I got your 2841 (ack 2842)"  (SYN-ACK)
Packet 3  [.]   client → server   "Got it (ack 7135), let's go"               (ACK)
          └─────────────────── connection is now ESTABLISHED (see Topic 08) ────────────
```

The `ack 2842` in packet 2 = client's seq (2841) + 1. The `ack 7135` in packet 3 = server's seq (7134) + 1. That "+1" is TCP counting the SYN as one byte. After packet 3 you'd see `[P.]` segments carrying the encrypted TLS handshake — you can watch them arrive but not read them.

**Save it for later instead of printing:**
```bash
sudo tcpdump -i any -nn -c 50 'tcp and host 93.184.216.34' -w handshake.pcap
# Read it back anytime, apply new filters without re-capturing:
tcpdump -nn -r handshake.pcap 'tcp[tcpflags] & tcp-syn != 0'   # show only SYN-bearing packets
```

---

## Example 2 — Production Scenario

**The situation:** `orders-service` calls `payments-service` over HTTP on port 8080. A few times an hour, `orders-service` logs `ECONNRESET — socket hang up`. It's intermittent, there's no pattern in the payloads, and both teams blame each other. Time to capture on **both ends simultaneously** and let the packets settle it.

**Step 1 — capture on both hosts, filtered tight, writing to disk.**

```bash
# On orders-service host (client side). payments IP = 10.0.4.30
sudo tcpdump -i any -nn -w /tmp/orders.pcap \
  'tcp port 8080 and host 10.0.4.30'

# On payments-service host (server side). orders IP = 10.0.3.15
sudo tcpdump -i any -nn -w /tmp/payments.pcap \
  'tcp port 8080 and host 10.0.3.15'

# Let them run until the error reproduces (watch the app logs), then Ctrl-C both.
```

Filtering to exactly the two IPs and the one port keeps the files small and the analysis clean — no unrelated traffic, no disk blowout.

**Step 2 — find the RST. Read only the reset packets first:**

```bash
# On the orders (client) capture:
tcpdump -nn -r /tmp/orders.pcap 'tcp[tcpflags] & tcp-rst != 0'
```

```
14:02:11.500000 IP 10.0.4.30.8080 > 10.0.3.15.51992: Flags [R.], seq 1, ack 349, win 0, length 0
                   └────── payments (server) is the SOURCE of the RST ──────┘
```

The RST's **source is `10.0.4.30.8080`** — the *payments* server sent it. So it's not the client aborting and not the network; payments actively reset the connection. Now find out *why*.

**Step 3 — reconstruct the full flow around that reset** (drop the RST filter, look at the whole conversation just before it):

```bash
tcpdump -nn -r /tmp/orders.pcap -ttt 'host 10.0.4.30 and port 8080'
```

```
 00:00:00.000000 IP 10.0.3.15.51992 > 10.0.4.30.8080: Flags [S],  seq 8001, length 0
 00:00:00.000420 IP 10.0.4.30.8080 > 10.0.3.15.51992: Flags [S.], seq 9001, ack 8002, length 0
 00:00:00.000090 IP 10.0.3.15.51992 > 10.0.4.30.8080: Flags [.],  ack 9002, length 0     ← handshake OK
 00:00:00.000300 IP 10.0.3.15.51992 > 10.0.4.30.8080: Flags [P.], seq 1:349, ack 1, length 348  ← client sends POST /charge
 00:05:00.400000 IP 10.0.4.30.8080 > 10.0.3.15.51992: Flags [R.], seq 1, ack 349, win 0, length 0  ← RST 5 MINUTES later
     └──────────── note the timestamp jump: exactly 5m00s after the request ───────────┘
```

**The tell:** the handshake succeeded, the client sent its request, the server **ACKed nothing of a response**, then reset **exactly 300 seconds later**. That is not a firewall (a firewall drop gives you *silence* / retransmits, not a polite RST) and not the client. It's the server's own socket timeout — payments has a **5-minute idle/keep-alive timeout** and tears down connections the client is still holding in its pool.

**Step 4 — confirm on the server capture.** The payments capture shows the identical RST leaving `10.0.4.30.8080`, and no application-level response was ever emitted — the connection was killed by the runtime, not by the handler. Cross-checking both captures rules out "the network ate the response."

**Diagnosis (from packets, not guesses):**

| Symptom in the trace | Rules out | Points to |
|----------------------|-----------|-----------|
| RST source = server I:port | Client abort, network loss | Server-side close |
| RST exactly 300s after last byte | Random glitch | A **timeout** (keep-alive idle) |
| Clean `[R.]`, not silence/retransmits | Firewall/blackhole | Deliberate socket reset by the app/runtime |

**The fix:** the client's HTTP connection pool holds connections open longer than the server's keep-alive timeout, so the server reaps an idle socket right as the client reuses it. Align them: set the **client's** idle timeout *below* the **server's** keep-alive timeout (e.g., client 30s, server 60s), or enable retry-on-idempotent-reset. This is a classic pool-vs-server timeout mismatch — invisible in app logs, obvious in the capture.

---

## Common Mistakes

### Mistake 1: Confusing capture (BPF) filters with Wireshark display filters
```
# WRONG — display-filter syntax handed to tcpdump:
sudo tcpdump -i any 'tcp.port == 443'
tcpdump: syntax error

# WRONG — BPF syntax typed into Wireshark's display bar → turns red, won't apply:
tcp port 443
```
**They are two different languages for two different moments.** BPF (`tcp port 443`, `host 1.2.3.4`) filters *at capture time*, in the kernel. Wireshark display filters (`tcp.port==443`, `ip.addr==1.2.3.4`) filter *already-captured* data in the app. tshark uses **both**: `-f` takes a capture filter, `-Y` takes a display filter.

---

### Mistake 2: Forgetting -n, so reverse DNS pollutes and slows the capture
```bash
# Without -n, tcpdump does a reverse-DNS lookup for EVERY IP it prints.
# Those lookups generate their OWN packets — which your capture then shows —
# and block output while they resolve. On a busy host it's a feedback mess.
sudo tcpdump -i any port 443          # slow, noisy, self-polluting

sudo tcpdump -i any -nn port 443      # fast, clean, IPs and ports stay numeric
```
**Rule:** basically always use `-n` (or `-nn`). You want raw addresses when debugging anyway.

---

### Mistake 3: Capturing on the wrong interface
```bash
# Traffic to 127.0.0.1 / localhost NEVER touches eth0 — it uses loopback.
sudo tcpdump -i eth0 'port 5432'      # ← sees NOTHING for a local Postgres
sudo tcpdump -i lo   'port 5432'      # ← correct for localhost traffic

# Not sure which interface? List them, or capture everything first:
ip -brief address        # Linux: show interfaces and their IPs
sudo tcpdump -i any -nn 'port 5432'   # -i any spans all interfaces (Linux)
```
Inside containers, in a VPN, or behind a bridge, the interface name is rarely the obvious one. `-i any` is the safe first move; narrow down once you see where the packets actually are.

---

### Mistake 4: Expecting to read TLS-encrypted payloads
```bash
sudo tcpdump -i any -nn -A 'tcp port 443 and host api.example.com'
# You'll see the ClientHello (with the SNI hostname in cleartext),
# ServerHello, cipher negotiation... then GARBAGE:
# ...^Q^X..9..k...z...
# The application data is ENCRYPTED. -A shows you random bytes, not JSON.
```
**To see plaintext you must capture the session keys.** Point curl/Chrome/Node at a key-log file and load it into Wireshark:
```bash
SSLKEYLOGFILE=/tmp/keys.log curl https://api.example.com/orders
# Wireshark → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename → /tmp/keys.log
# Now the HTTP inside TLS is fully decrypted in the GUI.
```
Or debug the app over plain HTTP on `localhost` before TLS is added. But raw `-A` on port 443 will *never* show request bodies.

---

### Mistake 5: A huge unfiltered capture that fills the disk
```bash
# On a busy prod server this writes gigabytes per minute and can fill /:
sudo tcpdump -i any -w everything.pcap        # DANGER, no filter, no limit

# Always bound it: tight filter + size/time/count limits + rotation.
sudo tcpdump -i any -nn 'host 10.0.4.30 and port 8080' \
  -C 100 -W 5 -w cap.pcap         # rotate at 100MB, keep 5 files (500MB cap)
sudo tcpdump -i any -nn -c 5000 'port 8080' -w cap.pcap   # stop after 5000 packets
```
`-C <MB>` rotates files by size, `-W <n>` keeps only the last n, `-c <count>` stops after N packets. Never run a naked `-w` on production.

---

### Mistake 6: Not using -w, so you can only analyze what scrolled past
Printing to the terminal is fine for a quick look, but you can't re-filter it, can't open it in Wireshark, and can't share it. **Capture to `-w file.pcap` whenever the problem is intermittent or you'll want a second opinion.** You can always `-r` the file and apply new filters as many times as you like without reproducing the bug.

---

## Hands-On Proof

```bash
# 1. Confirm you need privileges — this fails without sudo:
tcpdump -i any -c 1                 # likely: "You don't have permission to capture"
sudo tcpdump -i any -c 1            # works

# 2. List your interfaces so you capture on the right one:
sudo tcpdump -D                     # numbered list of capturable interfaces

# 3. Watch a live handshake to a real site (numeric, first 8 packets):
sudo tcpdump -i any -nn -c 8 'tcp and host 1.1.1.1' &
curl -sI https://1.1.1.1 > /dev/null
# Look for [S], [S.], [.] in the first three lines.

# 4. Show ONLY connection-opening SYNs (the tcpflags trick):
sudo tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'
# This matches SYN-without-ACK = brand-new outbound connection attempts.

# 5. Catch resets in real time (great for "connection reset" hunts):
sudo tcpdump -i any -nn 'tcp[tcpflags] & tcp-rst != 0'

# 6. Prove TLS is opaque: watch plaintext HTTP vs encrypted HTTPS
sudo tcpdump -i any -nn -A -c 20 'tcp port 80 and host neverssl.com' &
curl -s http://neverssl.com > /dev/null     # you'll SEE "GET / HTTP/1.1" in cleartext
# now the same on 443 shows encrypted bytes instead.

# 7. Save, then re-analyze without re-capturing:
sudo tcpdump -i any -nn -c 100 'port 443' -w /tmp/sample.pcap
tcpdump -nn -r /tmp/sample.pcap 'tcp[tcpflags] & tcp-rst != 0'   # RSTs only, from the saved file
tcpdump -nn -r /tmp/sample.pcap -A                              # ASCII view of the same file

# 8. If you have Wireshark installed, open the file for the full GUI:
wireshark /tmp/sample.pcap
# Right-click any packet → Follow → TCP Stream to see the whole conversation.
```

---

## Practice Exercises

### Exercise 1 — Easy: Capture and label a handshake
```bash
# Capture the first 3 TCP packets to github.com's IP with no name resolution.
# 1. Resolve the IP first:
dig +short github.com | head -1
# 2. Capture (replace the IP), while curling in another terminal:
sudo tcpdump -i any -nn -c 3 'tcp and host <IP>'
curl -sI https://github.com > /dev/null

# Answer from your output:
#   a. Which line is [S], which is [S.], which is [.]?
#   b. What is the RTT (gap between the [S] and the [S.] timestamps)?
#   c. In the [S.] line, how does 'ack' relate to the 'seq' from the [S] line?
```

### Exercise 2 — Medium: Distinguish "closed port" from "filtered port"
```bash
# Capture on a port where NOTHING is listening (e.g. 9999 on localhost),
# then on a port a firewall silently drops (pick a blocked host:port you know).
sudo tcpdump -i lo -nn 'port 9999' &
curl -s --max-time 3 http://127.0.0.1:9999 ; echo "exit: $?"

# Questions:
#   a. For the closed local port, what flag comes back immediately? (hint: [R.])
#      How does that map to the app error "connection refused" (ECONNREFUSED)?
#   b. For the firewalled host, what do you see instead — a RST, or SYNs that
#      just repeat with no answer? How does THAT map to "connection timed out"?
#   c. Write the one-sentence rule: RST vs silence → which error, which cause?
```

### Exercise 3 — Hard: Find where the packets die in a saved capture
```bash
# Capture ~30s of traffic to any API you call, writing to a file:
sudo tcpdump -i any -nn -w /tmp/api.pcap 'host <api-ip> and tcp'
# ... generate some load with curl in a loop, then Ctrl-C ...

# Now, using ONLY -r on the file, answer with a single filter each:
#   a. Are there any retransmissions? (hint: hard to see in tcpdump directly —
#      open in Wireshark and use display filter 'tcp.analysis.retransmission')
#   b. Any RSTs? Which side sent them?  'tcp[tcpflags] & tcp-rst != 0'
#   c. Any zero-window advertisements (receiver stalled)?
#      In Wireshark: 'tcp.analysis.zero_window'. What would cause that in YOUR app?
#   d. Pick one full conversation and Follow TCP Stream in Wireshark.
#      Can you see the HTTP request? If it's port 443, WHY not — and what would
#      you need to decrypt it?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **Where in the stack does packet capture tap, and why does that mean you can see a TCP handshake but not an HTTPS request body?**

2. **You run `tcpdump 'tcp.port == 443'` and get a syntax error. What did you do wrong, and what's the correct BPF filter? Where *would* `tcp.port == 443` be valid?**

3. **A capture line reads `Flags [S.]`. What two TCP flags are set, which side sent it, and which packet of the handshake is it?**

4. **Your app logs "connection reset." In the capture, the RST's source is your app's own IP and port. What does that tell you about who closed the connection — and does it point at the network, the peer, or your own process?**

5. **You capture on port 443 with `-A` and see garbage instead of JSON. Why? Name the two ways to actually read the plaintext.**

6. **Explain the difference between a capture filter and a display filter in terms of *when* each runs and whether it's reversible.** Why does that make capture filters cheaper but riskier to get wrong?

7. **A SYN goes out and nothing comes back (repeated SYNs, no SYN-ACK). Contrast that with a SYN that gets an immediate RST. What real-world cause does each point to, and which app error (`ETIMEDOUT` vs `ECONNREFUSED`) matches each?**

---

## Quick Reference Card

**tcpdump flags**

| Flag | Meaning |
|------|---------|
| `-i <iface>` / `-i any` | Interface to capture on (`any` = all, Linux) |
| `-n` / `-nn` | No DNS resolution / also no port-name resolution |
| `-c <count>` | Stop after N packets |
| `-w <file>` / `-r <file>` | Write raw to `.pcap` / read back a `.pcap` |
| `-v` / `-vv` / `-vvv` | Increasing verbosity |
| `-A` / `-X` | Payload as ASCII / hex+ASCII |
| `-s <snaplen>` | Bytes per packet (0 = whole packet) |
| `-e` / `-S` | Show MAC header / absolute seq numbers |
| `-C <MB>` / `-W <n>` | Rotate file at size / keep only n files |

**BPF capture filters (tcpdump)**

| Filter | Matches |
|--------|---------|
| `host 1.2.3.4` | Src OR dst is that IP |
| `src host X` / `dst host X` | Only source / only destination |
| `port 443` / `portrange 8000-8100` | That port / a range |
| `tcp` `udp` `icmp` | Protocol |
| `net 10.0.0.0/8` | Any IP in that subnet |
| `A and B`, `A or B`, `not A` | Boolean combinations |
| `tcp[tcpflags] & tcp-syn != 0` | Any packet with SYN set |
| `tcp[tcpflags] == tcp-syn` | SYN and *only* SYN (new conn attempt) |
| `tcp[tcpflags] & tcp-rst != 0` | Resets |

**Wireshark display filters (post-capture — different syntax!)**

| Filter | Matches |
|--------|---------|
| `ip.addr == 1.2.3.4` | Src or dst IP |
| `tcp.port == 443` | Either port is 443 |
| `tcp.flags.reset == 1` | RST packets |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | New connection attempts |
| `http.request` / `http.response.code == 500` | HTTP requests / 500s |
| `tcp.analysis.retransmission` | Retransmitted segments (loss) |
| `tcp.analysis.duplicate_ack` | Duplicate ACKs (loss/reorder) |
| `tcp.analysis.zero_window` | Receiver's buffer full (stalled) |

**Reading a line:** `time  src.port > dst.port: Flags [X], seq …, ack …, win …, length …`
Flags: `[S]`=SYN `[S.]`=SYN-ACK `[.]`=ACK `[P.]`=PSH-ACK `[F.]`=FIN-ACK `[R]`/`[R.]`=RST (`.` = ACK)

**Where packets die — the diagnostic table**

| What you see in the capture | What it means |
|-----------------------------|---------------|
| SYN out, **no** SYN-ACK (SYNs repeat) | Firewall dropping, or host down → `ETIMEDOUT` |
| SYN out, immediate **RST** back | Nothing listening on that port → `ECONNREFUSED` |
| **Retransmissions** | Packet loss on the path |
| **Duplicate ACKs** (3+) | Loss/reordering; triggers fast retransmit |
| **Zero window** advertised | Receiver's buffer is full — app not reading fast enough |
| **RST mid-connection** | One side killed it (app close, timeout, proxy) — check the source IP |

**Copy-paste cheat block**
```bash
# Quick live look, numeric, one interface, 20 packets:
sudo tcpdump -i any -nn -c 20 'tcp port 443 and host 1.2.3.4'

# Save for Wireshark, bounded so it can't fill disk:
sudo tcpdump -i any -nn 'host 10.0.4.30 and port 8080' -c 5000 -w cap.pcap

# Hunt resets and new-connection SYNs:
sudo tcpdump -i any -nn 'tcp[tcpflags] & tcp-rst != 0'
sudo tcpdump -i any -nn 'tcp[tcpflags] == tcp-syn'

# Re-analyze a saved file without recapturing:
tcpdump -nn -r cap.pcap 'tcp[tcpflags] & tcp-rst != 0'
wireshark cap.pcap    # Follow TCP Stream, expert info, I/O graphs
```

---

## When Would I Use This at Work?

### Scenario 1: "Connection reset by peer" from a downstream service
You capture on both hosts with a two-IP filter, wait for it to reproduce, and filter for `tcp[tcpflags] & tcp-rst != 0`. The RST's **source IP:port** tells you instantly who killed the connection, and the timestamp gap before it (a clean 30s/60s/300s) usually reveals a keep-alive/idle timeout mismatch between your connection pool and the server — exactly the Example 2 bug.

### Scenario 2: "The firewall team swears the port is open"
Run `tcpdump 'tcp[tcpflags] == tcp-syn'` on your host and try to connect. If you see your SYNs leaving but never a SYN-ACK, packets are being dropped somewhere on the path — silence, not a RST, is the firewall signature. Now you have proof, not opinions, to take to the network team. (Pair with `24-netstat-and-ss.md` to confirm the local socket state and `25-ping-traceroute-mtr.md` to locate *where* on the path they die.)

### Scenario 3: "The API is randomly slow and APM shows nothing"
Capture during a slow window, open in Wireshark, and apply `tcp.analysis.retransmission` and `tcp.analysis.zero_window`. Retransmissions mean packet loss (network/infra problem); zero-window means *your own app isn't reading from the socket fast enough* (a full receive buffer — often a blocked event loop or a slow consumer). This is the packet-level half of `27-debugging-a-slow-api.md`'s methodology.

### Scenario 4: "It works from curl but not from the app"
Capture both requests and diff them: different TLS version in the `ClientHello`, a missing or wrong SNI, IPv6 vs IPv4, an absent `Host`/auth header (visible if it's plaintext, or via `SSLKEYLOGFILE` decryption if TLS). The wire never lies about what your client actually sent.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the packet's whole journey, so you know what you're looking at
- `08-tcp-in-depth.md` — handshake, states, seq/ack, teardown — every flag you'll read here maps back to it

**Study alongside / next:**
- `24-netstat-and-ss.md` — connection *state* (LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT) — the static view that complements tcpdump's packet-by-packet view
- `25-ping-traceroute-mtr.md` — once a capture shows packets dying, find *where on the path* they die
- `27-debugging-a-slow-api.md` — the full slow-API methodology; tcpdump is its ground-truth instrument
- `09-tls-ssl-in-depth.md` — understand exactly which handshake fields are visible in the clear and why the body isn't

**This is the ground-truth tool.** When logs, metrics, and traces all disagree, you capture the packets — because the packets are what actually happened.

---

*Doc saved: `/docs/networking/23-tcpdump-and-wireshark.md`*
