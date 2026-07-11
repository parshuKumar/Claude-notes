# 04 — IP Addresses in Depth

> **Phase 1 — Topic 4 of 5**
> Prerequisites: `01-how-the-internet-works.md`, `02-osi-model.md`, `03-tcp-ip-model.md`

---

## ELI5 — The Simple Analogy

Imagine a huge apartment building in the middle of a city.

- The building has **one street address**: "742 Evergreen Terrace." That's the only address the outside world — the postal service, delivery drivers, your friends — knows about. This is your **public IP address**.
- Inside the building there are hundreds of apartments: 1A, 1B, 2A, 2B, ... 40Z. Nobody outside the building needs to know these numbers exist. They're only meaningful *inside* the building. This is your **private IP address**.
- At the front of the building sits a **front desk / concierge**. When a delivery arrives addressed to "742 Evergreen Terrace," the concierge doesn't know which apartment it's for just from the street address — so they keep a logbook: "the package that just left apartment 4B for the pizza place, I told the pizza place to deliver back to *me*, and I'll route it to 4B when it arrives." This logbook is **NAT (Network Address Translation)**.
- Two completely different apartment buildings, in two different cities, can both have an "Apartment 4B." That's fine — nobody outside either building ever addresses mail to "4B" directly. They always address it to the building's street address, and the front desk sorts it internally.

This is *exactly* how home networks, office networks, Docker containers, and AWS VPCs work:

```
Internet ("the city")
    │
    │  knows only the building's ONE public street address
    ▼
┌─────────────────────────────────────────────┐
│  Router / NAT Gateway  ("the front desk")    │
│  Public IP: 203.0.113.47                     │
│  Keeps a translation table:                  │
│    192.168.1.10:51234  ⇄  203.0.113.47:61000 │
│    192.168.1.15:52011  ⇄  203.0.113.47:61001 │
└─────────────────────────────────────────────┘
    │                    │                  │
    ▼                    ▼                  ▼
192.168.1.10        192.168.1.15       192.168.1.23
("Apt 4B")          ("Apt 7A")         ("Apt 12C")
Your laptop         Your phone         Your smart TV
```

Your laptop's `192.168.1.10` means nothing to Google's servers. Google only ever sees `203.0.113.47` — the front desk's address — and the front desk (your router) quietly rewrites every outbound packet's source address and translates every inbound reply back to the right apartment. That's the whole trick behind why the entire planet can reuse `192.168.1.x` without collisions.

---

## Where This Lives in the Network Stack

