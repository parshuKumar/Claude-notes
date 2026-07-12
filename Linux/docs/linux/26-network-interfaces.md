# 26 — Network Interfaces

## ELI5 — The Simple Analogy

Imagine a large office building.

- **The front door** is `eth0` (or `ens3`) — the door to the street. Everything going to or from the outside world passes through it.
- **The intercom** is `lo`, the loopback interface. You can call room 204 from room 118, and the call never leaves the building. It never touches the street. If someone outside dials the intercom, they get nothing — **the intercom has no connection to the outside at all.**
- **The internal courtyard door** is `docker0` — a door into a private courtyard where all your containers live. They talk to each other there. To reach the street, their mail has to be re-addressed and carried out the front door.
- **A private corridor between two rooms** is a `veth` pair — a tube with exactly two ends. Anything you push in one end pops out the other. One end is in the container, one end is in the host.
- **The receptionist's rulebook** is the routing table: "Mail for the 3rd floor → internal stairs. Mail for anywhere else → out the front door, hand it to the postman (the gateway)."
- **A separate apartment inside the building** is a network namespace. It has its OWN front door, its OWN intercom, its OWN receptionist, its OWN rulebook — and its **own room numbering**. Which is why apartment A can have a "room 3000" and apartment B can ALSO have a "room 3000" and nobody collides.

That last one is the whole reason two Docker containers can both bind port 3000.

---

## Where This Lives in the Linux Stack

```
Hardware (NIC — the actual network card, or a virtualized one in your cloud VM)
  └── KERNEL
       │    ├── NIC driver (e1000, virtio_net, ixgbe) — talks to the card
       │    ├── net_device struct  ◀◀◀ THIS TOPIC — "the interface" IS this kernel object
       │    ├── Routing table (FIB) ◀◀◀ THIS TOPIC — which interface does a packet leave by?
       │    ├── Neighbour table (ARP cache) ◀◀◀ THIS TOPIC
       │    ├── Network namespaces ◀◀◀ THIS TOPIC — isolated copies of ALL of the above
       │    └── TCP/IP stack (topic 28 lives here)
       │
       └── System Calls (socket(), bind(), ioctl(), and netlink sockets)
            │
            └── C Library (glibc)
                 │
                 └── iproute2 tools: ip, ss, bridge  ◀◀◀ THIS TOPIC (the commands)
                      │
                      └── Your Node app: server.listen(3000, '0.0.0.0')
```

**Key insight:** `ip` does not "configure the network." It sends a **netlink** message to the kernel asking the kernel to change a `net_device` or a routing entry. The kernel owns everything. `ip` is just a well-designed remote control.

---

## What Is This?

A **network interface** is a kernel object (a `struct net_device`) that represents one endpoint on a network. It has a name (`eth0`), a link-layer address (a MAC), an MTU, an up/down state, zero or more IP addresses, and packet counters.

Most interfaces are backed by a real NIC and its driver. But many are **purely software**: `lo` (loopback), `docker0` (a bridge), `veth123` (one end of a virtual cable), `wg0` (WireGuard), `tun0` (a VPN). The kernel treats all of them identically — they all send and receive packets, they all appear in `ip addr`, they can all have routes pointing at them.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| loopback vs 0.0.0.0 | Your Node app runs, the firewall is open, and it is still **unreachable from outside**. You will lose hours. |
| Network namespaces | Docker networking will be permanent magic to you |
| `localhost` inside a container | You'll try to reach a host Postgres at `127.0.0.1:5432` from a container and get `ECONNREFUSED` forever |
| The routing table | You won't be able to answer "why is my traffic leaving via the wrong interface / the VPN / not the VPN" |
| MTU and PMTU discovery | You'll hit the "SSH connects then hangs" / "small requests work, big ones stall" black hole and have no idea what happened |
| `ip` vs `ifconfig` | You'll `exec` into a container, type `ifconfig`, get `command not found`, and think the container is broken |
| ICMP being blocked | You'll declare a host "down" because `ping` failed, when the host is perfectly fine |

---

## The Physical Reality

An interface in the kernel is a struct. Here's what actually exists:

```
                    ┌────────────────────────────────────────────┐
                    │  struct net_device  (one per interface)     │
                    ├────────────────────────────────────────────┤
                    │  name          "ens3"                       │
                    │  ifindex       2          ← the real ID     │
                    │  dev_addr      06:5f:8a:2c:11:d4  (MAC)    │
                    │  mtu           1500                         │
                    │  flags         UP | BROADCAST | MULTICAST   │
                    │  operstate     UP    (carrier detected)     │
                    │  stats         rx_packets, tx_packets,      │
                    │                rx_dropped, tx_errors ...    │
                    │  netdev_ops    → driver functions           │
                    │  nd_net        → which NAMESPACE it's in    │
                    └────────────────────────────────────────────┘
                                      │
     ┌────────────────────────────────┼────────────────────────────────┐
     │                                │                                │
     ▼                                ▼                                ▼
 IP addresses                   Routing entries               ARP/neigh entries
 (a LIST — an interface        (point AT this ifindex)       (IP → MAC, per-iface)
  can have many)
```

**Prove the ifindex is the real identity, not the name:**

```bash
ls -l /sys/class/net/
# lrwxrwxrwx 1 root root 0 Jul 12 09:14 docker0 -> ../../devices/virtual/net/docker0
# lrwxrwxrwx 1 root root 0 Jul 12 09:14 ens3 -> ../../devices/pci0000:00/0000:00:03.0/net/ens3
# lrwxrwxrwx 1 root root 0 Jul 12 09:14 lo -> ../../devices/virtual/net/lo

cat /sys/class/net/ens3/ifindex     # 2
cat /sys/class/net/ens3/address     # 06:5f:8a:2c:11:d4
cat /sys/class/net/ens3/mtu         # 1500
cat /sys/class/net/ens3/operstate   # up
cat /sys/class/net/ens3/statistics/rx_dropped
```

Note that `ens3` lives under `devices/pci...` — it is backed by real (virtual) hardware on the PCI bus. `docker0` and `lo` live under `devices/virtual` — **there is no hardware at all**. That is the whole distinction, visible in the filesystem (remember topic 03 — Everything Is a File, and topic 02 — the `/sys` hierarchy).

---

## How It Works — Step by Step

### What happens when your Node app sends a packet to 8.8.8.8

