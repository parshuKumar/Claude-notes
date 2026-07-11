# 25 — ping, traceroute, mtr

> **Phase 4 — Topic 5 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `04-ip-addresses.md`

---

## ELI5 — The Simple Analogy

Imagine you mail a letter to a friend across the country and it arrives late. You want to know **why**.

- **ping** is you shouting "You there?!" across a canyon and timing the echo. If the echo comes back in half a second, your friend is close and awake. If it comes back slowly, or not at all, something is wrong — but you don't yet know *where*.

- **traceroute** is a clever trick to find *which road* the letter takes. You send a stack of letters. The first is addressed with instructions: "Postal worker #1, open me and mail me back a note saying where you are." The second says "Postal worker #2, do the same." And so on. Each postal worker along the route reports their location, so you build a map of every stop between you and your friend.

- **mtr** is standing at the canyon edge sending those location-request letters **over and over, forever**, keeping a live scoreboard: which stop drops the most letters, which stop is slowest, and whether the problem is getting worse or better.

That's the whole toolkit: ping tells you *if* and *how fast*, traceroute tells you *the path*, and mtr watches the path *continuously* to catch the exact stop where things fall apart.

---

## Where This Lives in the Network Stack

All three tools are built on **ICMP** (Internet Control Message Protocol), which lives at **Layer 3 — the Network layer**, right alongside IP. ICMP is not TCP and not UDP — it's a sibling protocol that rides directly inside IP packets.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, DNS, your app)                   │
│  Layer 4 — Transport     (TCP, UDP)          ◄─ traceroute -T/-U │
│  Layer 3 — Network       (IP + ICMP)         ◄─ ping, traceroute │  ← YOU ARE HERE
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                       │
│  Layer 1 — Physical      (fiber, copper, radio)                  │
└─────────────────────────────────────────────────────────────────┘
```

Key fact that trips people up: **ICMP is a separate protocol from TCP and UDP.** In the IP header's "Protocol" field (see Topic 01's packet anatomy), you'll see:

```
Protocol = 1   → ICMP   (ping, traceroute time-exceeded messages)
Protocol = 6   → TCP    (your API, SSH, HTTPS)
Protocol = 17  → UDP    (DNS, classic traceroute probes)
```

Because ICMP is its own protocol, a firewall can **block ICMP while allowing TCP**. That single fact is the source of most ping/traceroute confusion — a host can be perfectly healthy and serving traffic on port 443 while dropping every ping you send.

---

## What Is This?

Three tools that measure the **network path** between you and a remote host:

| Tool | One-line job | Underlying mechanism |
|------|-------------|----------------------|
| `ping` | Is the host reachable, and what's the round-trip time? | ICMP Echo Request → Echo Reply |
| `traceroute` | What is every router (hop) on the path, and its latency? | Packets with increasing TTL → ICMP Time Exceeded |
| `mtr` | Continuous, live per-hop loss% and latency | ping + traceroute, looped forever |

They all answer questions about **the network itself**, not about your application. They tell you whether packets can travel, how long they take, and where they get lost or delayed on the way. They do **not** tell you whether your HTTP server returned a 500, whether your TLS cert is valid, or whether your database is slow — those are higher-layer problems that need `curl` (Topic 21) and friends.

Think of them as the **plumbing inspection tools**. They check the pipes, not the water quality.

---

## Why Does It Matter for a Backend Developer?

You will reach for these constantly, and misreading them causes real outages to go undiagnosed:

- **"Users in one region report timeouts."** mtr from an affected location shows you *the exact hop* where loss begins — often an ISP peering link you don't own. This turns "the site is broken" into "there's congestion at hop 8, our upstream provider's border router" — an actionable ticket.
- **"Is the server down or is the network broken?"** ping and traceroute separate *host down* from *path broken* from *firewall dropping ICMP*. These need completely different fixes.
- **Latency budgeting.** ping gives you the raw RTT floor. If ping to your database is 40ms, no amount of query tuning gets a single round-trip below 40ms. (See Topic 33.)
- **Distinguishing physics from problems.** A traceroute hop that jumps to 150ms and *stays there through to the destination* is a real path problem. A middle hop that spikes to 150ms then drops back to 20ms at the next hop is almost always harmless ICMP deprioritization — and knowing the difference stops you from chasing ghosts.
- **Choosing the right probe.** Your service lives on TCP port 443. Testing it with an ICMP ping can lie to you in both directions. Knowing to use `traceroute -T -p 443` or `curl` instead is the difference between a real test and a fake one.

The single most valuable skill here is **not being fooled**: knowing that "ping fails" does not mean "host down," and that a scary middle hop is usually a red herring.

---

## The Packet/Protocol Anatomy

Everything here is ICMP. An ICMP message is tiny — it rides directly inside an IP packet with no TCP/UDP layer between them:

```
┌────────────────────────────────────────────────────────┐
│ IP HEADER (Layer 3)                                     │
│   Source IP, Destination IP, Protocol = 1 (ICMP), TTL   │
│  ┌───────────────────────────────────────────────────┐  │
│  │ ICMP MESSAGE                                       │  │
│  │   Type (1 byte)  │ Code (1 byte) │ Checksum        │  │
│  │   Identifier     │ Sequence Number                 │  │
│  │   Payload (timestamp + padding data)               │  │
│  └───────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### The ICMP message types you must know