IP addressing is the core job of **Layer 3 (Network)** in OSI terms, and the **Internet layer** in TCP/IP terms. It sits directly below the transport layer (where ports and TCP/UDP live) and directly above the link layer (where MAC addresses and physical delivery live).

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, DNS, WebSocket)                 │
│  Layer 6 — Presentation  (TLS/SSL)                               │
│  Layer 5 — Session       (session management)                    │
│  Layer 4 — Transport     (TCP, UDP — ports live here)            │
├─────────────────────────────────────────────────────────────────┤
│▶ Layer 3 — Network       (IP — addresses live HERE)          ◀▶ │  ← this topic
├─────────────────────────────────────────────────────────────────┤
│  Layer 2 — Data Link     (Ethernet, Wi-Fi — MAC addresses)      │
│  Layer 1 — Physical      (cables, fiber, radio waves)           │
└─────────────────────────────────────────────────────────────────┘
```

Every router on the internet makes exactly one decision, over and over, billions of times a second: **"Given this packet's destination IP address, which direction do I send it next?"** That's Layer 3's entire job. It doesn't care about ports (Layer 4's job), it doesn't care about HTTP methods (Layer 7's job) — it only reads the IP header and forwards.

---

## What Is This?

**IPv4 (Internet Protocol version 4)** is a 32-bit address written as four decimal numbers (0–255) separated by dots — "dotted decimal notation" — e.g. `192.168.1.10`. 32 bits gives you 2³² = **4,294,967,296** possible addresses (~4.3 billion). That sounds like a lot until you remember there are ~8 billion humans, most carrying 2–3 internet-connected devices each. IPv4 address space ran out at the IANA (the global allocator) level in **February 2011** — every regional registry has since exhausted its free pool. This exhaustion is the single biggest reason NAT and private address ranges exist at all: instead of giving every device a unique public address, we give most devices private addresses and share a much smaller pool of public addresses via NAT.

**IPv6 (Internet Protocol version 6)** is a 128-bit address written as eight groups of four hex digits separated by colons, e.g. `2001:0db8:85a3:0000:0000:8a2e:0370:7334`. 128 bits gives you 2¹²⁸ ≈ **340 undecillion** addresses (3.4 × 10³⁸) — effectively unlimited for any foreseeable future. IPv6 was designed specifically to make NAT unnecessary (every device can get its own globally unique address), though in practice NAT-like patterns (and private ranges) still show up in IPv6 networks too — more on that in Common Mistakes.

**Public vs private address space.** Not every IP address is allowed to appear on the public internet. A set of ranges is reserved by RFC 1918 (IPv4) specifically to be **non-routable on the internet** — routers on the public internet are configured to drop or reject packets addressed to these ranges. That's what makes them safe to reuse in millions of unrelated networks simultaneously:

```
10.0.0.0/8        →  10.0.0.0    – 10.255.255.255    (16,777,216 addresses)
172.16.0.0/12     →  172.16.0.0  – 172.31.255.255     (1,048,576 addresses)
192.168.0.0/16    →  192.168.0.0 – 192.168.255.255       (65,536 addresses)
```

Anything else (with a few more special exceptions covered in the Quick Reference Card) is potentially a **public IP** — globally unique, routable, reachable directly from anywhere on the internet (firewalls permitting).

**CIDR notation** (Classless Inter-Domain Routing, RFC 1518/1519, 1993) is the `/24`-style suffix you see everywhere — e.g. `192.168.1.0/24`. It replaced the old rigid **classful** system (Class A = `/8`, Class B = `/16`, Class C = `/24`, fixed sizes only) which wasted enormous amounts of address space — a company needing 300 addresses had to be given an entire Class B block of 65,536. CIDR lets you carve a network into *any* size prefix (`/22`, `/27`, `/29` — anything), which is why every AWS VPC, every `ip addr show`, and every subnet you'll ever configure uses CIDR notation today.

A **subnet mask** is the mechanism CIDR notation is shorthand for — a 32-bit pattern of 1s (network portion) followed by 0s (host portion) that a device ANDs against an IP address to determine which addresses are "local" to it. `/24` and `255.255.255.0` are two notations for the exact same mask — full binary breakdown is in the next section.

---

## Why Does It Matter for a Backend Developer?

You will hit IP addressing decisions constantly, often without realizing it's what's biting you:

- **Docker bridge network conflicts.** Docker's default bridge uses `172.17.0.0/16`, and `docker-compose` allocates additional networks (`172.18.0.0/16`, `172.19.0.0/16`, ...) as you create more projects. If your corporate VPN *also* routes `172.1x.0.0/16`, your containers become unreachable — or worse, silently route traffic into the wrong place — because your machine can't tell whether `172.18.5.4` means "a Docker container" or "a machine on the corporate VPN."
- **Kubernetes pod/service CIDR ranges.** Every cluster reserves a CIDR block for pod IPs (commonly `10.244.0.0/16` for Flannel, or `100.64.0.0/16`-style ranges for some CNIs) and another for Service IPs (commonly `10.96.0.0/12` for kubeadm defaults). If that range overlaps your VPC's CIDR or an on-prem network you're peering with over a VPN, routing breaks in ways that are miserable to debug.
- **AWS VPC subnet design.** Deciding what's a "public subnet" vs a "private subnet" is 100% an IP addressing and routing decision — it determines whether your database is *reachable from the internet at all*, which is a security question, not just a networking one.
- **`0.0.0.0` vs `127.0.0.1` vs a real private IP when binding a server.** `app.listen(3000, '127.0.0.1')` inside a Docker container means "only reachable from inside this exact container's own network namespace" — even `docker run -p 3000:3000` won't be able to reach it. This single misunderstanding causes an enormous fraction of "my container works locally but not with Docker" bug reports.
- **Debugging "why can't my container/pod reach the internet."** Usually the answer lives in this topic: wrong subnet, missing route to a NAT gateway, or a private-subnet resource that was never supposed to reach the internet directly in the first place.

---

## The Packet/Protocol Anatomy

### IPv4 Address Structure

An IPv4 address is 32 bits, split into 4 **octets** (8 bits each), written in decimal and separated by dots:

```
        192      .     168     .      1      .      10
     ┌────────┐     ┌────────┐    ┌────────┐    ┌────────┐
     │ octet 1│     │ octet 2│    │ octet 3│    │ octet 4│
     │ 8 bits │     │ 8 bits │    │ 8 bits │    │ 8 bits │
     └────────┘     └────────┘    └────────┘    └────────┘
    11000000       10101000       00000001      00001010