```
COMMAND: your app does fetch('http://8.8.8.8/')

Step 1: NODE          → socket(AF_INET, SOCK_STREAM, 0)  → gets fd 23
Step 2: NODE          → connect(23, {8.8.8.8:80})
Step 3: KERNEL        → ROUTE LOOKUP for destination 8.8.8.8
                        Walks the routing table, LONGEST PREFIX MATCH:
                          10.0.0.0/24   dev ens3      ← does 8.8.8.8 match? NO
                          172.17.0.0/16 dev docker0   ← NO
                          0.0.0.0/0     via 10.0.0.1 dev ens3  ← matches EVERYTHING
                        Result: send via gateway 10.0.0.1, out interface ens3,
                                use source IP 10.0.0.15
Step 4: KERNEL        → picks an ephemeral source port (topic 28), say 51234
Step 5: KERNEL        → needs the MAC of 10.0.0.1 (the gateway) to build the frame.
                        Checks the NEIGHBOUR TABLE (ARP cache).
                        MISS → broadcasts "who has 10.0.0.1?" (ARP request)
                        Gateway replies "06:1a:2b:3c:4d:5e"
                        Cached in the neigh table.
Step 6: KERNEL        → builds the frame:
                        [ eth: dst=gateway MAC | src=ens3 MAC ]
                        [ ip:  src=10.0.0.15  | dst=8.8.8.8  ]
                        [ tcp: src=51234      | dst=80  SYN  ]
Step 7: DRIVER        → hands the frame to the NIC's TX ring buffer
Step 8: NIC           → puts electrons/photons on the wire
```

**The single most important thing here:** the destination IP does NOT go in the Ethernet frame's destination. The frame goes to the **gateway's MAC**. The gateway then forwards it. This is why ARP exists, and why `ip neigh` matters.

### The two "up"s: admin state vs carrier

```
ip link set ens3 down      →  you administratively turned it OFF.  Flag: no UP.
ip link set ens3 up        →  you administratively turned it ON.   Flag: UP.

LOWER_UP                   →  the KERNEL/driver says: a cable is plugged in and
                              the physical link is negotiated. You do NOT control this.

┌──────────────┬────────────┬─────────────────────────────────────────┐
│ Flags        │ state      │ Meaning                                  │
├──────────────┼────────────┼─────────────────────────────────────────┤
│ UP,LOWER_UP  │ UP         │ Healthy. Admin up, carrier present.      │
│ UP (no       │ DOWN       │ You turned it on, but NO CABLE / no peer │
│  LOWER_UP)   │            │ → shows as NO-CARRIER. Very common on    │
│              │            │   docker0 when no containers are running │
│ (no UP)      │ DOWN       │ You (or a script) turned it off.         │
│ UP,LOWER_UP  │ UNKNOWN    │ Normal for `lo` and tun/tap — they have  │
│              │            │ no real carrier concept. Not a problem.  │
└──────────────┴────────────┴─────────────────────────────────────────┘
```

`state UNKNOWN` on `lo` alarms people. It is correct and expected.

---

## `ip` vs the Legacy net-tools

`ifconfig`, `netstat`, `route`, and `arp` come from a package called **net-tools**. It has been effectively unmaintained since ~2001. It cannot see modern kernel features (multiple IPs per interface are shown badly, policy routing not at all, namespaces not at all). **It is not installed by default on Ubuntu 20.04+, Debian 11+, or any slim/alpine Docker image.**

The replacement is **iproute2** (`ip`, `ss`, `bridge`, `tc`), which talks to the kernel over netlink.

| Legacy (net-tools) | Modern (iproute2) | Notes |
|---|---|---|
| `ifconfig` | `ip addr` (or `ip a`) | |
| `ifconfig eth0 up` | `ip link set eth0 up` | |
| `ifconfig eth0 10.0.0.5 netmask 255.255.255.0` | `ip addr add 10.0.0.5/24 dev eth0` | |
| `route -n` | `ip route show` (or `ip r`) | |
| `route add default gw 10.0.0.1` | `ip route add default via 10.0.0.1` | |
| `arp -n` | `ip neigh` | |
| `netstat -tulpn` | `ss -tulpn` | topic 28 |
| `netstat -i` | `ip -s link` | |

You still need to **read** `ifconfig` output, because every StackOverflow answer written before 2018 uses it. But **write** `ip`.

```bash
# The single most useful daily command — brief + colored:
ip -br -c addr
# lo               UNKNOWN        127.0.0.1/8 ::1/128
# ens3             UP             10.0.0.15/24 fe80::45f:8aff:fe2c:11d4/64
# docker0          DOWN           172.17.0.1/16
# veth9a1b2c3@if8  UP             fe80::acd1:2eff:fe33:1122/64
```

---

## Reading `ip addr` Field by Field

```
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 06:5f:8a:2c:11:d4 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.15/24 brd 10.0.0.255 scope global dynamic ens3
       valid_lft 84163sec preferred_lft 84163sec
    inet6 fe80::45f:8aff:fe2c:11d4/64 scope link
       valid_lft forever preferred_lft forever
```

```
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP ... qlen 1000
│   │     │                                 │        │              │              │
│   │     │                                 │        │              │              └── TX queue length (packets)
│   │     │                                 │        │              └── operational state (carrier)
│   │     │                                 │        └── queueing discipline (packet scheduler)
│   │     │                                 └── Max Transmission Unit — biggest payload, bytes
│   │     └── FLAGS: admin UP, LOWER_UP = carrier present, can broadcast/multicast
│   └── interface NAME
└── ifindex — the kernel's real ID for this device

    link/ether 06:5f:8a:2c:11:d4 brd ff:ff:ff:ff:ff:ff
    │          │                 │
    │          │                 └── L2 broadcast address (all-Fs = "everyone on this segment")
    │          └── this interface's MAC address (L2 identity)
    └── link layer type (ether / loopback / none for tun)

    inet 10.0.0.15/24 brd 10.0.0.255 scope global dynamic ens3
    │    │        │   │              │            │
    │    │        │   │              │            └── DYNAMIC = assigned by DHCP, has a lease
    │    │        │   │              └── scope: global (routable) | host (lo only) | link (fe80::)
    │    │        │   └── the broadcast address for this subnet
    │    │        └── prefix length: 24 bits of network, 8 bits of host
    │    └── the IPv4 address
    └── "inet" = IPv4 ("inet6" = IPv6)

       valid_lft 84163sec preferred_lft 84163sec
       │                  │
       │                  └── after this, prefer a different source address
       └── DHCP lease remaining. "forever" = statically configured, no lease.
```

If you see `dynamic` with a `valid_lft` countdown → **DHCP gave you this address.** Anything you set by hand with `ip addr add` shows `forever` and disappears on reboot.

### CIDR in one paragraph