```
┌──────┬──────┬─────────────────────────┬──────────────────────────────┐
│ Type │ Code │ Name                    │ Who sends it / when          │
├──────┼──────┼─────────────────────────┼──────────────────────────────┤
│  8   │  0   │ Echo Request            │ ping sends this OUT          │
│  0   │  0   │ Echo Reply              │ target sends this BACK       │
│ 11   │  0   │ Time Exceeded (TTL=0)   │ a router when TTL hits zero  │  ← traceroute's engine
│  3   │  x   │ Destination Unreachable │ router/host: can't deliver   │
│      │  3   │   └─ Port Unreachable   │ host: nothing on that UDP port│  ← traceroute's "arrived" signal
│      │  1   │   └─ Host Unreachable   │ no route to host             │
│      │  0   │   └─ Network Unreachable│ no route to network          │
└──────┴──────┴─────────────────────────┴──────────────────────────────┘
```

**How ping uses ICMP:**
```
YOU  ──── ICMP Type 8 (Echo Request) ────►  TARGET
YOU  ◄─── ICMP Type 0 (Echo Reply)    ────  TARGET
        (RTT = time between send and reply)
```

**How traceroute uses ICMP:** it does NOT primarily use Echo Requests (on Linux/macOS the default probe is a UDP packet). Instead it weaponizes the **TTL field** to make each router in turn confess its identity via a **Time Exceeded (Type 11)** message. When the probe finally reaches the destination, the destination replies differently:
- Classic UDP traceroute → destination sends **Port Unreachable (Type 3, Code 3)** because nothing listens on the weird high UDP port. That's the "we arrived" signal.
- `traceroute -I` (ICMP) → destination sends **Echo Reply (Type 0)**.
- `traceroute -T` (TCP) → destination sends **SYN-ACK** (open port) or **RST** (closed port).

**Because ICMP has no ports**, firewalls treat it as a single on/off category. Many networks rate-limit ICMP (a router will happily forward your TCP traffic at 10 Gbps but only generate, say, 100 ICMP errors per second to protect its own CPU). This rate-limiting is *the* reason middle hops in traceroute show inflated latency — more on that below.

---

## How It Works — Step by Step

### ping — the simple part

```
1. ping picks an Identifier (usually the process ID) and starts a sequence counter at 1.
2. It builds an ICMP Echo Request (Type 8), stamps the current time into the payload,
   and hands it to the OS to send to the target IP.
3. The target's OS receives it, flips it into an Echo Reply (Type 0), copies the
   payload back verbatim, and returns it.
4. ping receives the reply, matches it by Identifier + Sequence, and computes:
        RTT = now  -  timestamp-in-payload
5. It prints one line, waits ~1 second (the interval), increments the sequence, repeats.
6. On Ctrl-C it prints a summary: packets sent/received, loss %, and min/avg/max/mdev.
```

### traceroute — the TTL trick (the important part)

This is the clever bit, and it ties directly back to Topic 01's TTL section: *"TTL starts at 64 or 128, is decremented by 1 at every router, and when it hits 0 the packet is discarded and an ICMP Time Exceeded is sent back."* traceroute **abuses** that rule on purpose.

```
Goal: discover every router between YOU and the DESTINATION.

Trick: send probes with a deliberately tiny TTL, incrementing by 1 each round.
       Each probe "dies" exactly one hop further along, and the router that
       kills it is forced to send back an ICMP Time Exceeded — revealing itself.

┌─────┐   TTL=1    ┌────────┐          ┌────────┐          ┌────────┐        ┌──────────┐
│ YOU │──────────► │ Router │          │ Router │          │ Router │        │   DEST   │
└─────┘            │   R1   │          │   R2   │          │   R3   │        │          │
   ▲               └────────┘          └────────┘          └────────┘        └──────────┘
   │  TTL hits 0 at R1 ──► R1 sends back ICMP Time Exceeded (Type 11)
   └───────────────────────┘   "Hi, I'm R1, at 10.0.0.1, took 1.2ms"

┌─────┐   TTL=2    ┌────────┐  TTL=1   ┌────────┐
│ YOU │──────────► │   R1   │────────► │   R2   │  TTL hits 0 here
└─────┘            └────────┘          └────────┘──► R2 sends ICMP Time Exceeded
   ▲                                                 "Hi, I'm R2, 12ms"
   └─────────────────────────────────────────────────┘

┌─────┐   TTL=3    ┌────┐  TTL=2  ┌────┐  TTL=1  ┌────┐
│ YOU │──────────► │ R1 │───────► │ R2 │───────► │ R3 │  dies here ──► "I'm R3, 25ms"
└─────┘            └────┘         └────┘         └────┘

┌─────┐   TTL=4    ... probe finally reaches DEST ...
│ YOU │──────────►  DEST is reached. TTL never hits 0 mid-path.
└─────┘             DEST responds with Port Unreachable (UDP) / Echo Reply (-I) / SYN-ACK (-T)
   ▲                            = "you've arrived, trace complete"
   └─────────────────────────────────────────────────┘
```

