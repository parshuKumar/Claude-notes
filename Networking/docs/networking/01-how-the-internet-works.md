# 01 — How the Internet Works

> **Phase 1 — Topic 1 of 5**
> Prerequisites: none. This is the foundation everything else builds on.

---

## ELI5 — The Simple Analogy

Imagine you want to send a letter to a friend who lives in another city.

1. You write the letter and put it in an envelope (**your data**).
2. You write their address on it (**IP address**).
3. You drop it at your local post office (**your router**).
4. The post office doesn't drive it to your friend directly. It hands it to a regional sorting center (**ISP**), which hands it to another sorting center closer to your friend, and so on.
5. Eventually a postal worker delivers it to your friend's door (**their device**).

The internet works *exactly* like this — except instead of a letter it's millions of small chunks of data called **packets**, the delivery happens in milliseconds, and the route taken can change mid-delivery if a road is congested.

---

## Where This Lives in the Network Stack

This topic is the *entire* stack from a bird's-eye view. Every layer below is covered in later topics. For now, see the full picture:

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, WebSocket, DNS, SMTP)           │  ← your Node.js app lives here
│  Layer 6 — Presentation  (TLS/SSL, encoding, compression)       │
│  Layer 5 — Session       (session management)                   │
│  Layer 4 — Transport     (TCP, UDP — ports live here)           │
│  Layer 3 — Network       (IP — addresses live here)             │  ← routing decisions
│  Layer 2 — Data Link     (Ethernet, Wi-Fi — MAC addresses)      │  ← local delivery
│  Layer 1 — Physical      (cables, fiber, radio waves, photons)  │  ← actual electrons/light
└─────────────────────────────────────────────────────────────────┘
```

**This topic touches ALL layers** — we're tracing a packet from Layer 1 (you pressing Enter) all the way up and back down.

---

## What Is This?

The internet is a global network of networks. No single company owns it. It's built from:
- Billions of **end devices** (phones, laptops, servers)
- Millions of **routers** that forward packets toward their destination
- Thousands of **ISPs** (Internet Service Providers) connected to each other via peering agreements
- Physical infrastructure: fiber optic cables (including undersea ones), copper cables, cellular towers, and satellites

Data travels as **packets** — small chunks of binary data (typically up to ~1,500 bytes each) with a header that says where they're going and where they came from.

---

## Why Does It Matter for a Backend Developer?

Without understanding this, you can't:
- Reason about **why your API is slow** (is it DNS? TCP? TLS? routing?)
- Understand **why a request from Tokyo takes 200ms but one from Frankfurt takes 12ms**
- Debug **"connection refused"** vs **"connection timed out"** (they mean completely different things)
- Make **CDN decisions** — you need to know where packets actually travel
- Understand what happens when your **data center loses connectivity**

---

## The Physical Infrastructure — What Actually Carries Your Data

### Tier 1: Last Mile (Your Home/Office)
```
Your laptop
    │
    │  Wi-Fi (radio waves, 2.4GHz or 5GHz)
    ▼
Home Router
    │
    │  Copper wire (ADSL/cable) OR fiber optic cable
    ▼
ISP's Local Exchange / Street Cabinet
```

### Tier 2: ISP Network
```
ISP Local Exchange
    │
    │  Fiber optic cable (light pulses through glass)
    ▼
ISP Regional Hub
    │
    │  High-capacity fiber backbone (10-100 Gbps links)
    ▼
ISP's National Backbone
```

### Tier 3: Between ISPs — Internet Exchange Points (IXPs)
ISPs don't all connect to each other directly. They meet at **Internet Exchange Points** — buildings where hundreds of ISPs plug into the same physical switch and agree to swap traffic. Examples: AMS-IX (Amsterdam), DE-CIX (Frankfurt), LINX (London).

```
Your ISP ──────────────────► IXP (Internet Exchange Point)
                                        │
                          ┌─────────────┼─────────────┐
                          │             │             │
                    Tier-1 ISP    Target ISP    Another ISP
                    (AT&T, NTT)
```

### Tier 4: Undersea Cables
When your request goes to another continent:
```
London ──► Telehouse data center ──► Undersea cable landing station
               │
               │  Fiber optic cables under the Atlantic Ocean
               │  (signals travel at ~200,000 km/s through glass)
               ▼
