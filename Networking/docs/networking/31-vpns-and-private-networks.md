# 31 — VPNs and Private Networks

> **Phase 5 — Topic 4 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `04-ip-addresses.md`

---

## ELI5 — The Simple Analogy

Imagine two office buildings in different cities. Employees in Building A need to walk into Building B's private rooms as if they were on the same floor — but the only road between them is a public highway full of strangers who can watch anything that passes.

So you build an **armored, sealed tunnel** from Building A's basement straight into Building B's basement. It runs *over* the same public highway (you didn't build a new road), but:
- Nobody on the highway can see what's inside the tunnel (**encryption**).
- Once you exit the tunnel, you're standing inside the other building's private hallway as if you'd always been there (**you get a private IP on their network**).

That armored tunnel is a **VPN** (Virtual Private Network). "Virtual" because there's no real dedicated road — you're faking a private connection over shared, untrusted public infrastructure.

A **private network** is the building itself: rooms only reachable from inside, with no door onto the public street. Your production database lives in one of those rooms. The only way in is through the tunnel or from another room in the same building. A stranger on the highway can't even find the door.

---

## Where This Lives in the Network Stack

A VPN is mostly a **Layer 3 (Network)** trick — it moves whole IP packets — with the encryption sitting at **Layer 3 or Layer 4** depending on the protocol.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (your HTTP request rides inside)       │
│  Layer 6 — Presentation  (OpenVPN wraps TLS here)               │  ← OpenVPN encryption
│  Layer 5 — Session                                              │
│  Layer 4 — Transport     (WireGuard/OpenVPN ride over UDP)      │  ← tunnel transport
│  Layer 3 — Network       (IPsec encrypts here; VPN moves IPs)   │  ← THE VPN LAYER
│  Layer 2 — Data Link                                            │
│  Layer 1 — Physical                                             │
└─────────────────────────────────────────────────────────────────┘
```

The mind-bending part: a VPN takes a **Layer 3 packet** and stuffs it *inside* another **Layer 3 packet**. You end up with an IP header wrapped around an (encrypted) IP header. The network stack runs twice — once for the real journey across the internet, once for the "virtual" journey inside the tunnel.

A **VPC** (Virtual Private Cloud) is not a protocol at all — it's a Layer 3 *construct*: a software-defined private network your cloud provider gives you, with its own IP address range (CIDR), subnets, and routing rules. It builds directly on everything from `04-ip-addresses.md`.

---

## What Is This?

Two distinct-but-related ideas that always show up together in production:

**1. A VPN (Virtual Private Network)** is a secure tunnel that makes two networks — or a single device and a network — behave as if they were directly, privately connected, even though the traffic actually crosses the public, untrusted internet. It does two jobs at once:
- **Encapsulation** — wraps your private packets so they can travel across networks that don't know about your private IP ranges.
- **Encryption** — scrambles the payload so nobody in between can read it.

There are two flavors:

| Flavor | Connects | Example |
|--------|----------|---------|
| **Remote-access VPN** | One device → a network | Your laptop → the company/cloud network |
| **Site-to-site VPN** | A whole network ↔ another network | Office data center ↔ AWS VPC |

**2. A private network** is a network with no direct route to or from the public internet. In the cloud this is a **VPC** — your own isolated slice of the provider's infrastructure, with a private CIDR block (like `10.0.0.0/16`), carved into **public subnets** (things that face the internet, like load balancers) and **private subnets** (things that must never face the internet, like databases).

The VPN is *how you reach into* the private network from outside. The private network is *what you're protecting*.

---

## Why Does It Matter for a Backend Developer?

This is not "ops stuff you can ignore." Every one of these is a real ticket you will get:

- **"The database rejected my connection from my laptop."** — because it's in a private subnet with no public IP, by design. You need a bastion or VPN.
- **"Our app can't reach the Stripe API in production but it worked in staging."** — the app is in a private subnet with no NAT Gateway, so it has no outbound internet route.
- **"Security flagged our database as internet-exposed."** — someone gave RDS a public IP "to make it reachable," and now the entire internet is brute-forcing it.
- **"The site-to-site VPN to the new office won't come up."** — both networks use `10.0.0.0/16`, so the routing is ambiguous (overlapping CIDR).
- **"I need to run a migration against production Postgres from my machine, safely."** — SSH tunnel through a bastion, or connect via the VPN.

If you can't answer "how does a packet get from my laptop to a database that has no public IP," you can't reason about production access, security reviews, or why staging behaves differently from prod. This topic is the foundation of every "how do I reach X securely" conversation.

---

## The Packet/Protocol Anatomy

### The key concept: encapsulation (a packet inside a packet)

Normal routing sends your packet as-is. A VPN takes your **original packet**, encrypts it, and wraps it in a **brand-new outer packet** addressed between the two VPN endpoints' *public* IPs. Routers on the internet only ever see (and route on) the outer header.

```
PLAIN ROUTING (no VPN) — your private packet cannot even cross the internet:

┌───────────────────────────────────────────────┐
│ IP HEADER                                      │
│  src: 10.0.1.5   (private)  ─┐                 │
│  dst: 10.8.0.9   (private)  ─┴─ NOT ROUTABLE   │  ← the public internet
│ ─────────────────────────────────  on the      │    will DROP this — private
│  TCP + HTTP payload (readable by anyone)       │    IPs aren't globally routable
└───────────────────────────────────────────────┘


VPN TUNNEL — the same packet, encapsulated and encrypted:

┌──────────────────────────────────────────────────────────────────┐
│ OUTER IP HEADER  (this is what the internet routes on)            │
│  src: 203.0.113.7   (your public IP)                             │
│  dst: 198.51.100.2  (VPN gateway's public IP)                    │
│  protocol: UDP (WireGuard/OpenVPN) or ESP/50 (IPsec)             │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ ENCRYPTED PAYLOAD  ░░░░ nobody in between can read this ░░░ │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │ INNER IP HEADER  (the ORIGINAL private packet)        │  │  │
│  │  │  src: 10.0.1.5    (private)                           │  │  │
│  │  │  dst: 10.8.0.9    (private)                           │  │  │
│  │  │ ──────────────────────────────────────────────────   │  │  │
│  │  │  TCP + HTTP payload (your actual request)            │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

Outer header = public, plaintext, routable.   Inner packet = private, encrypted.
```

At the far end, the VPN gateway **strips the outer header, decrypts the payload, and re-injects the inner packet** onto its local network. From the destination's point of view, a packet just arrived from `10.0.1.5` — it has no idea it crossed the internet.

### The protocols (brief and accurate)

| Protocol | Layer / transport | Notes |
|----------|-------------------|-------|
| **IPsec** | Layer 3; uses **ESP** (IP protocol 50) | The classic standard. Two modes: **tunnel mode** (wraps the whole original IP packet — used for VPNs) and **transport mode** (encrypts only the payload, keeps original IP header — used host-to-host). AWS/Azure site-to-site VPNs use IPsec. |
| **WireGuard** | Rides over **UDP** (default port `51820`) | Modern, tiny (~4k lines), very fast, simple key-based config. Increasingly the default for remote access. |
| **OpenVPN** | **TLS-based**; typically over **UDP 1194** (can do TCP 443 to punch through restrictive firewalls) | Older, flexible, battle-tested. Uses X.509 certs like HTTPS. |

Why UDP for the tunnel? TCP inside TCP causes "TCP meltdown" — two reliability layers fighting each other, each retransmitting. UDP as the outer transport lets the *inner* TCP handle reliability once. (OpenVPN over TCP 443 exists only to disguise VPN traffic as HTTPS through hostile firewalls.)

---

## How It Works — Step by Step

Let's trace a packet from your laptop through a remote-access VPN to a private server, then through a VPC to its final destination.

**Setup:** Your laptop is at cafe Wi-Fi (public IP `203.0.113.7`). Your company runs a WireGuard VPN gateway at `198.51.100.2`. Once connected, your laptop is handed a private VPN IP `10.8.0.9`. The internal server you want is `10.0.1.5`.

### Step 1 — Establish the tunnel
```
1a. Your VPN client sends a handshake to 198.51.100.2:51820 (over UDP)
1b. Both sides authenticate (WireGuard: public keys; OpenVPN: certs; IPsec: IKE)
1c. Both sides derive a shared symmetric encryption key
1d. Your OS creates a VIRTUAL network interface (e.g. wg0 / utun3) with IP 10.8.0.9
1e. Routes are installed: "to reach 10.0.0.0/16, send via wg0"
```

### Step 2 — Your app makes a request
```
Your app: fetch("http://10.0.1.5:8080/health")

2a. OS builds the ORIGINAL packet:
        src: 10.8.0.9    dst: 10.0.1.5   (both private)
2b. Route table says: 10.0.1.5 matches 10.0.0.0/16 → send out interface wg0
2c. wg0 is the VPN — it hands the packet to the VPN process, not the real NIC
```

### Step 3 — Encapsulate + encrypt (the tunnel)
```
3a. VPN process ENCRYPTS the original packet with the shared key
3b. VPN process wraps it in a NEW outer packet:
        src: 203.0.113.7   dst: 198.51.100.2   (both PUBLIC)
        protocol: UDP, dst port 51820
        payload: ░ encrypted { src:10.8.0.9 dst:10.0.1.5 + HTTP } ░
3c. THIS outer packet goes out your REAL Wi-Fi NIC onto the internet
```

### Step 4 — Across the internet (normal routing)
```
Routers see only the outer header: 203.0.113.7 → 198.51.100.2
They route it hop by hop like any other packet (see topic 01).
Nobody can read the inner packet. Your ISP sees "encrypted UDP to 198.51.100.2".
```

### Step 5 — Decapsulate at the gateway
```
5a. Gateway 198.51.100.2 receives the outer packet
5b. Strips the outer header
5c. Decrypts the payload with the shared key
5d. Recovers the ORIGINAL packet: src:10.8.0.9 dst:10.0.1.5
5e. Injects it onto the internal network as if it originated there
```

### Step 6 — Inside the VPC (route table + subnets)
```
6a. The packet is now on the 10.0.0.0/16 network, headed for 10.0.1.5
6b. The VPC route table forwards it within the private subnet
6c. The target's SECURITY GROUP checks: is 10.8.0.0/24 allowed on port 8080? (topic 32)
6d. Server 10.0.1.5 receives the packet, replies — same journey in reverse
```

### The full picture
```
┌──────────────┐   encrypted tunnel over public internet   ┌─────────────────────────────┐
│ Your laptop  │                                            │   VPC  10.0.0.0/16          │
│ real IP:     │   outer: 203.0.113.7 → 198.51.100.2       │  ┌───────────────────────┐  │
│ 203.0.113.7  │═══════════════════════════════════════════╪═▶│ VPN Gateway           │  │
│              │   ░ inner: 10.8.0.9 → 10.0.1.5 ░           │  │ 198.51.100.2 (public) │  │
│ VPN IP:      │                                            │  │ + 10.0.0.1 (internal) │  │
│ 10.8.0.9     │                                            │  └──────────┬────────────┘  │
└──────────────┘                                            │             │ 10.0.1.5      │
                                                            │      ┌──────▼──────┐        │
                                                            │      │ App server  │        │
                                                            │      │ 10.0.1.5    │        │
                                                            │      └─────────────┘        │
                                                            └─────────────────────────────┘
```

---

## Exact Syntax Breakdown

### SSH local port-forward (reach a private DB through a bastion)

The most common way a backend dev reaches a private database from a laptop without a full VPN:

```
ssh  -L  5432:db.internal:5432   ec2-user@bastion.example.com  -N
│    │   │    │          │       │                              │
│    │   │    │          │       │                              └── -N: no shell, just forward
│    │   │    │          │       └── the BASTION you log into (has a public IP)
│    │   │    │          └── remote port on db.internal to connect to
│    │   │    └── hostname the BASTION resolves/reaches (the private DB)
│    │   └── LOCAL port opened on YOUR laptop (127.0.0.1:5432)
│    └── -L = local forward: bind a local port, tunnel it to a remote target
└── ssh
```

What this actually does:
```
Your laptop 127.0.0.1:5432  ──(encrypted SSH tunnel)──►  bastion  ──►  db.internal:5432
        (you connect here)                              (has public IP)  (private, no public IP)

Now: psql -h 127.0.0.1 -p 5432   → really talks to the private DB.
The bastion does the "last hop" to db.internal from INSIDE the VPC.
```

### Conceptual WireGuard peer config

WireGuard config is deliberately tiny. Client side (`/etc/wireguard/wg0.conf`):

```ini
[Interface]                       # ── YOUR side (the laptop) ──
PrivateKey = <your-private-key>   # your secret key
Address    = 10.8.0.9/32          # the private IP you get INSIDE the tunnel
DNS        = 10.0.0.2             # use the VPC's internal DNS once connected

[Peer]                            # ── the GATEWAY you connect to ──
PublicKey  = <gateway-public-key> # gateway's public key (authenticates it)
Endpoint   = 198.51.100.2:51820   # gateway's PUBLIC ip:port (the outer dst)
AllowedIPs = 10.0.0.0/16          # WHICH traffic goes through the tunnel
                                  #   10.0.0.0/16 = only VPC traffic (split tunnel)
                                  #   0.0.0.0/0   = ALL traffic (full tunnel)
PersistentKeepalive = 25          # keep NAT mappings alive (seconds)
```

The single most important line is `AllowedIPs`. It's a **routing filter**: any destination inside it goes through the tunnel; anything else goes out your normal internet connection. `10.0.0.0/16` = "only tunnel VPC traffic" (a *split tunnel*). `0.0.0.0/0` = "tunnel everything, including my web browsing" (a *full tunnel*).

Bring it up / down:
```bash
wg-quick up wg0      # create wg0, apply config, install routes
wg-quick down wg0    # tear it all down
wg                   # show handshake status + bytes transferred
```

---

## Example 1 — Basic

**Goal:** Reach a Postgres database that lives in a private subnet (no public IP) from your laptop, using an SSH tunnel through a bastion host.

```bash
# 1. Open the tunnel. -L binds local:5432 → db.internal:5432 via the bastion.
#    -N means "don't open a shell, just hold the tunnel open."
#    -f would background it; we keep it foreground here to watch it.
ssh -L 5432:db.internal.mycompany.com:5432 ec2-user@bastion.mycompany.com -N

# Leave that running. In a SECOND terminal:

# 2. Connect to the DB as if it were on your own machine.
#    You point psql at 127.0.0.1 — the tunnel carries it to the real DB.
psql -h 127.0.0.1 -p 5432 -U appuser -d production
```

What you should see and understand:
```
$ psql -h 127.0.0.1 -p 5432 -U appuser -d production
Password for user appuser:
psql (16.2)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, ...)
production=>

# You are now querying a database that has NO public IP and CANNOT be reached
# directly from the internet. Every byte went: your psql → SSH tunnel (encrypted)
# → bastion → private subnet → Postgres. If you close the ssh command, the psql
# connection dies instantly — proving the traffic really rides the tunnel.
```

Prove the DB is genuinely unreachable directly:
```bash
# Try to reach the private DB hostname WITHOUT the tunnel — it will hang then fail.
psql -h db.internal.mycompany.com -p 5432 -U appuser -d production
# psql: error: connection to server ... failed: Operation timed out
#   ↑ the private hostname doesn't even resolve/route from outside the VPC.
```

---

## Example 2 — Production Scenario

Three real, related incidents you *will* hit. Each is a different facet of "private networks."

### Scenario A — "Our app can't call the Stripe API in production"

Your app runs on EC2 in a **private subnet** (correct — app servers shouldn't have public IPs). It needs to make **outbound** HTTPS calls to `api.stripe.com`. It worked in staging (which was in a public subnet) but times out in prod.

```bash
# From the app server (reached via bastion), test outbound:
curl -v --max-time 5 https://api.stripe.com/v1/charges
#   *   Trying 34.x.x.x:443...
#   * connect timed out            ← packets leave but never come back
```

**Root cause:** A private subnet has no route to the internet at all. It needs a **NAT Gateway** — a device in a *public* subnet that lets private resources make **outbound-only** connections (it does source NAT, remembers the flow, allows the reply back). It does **not** allow unsolicited **inbound** connections — which is exactly what you want for an app server.

```
WRONG mental model: "just add an Internet Gateway"
   Internet Gateway = bidirectional. It would also let the internet initiate
   connections INTO your app server. That defeats the private subnet.

RIGHT fix: NAT Gateway in the public subnet + a route in the private subnet.
```

The fix is one route-table entry:
```bash
# Private subnet's route table: send internet-bound traffic to the NAT Gateway
aws ec2 create-route \
  --route-table-id rtb-0privateaaa \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0abc123

# Now the flow is:
#   app (10.0.2.5, private)  →  NAT GW (10.0.1.9, public subnet)
#   → Internet Gateway  →  api.stripe.com
#   Replies come back the same path. Inbound-initiated traffic is still BLOCKED.
```

The correct topology:
```
                   Internet
                       │
                ┌──────▼───────┐
                │ Internet GW  │   (attached to the VPC)
                └──────┬───────┘
      PUBLIC subnet    │  10.0.1.0/24
      ┌────────────────▼─────────────────┐
      │  ALB (public IP)   NAT Gateway    │   ← NAT lives HERE, in public subnet
      └────────┬────────────────┬─────────┘
               │ forwards        │ outbound-only for private subnet
      PRIVATE subnet 10.0.2.0/24 │
      ┌────────▼─────────────────▼─────────┐
      │  App servers (NO public IP)         │
      │  10.0.2.5, 10.0.2.6 ...             │
      └────────┬────────────────────────────┘
               │
      PRIVATE subnet 10.0.3.0/24
      ┌────────▼────────────────────────────┐
      │  RDS Postgres (NO public IP)         │   ← DB reachable only from inside VPC
      │  10.0.3.10                           │
      └──────────────────────────────────────┘
```

### Scenario B — "I need to run a DB migration against private RDS from my laptop"

You can't (and shouldn't) give RDS a public IP. Use the SSH tunnel from Example 1, but for a real migration tool:

```bash
# Open the tunnel to the private RDS endpoint via the bastion
ssh -f -N -L 5432:mydb.abc123.us-east-1.rds.amazonaws.com:5432 \
    ec2-user@bastion.mycompany.com

# Point your migration tool at localhost — it rides the tunnel to RDS
DATABASE_URL="postgres://appuser:pass@127.0.0.1:5432/production" \
  npx prisma migrate deploy

#   Prisma connects to 127.0.0.1 → SSH tunnel → bastion → RDS 10.0.3.10
#   The migration runs against production without RDS ever touching the internet.
```

### Scenario C — "The site-to-site VPN to the new office won't come up"

You set up an IPsec site-to-site VPN between your AWS VPC and the new office. The tunnel establishes, but no traffic flows.

```bash
# AWS VPC CIDR:
aws ec2 describe-vpcs --query 'Vpcs[].CidrBlock'
#   [ "10.0.0.0/16" ]

# The office LAN, from the office router:
ip route
#   10.0.0.0/16 dev eth0 ...     ← THE OFFICE ALSO USES 10.0.0.0/16
```

**Root cause: overlapping CIDR.** Both networks claim `10.0.0.0/16`. When an office machine sends to `10.0.3.10` (meaning AWS RDS), its own router thinks `10.0.3.10` is *local* and never sends it through the tunnel. Routing is ambiguous — the address exists on both sides. See `04-ip-addresses.md`: private ranges are only unique *within* a network; the moment you join two networks they must not overlap.

**Fix:** Re-IP one side onto a non-overlapping range (e.g. move the VPC to `10.20.0.0/16`, or the office to `192.168.10.0/24`). There is no clever routing trick that makes two identical `/16`s coexist cleanly — NAT hacks exist but are fragile. Plan CIDR ranges up front so every network you might ever connect is unique.

---

## Common Mistakes

### Mistake 1: Giving a database a public IP "to make it reachable"

```
WRONG:  RDS with "Publicly accessible: Yes" + a security group open to 0.0.0.0/0:5432
        → Your production database is now on the public internet. Within HOURS
          automated scanners find port 5432 and start brute-forcing credentials.
          This is one of the most common ways databases get breached.

RIGHT:  Database in a PRIVATE subnet, NO public IP. Reachable only from:
          - app servers inside the VPC (security group allows the app's SG)
          - your laptop via a bastion SSH tunnel or a VPN into the VPC
```
The urge is understandable ("I just need to connect to it"), but the correct answer is a tunnel, never a public IP. A database should never be directly addressable from the internet.

---

### Mistake 2: Confusing NAT Gateway with Internet Gateway

```
Internet Gateway (IGW):  BIDIRECTIONAL. Lets resources with public IPs both
                         receive inbound and make outbound connections.
                         Use for: load balancers, bastions — things that MUST
                         accept inbound traffic.

NAT Gateway (NAT GW):    OUTBOUND-ONLY. Lets private resources initiate outbound
                         connections; replies return, but nobody can START a
                         connection into them.
                         Use for: app servers that need to call external APIs
                         but must NEVER be reachable from the internet.
```
Adding an IGW route to a private subnet doesn't just enable outbound — it makes those instances internet-reachable (if they have public IPs), silently undoing the "private" in private subnet. Use a NAT Gateway for outbound-only.

---

### Mistake 3: Overlapping CIDR ranges when peering or VPN-ing networks

```
VPC-A: 10.0.0.0/16     ──peering──►   VPC-B: 10.0.0.0/16
                                              ▲
                               Both use 10.0.0.0/16. A packet to 10.0.1.5 is
                               ambiguous — it exists on BOTH sides. The peering
                               connection can't even be created with overlapping
                               CIDRs; site-to-site VPNs "come up" but drop traffic.
```
Every network you might ever connect (VPCs, offices, partner networks) must use non-overlapping ranges. Carve up `10.0.0.0/8` deliberately: prod `10.0.0.0/16`, staging `10.1.0.0/16`, office `10.2.0.0/16`, etc. (This is a direct application of CIDR math from `04-ip-addresses.md`.)

---

### Mistake 4: Misunderstanding what a VPN actually protects

```
MYTH A: "A VPN adds encryption my HTTPS didn't have."
        → HTTPS is already end-to-end encrypted. Over a VPN, your HTTPS
          traffic is double-wrapped, but the VPN's value here isn't extra
          crypto for that request — it's NETWORK ACCESS (reaching private
          IPs) and hiding metadata from the local network.

MYTH B: "A VPN makes me anonymous."
        → No. It moves the point where your traffic exits from "your ISP"
          to "the VPN provider." The VPN operator now sees your traffic and
          your real IP. You've shifted trust, not removed it. The destination
          site still sees a request; it just comes from the VPN's exit IP.
```
A VPN's real jobs are **network reach** (talk to private IPs as if local) and **confidentiality from the local/transit network** (cafe Wi-Fi, your ISP). It is not anonymity, and it is not a substitute for TLS on individual connections.

---

### Mistake 5: Leaving a bastion wide open (`0.0.0.0/0` on port 22)

```
WRONG:  Bastion security group: allow tcp/22 from 0.0.0.0/0
        → The whole internet can attempt SSH. Your bastion — the single
          door into your private network — is now the most attacked host
          you own. It's the exact thing the private subnet was protecting.

RIGHT:  Restrict tcp/22 to known IPs (office CIDR, your VPN's exit IP),
        or eliminate the bastion entirely with AWS SSM Session Manager
        (no open inbound port at all — it dials outbound to AWS).
```
A bastion concentrates all access into one host, which is good — *if* that host is locked down. Open to `0.0.0.0/0`, it's a liability. Lock port 22 to specific source IPs. (This is exactly the security-group discipline of `32-firewalls-and-security-groups.md`.)

---

## Hands-On Proof

```bash
# 1. See your VPN's virtual interface (when connected to any VPN)
ip addr show          # Linux — look for wg0, tun0, or ppp0
ifconfig | grep utun  # macOS — VPNs create utunN interfaces
#   The virtual interface has your INNER/private VPN IP (e.g. 10.8.0.9).

# 2. See which routes the VPN installed (what actually goes through the tunnel)
ip route              # Linux
netstat -rn           # macOS/Linux — look for routes pointing at the VPN iface
#   A split tunnel shows only 10.0.0.0/16 → wg0.  A full tunnel shows 0.0.0.0/1.

# 3. Prove your public exit IP changes when the VPN is up
curl https://ifconfig.me    # note the IP
# (connect VPN)
curl https://ifconfig.me    # now it's the VPN gateway's public IP

# 4. Watch WireGuard handshakes and byte counters
sudo wg                     # shows latest handshake time + transfer bytes

# 5. Open an SSH tunnel and prove it carries traffic
ssh -L 8080:internal-service:8080 user@bastion -N &
curl http://127.0.0.1:8080/health   # really hits the PRIVATE service
kill %1                              # close tunnel → the port stops working

# 6. See the encapsulation on the wire (the tunnel from the OUTSIDE)
sudo tcpdump -n -i en0 'udp port 51820'   # WireGuard
#   You see only: 203.0.113.7.xxxx > 198.51.100.2.51820: UDP, length N
#   The inner IPs and payload are invisible — that's the encryption working.

# 7. Confirm a "private" host truly has no public route
#    (run from your laptop, NOT connected to the VPN)
ping 10.0.3.10      # a private VPC IP → 100% packet loss (not routable publicly)
```

---

## Practice Exercises

### Exercise 1 — Easy: Read a VPN's routes

```bash
# Connect to any VPN you have access to (corporate, or a personal one).
# Then run:
netstat -rn          # macOS/Linux
# OR
ip route             # Linux

# Answer:
# 1. Which interface is the VPN? (tun0 / wg0 / utunN)
# 2. Is it a SPLIT tunnel or a FULL tunnel? (Does 0.0.0.0/0 point at the VPN
#    interface, or only a specific private range like 10.0.0.0/16?)
# 3. What private IP did the VPN assign to you? (check `ifconfig` / `ip addr`)
# 4. Before and after connecting, run `curl https://ifconfig.me`. Did your
#    public IP change? Explain why (or why not, for a split tunnel + public dest).
```

### Exercise 2 — Medium: Draw the packet

```
Your laptop (public IP 203.0.113.7, VPN IP 10.8.0.9) is on a WireGuard VPN whose
gateway is 198.51.100.2:51820. You run:

    curl http://10.0.5.20:3000/api

Draw the OUTER packet and the INNER packet, filling in EVERY field you can:
  - Outer: src IP, dst IP, protocol, dst port
  - Inner: src IP, dst IP, dst port, encrypted? (yes/no)

Then answer:
  1. Which IPs does a router on the public internet see and route on?
  2. Which IPs can an eavesdropper on the cafe Wi-Fi read?
  3. If AllowedIPs was 10.0.0.0/16, does this curl go through the tunnel? Why?
  4. If you instead ran `curl https://google.com` on a SPLIT tunnel, does that
     traffic go through the VPN? What does it go out instead?
```

### Exercise 3 — Hard (Production Simulation): Design the network

```
You're the backend dev designing the VPC for a new service. Requirements:
  - A public-facing Application Load Balancer (accepts internet traffic on 443)
  - App servers that must NOT be reachable from the internet, but MUST be able
    to call an external payments API (outbound HTTPS)
  - A Postgres database that must be reachable ONLY by the app servers
  - You (a developer) must be able to run migrations from your laptop
  - The company office (LAN 192.168.50.0/24) needs a site-to-site VPN into the VPC

Produce:
  1. A CIDR plan: the VPC CIDR and at least 3 subnet CIDRs, labeled
     public/private. Make sure NONE overlap with the office's 192.168.50.0/24.
  2. For EACH subnet, its route-table entries (where does 0.0.0.0/0 point?).
  3. Where does the NAT Gateway live, and which subnets route through it?
  4. How do YOU run migrations from your laptop? (name the mechanism + the exact
     ssh command shape)
  5. One sentence: why does the DB have no public IP, and what is its ONLY
     inbound path?

# Hint: only ONE subnet type routes 0.0.0.0/0 to the Internet Gateway.
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **In one sentence, what does "encapsulation" mean for a VPN, and which header does a router on the public internet actually route on — the inner or the outer?**

2. **What is the single functional difference between a NAT Gateway and an Internet Gateway, and which one belongs on the route table of a private subnet that needs to call an external API?**

3. **A database is in a private subnet with no public IP. List the two legitimate ways a developer's laptop can reach it, and why a public IP is not one of them.**

4. **Two networks you want to connect via site-to-site VPN both use `10.0.0.0/16`. Why does traffic fail even after the tunnel comes up, and what's the only real fix?**

5. **A colleague says "we don't need HTTPS on internal calls because they go over the VPN." Is the VPN a substitute for TLS on each connection? Explain the difference between what the VPN protects and what TLS protects.**

6. **What does `AllowedIPs = 0.0.0.0/0` do in a WireGuard config versus `AllowedIPs = 10.0.0.0/16`? Use the terms "split tunnel" and "full tunnel."**

7. **You leave a bastion's security group open to `0.0.0.0/0` on port 22. Why does this undermine the entire point of having a private network, and name one alternative that has no open inbound port at all.**

---

## Quick Reference Card

| Concept | What it is | Key fact |
|---------|-----------|----------|
| VPN | Encrypted tunnel over public internet | Encapsulates + encrypts whole IP packets |
| Remote-access VPN | Device → network | Your laptop → the VPC |
| Site-to-site VPN | Network ↔ network | Office ↔ VPC (usually IPsec) |
| Encapsulation | Packet inside a packet | Outer = public IPs; inner = private, encrypted |
| IPsec | L3 VPN standard, uses ESP (proto 50) | Tunnel mode wraps the whole packet |
| WireGuard | Modern, fast, simple VPN | UDP port 51820; key-based config |
| OpenVPN | TLS-based VPN | UDP 1194 (or TCP 443 to disguise) |
| VPC | Your isolated cloud network | Has a CIDR, e.g. 10.0.0.0/16 |
| Public subnet | Routes to Internet Gateway | Holds ALB, NAT GW, bastion |
| Private subnet | No direct internet route | Holds app servers, databases |
| Internet Gateway | Bidirectional internet access | For things that accept inbound |
| NAT Gateway | Outbound-only internet access | Lives in a public subnet |
| Bastion / jump host | A locked-down SSH entry point | Lock port 22 to known IPs |
| Overlapping CIDR | Two networks share a range | Breaks peering/VPN — re-IP one side |

**Cheat block:**
```bash
# ── Reach a private DB via a bastion (SSH local forward) ──
ssh -L 5432:db.internal:5432 user@bastion -N   # then psql -h 127.0.0.1 -p 5432

# ── WireGuard ──
wg-quick up wg0        # bring tunnel up
wg-quick down wg0      # tear it down
sudo wg                # show handshake + bytes

# ── Inspect the tunnel ──
ip addr / ifconfig     # find the virtual iface + your VPN IP
ip route / netstat -rn # see what routes through the tunnel
curl https://ifconfig.me   # your public exit IP (changes on a full tunnel)

# ── Route rules of thumb ──
public subnet  → 0.0.0.0/0 via Internet Gateway
private subnet → 0.0.0.0/0 via NAT Gateway (outbound only) — or no route at all
database       → NO public IP, reachable only from within the VPC / via tunnel
```

**Decision guide:**
```
Need inbound from internet?        → public subnet + Internet Gateway
Need outbound-only from private?   → NAT Gateway + route in the private subnet
Reach one private host quickly?    → SSH local port-forward through a bastion
Reach the whole private network?   → VPN into the VPC (WireGuard/OpenVPN/IPsec)
Connect two whole networks?        → site-to-site VPN or VPC peering (no CIDR overlap!)
No open SSH port at all?           → AWS SSM Session Manager (outbound dial)
```

---

## When Would I Use This at Work?

### Scenario 1: "New engineer can't connect to the staging database"
You now know the answer isn't "give the DB a public IP." Staging DB is in a private subnet by design. Set them up with the bastion SSH tunnel (`ssh -L 5432:db.internal:5432 user@bastion -N`) or add them to the company VPN. Five minutes, zero attack surface added.

### Scenario 2: "Production app can't reach a third-party webhook/API"
Works in staging, times out in prod. You immediately check: is the app in a private subnet, and does that subnet's route table have a `0.0.0.0/0 → NAT Gateway` entry? Missing NAT Gateway is the classic cause. You add the NAT Gateway (not an Internet Gateway — you don't want inbound) and the route.

### Scenario 3: "Security review flagged an internet-exposed database"
Someone set RDS "publicly accessible" and opened the security group to `0.0.0.0/0:5432`. You migrate it into a private subnet, remove the public IP, and lock the security group to the app servers' security group only. Then you set up a bastion/VPN for the humans who legitimately need occasional access.

### Scenario 4: "We're acquiring a company and need to connect our networks"
Before anyone touches a VPN config, you check both sides' CIDR ranges. If they both use `10.0.0.0/16`, you flag it *now* — the site-to-site VPN will never route correctly until one side is re-IP'd. This one question saves weeks of confusing debugging.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — packets, routing, and encapsulation basics
- `04-ip-addresses.md` — private ranges, CIDR notation, NAT — the entire foundation for VPCs and the overlapping-CIDR problem

**Study alongside / after:**
- `32-firewalls-and-security-groups.md` — security groups control who reaches what *inside* the VPC; the natural partner to private subnets and bastions
- `36-service-to-service-communication.md` — once services live in private subnets, internal DNS and mTLS govern how they talk to each other securely
- `09-tls-ssl-in-depth.md` — why a VPN is not a replacement for per-connection TLS

**The big idea:** A VPN is encapsulation + encryption that lets private packets cross the public internet. A VPC is the private network worth protecting. Databases live deep inside it with no public IP, and every legitimate path in — bastion, SSH tunnel, VPN, or app server — is deliberate and narrow. Get this right and "who can reach what" stops being guesswork.

---

*Doc saved: `/docs/networking/31-vpns-and-private-networks.md`*