`10.0.0.15/24` means: the first **24 bits** (`10.0.0.`) are the **network**, the remaining 8 bits are the **host**. That gives 2⁸ = **256 addresses**: `10.0.0.0` (the network address, not usable), `10.0.0.255` (the broadcast address, not usable), and `10.0.0.1`–`10.0.0.254` = **254 usable hosts**. `/24` = netmask `255.255.255.0`. Smaller number = bigger network: `/16` = 65,536 addresses (that's `docker0`'s `172.17.0.0/16`), `/32` = exactly one host, `/0` = literally every address (that's the default route). That is all the subnetting you need to read your own machine.

### Longest prefix match

When the kernel routes a packet, it does not take the first matching route. It takes the **most specific** one — the one with the longest prefix.

```
Destination: 172.17.0.5

Route table:
  0.0.0.0/0      via 10.0.0.1 dev ens3     ← /0  matches (matches everything)
  10.0.0.0/24    dev ens3                  ← /24 doesn't match
  172.17.0.0/16  dev docker0               ← /16 matches  ★ LONGEST → WINS

Winner: dev docker0. The /0 default route loses because /16 is more specific.
```

The **default route** (`0.0.0.0/0`) is simply the least-specific route possible: *"if nothing else matched, send it here."* It's the escape hatch to the internet. No default route = you can only reach your own subnets.

---

## MTU and the Black-Hole Bug

MTU = the largest payload a frame can carry. Ethernet's default is **1500 bytes**.

Now a VPN or an overlay network wraps your packet in ANOTHER header:

```
Normal Ethernet:      [ IP | TCP | 1460 bytes of data ]            = 1500  ✅
Through WireGuard:    [ IP | UDP | WG | IP | TCP | data ]
                        ^^^^^^^^^^^^^^ ~60 bytes of overhead
                      → your real payload must shrink, so wg0 has mtu 1420
```

The correct mechanism is **Path MTU Discovery**: the sender sets the "Don't Fragment" bit; if a router in the middle can't fit the packet, it sends back an **ICMP "Fragmentation Needed"** message saying *"max is 1420."* The sender shrinks and retries. Fine.

**The bug:** an over-zealous firewall somewhere blocks ALL ICMP. Now that message never comes back. The sender keeps sending 1500-byte packets that are silently dropped forever. This is a **PMTU black hole**, and its symptom is famous:

```
❌ SSH connects, you log in, then `cat bigfile` HANGS forever
❌ curl to the API: `GET /health` (small) → works.
                     `POST /upload` (big) → hangs, then times out
❌ TLS handshake completes, then the connection stalls on the first large record
❌ "It works on my laptop but not from the VPN"
```

Small packets fit; **large ones vanish**. Anything that works small and dies big is an MTU problem until proven otherwise.

```bash
# PROVE IT: find the real path MTU. -M do = set Don't Fragment. -s = payload size.
# 1472 payload + 8 ICMP header + 20 IP header = 1500 total.
ping -M do -s 1472 8.8.8.8
# ping: local error: message too long, mtu=1420   ← there's your answer

# Binary search downward until it succeeds; add 28 to get the MTU.
ping -M do -s 1392 8.8.8.8   # works → path MTU is 1392+28 = 1420

# Fix A: lower the interface MTU
sudo ip link set dev wg0 mtu 1420

# Fix B (the one that actually saves you): MSS clamping — rewrite the TCP MSS in
# every SYN so both ends negotiate a size that fits. Standard on any router/NAT box.
sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
     -j TCPMSS --clamp-mss-to-pmtu
```

---

## Interface Naming — And Why `eth0` Died

The kernel names interfaces in the order the drivers probe the hardware. That order **is not guaranteed to be stable across reboots** (it depends on parallel driver init, PCI enumeration timing, hotplug order). On a two-NIC box that means: `eth0` is your public NIC today, and after a kernel upgrade and a reboot, `eth0` is your *private* NIC. Your firewall rules now protect the wrong card. Your server is on the internet with the internal rules applied. This actually happened, a lot.

So systemd/udev **renames** interfaces at boot based on stable, physical facts:

```
  ens3                 enp0s3                     eno1                wlp2s0
  ││ │                 ││││ │                     ││ │                ││││ │
  ││ └── hotplug slot 3 ││││ └── slot 3            ││ └── onboard #1   ││││ └── slot 0
  ││                    │││└── PCI slot            ││                  │││└── ...
  ││                    ││└── PCI bus 0            ││                  ││└── PCI bus 2
  │└── "n" = ?          │└── "p" = PCI geography   │└── "o" = onboard  │└── "p"
  └── "en" = ETHERNET   └── "en" = ETHERNET        └── "en"            └── "wl" = WLAN
```

| Prefix | Meaning |
|---|---|
| `en` | Ethernet |
| `wl` | Wireless LAN |
| `ww` | Wireless WAN (cellular) |
| `o<n>` | On-board device index (from firmware/BIOS) |
| `s<n>` | Hotplug slot index (from firmware) |
| `p<b>s<n>` | PCI bus `<b>`, slot `<n>` — pure physical topology |

The name is now derived from **where the card physically is**, so it cannot change across a reboot. Ugly, but correct. (You can force the old scheme back with the kernel boot parameter `net.ifnames=0`, and some cloud images do — that's why you still see `eth0` on AWS.)

Interfaces you will meet:

| Name | What it is |
|---|---|
| `lo` | Loopback. `127.0.0.1/8` + `::1/128`. **Packets never leave the kernel.** |
| `eth0` | Classic kernel-assigned ethernet name (still common in cloud images and containers) |
| `ens3` / `enp0s3` / `eno1` | Predictable names — same thing, renamed by udev |
| `docker0` | The default Docker **bridge**. A virtual switch, `172.17.0.1/16`. |
| `veth9a1b2c3@if8` | One end of a **virtual cable**. The other end is inside a container. |
| `br-1a2b3c4d5e6f` | A **user-defined** Docker network's bridge (`docker network create`) |
| `wg0` | WireGuard VPN tunnel |
| `tun0` / `tap0` | OpenVPN/userspace tunnel (`tun` = L3/IP, `tap` = L2/ethernet) |
| `enx0242ac110002` | Name derived from the MAC (used when there's no other stable info) |

---

## THE BIG ONE — Loopback vs 0.0.0.0

This is the most expensive misunderstanding in backend development.

```
   server.listen(3000, '127.0.0.1')          server.listen(3000, '0.0.0.0')
   ────────────────────────────────          ─────────────────────────────
   Kernel binds the socket to the            Kernel binds the socket to
   loopback interface ONLY.                  EVERY interface.

   ┌──────────── HOST ───────────┐           ┌──────────── HOST ───────────┐
   │                              │           │                              │
   │  lo (127.0.0.1) ── :3000 ✅  │           │  lo (127.0.0.1) ── :3000 ✅  │
   │                              │           │                              │
   │  ens3 (10.0.0.15) ─── ✗      │           │  ens3 (10.0.0.15) ─ :3000 ✅ │
   │           │                  │           │           │                  │
   └───────────┼──────────────────┘           └───────────┼──────────────────┘
               │                                          │
          ✗ packets from outside                     ✅ reachable
            arrive on ens3 and find
            NOTHING listening there
```

`127.0.0.1` is not "this machine." It is **"the loopback interface."** A packet sent to it is handed straight back up the stack by the kernel and **never reaches a NIC**. There is no packet on the wire. There is nothing for a firewall to allow. Opening port 3000 in `ufw` changes **nothing** — the firewall was never the problem.

```bash
# The proof, in one command:
ss -tulpn | grep 3000
# tcp  LISTEN 0  511  127.0.0.1:3000  0.0.0.0:*  users:(("node",pid=8231,fd=23))
#                     ^^^^^^^^^ ← THERE IT IS. Bound to loopback. Unreachable.

# What you want to see:
# tcp  LISTEN 0  511    0.0.0.0:3000  0.0.0.0:*  users:(("node",pid=8231,fd=23))
#                       ^^^^^^^ = all interfaces
#      or *:3000 / [::]:3000  (dual-stack, also fine)
```

The identical bug inside a container: your Dockerfile app binds `127.0.0.1:3000`, which is the **container's** loopback. `docker run -p 3000:3000` forwards host:3000 → **container-IP**:3000, not container-loopback:3000. Nothing is listening there. `curl localhost:3000` on the host → `Connection refused`. **Inside a container you must bind `0.0.0.0`.**

---

## NETWORK NAMESPACES — The Payoff

A **network namespace** is a complete, isolated copy of the network stack. Not a filter, not a view — a genuinely separate instance of:

```
┌─────────────────────────────────────────────────────────────┐
│  ONE NETWORK NAMESPACE contains its own:                     │
├─────────────────────────────────────────────────────────────┤
│  • interfaces          (its own lo! its own eth0!)           │
│  • IP addresses                                              │
│  • routing table       (its own default route)               │
│  • ARP / neighbour table                                     │
│  • iptables/nftables rules                                   │
│  • socket table  ◀◀◀  ITS OWN PORT SPACE                     │
│  • /proc/net/*  and  /sys/class/net/*                        │
└─────────────────────────────────────────────────────────────┘
```

**That last line is the answer to a question you have definitely asked.** Container A binds port 3000. Container B binds port 3000. No conflict — because "port 3000" is an entry in a *socket table*, and each namespace has its **own socket table**. They are as unrelated as file `/tmp/x` on two different machines.

A container is *not* a lightweight VM. A container is a normal Linux process that has been placed in its own set of namespaces (net, pid, mnt, uts, ipc, user) and cgroups. The `net` namespace is the one that gives it "its own network."

### Building it by hand — this is literally what Docker does

```bash
# 1. Create a namespace called "app"
sudo ip netns add app
sudo ip netns list                       # app

# 2. Look inside. It is EMPTY except for a down loopback.
sudo ip netns exec app ip -br addr
# lo    DOWN

# 3. Create a veth pair — a virtual cable with two ends.
sudo ip link add veth-host type veth peer name veth-app

# 4. Push ONE END into the namespace. It vanishes from the host.
sudo ip link set veth-app netns app

# 5. Configure the container side (inside the ns)
sudo ip netns exec app ip addr add 10.10.0.2/24 dev veth-app
sudo ip netns exec app ip link set veth-app up
sudo ip netns exec app ip link set lo up          # ← don't forget lo!
sudo ip netns exec app ip route add default via 10.10.0.1

# 6. Configure the host side
sudo ip addr add 10.10.0.1/24 dev veth-host
sudo ip link set veth-host up

# 7. Prove it
sudo ip netns exec app ping -c1 10.10.0.1        # ✅ reaches the host
sudo ip netns exec app ip route show
# default via 10.10.0.1 dev veth-app
# 10.10.0.0/24 dev veth-app proto kernel scope link src 10.10.0.2

# 8. To get to the INTERNET it needs NAT (topic 29):
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o ens3 -j MASQUERADE
sudo ip netns exec app ping -c1 8.8.8.8           # ✅ now it works

# Cleanup
sudo ip netns del app     # deleting the ns deletes veth-app; veth-host dies with it
```

You just built a container's network from scratch. Docker adds a bridge so that many containers share one uplink.

### The full Docker picture

```
              CONTAINER A (netns)             CONTAINER B (netns)
            ┌────────────────────┐          ┌────────────────────┐
            │  eth0  172.17.0.2  │          │  eth0  172.17.0.3  │
            │  node listens :3000│          │  node listens :3000│  ← SAME PORT.
            │  lo    127.0.0.1   │          │  lo    127.0.0.1   │    NO CONFLICT.
            │  route: default    │          │  route: default    │
            │         via .0.1   │          │         via .0.1   │
            └─────────┬──────────┘          └─────────┬──────────┘
                      │  (veth pair)                  │  (veth pair)
    ══════════════════╪═══════════════════════════════╪══════════════════
                      │            HOST netns         │
              ┌───────┴────────┐             ┌────────┴───────┐
              │ vethA1B2C3@if5 │             │ vethD4E5F6@if7 │
              └───────┬────────┘             └────────┬───────┘
                      │                               │
                 ┌────┴───────────────────────────────┴────┐
                 │      docker0  (BRIDGE = virtual switch)  │
                 │      172.17.0.1/16                       │
                 │      = the containers' DEFAULT GATEWAY   │
                 └───────────────────┬─────────────────────┘
                                     │
                        iptables nat POSTROUTING:
                          -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
                        (rewrite source IP → 10.0.0.15)   [topic 29]
                                     │
                            ┌────────┴────────┐
                            │  ens3  10.0.0.15│
                            └────────┬────────┘
                                     │
                                 INTERNET
```

Inbound (`-p 8080:3000`) goes the other way, via **DNAT**:

```
iptables -t nat -L DOCKER -n
# Chain DOCKER (2 references)
# target  prot opt source     destination
# DNAT    tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:3000
#                                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                            "anything arriving for host port 8080 →
#                             rewrite the destination to the container"
```

**`-p 8080:3000` is not magic. It is one DNAT rule** (plus a `docker-proxy` userland process that Docker also starts as a fallback for cases iptables can't cover, like loopback-originated traffic — you'll see it in `ps aux | grep docker-proxy`).

### Docker network modes

| Mode | What happens | Consequence |
|---|---|---|
| `bridge` (default) | New netns + veth + docker0. NAT out, DNAT in. | Isolated. Needs `-p` to be reachable. |
| `host` (`--network host`) | **No new netns at all.** The container shares the HOST's network namespace. | `-p` is ignored and meaningless. The app binds the host's real ports directly. Zero isolation, zero NAT overhead. Two host-mode containers on :3000 **WILL** collide. |
| `none` | New netns with only `lo`. | No connectivity at all. For batch jobs. |
| `container:<name>` | Joins ANOTHER container's netns | They share `lo` — this is how a Kubernetes pod works. All containers in a pod share one netns, which is why they talk over `localhost`. |

### Why `localhost` inside a container is NOT the host

Because the container has **its own `lo`**. `127.0.0.1` inside the container resolves to the container's own loopback interface, in the container's own namespace. It has no relationship whatsoever to the host's `127.0.0.1`.

```
❌ WRONG — from inside a container, trying to reach Postgres running on the host:
   DATABASE_URL=postgres://user:pw@127.0.0.1:5432/app
   → ECONNREFUSED. You're talking to the container's own empty loopback.

✅ RIGHT — options, best first:
   1. Put Postgres in a container on the same user-defined network, use its NAME:
      DATABASE_URL=postgres://user:pw@postgres:5432/app        (topic 27 — DNS)
   2. Docker Desktop (Mac/Windows) provides a magic name:
      DATABASE_URL=postgres://user:pw@host.docker.internal:5432/app
      (on Linux, add: --add-host=host.docker.internal:host-gateway)
   3. Use the docker0 gateway IP directly (fragile, but works on Linux):
      DATABASE_URL=postgres://user:pw@172.17.0.1:5432/app
      ...AND make sure Postgres is bound to 0.0.0.0, not 127.0.0.1. (See above. Again.)
   4. --network host  (Linux only; then 127.0.0.1 really IS the host)
```

Docker doesn't register its container namespaces where `ip netns` looks, so to peek inside a running container from the host:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' my-api)
sudo nsenter -t $PID -n ip -br addr      # run `ip` INSIDE the container's netns
sudo nsenter -t $PID -n ss -tulpn        # ...even though the container has no `ss`
```

That last trick is gold: it lets you run **host tools inside a container's network** even when the image is a scratch/distroless image with no shell.

---

## Exact Syntax Breakdown

```
ip -br -c addr show dev ens3
│  │   │  │    │    │
│  │   │  │    │    └── limit to one interface
│  │   │  │    └── sub-subcommand (optional; `ip addr` alone implies `show`)
│  │   │  └── OBJECT: addr (addresses) | link (L2) | route | neigh | netns
│  │   └── -c = colorize output
│  └── -br = brief — one line per interface. The best flag in iproute2.
└── the ip binary (iproute2)
```

```
ip addr add 10.0.0.5/24 dev eth0
│  │    │   │           │   │
│  │    │   │           │   └── which interface gets the address
│  │    │   │           └── "dev" keyword — required
│  │    │   └── address WITH PREFIX. Forgetting /24 gives you /32 → nothing routes.
│  │    └── add (vs: del, show, flush)
│  └── object: addr
└── ip
    ⚠️ EPHEMERAL. Gone on reboot, gone if NetworkManager/netplan re-applies. See Config.
```

```
ip link set eth0 up
│  │    │   │    │
│  │    │   │    └── up | down | mtu 1420 | address 02:11:.. | promisc on | netns app
│  │    │   └── the device
│  │    └── "set" = modify (vs show / add / del)
│  └── object: link = layer 2 — MAC, MTU, admin state. NOT IP addresses.
└── ip
```

```
ip route get 8.8.8.8
│  │     │   │
│  │     │   └── the destination you're asking about
│  │     └── "get" = ASK THE KERNEL: what would you actually DO with this packet?
│  │            (vs "show", which just dumps the table and makes you do the math)
│  └── object: route
└── ip

Output:
8.8.8.8 via 10.0.0.1 dev ens3 src 10.0.0.15 uid 1000
        │            │        │
        │            │        └── the SOURCE IP the kernel will stamp on the packet
        │            └── the interface it will leave by
        └── the next-hop gateway

★ THIS IS THE BEST COMMAND IN THIS DOCUMENT.
  It answers "why is my traffic going out the wrong interface / not through the VPN"
  in one line, with zero guessing. Run it before you read a routing table by hand.
```

```
ip neigh show
│  │     │
│  │     └── dump the table
│  └── the NEIGHBOUR table = the ARP cache = "which MAC owns which IP, per interface"
└── ip

10.0.0.1    dev ens3    lladdr 06:1a:2b:3c:4d:5e REACHABLE
172.17.0.2  dev docker0 lladdr 02:42:ac:11:00:02 STALE
10.0.0.99   dev ens3    FAILED         ← ARP got no answer. That host is not there.
            │           │              │
            │           │              └── REACHABLE | STALE (ok, just old) | FAILED
            │           └── "link layer address" = the MAC
            └── which interface this mapping is valid on
```

```
ip -s link show ens3
│  │  │
│  │  └── object: link
│  └── -s = STATISTICS. Add it twice (-s -s) for even more detail.
└── ip

    RX: bytes      packets  errors  dropped  overrun  mcast
        482938471  1029384  0       12       0        0
    TX: bytes      packets  errors  dropped  carrier  collsns
        193847263  884722   0       0        0        0
                            ^^^^^^  ^^^^^^^
                            Non-zero errors/dropped that GROW = a real hardware,
                            driver, or ring-buffer problem. Check `ethtool -S ens3`.
                            Steady non-zero-but-flat = old, ignore.
```

```
tcpdump -i any -nn port 3000
│       │      │  │
│       │      │  └── BPF filter: only packets with src or dst port 3000
│       │      └── -nn = don't resolve hostnames OR port names. Fast, unambiguous.
│       └── -i any = capture on ALL interfaces (including lo and vethXXX)
└── tcpdump — needs root (or CAP_NET_RAW)

Add: -c 20   stop after 20 packets
     -A      print payload as ASCII (see the HTTP request!)
     -w f.pcap  write to a file for Wireshark
```

---

## Example 1 — Basic

```bash
# What interfaces exist? (the one-liner you'll type 1000 times)
ip -br -c addr
# lo               UNKNOWN        127.0.0.1/8 ::1/128
# ens3             UP             10.0.0.15/24 fe80::45f:8aff:fe2c:11d4/64
# docker0          DOWN           172.17.0.1/16

# What's my routing table?
ip route show
# default via 10.0.0.1 dev ens3 proto dhcp src 10.0.0.15 metric 100
# 10.0.0.0/24 dev ens3 proto kernel scope link src 10.0.0.15 metric 100
# 172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
#                                                                  ^^^^^^^^ no
#                                                     containers running → no carrier

# How would the kernel actually reach Google? (ASK IT, don't guess)
ip route get 8.8.8.8
# 8.8.8.8 via 10.0.0.1 dev ens3 src 10.0.0.15 uid 1000
#     cache

# ...and a container?
ip route get 172.17.0.2
# 172.17.0.2 dev docker0 src 172.17.0.1 uid 1000     ← direct, no gateway. Same L2.

# Who is my gateway, at layer 2?
ip neigh show
# 10.0.0.1 dev ens3 lladdr 06:1a:2b:3c:4d:5e REACHABLE

# Am I dropping packets?
ip -s link show ens3 | grep -A1 RX

# Add a second IP to an interface (temporarily — this is gone on reboot)
sudo ip addr add 10.0.0.99/24 dev ens3
ip -br addr show ens3
# ens3   UP   10.0.0.15/24 10.0.0.99/24 fe80::.../64    ← interfaces can have MANY IPs
sudo ip addr del 10.0.0.99/24 dev ens3
```

---

## Example 2 — Production Scenario

**02:14.** PagerDuty. Your Node API (`api` container) can't reach your Node auth service (`auth` container). The logs are full of:

```
Error: connect ECONNREFUSED 172.18.0.4:4000
    at TCPConnectWrap.afterConnect [as oncomplete] (net:1247:16)
```

`ECONNREFUSED` is very specific information: **a TCP packet reached a host, and that host's kernel sent back a RST** because nothing was listening on that port. It is NOT a timeout (which would mean a firewall or routing problem). So: the network is fine. Something is wrong at the destination. Walk the chain.

```bash
# ── 1. Is the auth container even alive?
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
# NAMES   STATUS          PORTS
# api     Up 12 minutes   0.0.0.0:8080->3000/tcp
# auth    Up 3 minutes    4000/tcp
#                         ^^^^^^^^ up. And it restarted 3 min ago — suspicious.

# ── 2. Is anything actually LISTENING inside auth? (topic 28)
docker exec auth ss -tulpn
# Netid State  Local Address:Port  Peer Address:Port  Process
# tcp   LISTEN 127.0.0.1:4000      0.0.0.0:*          users:(("node",pid=1,fd=21))
#              ^^^^^^^^^^^^^^^^
#              ★★★ FOUND IT ★★★
#              Bound to the CONTAINER'S OWN LOOPBACK.
#              Packets from `api` arrive on the container's eth0 (172.18.0.4).
#              Nothing is listening on eth0. Kernel replies RST → ECONNREFUSED.

# ── 3. Confirm from the network side. Are the packets even arriving?
#     Capture on the host, on ALL interfaces, port 4000:
sudo tcpdump -i any -nn port 4000 -c 6
# 02:16:41.882 IP 172.18.0.3.51204 > 172.18.0.4.4000: Flags [S], seq 91823...
# 02:16:41.882 IP 172.18.0.4.4000 > 172.18.0.3.51204: Flags [R.], seq 0, ack ...
#                                                            ^^^^ RST!
# The SYN ARRIVES. The auth container's kernel RESETS it.
# → The network is PERFECT. The app is bound wrong. This is a 100% conclusive answer,
#   obtained in 8 seconds, and it ends the "is it the network or the app?" argument.

# ── 4. Sanity-check the veth/bridge plumbing anyway (30 seconds, worth it)
docker network inspect app-net -f '{{range .Containers}}{{.Name}} {{.IPv4Address}}
{{end}}'
# api 172.18.0.3/16
# auth 172.18.0.4/16                     ← both on the bridge, correct IPs

ip -br link | grep -E 'br-|veth'
# br-1a2b3c4d5e6f  UP  02:42:1f:aa:bb:cc <BROADCAST,MULTICAST,UP,LOWER_UP>
# veth7f3a1c2@if12 UP  ce:11:22:33:44:55 <BROADCAST,MULTICAST,UP,LOWER_UP>
# veth9b2d4e6@if14 UP  ce:aa:bb:cc:dd:ee <BROADCAST,MULTICAST,UP,LOWER_UP>
#                                          both veths UP, both enslaved to the bridge ✅

# ── 5. THE FIX. Someone "hardened" the auth service in the last deploy:
git log --oneline -3 -- src/server.js
# 8a3f21c  chore: bind to localhost for security   ← 3 hours ago. There it is.

# src/server.js
# ❌  server.listen(4000, '127.0.0.1');
# ✅  server.listen(4000, '0.0.0.0');
#
# Inside a container, binding 0.0.0.0 is NOT insecure: the container's netns is
# already the isolation boundary, and nothing outside can reach 172.18.0.4 unless
# you published the port with -p. The "bind to localhost for security" instinct is
# correct on a bare host and WRONG inside a container.

docker compose up -d --build auth
docker exec auth ss -tulpn | grep 4000
# tcp LISTEN 0 511 0.0.0.0:4000 0.0.0.0:*  users:(("node",pid=1,fd=21))   ✅
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/health
# 200
```

**Total time: under five minutes**, because you knew that `ECONNREFUSED` means "reached the host, nothing listening," and you knew where to look.

---

## Common Mistakes

### Mistake 1 — "The firewall is open but my app still isn't reachable"

**Wrong:** `sudo ufw allow 3000` … still refused … `sudo ufw disable` … still refused … blame the cloud provider's security group … blame DNS … blame the universe.

**Root cause:** The socket is bound to `127.0.0.1:3000`. The kernel's `lo` interface is a dead end — packets sent to `127.0.0.1` are looped straight back up the stack and **never touch a NIC**. A packet arriving from outside lands on `ens3`. The kernel looks for a socket bound to `ens3`'s address (or to the wildcard `0.0.0.0`) on port 3000, finds none, and sends a TCP RST. The firewall never entered the picture — there was nothing to filter.

**Diagnose:** `ss -tulpn | grep :3000`. If the Local Address is `127.0.0.1`, you're done.

**Fix:** `server.listen(3000, '0.0.0.0')`. Or in Express: `app.listen(3000, '0.0.0.0')`. Or set `HOST=0.0.0.0`.

**Prevent:** Make the bind address an env var (`process.env.HOST || '0.0.0.0'`) and put `ss -tulpn | grep :3000` in your deploy smoke test.

---

### Mistake 2 — "`ping` fails, so the server is down"

**Wrong:** `ping api.example.com` → 100% packet loss → "the host is down, escalate."

**Root cause:** `ping` uses **ICMP echo**, a completely different protocol from TCP. A huge number of cloud security groups, corporate firewalls, and hardened hosts **drop ICMP by default** (AWS security groups do not allow ICMP unless you add it). The host can be serving 10,000 req/s on port 443 and still never answer a ping.

**Fix:** Test the thing you actually care about — the TCP port:

```bash
# ❌ ping api.example.com
# ✅ any of these:
nc -zv api.example.com 443            # -z = scan, don't send data; -v = tell me
curl -sS -o /dev/null -w '%{http_code} %{time_total}s\n' https://api.example.com/health
ss -tn state established '( dport = :443 )'
```

**Corollary:** "ping works" also doesn't mean your app works. ICMP is answered by the *kernel*, not by your process. A pingable host with a crashed Node app is still down.

---

### Mistake 3 — `ip addr add` and then a reboot

**Wrong:**
```bash
sudo ip addr add 10.0.0.99/24 dev ens3   # fixes the incident
# ...reboot 3 weeks later... the IP is gone, the service is down, nobody knows why
```

**Root cause:** `ip` writes to the **running kernel** via netlink. It writes to no file. Nothing persists it. Furthermore, if NetworkManager or `netplan apply` runs, it will reconcile the interface back to its config file and **delete your address immediately**, without a reboot.

**Fix — make it persistent (Ubuntu / netplan):**

```yaml
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: true
      addresses:
        - 10.0.0.99/24          # the extra static IP
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

```bash
sudo netplan generate     # render to systemd-networkd/NM config; syntax check
sudo netplan try          # ★ APPLY, but AUTO-REVERT after 120s unless you press Enter.
                          #   This is what saves you from locking yourself out over SSH.
sudo netplan apply        # commit for real
```

**Prevent:** Use `ip addr add` for *diagnosis and temporary tests only*. Anything that must survive a reboot goes in a config file. And **always** use `netplan try`, never `netplan apply`, when you are SSH'd in.

---

### Mistake 4 — "small requests work, big ones hang" (the MTU black hole)

**Wrong:** Assume the app is slow, add timeouts, add retries, blame the database.

**Root cause:** A tunnel (WireGuard, IPsec, a cloud overlay, a Docker overlay network) reduced the path MTU. The sender is emitting 1500-byte packets with DF set. A middlebox needs to fragment, can't, and sends back an ICMP "fragmentation needed" — **which a firewall drops**. The sender never learns. Every large packet is silently discarded. TCP retransmits the same too-large packet forever.

**Symptom signature:** the connection *establishes* (SYN/ACK are tiny), small responses work, and it dies the instant a response exceeds ~1400 bytes.

**Diagnose:**
```bash
ping -M do -s 1472 <peer>       # message too long, mtu=1420 → confirmed
ip link show wg0 | grep mtu     # mtu 1420
ss -tin dst <peer> | grep mss   # look at the negotiated MSS
```

**Fix:** lower the MTU on the tunnel interface, and/or clamp the MSS (`--clamp-mss-to-pmtu`). Never fix it by disabling PMTUD.

**Prevent:** Don't blanket-drop ICMP. Allow type 3 code 4 (fragmentation needed) and type 11 (time exceeded), always.

---

### Mistake 5 — Reaching for `ifconfig` inside a container

**Wrong:**
```bash
docker exec -it api ifconfig
# OCI runtime exec failed: exec: "ifconfig": executable file not found in $PATH
```
…followed by 20 minutes of `apt-get install net-tools` inside a production container.

**Root cause:** net-tools is legacy and not in slim/alpine/distroless images. Often `ip` isn't either.

**Fix:** Don't install tools into the container. **Enter the container's namespace from the host**, where you have every tool:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' api)
sudo nsenter -t $PID -n ip -br addr      # host's `ip`, container's network
sudo nsenter -t $PID -n ss -tulpn        # host's `ss`, container's sockets
sudo nsenter -t $PID -n tcpdump -nn -i eth0 port 3000
```

Or, for a throwaway toolbox that shares the target's netns:

```bash
docker run --rm -it --net container:api nicolaka/netshoot ss -tulpn
```

**Prevent:** Learn `nsenter`. It turns every "I can't debug this container, it has no tools" into a non-problem — and it works on scratch images with no shell at all.

---

## Hands-On Proof

```bash
# PROVE IT: an interface is a kernel object, not a file you edit
ls /sys/class/net/                          # one entry per interface
cat /sys/class/net/lo/type                  # 772 = ARPHRD_LOOPBACK
cat /sys/class/net/ens3/type                # 1   = ARPHRD_ETHER
readlink /sys/class/net/lo                  # .../devices/virtual/net/lo  ← VIRTUAL
readlink /sys/class/net/ens3                # .../pci0000:00/... ← real PCI hardware

# PROVE IT: loopback traffic never reaches a NIC
# Terminal 1 — watch the REAL interface only:
sudo tcpdump -i ens3 -nn port 9999
# Terminal 2:
nc -l 127.0.0.1 9999 &  ; echo hi | nc 127.0.0.1 9999
# Terminal 1 shows ZERO packets. Now:
sudo tcpdump -i lo -nn port 9999      # ← the packets are HERE. It never left the kernel.

# PROVE IT: `ip route get` tells you the truth about routing
ip route get 8.8.8.8            # → via the gateway, out ens3
ip route get 127.0.0.1          # → dev lo src 127.0.0.1
ip route get 172.17.0.2         # → dev docker0 (if Docker is installed)

# PROVE IT: the ARP cache is populated on demand
ip neigh flush all
ip neigh show                   # empty (or nearly)
ping -c1 $(ip route | awk '/default/{print $3}')   # ping the gateway
ip neigh show                   # the gateway's MAC is now cached  ← ARP happened

# PROVE IT: each container has its OWN port space
docker run -d --name a -p 8081:3000 node:20-slim \
  node -e "require('http').createServer((_,r)=>r.end('a')).listen(3000,'0.0.0.0')"
docker run -d --name b -p 8082:3000 node:20-slim \
  node -e "require('http').createServer((_,r)=>r.end('b')).listen(3000,'0.0.0.0')"
# BOTH bound port 3000. No conflict. Different namespaces = different socket tables.
curl -s localhost:8081; echo    # a
curl -s localhost:8082; echo    # b

# PROVE IT: the veth pairs are visible on the host
ip -br link | grep veth         # one veth per running container
docker exec a cat /sys/class/net/eth0/iflink     # e.g. 14
ip -br link | grep '@if'        # find the host veth whose index is 14 → that's the pair

# PROVE IT: -p is just a DNAT rule
sudo iptables -t nat -L DOCKER -n --line-numbers
# DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:8081 to:172.17.0.2:3000
ps aux | grep [d]ocker-proxy    # and the userland fallback proxy

# PROVE IT: host mode has no isolation
docker run --rm --net host node:20-slim ip -br addr
# You will see the HOST's ens3, docker0, everything. It IS the host's namespace.

docker rm -f a b
```

---

## Practice Exercises

### Exercise 1 — Easy

In the terminal:

1. Print all interfaces in brief colored form. Identify which is loopback, which is your real NIC, which are virtual.
2. Print your routing table. Identify the default route and name the gateway IP.
3. Run `ip route get 1.1.1.1` and `ip route get 127.0.0.1`. Explain, out loud, why the outputs differ.
4. Find your NIC's MAC address **twice**: once with `ip link`, once by `cat`-ing a file under `/sys`.
5. Run `ip neigh`. How many neighbours does your machine know about? Ping something on your LAN and check again.

Paste your output.

### Exercise 2 — Medium

Reproduce the loopback bug and fix it, with proof at each step.

```bash
# 1. Start a Node server bound to loopback only
node -e "require('http').createServer((_,r)=>r.end('ok')).listen(3000,'127.0.0.1')" &

# 2. Prove it works locally
curl -s localhost:3000

# 3. Prove it does NOT work from your machine's real IP
MYIP=$(ip route get 1.1.1.1 | awk '{print $7; exit}')
curl -s --max-time 3 http://$MYIP:3000       # ← should FAIL

# 4. Prove WHY with ss. Show the Local Address column.
ss -tulpn | grep 3000

# 5. Kill it, rebind to 0.0.0.0, and prove BOTH now work.
# 6. While it's running on 0.0.0.0, run `sudo tcpdump -i any -nn port 3000` in a second
#    terminal, curl it from $MYIP, and show the SYN/SYN-ACK/ACK handshake.
```

**Deliverable:** the `ss` output before and after, and the tcpdump handshake.

### Exercise 3 — Hard (Production Simulation)

**Build a container's network from scratch, by hand, with no Docker.** Then break it three ways and fix each.

```bash
# PART A — Build it
#   Create a netns "prod". Create a veth pair. Put one end in the ns.
#   Give the ns 10.99.0.2/24, bring up its lo AND its veth, add a default route.
#   Give the host end 10.99.0.1/24 and bring it up.
#   Enable ip_forward and add a MASQUERADE rule so the ns can reach the internet.
#   PROVE:  ip netns exec prod ping -c1 8.8.8.8

# PART B — Run a real server inside it
#   ip netns exec prod node -e "require('http')
#     .createServer((_,r)=>r.end('from the namespace\n'))
#     .listen(3000,'0.0.0.0')" &
#   PROVE from the host:  curl 10.99.0.2:3000

# PART C — Break it three times. For EACH: state the symptom, find it with a command,
#          and fix it.
#   BREAK 1:  ip netns exec prod ip link set lo down
#             (what breaks? why? what error do you get?)
#   BREAK 2:  ip netns exec prod ip route del default
#             (what still works? what stops? which command reveals it in one line?)
#   BREAK 3:  re-listen on 127.0.0.1 instead of 0.0.0.0
#             (what does `ip netns exec prod ss -tulpn` show?)

# PART D — Clean up completely and prove nothing is left:
#   ip netns list ; ip -br link | grep veth ; iptables -t nat -S POSTROUTING
```

Write it as a shell script with a comment above every command explaining what the kernel does.

---

## Mental Model Checkpoint

1. **What is a network interface, from the kernel's point of view?** Why do `lo` and `docker0` appear under `/sys/devices/virtual` while `ens3` appears under `/sys/devices/pci...`?
2. **A packet is destined for `127.0.0.1:3000`. Trace it.** Does it reach a NIC? Does a firewall rule on `ens3` ever apply to it? Why is a service bound there unreachable from another machine even with every port open?
3. **What is the difference between `UP` and `LOWER_UP`?** Which one can you change with `ip link set`?
4. **The routing table has `0.0.0.0/0`, `10.0.0.0/24`, and `172.17.0.0/16`. Which one wins for destination `172.17.0.9`, and why?** What single command makes the kernel tell you the answer directly?
5. **Two containers both run a server on port 3000 and neither fails.** Explain, in terms of kernel objects, exactly why there is no conflict.
6. **Draw from memory:** container `eth0` → veth → docker0 → host NIC → internet. Label where NAT happens and where DNAT happens.
7. **Your app inside a container connects to `127.0.0.1:5432` looking for the host's Postgres.** What does the kernel do with that packet, and what are the three ways to fix it?

---

## Quick Reference Card

| Command | What it does | Key flags |
|---|---|---|
| `ip addr` | Show interfaces + IP addresses | `-br` brief, `-c` color, `-4`/`-6` |
| `ip addr add 10.0.0.5/24 dev eth0` | Add an IP (**ephemeral!**) | `del` to remove, `flush dev X` |
| `ip link` | L2: MAC, MTU, admin state | `-s` stats, `-br`, `set X up\|down\|mtu N` |
| `ip route` | Show/modify the routing table | `add default via IP`, `del` |
| **`ip route get 8.8.8.8`** | **Ask the kernel which iface/src/gw it WOULD use** | — (the best debugging command here) |
| `ip neigh` | ARP cache (IP → MAC) | `flush all`, `show dev eth0` |
| `ip netns` | Manage network namespaces | `add`, `del`, `list`, `exec NS CMD` |
| `ip -s link show eth0` | RX/TX packet + error + drop counters | `-s -s` for more |
| `nsenter -t PID -n CMD` | Run a HOST command inside a container's netns | `-n` = net ns only |
| `ping` | ICMP echo (**often blocked ≠ host down**) | `-c N`, `-M do -s N` (PMTU test) |
| `traceroute` / `mtr` | Path to a destination, hop by hop | `mtr -rwc 20 host` (report mode) |
| `tcpdump -i any -nn port 3000` | See if packets actually arrive | `-c N`, `-A`, `-w file.pcap` |
| `ss -tulpn` | What's listening (topic 28) | see topic 28 |
| `ethtool eth0` | Link speed, duplex, driver | `-S` (NIC hardware stats), `-i` (driver) |
| `arping -I eth0 10.0.0.1` | ARP-level reachability (works when ICMP is blocked) | `-c N` |
| `netplan try` | Apply netplan config, **auto-revert in 120s** | `apply`, `generate` |
| `ifconfig` / `route -n` / `arp -n` | **LEGACY.** Read-only knowledge. | Use `ip` instead |

---

## When Would I Use This at Work?

### Scenario 1: New EC2 box, app is up, nothing can reach it
You deploy, `pm2 list` says `online`, `curl localhost:3000` works from inside the box, and the load balancer health check fails. You run `ss -tulpn | grep 3000`, see `127.0.0.1:3000`, and you're done in 15 seconds instead of spending an hour on security groups. Fix: `HOST=0.0.0.0` in the systemd unit's `Environment=` (topic 22).

### Scenario 2: Your CI pipeline can't reach the private artifact registry over the VPN
`curl` to it hangs after the TLS handshake. Small `GET`s work; the tarball download hangs. You recognize the MTU black hole instantly, run `ping -M do -s 1472 registry.internal`, get `mtu=1380`, set `ip link set wg0 mtu 1380` (and make it permanent in the WireGuard config), and add an MSS clamp on the gateway. Nobody else on the team would have found this.

### Scenario 3: A container can't talk to the database and the on-call is 40 minutes in
You run `docker exec db ss -tulpn` (or `nsenter` if the image has no tools), see the bind address, check `docker network inspect` for whether both containers are even on the same user-defined network, and confirm with `tcpdump -i any -nn port 5432` whether the SYN is arriving and being RST'd (app problem) or vanishing (network/firewall problem). That single distinction — **RST vs silence** — routes the whole investigation, and you can make it in one command.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 27 — DNS from the Command Line | You now know *which interface* a packet leaves by. Next: how a **name** becomes the IP you're routing to. |
| **Next** | 28 — Ports and Sockets | An interface is the *where*. A socket bound to a port is the *what*. `ss -tulpn` is the other half of every diagnosis here. |
| **Builds on** | 01 — What Linux Actually Is | The kernel owns the network stack; `ip` is a userspace program sending netlink messages across the syscall boundary. |
| **Builds on** | 02 — The Filesystem Hierarchy | `/sys/class/net/` and `/proc/net/` are how the kernel *exposes* interfaces as files. |
| **Builds on** | 03 — Everything Is a File | Interfaces, and the sockets bound to them, are surfaced through the filesystem. |
| **Used by** | 29 — Firewalls | iptables rules operate **per interface** (`-i eth0`, `-o docker0`) and per chain. Docker's NAT/DNAT rules are what make the diagram in this doc work. |
| **Used by** | 30 — curl and wget | Every `curl` starts with a route lookup and an interface choice you can now predict. |
| **Used by** | 33 — Node in Production | The bind address (`0.0.0.0` vs `127.0.0.1`) is a production config decision, set in your systemd unit or Docker env. |
| **Used by** | 35 — Security Basics | Binding to `127.0.0.1` *is* a legitimate hardening technique — for a service behind a local nginx. Knowing when it's protection vs. when it's an outage is the whole skill. |