Full 32-bit binary:  11000000.10101000.00000001.00001010
Decimal:                  192 .     168 .      1 .     10
```

Each octet ranges 0–255 (2⁸ = 256 possible values, 0-indexed). That's why you never see `192.168.1.400` — it's not representable in 8 bits.

### CIDR Notation, Piece by Piece

```
                192.168.1.0/24
                │              │
                │              └── prefix length: how many of the 32 bits,
                │                  counting from the LEFT, are the "network"
                │                  portion (fixed, shared by every host here)
                │
                └── the network address itself (host bits all zeroed out)

/24 means: first 24 bits = network, last 8 bits = host
     network bits (fixed)         host bits (variable, 0-255)
     ┌──────────────────────┐    ┌────────┐
      11000000.10101000.00000001 . 00000000
      └──────┘ └──────┘ └──────┘   └──────┘
       192       168       1          0
     ────────────── /24 ─────────────┘
```

With 8 host bits free, this subnet contains 2⁸ = 256 total addresses (`192.168.1.0` through `192.168.1.255`).

### Subnet Mask ANDing — Worked Binary Example

A subnet mask is just the CIDR prefix expressed as its own 32-bit number — 1s for every network bit, 0s for every host bit. `/24` **is** `255.255.255.0`:

```
/24  in binary:  11111111.11111111.11111111.00000000
/24  in decimal:      255 .     255 .     255 .      0
```

To find out which "network" an address belongs to, a device performs a **bitwise AND** between the IP address and the subnet mask:

```
   IP address:    192.168.1.10   =  11000000.10101000.00000001.00001010
   Subnet mask:   255.255.255.0  =  11111111.11111111.11111111.00000000
   ────────────────────────────────────────────────────────────────────
   AND result:    192.168.1.0    =  11000000.10101000.00000001.00000000
                  └───────────┘
                  This is the NETWORK ADDRESS — every device with a
                  192.168.1.x address on this /24 produces this same result.
```

Every bit position: `1 AND 1 = 1`, `1 AND 0 = 0`, `0 AND anything = 0`. The mask's 1-bits "pass through" the IP's bits unchanged; the mask's 0-bits force the result to 0, wiping out the host-specific portion. This single operation is how every router, every OS, and every switch decides "is this destination local to me or not" — you'll use it again in the next section.

### IPv6 Address Anatomy (Brief)

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
└──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘
 16   16   16   16   16   16   16   16   bits per group × 8 groups = 128 bits

Compressed form (leading zeros dropped, one run of all-zero groups → "::"):
2001:db8:85a3::8a2e:370:7334
```