New York ──► Data center ──► AWS us-east-1 ──► Your target server
```
There are ~400 undersea cable systems carrying ~99% of intercontinental internet traffic. Satellites are used only where cables can't reach.

---

## How It Works — Step by Step

Let's trace what happens when you type `https://api.myapp.com/users` in your Node.js app or browser and press Enter. Every step. No shortcuts.

```
YOU TYPE: fetch("https://api.myapp.com/users")
```

### Step 1 — DNS Resolution (finding the address)
```
Your app asks: "What is the IP address of api.myapp.com?"

1a. Check local DNS cache → not found
1b. Check /etc/hosts → not found  
1c. Ask your configured DNS resolver (e.g., 8.8.8.8)
1d. Resolver asks root nameserver → "ask .com TLD"
1e. Resolver asks .com TLD server → "ask myapp.com's nameserver"
1f. Resolver asks myapp.com nameserver → "142.250.80.46"
1g. Answer cached locally for TTL seconds (e.g., 300s = 5 min)

Result: api.myapp.com = 142.250.80.46
```
*DNS lives at Layer 7 (Application). It uses UDP on port 53. More in Topic 06.*

### Step 2 — TCP Connection (establishing the channel)
```
Your OS: "I need to talk to 142.250.80.46 on port 443"

2a. OS picks an ephemeral source port, e.g. 54231
2b. SYN packet sent:      [your IP:54231] → [142.250.80.46:443]  flags=SYN
2c. SYN-ACK received:     [142.250.80.46:443] → [your IP:54231]  flags=SYN,ACK
2d. ACK sent:             [your IP:54231] → [142.250.80.46:443]  flags=ACK

TCP connection is now ESTABLISHED. A reliable, ordered byte stream exists.
```
*TCP lives at Layer 4 (Transport). More in Topic 08.*

### Step 3 — TLS Handshake (encrypting the channel)
```
Because this is HTTPS (port 443), TLS negotiation happens now:

3a. ClientHello:  "I support TLS 1.3, here are my cipher suites"
3b. ServerHello:  "Let's use TLS_AES_256_GCM_SHA384"
3c. Server sends its X.509 certificate (contains its public key)
3d. Your OS verifies the certificate chain up to a trusted root CA
3e. Key exchange (ECDH): both sides compute the same shared secret
3f. Both sides switch to symmetric encryption using that secret
3g. Finished messages exchanged — channel is now encrypted

All further data is encrypted. Even your ISP can't read it.
```
*TLS lives at Layer 6 (Presentation). More in Topic 09.*

### Step 4 — HTTP Request (sending the actual question)
```
Inside the encrypted TLS tunnel, your app sends:

GET /users HTTP/1.1
Host: api.myapp.com
Accept: application/json
Authorization: Bearer eyJhbGci...
User-Agent: node-fetch/3.0

(blank line — signals end of headers)
```
*HTTP lives at Layer 7 (Application). More in Topic 10.*

### Step 5 — Packet Routing (traveling across the internet)
```
Your GET request leaves as one or more IP packets.
Each packet's journey:

Packet: [src: 203.0.113.5:54231] [dst: 142.250.80.46:443] [data: encrypted HTTP]

Hop 1:  Your laptop        → Your home router         (Wi-Fi)
Hop 2:  Your home router   → ISP edge router           (fiber/copper)
Hop 3:  ISP edge router    → ISP backbone router       (fiber, BGP routing)
Hop 4:  ISP backbone       → IXP                       (peering)
Hop 5:  IXP                → Target's ISP backbone     (peering)
Hop 6:  Target ISP         → Target data center        (fiber)
Hop 7:  Data center router → Target server's NIC       (Ethernet)

Each router reads the DESTINATION IP only, picks the best next hop, 
and forwards. Routers do NOT read your HTTP data — they only see Layer 3.
```

### Step 6 — Server Processes the Request
```
6a. Server's NIC receives Ethernet frame
6b. OS extracts IP packet from frame
6c. OS extracts TCP segment from IP packet
6d. OS decrypts TLS record
6e. HTTP server (nginx, Node.js) reads the request
6f. Your route handler runs: db query, business logic
6g. Response built: HTTP 200 + JSON body
6h. Response travels back down: HTTP → TLS → TCP → IP → Ethernet
6i. Same routing process in reverse
```

### Step 7 — Connection Teardown
```
After response is received (or after timeout/close):

7a. FIN sent:     [server] → [client]   flags=FIN,ACK
7b. ACK sent:     [client] → [server]   flags=ACK
7c. FIN sent:     [client] → [server]   flags=FIN,ACK
7d. ACK sent:     [server] → [client]   flags=ACK

Connection enters TIME_WAIT state for 2×MSL (~60-120 seconds)
then fully closes. (With HTTP keep-alive, connection reused for next request.)
```