Step by step:

```
1. traceroute sets TTL = 1, sends 3 probe packets to the destination.
2. The FIRST router decrements TTL to 0, drops the packet, and sends back
   ICMP Time Exceeded. traceroute records that router's IP + the RTT for each
   of the 3 probes. That's hop line #1.
3. traceroute sets TTL = 2. The first router passes it (TTL→1), the SECOND
   router drops it and reports. That's hop line #2.
4. This repeats, TTL = 3, 4, 5..., each round revealing one more router.
5. Eventually TTL is large enough that the probe reaches the DESTINATION.
   The destination does NOT send Time Exceeded — it sends the "arrived" signal
   (Port Unreachable / Echo Reply / SYN-ACK depending on mode). traceroute
   sees this, prints the final hop, and stops.
6. Each hop shows 3 latency numbers because it sends 3 probes per TTL value.
   A "*" means that probe got no reply within the timeout.
```

**Why 3 probes per hop?** Redundancy and a rough sense of jitter. If one shows `*` and two show times, the router is just rate-limiting some ICMP — not a real problem.

**Why later hops sometimes show *lower* latency than earlier hops** — because the numbers are independent measurements of *different routers* deprioritizing ICMP by different amounts, not a cumulative total. (Critical subtlety below.)

### mtr — traceroute in a loop, with statistics

```
1. mtr runs a traceroute to map the path (hop 1, hop 2, ... hop N).
2. Then it continuously sends probes to EVERY hop at once, round after round.
3. After each round it updates a live table: for each hop it tracks
   packets Sent, packets lost (Loss%), and Last/Avg/Best/Worst/StdDev latency.
4. It refreshes the screen ~once per second. You watch loss% climb or hold steady.
5. --report mode does this non-interactively for a fixed number of cycles,
   then prints a single clean snapshot (perfect for pasting into a ticket).
```

mtr's superpower: because it pings *every hop continuously*, you can watch loss% **appear at a specific hop and persist to the end** — that persistence is the fingerprint of a real problem.

---

## Exact Syntax Breakdown