Rules worth knowing:
- `::` can be used **only once** per address (otherwise it's ambiguous how many zero-groups it represents).
- `::1` is the IPv6 loopback (equivalent of `127.0.0.1`).
- `::` alone is the "unspecified address" (equivalent of `0.0.0.0`).
- `fe80::/10` is **link-local** — every IPv6 interface auto-assigns one; not routable beyond the local link.
- `fc00::/7` (in practice `fd00::/8`) is **Unique Local Address (ULA)** space — IPv6's rough equivalent of RFC 1918 private ranges, not routed on the public internet.
- `2000::/3` is the current publicly routable **global unicast** range.

---

## How It Works — Step by Step

Every device on a network (your laptop, an EC2 instance, a container) is configured with three pieces of information — via DHCP or statically:
1. Its own **IP address** (e.g. `192.168.1.50`)
2. Its own **subnet mask / CIDR prefix** (e.g. `/24`)
3. A **default gateway** — the router to hand off anything that isn't local (e.g. `192.168.1.1`)

When your device wants to send a packet, it must decide: *"Can I deliver this directly on my local network segment, or does it have to go through the router?"*

```
Host:            192.168.1.50 / 255.255.255.0
Default gateway: 192.168.1.1

Step 1 — Compute MY network address:
   192.168.1.50  AND  255.255.255.0  =  192.168.1.0

Step 2 — Compute DESTINATION's network address (using MY mask):
```

**Case A — destination `192.168.1.75` (same subnet):**
```
   192.168.1.75  AND  255.255.255.0  =  192.168.1.0

   192.168.1.0  ==  192.168.1.0   →  MATCH → destination is LOCAL

Step 3a: Device broadcasts an ARP request: "Who has 192.168.1.75?"
Step 3b: 192.168.1.75 replies with its MAC address
Step 3c: Packet is sent DIRECTLY over the local network (Ethernet/Wi-Fi),
         no router involved. Destination IP in the packet = 192.168.1.75.
         Destination MAC in the frame = 192.168.1.75's own MAC.
```

**Case B — destination `93.184.216.34` (example.com, out on the internet):**
```
   93.184.216.34  AND  255.255.255.0  =  93.184.216.0

   192.168.1.0  !=  93.184.216.0   →  NO MATCH → destination is REMOTE

Step 3a: Device already knows (via ARP) the MAC address of its
         default gateway, 192.168.1.1
Step 3b: Packet is sent to the GATEWAY's MAC address at Layer 2 —
         but the packet's DESTINATION IP is still 93.184.216.34, unchanged!
Step 3c: The gateway receives it, looks at ITS OWN routing table,
         and forwards it toward the next hop — repeating this exact
         same local-vs-remote decision at every router along the path.
```

This is the single most important mental model in this entire topic: **the destination IP never changes as a packet crosses the internet, but the destination MAC address changes at every hop.** The IP address is the "where it's ultimately going" (Layer 3, end-to-end); the MAC address is the "who's the very next physical device to hand it to" (Layer 2, hop-by-hop). Confusing these two is the root of a lot of networking confusion — see topic `02-osi-model.md` for the full Layer 2 vs Layer 3 breakdown.

---

## Exact Syntax Breakdown

### `192.168.1.0/24` — Piece by Piece
```
192.168.1.0  /  24
│              │
│              └── prefix length in bits (0–32). Higher number = smaller
│                  network (fewer host bits = fewer usable addresses).
│
└── the network address: the IP with ALL host bits set to zero.
    This is never assigned to a real device — it identifies the subnet itself.
```

### `ip addr show` — Line by Line
```bash
$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link
       valid_lft forever preferred_lft forever
```
```
2: eth0:                     ← interface index and name
<BROADCAST,MULTICAST,UP,...> ← interface flags (UP = active, LOWER_UP = link detected)
mtu 1500                     ← Maximum Transmission Unit — largest packet size, in bytes
link/ether 02:42:ac:...      ← this interface's MAC address (Layer 2)
inet 172.17.0.2/16           ← IPv4 address + CIDR prefix (Layer 3)  ← what THIS topic is about
brd 172.17.255.255           ← this subnet's broadcast address
inet6 fe80::.../64           ← auto-assigned IPv6 link-local address
scope global                 ← reachable beyond just this host (vs "scope link")
```

### Subnet Mask: Two Notations, Same Thing
```
CIDR prefix   Dotted-decimal mask     Binary
   /24     =    255.255.255.0     =  11111111.11111111.11111111.00000000
   /16     =    255.255.0.0       =  11111111.11111111.00000000.00000000
   /8      =    255.0.0.0         =  11111111.00000000.00000000.00000000
   /25     =    255.255.255.128   =  11111111.11111111.11111111.10000000
   /28     =    255.255.255.240   =  11111111.11111111.11111111.11110000
```
Linux tools (`ip addr`) almost always show `/24`. Older tools and some GUIs (`ifconfig`, Windows `ipconfig`, router admin panels) show `255.255.255.0`. They are mathematically identical — just count the leading 1-bits to convert between them.

---

## Example 1 — Basic: Calculate a Subnet by Hand

**Given:** `10.0.4.0/22` — find the network address, broadcast address, and usable host range.

**Step 1 — Convert the prefix to a mask.**
```
/22  =  22 ones followed by 10 zeros
     =  11111111.11111111.11111100.00000000
     =  255.255.252.0
```

**Step 2 — Confirm `10.0.4.0` is a valid network boundary for this mask.**
```
Third octet mask bits: 11111100  →  block size = 256 − 252 = 4
Valid third-octet network boundaries: 0, 4, 8, 12, 16, 20, ... (multiples of 4)
Third octet given is 4 → yes, valid boundary. ✓
```

**Step 3 — Determine the range this subnet covers.**
```
Block size = 4 in the third octet, so this subnet spans third-octet
values 4, 5, 6, and 7 — with ALL 256 values of the fourth octet in each.

Total addresses = 2^(32-22) = 2^10 = 1,024

First address (Network Address):    10.0.4.0
Last address  (Broadcast Address):  10.0.7.255
```

**Step 4 — Subtract network + broadcast to get usable hosts.**
```
Total addresses:            1,024
Minus network address:       -  1   (10.0.4.0   — identifies the subnet, unusable)
Minus broadcast address:     -  1   (10.0.7.255 — "send to everyone", unusable)
────────────────────────────────
Usable host range:          1,022  addresses

Usable range: 10.0.4.1  →  10.0.7.254
```

**Binary sanity check (network address):**
```
   10.0.4.0        =  00001010.00000000.00000100.00000000
   255.255.252.0   =  11111111.11111111.11111100.00000000
   AND              =  00001010.00000000.00000100.00000000  =  10.0.4.0  ✓
```

---

## Example 2 — Production Scenario: Designing an AWS VPC

**The situation:** You're standing up a new backend service on AWS: an API behind a load balancer, a fleet of app servers, and an RDS Postgres database. You need to decide the subnet layout.

```
VPC: 10.0.0.0/16   (65,536 addresses total)   region: us-east-1
│
├── Public Subnet    10.0.1.0/24   (us-east-1a)
│   Contents: Application Load Balancer, NAT Gateway, bastion host
│   Route table:  10.0.0.0/16 → local
│                 0.0.0.0/0   → Internet Gateway (igw-0abc123)
│   → Instances here CAN have a public IP and ARE directly reachable
│     from the internet (subject to security groups).
│
├── Private Subnet   10.0.2.0/24   (us-east-1a)
│   Contents: App servers (EC2/ECS/EKS nodes running your Node.js app)
│   Route table:  10.0.0.0/16 → local
│                 0.0.0.0/0   → NAT Gateway (nat-0def456, sits in the public subnet)
│   → Instances here have NO public IP. They can reach OUT to the
│     internet (npm install, calling external APIs) via the NAT Gateway,
│     but nothing on the internet can initiate a connection IN to them.
│
└── Private Subnet   10.0.3.0/24   (us-east-1a)  — "isolated" / DB subnet
    Contents: RDS Postgres (primary + read replica)
    Route table:  10.0.0.0/16 → local
                  (no 0.0.0.0/0 route at all)
    → No path to the internet in EITHER direction. Only reachable from
      inside the VPC — i.e., from the app servers in 10.0.2.0/24, or
      from an engineer connected via bastion host or VPN.
```

**Why the database has no public IP:** "Public vs private subnet" in AWS isn't a property you set on the subnet directly — it's entirely determined by the **route table**. A subnet is "public" only if its route table sends `0.0.0.0/0` traffic to an **Internet Gateway**. RDS instances in `10.0.3.0/24` simply have no route to an IGW at all, so even if you fat-fingered a public IP onto one, there would be no path in or out. This is defense in depth: network topology enforces the "database should never be internet-facing" rule at a layer no application bug can override.

**How the NAT Gateway threads the needle:** the NAT Gateway lives in the *public* subnet and has an Elastic (public) IP. When an app server in `10.0.2.0/24` calls `npm install`, its packet's source address gets rewritten by the NAT Gateway to the NAT Gateway's public IP before it leaves the VPC — the exact same translation trick from the ELI5 section. The npm registry replies to the NAT Gateway's public IP; the NAT Gateway looks up which private instance made that request and forwards the reply back. But — critically — NAT Gateway only translates **outbound-initiated** connections. Nobody outside can initiate a *new* connection to an app server through it. That asymmetry (outbound yes, inbound no) is what makes "private subnet + NAT Gateway" the standard pattern for app servers that need internet access but must never be directly attackable from it.

---

## Common Mistakes

### Mistake 1: Assuming your private IP is globally unique
```
Belief:  "My laptop's IP is 192.168.1.5, that must uniquely identify my machine."
Reality: Millions of home networks worldwide are ALSO using 192.168.1.5 right
         now. It's only unique within YOUR local network. The only globally
         unique thing about your connection is your router's public IP
         (assigned by your ISP), which NAT translates on your behalf.
```
Verify it yourself: `ip addr show` (your private IP) vs `curl https://ifconfig.me` (your actual public IP) will almost always print two completely different addresses.

### Mistake 2: Confusing network/broadcast addresses with usable hosts
```
Subnet: 192.168.1.0/24

WRONG:  "This subnet has 256 usable hosts, from .0 to .255"
RIGHT:  192.168.1.0    = network address    — reserved, never assign to a host
        192.168.1.255  = broadcast address  — reserved, never assign to a host
        192.168.1.1 – 192.168.1.254         — 254 usable host addresses

Assigning a server the IP 192.168.1.255 will silently break — packets meant
for just that server get delivered to (or interfere with) every device on
the subnet, because .255 means "broadcast to everyone" at the IP layer.
```
(Note: `/31` and `/32` are special-cased exceptions used for point-to-point links and single-host routes — see the Quick Reference Card.)

### Mistake 3: Thinking IPv6 adoption removes the need to understand NAT/private ranges
```
Belief:  "IPv6 gives everything a public address, so private ranges and NAT
          are legacy knowledge I can skip."
Reality: Most production networks in 2026 are still dual-stack (IPv4 + IPv6)
         or IPv4-only. Cloud VPCs, Docker, Kubernetes, and most corporate
         networks are still fundamentally built on RFC 1918 + NAT today.
         Even IPv6 networks commonly use ULA (fc00::/7) private-style
         addressing internally and NAT66/NPTv6 in some enterprise designs.
         You cannot debug a modern backend's networking without IPv4/NAT
         fluency — it isn't going away on any timeline you'll work within.
```

### Mistake 4: Binding to `127.0.0.1` instead of `0.0.0.0` inside Docker
```javascript
// Inside a Dockerfile / container entrypoint:
app.listen(3000, '127.0.0.1');   // ✗ WRONG for containers
app.listen(3000, '0.0.0.0');     // ✓ CORRECT for containers
```
```
127.0.0.1 inside a container means "only reachable from THIS container's
own network namespace." Docker's port mapping (-p 3000:3000) forwards
traffic to the container's external-facing interface, NOT to its loopback.
Even though `docker run -p 3000:3000 myapp` looks correct, curling
http://localhost:3000 from your host machine will hang or get connection
refused, because nothing is actually listening on the interface Docker's
NAT rule is targeting.

0.0.0.0 means "listen on ALL interfaces of this container" — including
the one Docker's port-forwarding rule talks to. Locally on your own
laptop (no Docker), 127.0.0.1 is usually fine since your terminal and
your server share the same loopback. Inside a container, it almost
never is.
```

### Mistake 5: Assuming any `172.x.x.x` address is private
```
Belief:  "172 is one of the private prefixes, so any 172.x.x.x address is
          internal/safe."
Reality: Only 172.16.0.0 – 172.31.255.255 (172.16.0.0/12) is private.
         172.32.0.0 and above — e.g. 172.64.0.0, 172.217.0.0 (used by
         some real Google services) — are ordinary PUBLIC addresses.
         Second-octet range 16-31 only. This trips people up constantly
         when eyeballing IP lists instead of doing the actual CIDR math.
```

---

## Hands-On Proof

```bash
# 1. See your machine's private IP address(es) and interfaces
ip addr show          # Linux (modern)
ifconfig               # macOS / older Linux
# Look for "inet" lines — that's your IPv4 address on each interface.

# 2. See your routing table — where does traffic go by default?
ip route show          # Linux
netstat -rn             # macOS
# Look for the line starting "default" — that's your gateway, the
# device everything non-local gets handed off to.

# 3. Ping a private IP — only works within your own local network
ping -c 4 192.168.1.1   # typically your home router's gateway IP
# 64 bytes from 192.168.1.1: icmp_seq=0 ttl=64 time=1.234 ms
# This IP is meaningless to anyone outside your house. Someone else's
# 192.168.1.1, in a different city, is a completely different device.

# 4. See the public IP NAT presents to the world
curl https://ifconfig.me
# 203.0.113.47
# Compare this to step 1's output — they will almost never match.
# This is the ONE address the entire internet can use to reach you,
# and your router's NAT table quietly maps it back to your machine.

# 5. Inspect Docker's default bridge network and its CIDR block
docker network inspect bridge
# [
#     {
#         "Name": "bridge",
#         "Driver": "bridge",
#         "IPAM": {
#             "Config": [
#                 { "Subnet": "172.17.0.0/16", "Gateway": "172.17.0.1" }
#             ]
#         },
#         "Containers": {
#             "3e1c9f...": {
#                 "Name": "my-node-app",
#                 "IPv4Address": "172.17.0.2/16"
#             }
#         }
#     }
# ]
# Every container gets a private IP inside 172.17.0.0/16. docker0 (the
# bridge, 172.17.0.1) acts as that mini-network's gateway/NAT device —
# same pattern as your home router, one layer deeper.

# 6. Confirm the AND-with-mask logic yourself
python3 -c "
ip = '192.168.1.200'
mask = '255.255.255.0'
ip_bytes = [int(x) for x in ip.split('.')]
mask_bytes = [int(x) for x in mask.split('.')]
network = [ip_bytes[i] & mask_bytes[i] for i in range(4)]
print('.'.join(str(x) for x in network))
"
# 192.168.1.0
```

---

## Practice Exercises

### Exercise 1 — Easy: Subnet math by hand
```
Given the CIDR block:  172.16.16.0/20

Calculate, showing your binary work:
1. The subnet mask, in both /prefix and dotted-decimal form
2. The total number of addresses in this block
3. The network address
4. The broadcast address
5. The usable host range, and how many usable hosts that is

(Check yourself: the block size in the third octet should come out
to 16 — meaning valid network boundaries are .0, .16, .32, .48, ...
Confirm 172.16.16.0 lands exactly on one of those boundaries.)
```

### Exercise 2 — Medium: Public or private?
```
Classify each address below as PUBLIC or PRIVATE (non-routable),
and briefly justify each answer using the ranges from this topic:

1.  10.0.5.23
2.  172.20.10.5
3.  8.8.8.8
4.  192.168.50.2
5.  172.32.5.5      ← careful, this one is a common trap
6.  169.254.1.1      ← is this the same thing as "private"? why or why not?
7.  100.64.1.5        ← research this one; it's neither classic RFC 1918
                         private space NOR ordinary public space
8.  127.0.0.1
```

### Exercise 3 — Hard (Production Simulation): Design a 3-tier VPC
```
You're designing the network for a new production service on AWS:
- CIDR budget: you've been allocated 10.20.0.0/16 for the whole VPC
- The service needs to run across TWO availability zones for redundancy
- Three tiers: a public-facing load balancer tier, a private application
  tier, and a private, fully isolated database tier

Your task:
1. Split 10.20.0.0/16 into subnets — one of each tier PER availability
   zone (6 subnets total). Use /24 blocks. Write out each subnet's CIDR.
2. For each subnet, state: does its route table point 0.0.0.0/0 at an
   Internet Gateway, a NAT Gateway, or nothing at all?
3. An engineer needs to run a one-off migration script against the
   database from their laptop. The database subnet has no route to the
   internet at all. Name TWO legitimate ways to reach it anyway
   (hint: this topic's ELI5 mentioned a front desk — what's the
   equivalent of "someone let in through a side door"?)
4. A teammate suggests putting the RDS database in the PUBLIC subnet
   "to make it easier to connect to from our laptops." Explain, in
   networking terms (not just "it's insecure"), exactly what changes
   about the database's reachability if you do this.
```

---

## Mental Model Checkpoint

Answer these from memory. If you can't, re-read the relevant section.

1. **Two unrelated home networks both use `192.168.1.5` for a laptop. Why does this never cause a conflict on the public internet?**
2. **Given a subnet, what's the difference between its network address, its broadcast address, and its usable host range? Which two addresses can never be assigned to a real device?**
3. **A device has IP `10.0.2.15/24` and default gateway `10.0.2.1`. It wants to send a packet to `10.0.2.200` and, separately, to `52.94.0.1`. Walk through the AND-with-mask decision for each — which one goes straight to the destination, and which one goes to the gateway first?**
4. **Why would a production database in an AWS private subnet have no public IP address, and what specifically prevents someone on the internet from reaching it directly?**
5. **Why does binding a Node.js server to `127.0.0.1` inside a Docker container break `docker run -p 3000:3000`, when the exact same bind works fine for a server running directly on your laptop?**
6. **What does `/24` literally mean, in terms of bits? What does `/22` mean, and how many total addresses does it contain?**
7. **Why doesn't the growing rollout of IPv6 make RFC 1918 private ranges and NAT irrelevant knowledge for a backend developer today?**

---

## Quick Reference Card

**Reserved IPv4 ranges:**

| Range | CIDR | # Addresses | Purpose |
|---|---|---|---|
| 10.0.0.0 – 10.255.255.255 | `10.0.0.0/8` | 16,777,216 | Private — large networks, AWS VPCs, k8s clusters |
| 172.16.0.0 – 172.31.255.255 | `172.16.0.0/12` | 1,048,576 | Private — Docker's default bridge range lives here |
| 192.168.0.0 – 192.168.255.255 | `192.168.0.0/16` | 65,536 | Private — home routers, small office LANs |
| 127.0.0.0 – 127.255.255.255 | `127.0.0.0/8` | 16,777,216 | Loopback — `127.0.0.1` = "this machine," never leaves the host |
| 169.254.0.0 – 169.254.255.255 | `169.254.0.0/16` | 65,536 | Link-local (APIPA) — auto-assigned when DHCP fails; NOT the same as "private" |

**CIDR-to-host-count cheat sheet:**

| CIDR | Total addresses | Usable hosts (total − 2) |
|---|---|---|
| `/32` | 1 | 1 (single host route, no subtraction) |
| `/31` | 2 | 2 (point-to-point link, RFC 3021, no subtraction) |
| `/30` | 4 | 2 |
| `/29` | 8 | 6 |
| `/28` | 16 | 14 |
| `/27` | 32 | 30 |
| `/26` | 64 | 62 |
| `/25` | 128 | 126 |
| `/24` | 256 | 254 |
| `/23` | 512 | 510 |
| `/22` | 1,024 | 1,022 |
| `/20` | 4,096 | 4,094 |
| `/16` | 65,536 | 65,534 |
| `/12` | 1,048,576 | 1,048,574 |
| `/8` | 16,777,216 | 16,777,214 |

**Key commands:**
```bash
ip addr show                  # your IP addresses per interface (Linux)
ifconfig                       # same idea (macOS / older Linux)
ip route show                  # your routing table / default gateway
curl https://ifconfig.me       # the public IP NAT presents to the world
docker network inspect bridge  # Docker's private subnet + container IPs
```

**Special single addresses worth memorizing:**
```
0.0.0.0          "all interfaces" when binding a server;
                 "default route" when seen in a routing table
127.0.0.1        loopback — always means "this machine, right now"
255.255.255.255  limited broadcast — "everyone on this local link"
::1              IPv6 loopback (equivalent of 127.0.0.1)
```

---

## When Would I Use This at Work?

### Scenario 1: Designing a new service's VPC
Every time you spin up infrastructure for a new production service, you're making Layer 3 decisions: how big should the VPC's CIDR block be (leave room to grow — `/16` is standard), how many AZs, which subnets are public (load balancer, NAT gateway) vs private (app tier, database tier). Getting this wrong early means a painful re-IP migration later — CIDR blocks are hard to resize once other subnets, peering connections, and route tables depend on them.

### Scenario 2: "My Docker containers can't reach the internet" after connecting to the company VPN
Classic real-world bug: your VPN client pushes routes for `172.20.0.0/16` (or similar), and Docker happens to have allocated a network in the same range for one of your `docker-compose` projects. Your machine's routing table now has an ambiguous/overlapping route, and traffic silently goes to the wrong place. The fix is knowing exactly what this topic taught: inspect `docker network inspect`, compare against `ip route show` while the VPN is connected, and either change Docker's subnet (`docker network create --subnet=...`) or ask your network team to route around the conflict.

### Scenario 3: Kubernetes cluster CIDR overlaps during VPN/VPC peering
You're peering your EKS cluster's VPC with an on-prem network (or another VPC) over a VPN or Transit Gateway so services can talk to each other privately. If the cluster's pod CIDR (e.g. `10.244.0.0/16`) overlaps the CIDR on the other side of the peering connection, routing becomes ambiguous and packets get silently dropped or misdelivered. This is a top-5 real-world reason cross-network Kubernetes connectivity fails, and diagnosing it requires exactly the subnet-math fluency this topic covers — comparing CIDR blocks for overlap before you ever open a support ticket.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the big picture this topic zooms into
- `02-osi-model.md` — Layer 3 vs Layer 2 vs Layer 4, why IP and MAC addresses are different tools for different jobs
- `03-tcp-ip-model.md` — where the Internet layer sits in the 4-layer model backend developers actually use daily

**Study next:**
- `05-ports.md` — IP address gets you to the right *machine*; the port gets you to the right *process* on it

**Builds directly on this topic later:**
- `31-vpns-and-private-networks.md` — tunneling between private networks, the VPC/peering concepts touched on in Example 2 and Scenario 3 above, covered in full depth
- `32-firewalls-and-security-groups.md` — what `0.0.0.0/0` means in a security group rule, inbound vs outbound rules, and how they interact with the public/private subnet design from this topic

---

*Doc saved: `/docs/networking/04-ip-addresses.md`*