---

## The Complete End-to-End Picture

```
┌──────────────┐         INTERNET          ┌──────────────────────┐
│  Your Laptop │                           │   api.myapp.com      │
│              │                           │   142.250.80.46      │
│  Node.js app │──DNS──►┌──────────┐       │                      │
│  port 54231  │        │ 8.8.8.8  │       │  Node.js server      │
│              │        │ DNS      │       │  port 443 (HTTPS)    │
│              │        └──────────┘       │                      │
│              │                           │                      │
│              │──TCP SYN──────────────────►│                      │
│              │◄──TCP SYN-ACK─────────────│                      │
│              │──TCP ACK──────────────────►│                      │
│              │                           │                      │
│              │──TLS ClientHello──────────►│                      │
│              │◄──TLS ServerHello+Cert────│                      │
│              │──TLS Finished─────────────►│                      │
│              │                           │                      │
│              │──HTTP GET /users──────────►│  route handler runs  │
│              │◄──HTTP 200 + JSON─────────│                      │
│              │                           │                      │
│              │──TCP FIN──────────────────►│                      │
└──────────────┘                           └──────────────────────┘

Total time from pressing Enter to receiving response: ~50–200ms (typical)
Breakdown:
  DNS lookup:      1–50ms  (0ms if cached)
  TCP handshake:   1×RTT   (e.g., 20ms for nearby server)
  TLS handshake:   1×RTT   (TLS 1.3) or 2×RTT (TLS 1.2)
  HTTP request:    1×RTT   + server processing time
```

---

## Packet Anatomy — What's Actually on the Wire

Every piece of data on the internet is wrapped in multiple layers of headers (like envelopes inside envelopes). This is called **encapsulation**.

```
When your HTTP GET /users is sent, the actual bytes look like this:

┌────────────────────────────────────────────────────────────────┐
│ ETHERNET FRAME (Layer 2)                                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ IP PACKET (Layer 3)                                      │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ TCP SEGMENT (Layer 4)                              │  │  │
│  │  │  ┌─────────────────────────────────────────────┐  │  │  │
│  │  │  │ TLS RECORD (Layer 6)                        │  │  │  │
│  │  │  │  ┌──────────────────────────────────────┐  │  │  │  │
│  │  │  │  │ HTTP REQUEST (Layer 7)               │  │  │  │  │
│  │  │  │  │ GET /users HTTP/1.1                  │  │  │  │  │
│  │  │  │  │ Host: api.myapp.com                  │  │  │  │  │
│  │  │  │  └──────────────────────────────────────┘  │  │  │  │
│  │  │  └─────────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘

Each layer adds its own header (and sometimes trailer).
Each layer on the receiving end strips its header and passes the payload up.
This process is called DE-encapsulation.
```

**IPv4 Packet Header (simplified):**
```
┌─────────────────────────────────────────┐
│ Version (4 bits)      │ IHL (4 bits)    │  ← IPv4, header length
│ DSCP/ECN (8 bits)     │ Total Length    │  ← QoS, packet size
│ Identification (16 bits)                │  ← for reassembling fragments
│ Flags (3) │ Fragment Offset (13 bits)   │  ← fragmentation control
│ TTL (8 bits)    │ Protocol (8 bits)     │  ← hops remaining, TCP=6/UDP=17
│ Header Checksum (16 bits)               │  ← error detection
│ Source IP Address (32 bits)             │  ← e.g. 203.0.113.5
│ Destination IP Address (32 bits)        │  ← e.g. 142.250.80.46
│ Options (variable) │ Data (payload)     │
└─────────────────────────────────────────┘
```

**TTL (Time to Live):** starts at 64 or 128, decremented by 1 at every router. When it hits 0, the packet is discarded and an ICMP "Time Exceeded" is sent back. This prevents packets looping forever and is how `traceroute` works.

---

## Exact Syntax Breakdown

### curl -v https://api.myapp.com/users
```
curl  -v       https://api.myapp.com/users
│     │        │        │             │
│     │        │        │             └── path (sent in HTTP request line)
│     │        │        └── hostname (used for DNS lookup + Host header)
│     │        └── https = port 443 + TLS negotiation
│     └── verbose: print all headers, TLS info, timing
└── transfer URL data
```