### ping
```
ping  -c 5   -i 0.2   -s 1400   -W 2    example.com
│     │      │        │         │       │
│     │      │        │         │       └── target hostname or IP
│     │      │        │         └── -W 2  : wait max 2 seconds for each reply
│     │      │        └── -s 1400         : payload size = 1400 bytes (test MTU/fragmentation)
│     │      └── -i 0.2                   : interval 0.2s between pings (default 1s; <1s needs root on Linux)
│     └── -c 5                            : send exactly 5 packets then stop (default: forever)
└── send ICMP Echo Requests
```
Other flags worth knowing: `-n` (numeric, skip reverse-DNS on the target), `-D` (print timestamps, Linux), `-t 1` (set outgoing TTL, macOS/BSD), `-M do` (set Don't-Fragment bit, for MTU discovery, Linux).

> **Note:** `-W` is *seconds* on Linux and *milliseconds* on macOS. Flag meanings differ between GNU ping (Linux) and BSD ping (macOS) — always sanity-check with `man ping` on the box you're actually on.

### traceroute
```
traceroute  -T   -p 443   -n   -q 1   -m 20    example.com
│           │    │        │    │      │        │
│           │    │        │    │      │        └── target
│           │    │        │    │      └── -m 20 : max 20 hops (default 30/64)
│           │    │        │    └── -q 1         : 1 probe per hop instead of 3 (faster, cleaner)
│           │    │        └── -n                : numeric — do NOT reverse-DNS each hop (much faster)
│           │    └── -p 443                     : probe port 443 (with -T = TCP SYN to HTTPS)
│           └── -T                              : use TCP SYN probes (needs sudo)
└── trace the path
```
The probe-type flags are the ones that matter most:

| Flag | Probe type | When to use | Notes |
|------|-----------|-------------|-------|
| *(none)* | UDP high ports | default on Linux/macOS | often blocked; arrival = Port Unreachable |
| `-I` | ICMP Echo | mimics `ping` path | blocked wherever ICMP is filtered |
| `-T` | TCP SYN | **firewalled hosts, real services** | `sudo`; `-p 443` to test HTTPS reachability |
| `-U` | UDP to a fixed port | test a specific UDP service | |

`tcptraceroute example.com 443` is a dedicated tool equivalent to `sudo traceroute -T -p 443`.

**Windows:** `tracert example.com` — uses **ICMP Echo** by default (like `traceroute -I`), and `-d` is the numeric flag (equivalent to `-n`).

### mtr
```
mtr  --report   -c 100   -w   -b    example.com
│    │          │        │    │     │
│    │          │        │    │     └── target
│    │          │        │    └── -b : show both hostname AND IP for each hop
│    │          │        └── -w      : wide report (don't truncate long hostnames)
│    │          └── -c 100           : send 100 cycles of probes, then stop
│    └── --report                    : non-interactive; run cycles, print snapshot, exit
└── continuous ping + traceroute
```
- Interactive mode (no `--report`): `mtr example.com` opens a live, auto-refreshing table. Press `q` to quit.
- `--report-wide` = `--report` + `-w`. `-n` skips DNS. `-T` / `-P 443` makes mtr use TCP probes (same firewall benefit as traceroute -T). `-u` uses UDP.

**Canonical commands to memorize:**
```bash
mtr --report -c 100 example.com          # 100-cycle path health snapshot for a ticket
sudo traceroute -T -p 443 example.com    # trace to a real HTTPS service through firewalls
```
`mtr --report -c 100` means: probe all hops for 100 rounds (~100 seconds at the default 1s interval), then print one table where `Snt` will read 100 and `Loss%` is trustworthy because it's averaged over 100 samples — not a noisy single shot.

---

## Example 1 — Basic

### ping a well-known host and read every field

```bash
ping -c 4 1.1.1.1
```

Output (annotated):
```
PING 1.1.1.1 (1.1.1.1): 56 data bytes
64 bytes from 1.1.1.1: icmp_seq=0 ttl=57 time=12.3 ms
64 bytes from 1.1.1.1: icmp_seq=1 ttl=57 time=11.8 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=57 time=45.1 ms   ← one slow reply = jitter
64 bytes from 1.1.1.1: icmp_seq=3 ttl=57 time=12.0 ms

--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 11.8/20.3/45.1/14.2 ms
```

Reading it field by field:
```
64 bytes         → size of the reply (56 data + 8 ICMP header). Not important day-to-day.
icmp_seq=0..3    → sequence number. GAPS here = packet loss (a missing seq = a dropped packet).
ttl=57           → the TTL REMAINING in the reply. See the TTL detective trick below.
time=12.3 ms     → the round-trip time (RTT) for THIS packet. This is the number you care about.

0.0% packet loss → the headline reliability number.
min/avg/max      → 11.8 / 20.3 / 45.1 — a big gap between min and max = jitter (unstable latency).
stddev (mdev)    → 14.2 ms — how "spread out" the times are. High = jittery. Low = rock steady.
```

**The TTL detective trick.** The reply shows `ttl=57`. Common OS starting TTLs are **64 (Linux/macOS)**, **128 (Windows)**, and **255 (many routers/network gear)**. The reply lost some TTL crossing the network back to you:
```
Reply TTL = 57  →  nearest starting value above it is 64  →  64 − 57 = ~7 hops away,
                   and the sender is probably a Linux/Unix box.
Reply TTL = 117 →  nearest above is 128  →  128 − 117 = ~11 hops, probably Windows.
Reply TTL = 250 →  nearest above is 255  →  ~5 hops, probably a router/appliance.
```
It's a heuristic, not proof (paths can be long), but it's a fast gut-check for OS and distance.

> **Note on jitter:** the third packet at 45.1ms while the others sit at ~12ms is *jitter* — variance in latency. For a REST API a little jitter is invisible. For a voice/video/game backend, jitter is often worse than a higher-but-stable latency. `stddev`/`mdev` in the summary is your jitter number.

### traceroute the path

```bash
traceroute -n 1.1.1.1
```
```
traceroute to 1.1.1.1 (1.1.1.1), 64 hops max, 52 byte packets
 1  192.168.1.1     1.201 ms   0.998 ms   0.951 ms   ← your home router (private IP)
 2  100.64.0.1      9.882 ms   8.771 ms   9.104 ms   ← ISP CGNAT gateway
 3  10.15.32.1     10.443 ms  11.002 ms   9.998 ms   ← inside ISP network
 4  * * *                                            ← this hop won't answer ICMP (normal!)
 5  203.0.113.9    12.115 ms  11.887 ms  12.334 ms   ← ISP border / peering
 6  1.1.1.1        12.008 ms  11.945 ms  12.101 ms   ← DESTINATION reached
```
```
Column layout per hop:  <hop#>  <router IP>   <probe1 RTT>  <probe2 RTT>  <probe3 RTT>

Hop 4 shows "* * *": that router is configured to NOT send ICMP Time Exceeded
(or rate-limits it away). This is EXTREMELY common and does NOT mean packets are
lost — notice the trace continues cleanly to hop 5 and reaches the destination.
```

---

## Example 2 — Production Scenario

### "Users in Mumbai report intermittent timeouts to our API. New York users are fine."

Your API runs in AWS `us-east-1` (Virginia). New York users are happy. Mumbai users get sporadic timeouts and 3–8 second stalls. Your server logs look clean — CPU low, DB queries 4ms, no errors. Is it your service? The network? Their ISP?

**Step 1 — ping from an affected location to establish loss & baseline.**

SSH into a box in Mumbai (or use a jump host / a colleague's machine there) and run a long ping:
```bash
ping -c 100 api.myapp.com
```
```
--- api.myapp.com ping statistics ---
100 packets transmitted, 89 packets received, 11.0% packet loss
round-trip min/avg/max/stddev = 210.4/268.9/612.7/71.3 ms
```
Two red flags: **11% packet loss** and a huge **max (612ms) vs min (210ms)** spread. 210ms average is expected physics (Mumbai↔Virginia is far — see Topic 33), but loss and that jitter are not. Loss + jitter → retransmissions and stalls → the timeouts users see. Now: **where** is the loss happening?

**Step 2 — mtr to localize the loss to a specific hop.**

```bash
mtr --report -c 200 api.myapp.com
```
```
Start: 2026-07-12T09:14:03+0530
HOST: mumbai-jump-01              Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 10.0.0.1                   0.0%   200    0.4   0.5   0.3   1.9   0.2
  2.|-- 172.16.4.1                 0.0%   200    2.1   2.4   1.9   9.8   0.6
  3.|-- isp-mum-border.net         0.0%   200    3.8   4.1   3.5  14.2   1.1
  4.|-- 115.110.x.x  (ISP core)    0.0%   200    4.9   5.2   4.6  22.0   1.4
  5.|-- peer-mum-sing.isp.net     13.5%   200  118.3 121.7 116.0 402.1  38.9   ◄── LOSS STARTS HERE
  6.|-- sing-border.tier1.net     13.0%   200  120.1 124.5 117.2 410.5  40.1
  7.|-- 63.243.x.x  (transatl.)   12.5%   200  198.7 205.3 195.0 588.0  55.2
  8.|-- ae-1.us-east.tier1.net    12.0%   200  260.4 268.0 258.1 620.0  62.4
  9.|-- aws-edge.us-east-1        11.5%   200  262.0 267.9 259.0 615.0  61.0
 10.|-- api.myapp.com             11.0%   200  263.5 269.2 260.0 612.7  71.3   ◄── loss PERSISTS to dest
```

**How to read this correctly — the make-or-break skill:**

```
Hops 1–4: Loss 0.0%. Latency single-digit ms. Your local network + ISP entry: healthy.

Hop 5: Loss JUMPS to 13.5% and latency jumps to ~120ms. This is the peering link
       between the Mumbai ISP and a Singapore Tier-1 carrier.

CRUCIAL: does the loss STAY, or does it recover at the next hop?
       → Hops 6, 7, 8, 9, 10 ALL show ~11–13% loss, all the way to the destination.
       → The loss STARTS at hop 5 and PERSISTS to the end.
       → THIS IS A REAL PROBLEM. Hop 5 (the peering link) is congested/dropping packets.
```

**Contrast — what a false alarm looks like:**
```
  6.|-- some-router.net           40.0%   200  118  125  110  300  ...   ◄── scary 40% loss!
  7.|-- next-router.net            0.0%   200  120  124  116  210  ...   ◄── back to 0%
  8.|-- api.myapp.com              0.0%   200  262  268  259  310  ...   ◄── 0% at destination
```
Here hop 6 screams 40% loss and high latency, but **hop 7 and the destination are 0%**. That means hop 6's router is simply **deprioritizing/rate-limiting the ICMP replies to itself** — it forwards the actual through-traffic fine (proven by the clean hops after it). **This is NOT a problem. Ignore hop 6.** Only loss/latency that *appears and persists through to the destination* is real.

**Step 3 — confirm it's the network path, not your service, with a TCP trace to the real port.**

ICMP could be lying either way, so probe the actual service port:
```bash
sudo traceroute -T -p 443 -n api.myapp.com
```
If the TCP SYN probes show the same latency/loss profile starting at hop 5, the path problem is real for *actual application traffic*, not just ICMP.

**Step 4 — decide the fix.** The loss is at hop 5, a **peering link between the Mumbai ISP and a Singapore carrier** — infrastructure you do not own. Options:
```
1. You can't fix their router. But you can ROUTE AROUND it:
   → Put the API behind a CDN / global anycast edge (Topic 30). Mumbai users then
     hit a nearby edge PoP over a healthier path; the congested peering link is bypassed.
2. Add an AWS region closer to India (ap-south-1, Mumbai) so traffic never crosses
     that Singapore↔US peering link at all. Biggest win for Indian users.
3. If you have an enterprise relationship: report the specific hop (with the mtr
     --report output pasted in) to your transit provider / the ISP. The mtr snapshot
     is exactly the evidence a NOC needs.
4. Short term: make clients more resilient — sane timeouts + retries with backoff —
     so 11% loss degrades gracefully instead of producing hard user-visible timeouts.
```

**The lesson:** mtr localized a fuzzy "the app is broken for Mumbai" into "packet loss originates at the ISP↔Singapore peering link (hop 5) and persists to the destination." That precise finding is what lets you choose CDN vs. new region vs. escalation — instead of blaming your (innocent) code.

---

## Common Mistakes

### Mistake 1: "ping fails, therefore the host is down"
```
$ ping api.myapp.com
PING api.myapp.com (203.0.113.9): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
...
--- api.myapp.com ping statistics ---
5 packets transmitted, 0 packets received, 100.0% packet loss
```
**Wrong conclusion:** "The server is down."
**Reality:** Loads of hosts (AWS EC2 by default, Google, Cloudflare origins, corporate firewalls) **drop ICMP Echo on purpose** while happily serving TCP. ICMP is a *separate protocol* (proto 1) and firewalls block it independently of TCP (proto 6).
**Prove the host is actually up by testing the real port:**
```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://api.myapp.com/health   # 200? it's UP.
nc -vz api.myapp.com 443                                                  # "succeeded" = port open
sudo traceroute -T -p 443 api.myapp.com                                   # TCP path reaches it
```
`ping fails` ≠ `host down`. It only ever means "no ICMP Echo Reply came back."

---

### Mistake 2: Panicking over one high-latency middle hop
```
 6  edge-router.isp.net   198 ms   201 ms   205 ms   ← "hop 6 is at 200ms, found the problem!"
 7  core.tier1.net         24 ms    23 ms    24 ms   ← wait... hop 7 is back to 24ms??
 8  api.myapp.com          26 ms    25 ms    26 ms
```
**Wrong conclusion:** "Hop 6 is the bottleneck, it's adding 200ms."
**Reality:** Each hop's number is that router's *own* time to generate an ICMP reply to **you**, not the cumulative latency of traffic passing through it. Router 6 was busy and slow to *reply to ICMP* (deprioritized control-plane work), but it *forwarded the real packets* fine — proven because hops 7 and 8 are fast. If hop 6 were truly adding 200ms, everything after it would be 200ms+ too. **A spike that recovers is noise.** Only latency that rises *and stays high through to the destination* is real.

---

### Mistake 3: Reading `*` as packet loss
```
 4  * * *
 5  203.0.113.9   12 ms   11 ms   12 ms
```
**Wrong conclusion:** "Hop 4 is dropping 100% of packets!"
**Reality:** `*` means *that probe got no ICMP reply within the timeout*. The overwhelmingly common cause is a router **configured not to send ICMP Time Exceeded** (or rate-limiting it). Since the trace **continues past hop 4 and reaches the destination**, no user traffic is being lost — that router just doesn't announce itself. Real end-to-end loss shows up as `*`s (or high Loss% in mtr) that **persist through to the final hop**, not a single silent middle hop.

---

### Mistake 4: Using ICMP ping to test a service that lives on a TCP port
```bash
ping db.internal          # "it pings, so the database is fine!"   ← NO
```
**Wrong conclusion:** A successful ping means your Postgres on 5432 is reachable and healthy.
**Reality:** ping tests Layer 3 reachability to the *host*, not whether a *service* is listening, accepting connections, or being blocked by a security group on its specific port. A host can ping perfectly while port 5432 is firewalled shut or the DB process is dead.
**Test the actual port/service:**
```bash
nc -vz db.internal 5432                       # TCP handshake to the real port
sudo traceroute -T -p 5432 db.internal        # TCP path to the real port
psql -h db.internal -c 'select 1'             # the real application-level check
```

---

### Mistake 5: Forgetting firewalls/rate-limits distort *all* ICMP numbers
```
Symptom: traceroute shows wildly bouncing latencies and lots of *,
         but curl to the service is fast and reliable.
```
**Wrong conclusion:** "The network is a mess."
**Reality:** Corporate firewalls, cloud security groups, and router control-plane rate-limits all mangle ICMP specifically. If your *actual* traffic (TCP `curl`, real requests) is healthy, believe the traffic — not the deprioritized ICMP diagnostics. Cross-check with `traceroute -T`/`mtr -T` on the real port, and trust end-to-end application metrics over any single ICMP hop.

---

## Hands-On Proof

Run these and watch the concepts become real:

```bash
# 1. ping and read TTL to guess the remote OS + hop distance
ping -c 4 1.1.1.1
#    Look at ttl=NN. Subtract from the nearest of 64/128/255 → approx hop count.

# 2. See a router that hides from traceroute (the "* * *" phenomenon)
traceroute -n 8.8.8.8
#    Almost every trace has at least one "* * *" hop that still reaches the destination.

# 3. Prove "ping fails ≠ host down": find a host that blocks ICMP but serves HTTPS
ping -c 3 github.com                      # may time out or be filtered
curl -sS -o /dev/null -w "HTTP %{http_code}\n" https://github.com   # yet HTTPS works fine

# 4. Watch mtr localize behavior live (Ctrl-C / q to quit)
mtr 8.8.8.8
#    Watch Loss% and Avg per hop update every second. Note that any loss on a middle
#    hop that does NOT persist to the last hop is harmless ICMP deprioritization.

# 5. Snapshot report you could paste into a ticket
mtr --report -c 50 -w 1.1.1.1

# 6. Compare ICMP path vs TCP-to-443 path (firewalls treat them differently)
traceroute -I 1.1.1.1                      # ICMP probes
sudo traceroute -T -p 443 1.1.1.1          # TCP SYN probes to HTTPS

# 7. Measure jitter: send 20 fast pings and read stddev/mdev in the summary
ping -c 20 -i 0.2 1.1.1.1                   # (sub-second interval may need sudo on Linux)

# 8. See TTL do its job: force a tiny TTL and watch a near router kill the packet
ping -c 1 -t 1 1.1.1.1                      # macOS: -t sets TTL; expect "Time to live exceeded"
#    Linux equivalent:  ping -c 1 -T tsonly ... is different; use:  traceroute -m 1 1.1.1.1
```

---

## Practice Exercises

### Exercise 1 — Easy: Read a ping summary
```bash
ping -c 10 1.1.1.1
```
From YOUR output, answer:
```
1. What is your packet loss %? Is it 0? If not, is the loss consistent or one-off?
2. What is the min/avg/max? How big is the gap between min and max (that's your jitter)?
3. Look at the ttl= value. Using the 64/128/255 rule, roughly how many hops away is
   the responder, and is it likely Linux, Windows, or a router?
4. Run it again. Did avg change much? Is the path stable minute-to-minute?
```

### Exercise 2 — Medium: Interpret a traceroute correctly
```bash
traceroute -n 8.8.8.8
```
Answer from your output:
```
1. How many total hops to reach 8.8.8.8?
2. Which hop shows the biggest latency JUMP? (Usually a long-distance fiber link.)
3. Are there any "* * *" hops? Did the trace still reach the destination afterward?
   Explain in one sentence why "* * *" does NOT mean packets are being lost.
4. Is there any hop where latency spikes then RECOVERS at the next hop? What does
   that tell you about whether it's a real problem?
5. Now run:  sudo traceroute -T -p 443 8.8.8.8
   Does the TCP path differ from the default/ICMP path? Why might it?
```

### Exercise 3 — Hard (Production Simulation): Localize loss with mtr
```bash
# Pick a distant host (e.g., a server on another continent, or a known-far CDN edge).
mtr --report -c 100 -b <distant-host>
```
Then reason through it:
```
1. Identify the FIRST hop where Loss% becomes non-zero.
2. Does that loss PERSIST all the way to the final hop, or does it drop back to 0%
   at a later hop?
       - If it recovers → explain why it's harmless (which mechanism?).
       - If it persists → that hop is your suspect. What KIND of link is it likely to
         be (your LAN? your ISP? a peering/border link? the destination network?)
         based on its position in the path and the IP/hostname?
3. You determine the loss is on a peering link you do NOT own. List THREE distinct
   actions you could take (hint: one bypasses the link, one escalates, one makes
   clients resilient).
4. Bonus: re-run with  mtr --report -c 100 -T -P 443 <host>  and compare. If the TCP
   result is clean but the default ICMP result showed loss, what do you conclude?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **ICMP is which layer, and what are its protocol number and its relationship to TCP/UDP?** Why does that relationship explain how a host can serve HTTPS perfectly while every ping times out?

2. **Explain the TTL trick that makes traceroute work** — in your own words, why does sending packets with TTL = 1, 2, 3… reveal each router on the path? Which ICMP message does each router send back?

3. **You run traceroute and hop 7 shows 300ms while hop 6 shows 40ms and hop 8 (the destination) shows 45ms. Is hop 7 a problem? Why or why not?** What single rule tells you when a hop's latency IS a real problem?

4. **What does a `*` in traceroute output mean?** Give the most common benign cause, and describe what real end-to-end packet loss would look like instead.

5. **In `mtr --report -c 100 example.com`, what do the columns Loss%, Snt, Avg, Best, Wrst, and StDev each tell you?** Which single column, and which pattern in it, pinpoints the exact hop where a real problem begins?

6. **Your service is on TCP port 443 and users report failures, but ping to the host succeeds and traceroute (default) looks clean. What are you possibly missing, and which two commands would you run to test the ACTUAL service path?**

7. **You see `ttl=52` in a ping reply.** Roughly how many hops away is the responder, and what OS family is it likely running? Show your reasoning.

---

## Quick Reference Card

### ICMP message types
| Type | Name | Role |
|------|------|------|
| 8 → 0 | Echo Request → Echo Reply | ping: reachability + RTT |
| 11 | Time Exceeded (TTL=0) | traceroute: each router reveals itself |
| 3/3 | Destination Unreachable / Port Unreachable | classic UDP traceroute "arrived" signal |
| 3/1, 3/0 | Host / Network Unreachable | no route — real delivery failure |

### ping output fields
| Field | Meaning | What to watch for |
|-------|---------|-------------------|
| `icmp_seq` | sequence number | gaps = dropped packets |
| `ttl` | TTL remaining in reply | 64/128/255 minus this ≈ hop count + OS hint |
| `time` | round-trip time (ms) | the core latency number |
| `% packet loss` | reliability | >0% and persistent = trouble |
| `min/avg/max/mdev` (stddev) | latency spread | big max−min or high mdev = jitter |

### traceroute probe modes
| Command | Probe | Use when |
|---------|-------|----------|
| `traceroute host` | UDP high ports (Linux/mac) | default quick trace |
| `traceroute -I host` | ICMP Echo | mimic ping's path; Windows `tracert` default |
| `sudo traceroute -T -p 443 host` | TCP SYN → 443 | through firewalls; test a real service |
| `traceroute -n host` | (any) numeric | skip slow reverse-DNS |
| `tracert host` (Windows) | ICMP Echo | Windows equivalent (`-d` = numeric) |

### mtr columns
| Column | Meaning |
|--------|---------|
| `Loss%` | % probes to this hop with no reply |
| `Snt` | probes sent to this hop |
| `Last` | most recent RTT |
| `Avg` | average RTT |
| `Best` / `Wrst` | min / max RTT seen |
| `StDev` | RTT standard deviation (jitter) |

**The one rule that matters:** loss/latency that **appears at a hop AND persists through to the destination** = real. A spike or loss that **recovers at a later hop** = harmless ICMP deprioritization. Ignore it.

**Cheat block:**
```bash
ping -c 5 host                        # 5 pings, then summary
ping -c 100 host                      # loss %/jitter over a good sample
traceroute -n host                    # trace path, numeric (fast)
sudo traceroute -T -p 443 host        # TCP trace to a real HTTPS service
tracert host                          # Windows (ICMP by default)
mtr host                              # live continuous path table
mtr --report -c 100 host              # 100-cycle snapshot for a ticket
mtr --report -c 100 -T -P 443 host    # continuous TCP-to-443 path health

# "ping fails" ≠ "host down" — confirm with the real port:
nc -vz host 443
curl -sS -o /dev/null -w "%{http_code}\n" https://host/health
```

---

## When Would I Use This at Work?

### Scenario 1: "The site is down for our EU users!"
ping and mtr from an EU location (a colleague's box, a cloud VM in that region, or a service like a looking-glass). If mtr shows loss starting at a specific hop and persisting to your origin, you have pinpointed a path problem — likely an ISP or peering link. If instead everything is clean end-to-end and only the app returns errors, it's not the network — pivot to `curl` and your application logs (Topic 27).

### Scenario 2: "Is it the database host or the database service?"
A host can ping fine while the DB port is firewalled or the process is dead. `ping db.internal` proves the *host* is on the network; `nc -vz db.internal 5432` (or `traceroute -T -p 5432`) proves the *service port* is reachable. Two different failures, two different fixes — never conflate them.

### Scenario 3: "New region rollout — is latency acceptable?"
Before promising SLAs, `ping -c 100` and `mtr --report` from the new region to your services give you the raw RTT floor and path stability. If ping avg is 180ms, no code change beats that single-round-trip cost — you now know whether you need a closer region, a CDN, or read replicas (Topics 30, 33).

### Scenario 4: "Escalating to our transit provider"
NOCs want evidence, not vibes. `mtr --report -c 200` output showing loss originating at their border router and persisting downstream is exactly the artifact that gets a peering/congestion issue actioned. Paste the report straight into the ticket.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the TTL field and packet routing that traceroute exploits; the hop-by-hop journey these tools measure.
- `04-ip-addresses.md` — public vs private IPs you'll recognize in each traceroute hop (192.168.x, 10.x, CGNAT 100.64.x).

**Study alongside / next:**
- `21-curl-mastery.md` — the higher-layer companion: once ping/traceroute prove the *path* is healthy, curl tests the *service* (DNS, TCP, TLS, HTTP timing).
- `23-tcpdump-and-wireshark.md` — when you need to see the actual packets (including the ICMP messages) on the wire.
- `27-debugging-a-slow-api.md` — the full methodology that folds ping/traceroute/mtr into a systematic latency breakdown across DNS, TCP, TLS, and TTFB.
- `33-network-latency.md` — the physics behind the RTT numbers ping shows you: speed-of-light limits, distance math, and what you can vs. cannot optimize.

**This topic's core takeaway:** ping tells you *if and how fast*, traceroute tells you *the path*, mtr watches *the path continuously* — and the master skill is refusing to be fooled by blocked ICMP, silent `*` hops, and deprioritized middle-hop latency. Only problems that persist end-to-end are real.

---

*Doc saved: `/docs/networking/25-ping-traceroute-mtr.md`*
