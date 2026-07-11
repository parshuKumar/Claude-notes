# 03 — The TCP/IP Model

> **Phase 1 — Topic 3 of 5**
> Prerequisites: `01-how-the-internet-works.md`, `02-osi-model.md`

---

## ELI5 — The Simple Analogy

Imagine a consulting firm draws up the "official" org chart for how a shipping company *should* work. On paper it has seven distinct roles: a Sales Rep who takes the order, a Packaging Designer who decides how to wrap it, a Session Coordinator who keeps track of the ongoing conversation with the customer, a Logistics Planner who decides shipping method, a Route Planner who picks the highway, a Local Dispatcher who assigns the delivery truck, and a Driver who physically carries the box. Seven clean, separate jobs. That's the **OSI model** — a model designed by a standards committee (ISO) to teach and document networking in the cleanest possible way.

Now walk into an actual FedEx warehouse. Nobody has seven separate job titles. In practice, one "customer service" person handles sales, packaging decisions, *and* keeping track of the conversation — those three jobs blur together because they always happen in the same breath. Meanwhile "which highway" and "which local truck" are handled by the same dispatch software. So the real, working company only has **four practical departments**: Customer-Facing, Shipping Method, Routing, and Local Delivery.

That's the **TCP/IP model**. It wasn't designed by a standards committee trying to be theoretically clean — it was built by engineers (DARPA, then Vint Cerf and Bob Kahn) who needed something that actually moved packets *today*. They merged the layers that, in real protocol implementations, never usefully separate. The result is the model the internet is literally built on — not a teaching aid describing it from the outside.

---

## Where This Lives in the Network Stack

Here is the exact collapse, layer by layer. This diagram is the single most important picture in this document — memorize it.

```
        OSI MODEL (theoretical, 7 layers)             TCP/IP MODEL (practical, 4 layers)
        ISO standard, 1984                             RFC 1122/1123, deployed since ~1983
        ───────────────────────────────                ───────────────────────────────────

        ┌───────────────────────────┐
        │ L7  Application           │  ─┐
        ├───────────────────────────┤   │
        │ L6  Presentation          │  ─┼──────►        ┌───────────────────────────────┐
        ├───────────────────────────┤   │               │  APPLICATION                  │  HTTP, DNS, TLS,
        │ L5  Session               │  ─┘               │  (your Node.js code lives here)│  SMTP, FTP, SSH, gRPC
        ├───────────────────────────┤                   ├───────────────────────────────┤
        │ L4  Transport             │  ──────────►      │  TRANSPORT                    │  TCP, UDP
        ├───────────────────────────┤                   ├───────────────────────────────┤
        │ L3  Network               │  ──────────►      │  INTERNET                     │  IP, ICMP
        ├───────────────────────────┤                   ├───────────────────────────────┤
        │ L2  Data Link             │  ─┐               │  LINK  (a.k.a. Network Access) │  Ethernet, Wi-Fi
        ├───────────────────────────┤   ├──────►        └───────────────────────────────┘  (802.11), PPP
        │ L1  Physical              │  ─┘
        └───────────────────────────┘

        7 layers, designed to teach            4 layers, designed to ship
```

**The mapping in one sentence each:**
- TCP/IP **Application** = OSI Layers 7 + 6 + 5 (Application + Presentation + Session merged — HTTP, TLS, and "session state" are all just "stuff the app handles")
- TCP/IP **Transport** = OSI Layer 4 (unchanged — TCP and UDP map directly)
- TCP/IP **Internet** = OSI Layer 3 (unchanged — IP maps directly)
- TCP/IP **Link** = OSI Layers 2 + 1 (Data Link + Physical merged — "getting a frame onto the wire" is one job in practice)