### What -v shows you:
```
*   Trying 142.250.80.46:443...          ← TCP connect attempt (after DNS)
* Connected to api.myapp.com (142.250.80.46) port 443  ← TCP handshake complete
* SSL connection using TLSv1.3           ← TLS version negotiated
* Server certificate: api.myapp.com      ← certificate verified
> GET /users HTTP/2                      ← HTTP request sent (> = outgoing)
> Host: api.myapp.com
> User-Agent: curl/8.4.0
>
< HTTP/2 200                             ← HTTP response (< = incoming)
< content-type: application/json
< content-length: 342
<
[{"id":1,"name":"Alice"}, ...]           ← response body
```

---

## Example 1 — Basic: Trace Your Own Request

Run this in your terminal and watch every step happen:

```bash
# -v = verbose (shows TLS, headers, timing)
# --trace-time = adds timestamps to every step
curl -v --trace-time https://httpbin.org/get 2>&1

# What to look for:
# Line starting with "*   Trying" = TCP connect starting (DNS already done)
# Line "* Connected" = TCP 3-way handshake complete
# Line "* SSL connection using" = TLS negotiated
# Lines starting ">" = your request headers
# Lines starting "<" = server response headers
# Last section = response body
```

Expected output excerpt:
```
00:00:00.000000 *   Trying 54.208.105.20:443...
00:00:00.023000 * Connected to httpbin.org port 443
00:00:00.089000 * SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
00:00:00.090000 > GET /get HTTP/2
00:00:00.090000 > Host: httpbin.org
00:00:00.090000 >
00:00:00.156000 < HTTP/2 200
00:00:00.156000 < content-type: application/json
```

Notice the timestamps: DNS + TCP took ~23ms, TLS took ~66ms, response took ~67ms. That's your latency breakdown.

---

## Example 2 — Production Scenario: "Why is our API slow for users in Asia?"

**The situation:** Your Node.js API is hosted in `us-east-1` (Virginia). Users in Tokyo are complaining about 800ms response times. Users in New York see 50ms. Your code is fast — database queries take 5ms. What's happening?

**Step 1: Measure where the time goes**
```bash
# curl's built-in timing breakdown
curl -o /dev/null -s -w "
DNS lookup:        %{time_namelookup}s
TCP connect:       %{time_connect}s
TLS handshake:     %{time_appconnect}s
Time to first byte:%{time_starttransfer}s
Total:             %{time_total}s
" https://api.myapp.com/users
```

**Output from a server in Tokyo:**
```
DNS lookup:         0.023s
TCP connect:        0.185s    ← TCP handshake = 1×RTT = ~170ms (Tokyo→Virginia)
TLS handshake:      0.370s    ← TLS = another 1×RTT (TLS 1.3)
Time to first byte: 0.380s    ← server processing: 10ms
Total:              0.381s
```

**Root cause:** Physics. Tokyo to Virginia is ~10,000 km. Light travels at ~200,000 km/s through fiber (with bends and amplifiers, effective speed is ~2/3 of light speed in vacuum). That's **~100ms one-way**, so each round-trip costs **~200ms**.

- TCP handshake = 1 RTT = 200ms
- TLS 1.3 handshake = 1 RTT = another 200ms
- HTTP request/response = 1 RTT = 200ms
- Total: ~600ms just from distance. Plus processing = 800ms ✓

**Fix:** Deploy to `ap-northeast-1` (Tokyo) or use a CDN/edge function closer to your users.

**The lesson:** No amount of code optimization fixes geography. This is a network topology problem.

---

## Common Mistakes

### Mistake 1: Confusing "connection refused" with "connection timed out"
```
Error: connect ECONNREFUSED 203.0.113.5:3000
```
**Root cause:** The server is reachable at the IP level (you got a TCP RST back immediately). The port is closed or nothing is listening. Your Node.js server isn't running, or it's on a different port.

```
Error: connect ETIMEDOUT 203.0.113.5:3000
```
**Root cause:** Packets are going out but no response is coming back. Could be: firewall dropping packets, wrong IP, server is down completely, or routing failure.

**Diagnosis:**
```bash
# ECONNREFUSED → something is listening but rejecting
telnet 203.0.113.5 3000   # Should say "Connection refused" instantly

# ETIMEDOUT → packets are being dropped
ping 203.0.113.5          # If ping responds, IP is reachable
traceroute 203.0.113.5    # Find where packets die
```

---

### Mistake 2: Thinking HTTPS hides the destination IP
```
Common belief: "HTTPS encrypts everything — nobody knows where I'm connecting"
Reality: The DESTINATION IP ADDRESS is in every packet header — in plaintext.
         Your ISP knows you connected to 142.250.80.46.
         They don't know what you requested, but they know who you talked to.
         (SNI leaks the hostname too — until ECH/Encrypted SNI is deployed)
```

---

### Mistake 3: Thinking DNS is instant
```javascript
// This DNS lookup can take 50ms+ if:
// - The record is not cached locally
// - Your DNS server is far away
// - TTL expired and must re-query

const response = await fetch("https://api.external-service.com/data");
// ↑ if this is in a hot loop, DNS lookup happens once then caches.
// But if TTL is 60 seconds, every 60s there's a DNS round-trip penalty.
```
**Fix:** Set reasonable TTLs (300s minimum) on your DNS records. Use connection pooling (reuses TCP connections, so DNS is only looked up once).

---

### Mistake 4: Expecting packet delivery order/reliability from IP
```
IP does NOT guarantee:
- Packets arrive in order (they can take different routes)
- Packets arrive at all (routers drop packets when congested)
- Packets arrive only once (duplicates can happen)

TCP solves all of these. UDP does not.
If you use raw UDP (or build on it), YOU handle ordering/reliability.
```

---

## Hands-On Proof

```bash
# 1. See the route your packets actually take to Google
tracert 8.8.8.8              # Windows
traceroute 8.8.8.8           # macOS/Linux
# Each line is one router hop. Watch the latency grow with distance.

# 2. See DNS resolution happening
nslookup google.com          # Windows/macOS/Linux
# Then try a second time — much faster (cached)

# 3. See the full HTTP conversation
curl -v https://httpbin.org/get 2>&1 | more
# Read every line. Find: Trying, Connected, SSL, >, <

# 4. Measure latency to different parts of the world
ping google.com              # ~10ms if nearby, ~150ms if far
ping 1.1.1.1                 # Cloudflare — usually very fast worldwide

# 5. See your own public IP (what the internet sees)
curl https://ifconfig.me

# 6. See the full timing breakdown
curl -o /dev/null -s -w "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | Total: %{time_total}s\n" https://google.com
```

---

## Practice Exercises

### Exercise 1 — Easy: Map your own path to the internet
```bash
# Run this:
tracert 8.8.8.8       # Windows
# OR
traceroute 8.8.8.8    # macOS/Linux

# Answer these questions from the output:
# 1. How many hops until you reach Google's network?
# 2. Where does latency jump significantly? (That's likely a long fiber link)
# 3. What is the total latency to 8.8.8.8?
# 4. Can you identify your home router (hop 1) and your ISP (hops 2-3)?

# Paste your output.
```

### Exercise 2 — Medium: Measure your request latency breakdown
```bash
# Run this against 3 different servers:
curl -o /dev/null -s -w "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" https://google.com

curl -o /dev/null -s -w "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" https://github.com

curl -o /dev/null -s -w "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" https://httpbin.org/get

# Answer:
# 1. Which step takes the most time for each?
# 2. Run each command TWICE in a row. Does DNS time drop the second time? Why?
# 3. Calculate: what is the estimated RTT (round trip time) based on the TCP connect time?
```

### Exercise 3 — Hard (Production Simulation): Diagnose a slow endpoint
```bash
# Your job: figure out WHERE the latency is coming from

# Step 1: run the timing command 5 times, record results
for i in 1 2 3 4 5; do
  curl -o /dev/null -s -w "Run $i — DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | Total: %{time_total}s\n" https://api.github.com/zen
done

# Step 2: Now try with a pre-resolved IP (bypassing DNS) to isolate DNS cost:
# First get the IP:
nslookup api.github.com     # Windows
# OR
dig +short api.github.com   # macOS/Linux

# Then connect directly to the IP (using --resolve to keep the TLS hostname):
# Replace IP_HERE with the actual IP from above
curl -o /dev/null -s -w "No-DNS — TCP: %{time_connect}s | TLS: %{time_appconnect}s | Total: %{time_total}s\n" \
  --resolve "api.github.com:443:IP_HERE" \
  https://api.github.com/zen

# Question: 
# Compare DNS vs no-DNS results.
# Is DNS adding meaningful latency? 
# What would you do if DNS was adding 100ms to every request?
# (Hint: connection pooling keeps the connection alive so DNS is only done once)
```