A detail worth knowing: **TCP/IP is older than OSI.** ARPANET switched to TCP/IP on January 1, 1983 (the famous "flag day"). ISO didn't publish the OSI model until 1984 — a year *after* TCP/IP was already running production traffic. OSI was meant to become the eventual global standard; TCP/IP was the pragmatic thing that shipped first, worked, and never got replaced. Today OSI survives as the universal *teaching vocabulary* ("that's a Layer 3 problem"), while TCP/IP is the actual *specification* every internet protocol is written against.

---

## What Is This?

The **TCP/IP model** (also called the **Internet Protocol Suite**, or historically the **DoD model** since it was funded by the U.S. Department of Defense's DARPA) is the four-layer architecture that describes how real internet protocols are organized. It is formally defined in **RFC 1122** and **RFC 1123** ("Requirements for Internet Hosts"). Unlike OSI, it was not designed top-down as an abstract reference — it was written to describe protocols (TCP, IP, and friends) that were *already running*.

The four layers, each with a one-line job description:

| Layer | Job in one line |
|---|---|
| **Application** | Format and exchange the data your business logic actually cares about |
| **Transport** | Get data end-to-end between the correct *processes* (ports), with or without reliability |
| **Internet** | Get data across many interconnected networks to the correct *host* (IP addressing + routing) |
| **Link** (Network Access) | Get data across one *physical/local* network segment (frames, MAC addresses, media) |

Note: some textbooks draw a **5-layer hybrid model** that splits Link back into "Physical" and "Data Link" purely to make the diagram line up 1:1 with OSI's 7 layers. That's a teaching convenience, not how RFC 1122 defines it. The real specification — the one every socket API, every router, and every RFC assumes — has **four** layers. That is the model this document teaches.

---

## Why Does It Matter for a Backend Developer?

This is the most practically useful section in this doc, because it tells you **where to invest your learning time**. Not all four layers deserve equal attention from an application developer.

| TCP/IP Layer | How often you touch it | What you actually do there |
|---|---|---|
| **Application** | Constant — roughly 90% of your job | Write HTTP route handlers, parse/set headers, handle WebSocket upgrades and gRPC calls, configure TLS certificates, call services by DNS name, design REST APIs, read response codes |
| **Transport** | Occasional — tuning, not building | Set socket options (`SO_REUSEADDR`, `TCP_NODELAY`), configure keep-alive timeouts, size connection pools, understand L4 load balancer behavior, diagnose `TIME_WAIT` exhaustion |
| **Internet** | Rare — mostly an infra/DevOps concern | Design VPC CIDR ranges, configure subnets and route tables, read/write security group IP rules, understand NAT for outbound traffic |
| **Link** | Almost never | You know it exists (MTU limits, NIC behavior, VLAN tagging) but you essentially never configure it — that's owned by network engineers, the cloud provider, or on-prem hardware teams |

**Deserves deep, ongoing attention:** Application layer (it's where your code lives) and, to a lesser but real degree, Transport layer (a misconfigured connection pool or missing keep-alive will page you at 3am; a misunderstood `ECONNRESET` will waste your afternoon).

**Deserves surface awareness, not mastery:** Internet layer (you should be able to read a route table and understand why `10.0.1.5` can't reach `10.0.2.9` across subnets, but you're not usually the one designing the VPC) and Link layer (you should know MTU mismatches can silently break large packets, but you will almost never touch a NIC driver or an Ethernet frame directly).

The practical rule: **if an interview or an incident is about your API being slow or broken, it's almost always Application or Transport. If it's about two servers not being able to see each other at all, it's almost always Internet or Link — and that's usually a DevOps/platform-team problem, not a "fix the code" problem.**

---

## The Packet/Protocol Anatomy

Encapsulation works exactly the same way it did in the OSI walkthrough — each layer wraps the layer above it in its own header — but now there are only four wrappers instead of seven, and each one has a specific name.

```
Your data:     { "userId": 42, "action": "purchase" }
                              │
                              ▼   APPLICATION wraps it in an HTTP request
┌────────────────────────────────────────────────────────────────────────┐
│ APPLICATION  →  called: "message" / "data"                             │
│  POST /orders HTTP/1.1                                                 │
│  Host: api.myapp.com                                                   │
│  Content-Type: application/json                                        │
│                                                                          │
│  { "userId": 42, "action": "purchase" }        ← your actual payload   │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼   TRANSPORT wraps it in a TCP segment
┌────────────────────────────────────────────────────────────────────────┐
│ TRANSPORT  →  called: "segment" (TCP) or "datagram" (UDP)              │
│  src port: 54231   dst port: 443                                       │
│  seq: 8842001   ack: 1   flags: PSH,ACK   window: 65535                │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  [ everything from APPLICATION above ]                          │   │
│  └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼   INTERNET wraps it in an IP packet
┌────────────────────────────────────────────────────────────────────────┐
│ INTERNET  →  called: "packet"                                          │
│  src: 203.0.113.5   dst: 142.250.80.46   ttl: 64   proto: 6 (TCP)      │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  [ everything from TRANSPORT above ]                            │   │
│  └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼   LINK wraps it in an Ethernet frame
┌────────────────────────────────────────────────────────────────────────┐
│ LINK  →  called: "frame"                                               │
│  src MAC: 02:42:ac:11:00:02   dst MAC: 5c:52:1e:aa:bb:cc   type: IPv4  │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  [ everything from INTERNET above ]                             │   │
│  └────────────────────────────────────────────────────────────────┘   │
│  ...FCS (frame check sequence / CRC trailer for error detection)       │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼   electrons on copper / light on fiber / radio waves
```

**Real protocols living at each layer:**

| Layer | Protocols you'll actually see |
|---|---|
| **Application** | HTTP/HTTPS, DNS, TLS (yes — TLS is Application here, even though OSI called it Presentation), SMTP, FTP, SSH, gRPC, WebSocket, MQTT |
| **Transport** | TCP, UDP (QUIC is arguably a new transport, but it's built *on top of* UDP) |
| **Internet** | IP (v4 and v6), ICMP (used by `ping`/`traceroute`, carried inside IP packets, IP protocol number 1) |
| **Link** | Ethernet, Wi-Fi (802.11), PPP |

**The ARP asterisk:** Address Resolution Protocol (ARP) resolves an IP address to a MAC address — logically it sits *between* the Internet and Link layers. It doesn't fit cleanly into either: it operates on local-segment addressing (a Link-layer concern) but its whole purpose is to serve IP (an Internet-layer concern). Technically, ARP packets are **not** carried inside IP packets at all — they get their own EtherType (`0x0806`) directly inside the Ethernet frame. Most references (including RFC 826) file ARP under Link, but it's genuinely the model's one awkward seam. Keep this in your back pocket — it's a favorite "gotcha" interview question.

---

## How It Works — Step by Step

Let's retrace the exact same HTTPS API request from Topic 01 — but this time every step is labeled by its **TCP/IP** layer, not its OSI layer. Watch how DNS, TLS, and HTTP — three things that lived on *three different OSI layers* (7, 6, and arguably 5) — all collapse into **one single layer** here.

```
YOU RUN: fetch("https://api.myapp.com/orders", { method: "POST", body: {...} })
```

### Step 1 — APPLICATION layer: DNS, TLS, and HTTP all happen here
```
1a. DNS query:      api.myapp.com → 142.250.80.46          (its own Application protocol, UDP/53)
1b. TLS handshake:  ClientHello → ServerHello+Cert → Finished  (encrypts the channel)
1c. HTTP request:   POST /orders HTTP/1.1
                     Host: api.myapp.com
                     Content-Type: application/json
                     { "userId": 42, "action": "purchase" }

In OSI these were spread across Layer 7 (DNS, HTTP), Layer 6 (TLS), and arguably
Layer 5 (keeping the logical session alive). In TCP/IP, there is no separate
Presentation or Session layer — ALL of this is just "the Application's problem."
The Transport layer below doesn't know or care that TLS or DNS even exist.
```

### Step 2 — TRANSPORT layer: TCP delivers it reliably, in order, to the right process
```
2a. OS picks ephemeral source port 54231
2b. SYN:      [you:54231] → [server:443]   flags=SYN
2c. SYN-ACK:  [server:443] → [you:54231]   flags=SYN,ACK
2d. ACK:      [you:54231] → [server:443]   flags=ACK
2e. Your HTTP bytes ride inside TCP segments, sequenced and acknowledged

The Transport layer's ONLY job: get these bytes to port 443 on that host,
in order, without loss — and hand the reassembled stream up to Application.
It does not know or care that the payload is HTTP, TLS, or JSON.
```

### Step 3 — INTERNET layer: IP gets the packet across the network of networks
```
Each TCP segment is wrapped in an IP packet:

  [src: 203.0.113.5]  →  [dst: 142.250.80.46]   ttl=64  proto=TCP

Routers along the path read ONLY the destination IP address and forward
toward the next hop. They never look inside at the TCP header, let alone
the HTTP request. This is the Internet layer's entire job: best-effort,
hop-by-hop forwarding based on IP address alone.
```

### Step 4 — LINK layer: local, one-hop-at-a-time delivery
```
On EVERY single hop (your laptop → home router → ISP router → ... →
destination NIC), the IP packet is wrapped in a fresh Link-layer frame
scoped to that one physical segment:

  Hop 1 (Wi-Fi):      802.11 frame,   src/dst = MAC addresses on your LAN
  Hop 2 (fiber/DSL):  PPP or Ethernet frame to the ISP's router
  ...
  Final hop:          Ethernet frame, src/dst = MAC addresses on the server's LAN

Before each hop, ARP (or its IPv6 cousin, NDP) is used to discover the
next-hop MAC address for a given IP address. The IP packet itself is
UNCHANGED across the whole journey — only the Link-layer frame around it
is stripped and rebuilt at every single hop.
```

**The key insight:** the same journey Topic 01 described across 7 OSI layers is exactly the same set of bytes on the wire — TCP/IP just groups the *concepts* differently. DNS, TLS, and HTTP are three separate protocols but one single layer. That's not a simplification that loses information — it reflects how the protocols are actually specified and implemented.

---

## Exact Syntax Breakdown

Four commands, one per TCP/IP layer, that you'll use constantly as a backend developer:

```
curl -v URL          →  APPLICATION layer  (DNS + TLS + HTTP visible in one command)
ss -tlnp             →  TRANSPORT layer    (what's listening on which port)
ip route show        →  INTERNET layer     (how packets get routed off this host)
ip link show         →  LINK layer         (physical/virtual network interfaces)
```

### `ss -tlnp` — line by line
```
$ ss -tlnp
State   Recv-Q  Send-Q   Local Address:Port    Peer Address:Port   Process
LISTEN  0       128      0.0.0.0:22            0.0.0.0:*           users:(("sshd",pid=812,fd=3))
LISTEN  0       511      127.0.0.1:5432        0.0.0.0:*           users:(("postgres",pid=1044,fd=6))
LISTEN  0       511      0.0.0.0:3000          0.0.0.0:*           users:(("node",pid=2201,fd=19))
```
```
ss     -t         -l          -n          -p
│      │          │           │           │
│      │          │           │           └── show the owning Process (name + pid)
│      │          │           └── Numeric ports/addresses (skip slow DNS reverse-lookups)
│      │          └── only LISTENing sockets (not established connections)
│      └── TCP sockets only (add -u for UDP)
└── socket statistics
```
Column meaning: **Local Address:Port** is what's bound and listening on *this* host. `0.0.0.0:3000` means the Node.js process on port 3000 accepts connections on **any** network interface; `127.0.0.1:5432` means Postgres only accepts connections from **localhost** — it is deliberately not reachable from outside this machine. This single line is how you diagnose "why can't another container reach my database" — if it's bound to `127.0.0.1`, nothing outside the host (or outside the container's network namespace) can ever reach it, no matter what the firewall says.

### `ip route show` — quick read
```
$ ip route show
default via 192.168.1.1 dev eth0 proto dhcp metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.42 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```
The first line is the **default route** — "if the destination isn't covered by any more specific rule below, send it to gateway `192.168.1.1` out interface `eth0`." The other lines are directly-connected local subnets (`192.168.1.0/24` on your LAN, `172.17.0.0/16` on Docker's bridge network). This is the Internet layer's routing table — the exact decision every packet leaving this host is checked against.

---

## Example 1 — Basic

Classify each of these into its TCP/IP layer:

| Protocol / Tool | TCP/IP Layer | Why |
|---|---|---|
| `HTTP` | Application | Formats the request/response your app logic reads |
| `DNS` | Application | Resolves names to IPs — a protocol conversation, same as HTTP |
| `TLS` | Application | Encrypts the Application-layer conversation; OSI called it Presentation, TCP/IP doesn't separate it |
| `TCP` | Transport | End-to-end, ordered, reliable delivery between two ports |
| `UDP` | Transport | End-to-end, unordered, best-effort delivery between two ports |
| `IP` (v4/v6) | Internet | Logical addressing + routing across networks |
| `ARP` | Link *(the one exception — see note above)* | Resolves IP → MAC on the local segment, framed directly in Ethernet |
| `Ethernet` / `Wi-Fi (802.11)` | Link | Frames data for delivery across one physical/local network segment |

---

## Example 2 — Production Scenario

**The situation:** In a Kubernetes cluster, the `orders-service` Pod calls `payments-service` on port `8443` and gets a connection failure. You don't know yet whether the problem is DNS, the port, the network path, or the node itself. Walk it layer by layer, bottom-up in terms of "which layer works" — top-down in terms of where you check first, since Application-layer failures are both the most common and the cheapest to test.

### 1. Application layer — does the name even resolve?
```bash
kubectl exec -it orders-7d9f8c-abcd -- nslookup payments-service.default.svc.cluster.local
```
```
Server:    10.96.0.10
Address:   10.96.0.10:53

** server can't find payments-service.default.svc.cluster.local: NXDOMAIN
```
`NXDOMAIN` means CoreDNS (the cluster's DNS server, itself an Application-layer service) has no record for that name. Check the Service actually exists and is in the namespace you think it's in:
```bash
kubectl get svc payments-service -n default
kubectl get endpoints payments-service -n default   # empty = no healthy pods match the selector
```
If DNS resolves fine and returns a ClusterIP, Application layer is cleared — move down.

### 2. Transport layer — is the port actually open?
```bash
kubectl exec -it orders-7d9f8c-abcd -- nc -zv payments-service 8443
```
```
nc: connect to payments-service port 8443 (tcp) failed: Connection refused
```
`Connection refused` (an immediate TCP RST) means *something* answered at the IP level but nothing is listening on `8443` — check the container's actual listening port vs. the Service's `targetPort`:
```bash
kubectl exec -it payments-<pod> -- ss -tlnp   # is it really listening on 8443, or on 8080?
```
A mismatched `targetPort` in the Service spec vs. the port the container actually binds is one of the single most common Kubernetes networking bugs — and it's purely a Transport-layer symptom with an Application-layer (misconfigured YAML) root cause.

### 3. Internet layer — can the packets even route between pods?
If instead you saw a **timeout** rather than "refused," suspect routing:
```bash
kubectl get pods -o wide                       # get the real pod IPs
kubectl exec -it orders-<pod> -- ip route       # what does this pod think its routes are?
# on the node itself:
iptables -t nat -L KUBE-SERVICES -n | grep payments   # does kube-proxy have a NAT rule for the ClusterIP?
```
kube-proxy's whole job is translating the virtual ClusterIP into a real Pod IP via iptables (or IPVS) rules — that translation *is* Internet-layer NAT. If the rule is missing, or the CNI plugin (Calico, Cilium, Flannel) hasn't programmed a route to the destination pod's node, packets simply vanish — a pure Internet-layer failure.

### 4. Link layer — is there a physical/virtual transport problem underneath it all?
The classic Link-layer gotcha in overlay networks: an **MTU black hole**. Many CNI plugins tunnel pod traffic over VXLAN, which adds ~50 bytes of overhead — the effective MTU drops from 1500 to around 1450, but the pod's `veth` interface may still advertise 1500. Small packets (like a DNS query) work fine; large packets silently vanish because a router in between drops oversized frames without sending back the ICMP message that would let TCP discover the smaller size.
```bash
ip link show cni0        # check the bridge/interface MTU on the node
ip -d link show veth123   # check the specific veth pair for the pod
```
Symptom: small requests succeed, large POST bodies or big TLS certificates hang forever. This is the layer nobody thinks to check — and exactly why "95% of the time it's Application/Transport, but when it's not, it's usually this."

---

## Common Mistakes

### Mistake 1: Treating "TCP/IP model" and "OSI model" as interchangeable
```
Wrong: "TLS is a Layer 6 protocol, so in TCP/IP terms it's the 6th layer."
Right: TCP/IP has no Layer 6. TLS is simply part of the Application layer.
       Layer NUMBERS are an OSI concept. TCP/IP layers are named, not numbered,
       precisely because they don't map 1:1 onto OSI's seven slots.
```
When someone says "that's a Layer 4 load balancer," they're using OSI numbering as shorthand even in a TCP/IP context — that's fine and extremely common in industry, but know that you're borrowing OSI's *numbers* to describe a TCP/IP *layer* (Transport). Don't assume the two models are the same thing wearing different names.

### Mistake 2: Assuming "Application layer" means "just HTTP"
```
Wrong: "The Application layer is for HTTP APIs."
Right: The Application layer is EVERYTHING above the transport handshake:
       HTTP, DNS, TLS, SMTP, FTP, SSH, gRPC, WebSocket, MQTT — all of it.
```
TLS in particular trips people up. Because OSI taught you TLS is "Presentation," it's tempting to think TCP/IP's Application layer is somehow "below" TLS. It isn't — TLS **is** Application layer here, full stop.

### Mistake 3: Forgetting ARP doesn't fit cleanly into either model
```
ARP resolves an IP address (Internet-layer concept) to a MAC address
(Link-layer concept) — and it does so using its own EtherType, not
carried inside an IP packet at all. It's the model's one genuine seam.
Interviewers love asking "what layer is ARP?" specifically because
there is no fully clean answer.
```

### Mistake 4: Over-focusing on Internet/Link-layer trivia
```
It's satisfying to memorize IP header field widths, VLAN tagging rules,
and MTU black-hole mechanics — but for 95% of backend work, the bug is
in your HTTP handler, your headers, your TLS config, or your connection
pool (Application/Transport). Learn Internet/Link layer well enough to
read a route table and reason about a Kubernetes CNI issue — don't burn
weeks becoming a router-configuration expert unless that's literally your job.
```

---

## Hands-On Proof

Run one command per layer, right now, and connect each output to the layer it belongs to:

```bash
# 1. LINK layer — see your machine's physical/virtual network interfaces
ip link show                    # Linux
ifconfig -a                     # macOS
# Look for: interface names (eth0, en0, docker0), MAC addresses, MTU, UP/DOWN state

# 2. INTERNET layer — see your routing table and IP addresses
ip route show                   # Linux
netstat -nr                     # macOS
ip addr show                    # Linux — your assigned IP addresses
ifconfig                        # macOS — same info

# 3. TRANSPORT layer — see every listening socket on your machine
ss -tlnp                        # Linux
netstat -an | grep LISTEN       # macOS/Linux (no process names on macOS without sudo)
# Notice: some things listen on 127.0.0.1 only (local-only), others on 0.0.0.0 (everywhere)

# 4. APPLICATION layer — see DNS + TLS + HTTP happen in one shot
curl -v https://httpbin.org/get 2>&1
# "* Trying..."      = INTERNET/TRANSPORT (IP found, TCP connecting)
# "* Connected"      = TRANSPORT handshake done
# "* SSL connection" = APPLICATION (TLS)
# "> GET ..."        = APPLICATION (HTTP request)
# "< HTTP/2 200"     = APPLICATION (HTTP response)
```

Run all four back to back and you've just personally observed all four TCP/IP layers on your own machine, top to bottom.

---

## Practice Exercises

### Exercise 1 — Easy: Layer-classify your own machine's output
```bash
# Run these four commands:
ip link show          # or: ifconfig -a
ip route show          # or: netstat -nr
ss -tlnp                # or: netstat -an | grep LISTEN
curl -v https://example.com 2>&1

# For each command's output, answer:
# 1. Which TCP/IP layer does this command primarily inspect?
# 2. Point to ONE specific line/field in the output that proves it belongs to that layer.
# 3. Which command's output would change if you plugged into a different Wi-Fi network?
#    Which would change if your app started listening on a new port?
```

### Exercise 2 — Medium: Watch encapsulation happen live
```bash
# In one terminal, start a tiny local server:
python3 -m http.server 8000
# or: npx http-server -p 8000

# In a second terminal, BEFORE making a request, check what's listening:
ss -tlnp | grep 8000
# This is pure TRANSPORT layer — a bound, listening socket, no connection yet.

# Now make a request while watching connections open in real time:
curl -v http://localhost:8000/ &
ss -tn | grep 8000
# You should briefly see an ESTABLISHED connection appear (TRANSPORT layer),
# while curl's -v output simultaneously shows the HTTP request/response (APPLICATION layer).

# Questions:
# 1. What TRANSPORT-layer state does the connection end up in after curl exits? (hint: try `ss -tn` again a few seconds later)
# 2. Why does localhost traffic never touch the LINK layer the way a real network request would?
#    (Hint: what interface does 127.0.0.1 route through? Check with `ip route get 127.0.0.1`.)
```

### Exercise 3 — Hard (Production Simulation): Diagnose by layer, blind
```bash
# Start a Node/Python server on port 4000 that responds to GET /health.
# Then, one at a time, break it THREE different ways and re-test with:
#   curl -v http://localhost:4000/health
#
# Break #1: Stop the server process entirely, then curl.
# Break #2: Keep the server running, but curl a WRONG port, e.g. 4001.
# Break #3: Keep the server running and correct, but curl a hostname that
#           doesn't exist, e.g. http://totally-fake-host-xyz:4000/health
#
# For each of the 3 breaks:
# 1. What is the exact error curl reports?
# 2. Which TCP/IP layer is actually responsible for that failure — Application,
#    Transport, Internet, or Link? Justify your answer using what you know
#    about what each layer is responsible for detecting/reporting.
# 3. Which of the three breaks would a firewall rule (Internet layer) also produce,
#    and how would you tell a firewall block apart from a genuinely dead server?
```

---

## Mental Model Checkpoint

Answer these from memory. If you can't, re-read the relevant section.

1. **Name the four TCP/IP layers in order, top to bottom, and give the one-line job of each.**

2. **OSI Layer 5 (Session) collapses into which TCP/IP layer? What about OSI Layer 2 (Data Link)?**

3. **Why does TLS live at the Application layer in the TCP/IP model, even though OSI puts it at Layer 6 (Presentation)?**

4. **Which TCP/IP layer does ARP belong to, and why is that answer genuinely disputed rather than clean?**

5. **A backend developer spends most of their career deep in which ONE TCP/IP layer? Which layer do they need only surface awareness of, and who typically owns that layer instead?**

6. **`ss -tlnp` shows a process listening on `127.0.0.1:5432` instead of `0.0.0.0:5432`. What does that mean for reachability from another container on the same host?**

7. **In the Kubernetes scenario, a `Connection refused` error pointed to which layer's problem, and a multi-second timeout on large payloads only (small ones worked) pointed to which layer's problem?**

---

## Quick Reference Card

| TCP/IP Layer | Corresponding OSI Layer(s) | Example Protocols | What a Backend Dev Does Here |
|---|---|---|---|
| **Application** | 7 (Application) + 6 (Presentation) + 5 (Session) | HTTP, DNS, TLS, SMTP, FTP, SSH, gRPC, WebSocket | Writes almost all their code here — routing, headers, auth, serialization |
| **Transport** | 4 (Transport) | TCP, UDP | Tunes socket options, keep-alive, connection pools; reads `ss`/`netstat` output |
| **Internet** | 3 (Network) | IP, ICMP | Reads route tables, understands VPC/subnet/NAT design (usually DevOps-owned) |
| **Link** | 2 (Data Link) + 1 (Physical) | Ethernet, Wi-Fi (802.11), PPP | Almost nothing directly; knows MTU can silently break large packets |

**Compact cheat block:**
```
Application  ← HTTP, DNS, TLS, gRPC, WebSocket        ← 90% of your code
Transport    ← TCP, UDP                                ← occasional tuning
Internet     ← IP, ICMP                                 ← mostly infra's job
Link         ← Ethernet, Wi-Fi, PPP                     ← almost never touched

ARP  = the exception that belongs to neither cleanly (Internet concern, Link framing)
"Layer 4 LB" / "Layer 7 LB"  = industry borrows OSI NUMBERS even in TCP/IP conversations
```

**Key commands, one per layer:**
```bash
ip link show      # LINK        — physical/virtual interfaces
ip route show     # INTERNET    — routing table
ss -tlnp          # TRANSPORT   — listening sockets
curl -v URL        # APPLICATION — DNS + TLS + HTTP in one shot
```

---

## When Would I Use This at Work?

### Scenario 1: A code review argument about "Layer 7 routing"
A teammate says the new API gateway does "Layer 7 routing." You now know they're borrowing OSI's numbering (Layer 7 = Application) to describe a TCP/IP-model behavior: the gateway reads the HTTP path/headers (Application layer) to decide where to route, not just the IP/port (which would be "Layer 4" / Transport-Internet routing, i.e., a plain TCP load balancer). This vocabulary mixing is completely normal in industry — now you can use it correctly instead of just nodding along.

### Scenario 2: "Pod-to-pod networking is broken" in Kubernetes
Instead of randomly restarting things, you diagnose top-down through the four layers exactly like Example 2 above: does DNS resolve (Application)? Does the port accept a connection (Transport)? Do the IPs actually route between nodes (Internet)? Is there an MTU mismatch on the overlay network (Link)? This ordering — cheapest/most-likely-culprit first — is the TCP/IP model turned into an actual debugging checklist.

### Scenario 3: Justifying a database security decision to a security reviewer
You explain that Postgres is bound to `127.0.0.1:5432` (visible via `ss -tlnp`, a Transport-layer fact) and additionally sits in a private subnet with no route to the internet (an Internet-layer fact, visible in the VPC route table). You're using two different TCP/IP layers together to make a real defense-in-depth argument, instead of relying on a single firewall rule.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the end-to-end journey this doc re-labels by layer
- `02-osi-model.md` — the 7-layer reference vocabulary this doc maps into 4 practical layers

**Study next:**
- `04-ip-addresses.md` — a full deep dive into the Internet layer: IPv4, IPv6, CIDR, NAT
- `05-ports.md` — a full deep dive into the Transport layer: what a port is, well-known ports, ephemeral ports

---

*Doc saved: `/docs/networking/03-tcp-ip-model.md`*