---

## Mental Model Checkpoint

Answer these from memory. If you can't, re-read the relevant section.

1. **A packet from your laptop reaches its destination in ~20ms. What is the approximate physical distance to that server?** (Think: speed of light in fiber is ~200,000 km/s, RTT = 2 × one-way)

2. **Your app gets `ETIMEDOUT` when connecting to a remote server. List 3 possible root causes and how you'd distinguish between them.**

3. **Why does HTTPS NOT hide the fact that you're connecting to google.com from your ISP?** (Two reasons — IP address and what else?)

4. **You deploy your Node.js API. A user in Brazil hits your endpoint 100 times per second. Your server takes 1ms per request. But users see 200ms latency. Why, and what is the fix?**

5. **What is TTL in an IP packet? What happens when it hits zero? What tool exploits this behavior?**

6. **Explain encapsulation in one sentence. What is de-encapsulation?**

7. **Your Node.js app makes 50 API calls to an external service per second. Each call does a fresh DNS lookup. The external service has a 60s TTL. Approximately how many DNS queries leave your server per minute?**

---

## Quick Reference Card

| Concept | What it is | Key number to remember |
|---------|-----------|----------------------|
| Packet | Basic unit of data on internet | Max ~1,500 bytes (Ethernet MTU) |
| TTL | Hop counter in IP packet | Starts at 64 (Linux) or 128 (Windows), decremented per router |
| RTT | Round-trip time | NYC→London ≈ 70ms, NYC→Tokyo ≈ 170ms |
| ISP | Internet Service Provider | Carries your packets to/from internet |
| IXP | Internet Exchange Point | Where ISPs physically connect and exchange traffic |
| DNS | Translates hostname → IP | UDP port 53, cached by TTL |
| TCP handshake | Establish connection | 1 RTT cost before you can send data |
| TLS handshake | Encrypt connection | 1 RTT (TLS 1.3) or 2 RTT (TLS 1.2) |
| Ephemeral port | Source port your OS picks | Range 49152–65535 |
| Speed of light in fiber | Effective speed | ~200,000 km/s (~2/3 of vacuum speed) |

**Latency cheat sheet:**
```
Same data center:        <1ms
Same city:               1–5ms
Cross-country (US):      30–60ms
US ↔ Europe:             70–100ms
US ↔ Asia:               150–200ms
US ↔ Australia:          180–250ms
```

**Key commands:**
```bash
curl -v URL                                    # See full HTTP conversation
curl -o /dev/null -s -w "%{time_total}\n" URL # Total request time
tracert/traceroute IP                          # Trace packet path
ping IP                                        # Measure RTT
nslookup HOSTNAME                              # Quick DNS lookup
```

---

## When Would I Use This at Work?

### Scenario 1: "Our API is fast in the US but slow in Europe"
You now know to immediately ask: is this a latency problem (physics/distance) or a throughput problem (bandwidth)? Run `curl` with timing from a European server. If TCP connect time is 80ms, that's just the Atlantic Ocean — deploy closer. If TCP connect time is 5ms but TTFB is 3 seconds, it's your server code.

### Scenario 2: "Microservice A can't connect to Microservice B after deployment"
`ECONNREFUSED` = service B's process isn't running or is on the wrong port.
`ETIMEDOUT` = firewall/security group blocking the traffic, or wrong IP.
`ENOTFOUND` = DNS isn't resolving the service name — Kubernetes service not created, or wrong namespace.

### Scenario 3: "Our server costs spiked — we're pushing 10TB of data this month"
You now understand data travels through your ISP, and ISPs charge for egress bandwidth. CDNs cache your static assets at edge nodes, so users fetch from the CDN (cheap) rather than your origin server (expensive). This is exactly why companies use CDNs — not just for speed, but to reduce origin bandwidth costs.

---

## Connected Topics

**Study before this:** Nothing — this is the foundation.

**Study next (in order):**
- `02-osi-model.md` — Drill into each of the 7 layers we flew over here
- `03-tcp-ip-model.md` — The practical 4-layer model developers actually use
- `04-ip-addresses.md` — Deep dive on IP addresses, subnets, NAT
- `05-ports.md` — What ports are, which ones matter, port conflicts

**This topic connects forward to everything.** Every topic in Phase 2–6 is a deep dive into one piece of the picture painted here.

---

*Doc saved: `/docs/networking/01-how-the-internet-works.md`*
