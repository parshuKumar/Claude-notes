# 29 — Firewalls

## ELI5 — The Simple Analogy

Imagine an office building with a **security desk** at the entrance.

- **The security desk itself is built into the building's structure** — it's part of the walls, part of the doors, welded in. You cannot remove it. This is **netfilter**: a framework baked into the Linux kernel.
- **The rulebook on the desk** is the list of rules: "let in anyone with a badge," "nobody enters through the loading dock," "block anyone from that one competitor."
- **The office manager who writes the rulebook** is `iptables`. She doesn't stand at the door — she just writes rules into the book that the guards follow.
- **The friendly intern who writes rules for the manager in plain English** is `ufw`. You say "let the plumber in," and the intern translates that into the manager's formal notation, who writes it in the guard's book.

The critical insight: **the guard (netfilter) is the only one who actually stops anybody.** iptables and ufw never touch a single packet. They just edit the rulebook.

And two more rules of the building:
- **The guard reads the rulebook top to bottom and stops at the first line that matches.** If line 3 says "let everyone in" and line 40 says "block competitors," line 40 never gets read. Order is everything.
- **There are two ways to say no.** The guard can slam the door and say nothing (the visitor stands there confused for 30 seconds, then gives up) — that's **DROP**. Or the guard can say "you're not on the list, go away" — the visitor leaves immediately — that's **REJECT**.

---

## Where This Lives in the Linux Stack

```
Hardware (NIC — the network card, receiving electrical signals)
  │
  └── KERNEL
       │
       ├── Network Driver (turns signals into packets)
       │
       ├── NETFILTER ◀◀◀ THIS TOPIC (the kernel part)
       │    Hooks inside the kernel's network stack. Every packet
       │    passes through these hooks. Rules live HERE, in kernel
       │    memory. This is where ACCEPT/DROP/REJECT actually happens.
       │
       ├── TCP/IP Stack (routing, TCP state machine, conntrack)
       │
       └── Sockets ────────────────────────┐
            │                              │
            └── System Calls (setsockopt, and NETLINK sockets ─────┐
                 │                                                 │
                 └── C Library                                     │
                      │                                            │
                      └── Shell                                    │
                           │                                       │
                           └── iptables / nft / ufw ◀◀◀ THIS TOPIC │
                               (the USERSPACE part — these are     │
                                just programs that send rules      │
                                DOWN into netfilter over a         │
                                netlink socket) ───────────────────┘
```

**The single most important sentence in this document:**
**netfilter is the kernel framework that filters packets. `iptables`, `nftables`, and `ufw` are just userspace programs that install rules into it. They are configuration tools, not firewalls.**

If you kill the `iptables` binary, your firewall keeps working. If you `rm /usr/sbin/iptables`, your firewall keeps working. The rules live in the kernel.

---

## What Is This?

A Linux firewall is a set of rules stored inside the kernel that decide, for every single packet that touches the machine, whether to ACCEPT it, DROP it, REJECT it, or rewrite it. The kernel framework is **netfilter**. You never talk to netfilter directly — you use a userspace tool.

The tools, from lowest to highest level:

```
┌──────────────────────────────────────────────────────────────┐
│  ufw          "allow 22/tcp"        ← friendly, what you use │
│    └── translates to ──▼                                     │
│  iptables     "-A INPUT -p tcp --dport 22 -j ACCEPT"         │
│    └── sends over netlink to ──▼                             │
│  nftables     (the modern kernel backend on Ubuntu 22.04+)   │
│    └── which IS a frontend to ──▼                            │
│  NETFILTER    (kernel hooks — where the packet is judged)    │
└──────────────────────────────────────────────────────────────┘
```

**Modern twist that confuses everyone:** On Ubuntu 22.04 and Debian 12, the `iptables` command you type is actually `iptables-nft` — a **compatibility shim**. It speaks the iptables syntax you know, but writes **nftables** rules into the kernel. Check it:

```bash
update-alternatives --display iptables
# iptables - auto mode
#   link best version is /usr/sbin/iptables-nft   ← this is the shim
```

So on a modern box, `iptables -L` and `nft list ruleset` are looking at the *same* rules from two different angles.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| The chains (INPUT vs FORWARD) | You will "block" a port with ufw, and your Docker container will still be wide open to the internet — because Docker traffic goes through FORWARD, not INPUT |
| Rule order (first match wins) | You'll add a DENY rule that does nothing, because an ACCEPT rule above it already matched |
| Conntrack / ESTABLISHED | You'll set `-P INPUT DROP`, and the box will instantly lose all outbound internet — `apt update` hangs forever — because you blocked the *replies* to your own requests |
| DROP vs REJECT | Your app will hang for 30 seconds on a call to an internal service instead of failing instantly, and you won't know why |
| Persistence | Your carefully-built iptables rules will vanish on the next reboot, and the postmortem will say "the firewall was fine until we restarted" |
| SSH-before-enable | You will run `ufw enable` on a remote box and permanently lock yourself out. This is not hypothetical. It happens constantly |
| Cloud security groups | You'll open port 443 in ufw, still get a timeout, and burn two hours before checking the AWS console |

---

## The Physical Reality

Rules do not live in a file. They live in **kernel memory**, in a set of tables and chains.

```
                  ┌─────────────────────────────────────────────┐
                  │           KERNEL MEMORY                      │
                  │                                              │
                  │  TABLE: filter          TABLE: nat           │
                  │  ┌────────────────┐    ┌─────────────────┐  │
                  │  │ CHAIN: INPUT   │    │ CHAIN: PREROUTING│ │
                  │  │  policy: DROP  │    │  policy: ACCEPT  │ │
                  │  │  ┌──────────┐  │    │  ┌────────────┐  │ │
                  │  │  │ rule 1   │  │    │  │ DOCKER     │  │ │
                  │  │  │ rule 2   │  │    │  │ (jump)     │  │ │
                  │  │  │ rule 3   │  │    │  └────────────┘  │ │
                  │  │  └──────────┘  │    │                  │ │
                  │  ├────────────────┤    ├─────────────────┤  │
                  │  │ CHAIN: FORWARD │    │ CHAIN: POSTROUT │  │
                  │  │ CHAIN: OUTPUT  │    │ CHAIN: OUTPUT   │  │
                  │  └────────────────┘    └─────────────────┘  │
                  │                                              │
                  │  Each rule has: MATCH CRITERIA + a TARGET   │
                  │  Each rule also has: pkts counter, bytes    │
                  │                       counter ← GOLD for    │
                  │                       debugging             │
                  └─────────────────────────────────────────────┘
```

**Two counters per rule.** Every rule keeps a packet count and a byte count. When you're debugging "is the firewall dropping my traffic?", these counters are the *definitive proof*. If the counter on the DROP rule goes up when you make a request, the firewall is your problem. If it doesn't move, the firewall is innocent — look elsewhere.

**Nothing here is on disk.** That's why persistence is a separate problem (see below).

---

## How It Works — Step by Step

### The Packet's Journey Through the Chains

This diagram is the whole topic. Learn to draw it.

```
                              ┌─────────────────┐
   PACKET ARRIVES ──────────▶ │   PREROUTING    │  tables: raw, mangle, NAT
   on the network card        │                 │  ← DNAT happens here
                              │  "before any    │    (docker -p 3000:3000
                              │   decision"     │     rewrites the DEST here)
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │ ROUTING DECISION│  Kernel asks:
                              │                 │  "Is this packet's destination
                              │  Is the dest IP │   IP one of MY addresses?"
                              │  mine, or not?  │
                              └──┬───────────┬──┘
                                 │           │
                    YES, it's    │           │  NO, it's for
                    FOR ME       │           │  someone else —
                                 │           │  I must ROUTE it
                        ┌────────▼───────┐  ┌▼─────────────────┐
                        │     INPUT      │  │     FORWARD      │
                        │  table: filter │  │  table: filter   │
                        │                │  │                  │
                        │  ← ufw's rules │  │  ← DOCKER lives  │
                        │    live HERE   │  │    HERE. Routers │
                        │                │  │    live here.    │
                        │  nginx :443    │  │    Containers.   │
                        │  sshd  :22     │  │    K8s.          │
                        │  node  :3000   │  │                  │
                        └────────┬───────┘  └────────┬─────────┘
                                 │                   │
                        ┌────────▼───────┐           │
                        │  LOCAL PROCESS │           │
                        │  (your Node    │           │
                        │   app's socket)│           │
                        └────────┬───────┘           │
                                 │                   │
                        ┌────────▼───────┐           │
                        │     OUTPUT     │           │
                        │  table: filter │           │
                        │                │           │
                        │  packets your  │           │
                        │  box GENERATES │           │
                        └────────┬───────┘           │
                                 │                   │
                                 └────────┬──────────┘
                                          │
                                 ┌────────▼────────┐
                                 │   POSTROUTING   │  table: NAT
                                 │                 │  ← SNAT / MASQUERADE
                                 │  "last chance   │    happens here (this is
                                 │   before wire"  │     how a container gets
                                 └────────┬────────┘     to the internet)
                                          │
                                          ▼
                                   OUT ON THE WIRE
```

**Read it as three journeys:**

| Journey | Chains traversed | Real example |
|---|---|---|
| Packet **FOR** this host | PREROUTING → INPUT | Someone curls your nginx on :443 |
| Packet **THROUGH** this host | PREROUTING → FORWARD → POSTROUTING | Traffic to your Docker container. **Never touches INPUT.** |
| Packet **FROM** this host | OUTPUT → POSTROUTING | Your Node app calling the Stripe API |

That middle row is the whole reason the Docker/ufw disaster (below) exists.

### The Tables

| Table | Purpose | Targets you'll use | When you touch it |
|---|---|---|---|
| **filter** | Allow/deny. The default table. | ACCEPT, DROP, REJECT | 95% of what you do |
| **nat** | Rewrite addresses | DNAT, SNAT, MASQUERADE, REDIRECT | Docker port publishing, container internet access, port forwarding |
| **mangle** | Modify packet headers (TOS, TTL, marks) | MARK, TOS | Rarely. QoS, policy routing |
| **raw** | Skip conntrack for performance | NOTRACK | Almost never. High-throughput edge cases |

`iptables -L` with no `-t` shows the **filter** table. That's why people don't notice Docker's rules — they're in **nat** and in the FORWARD chain.

### Rule Evaluation — Top to Bottom, First Match Wins

```
INPUT chain:
  ┌───────────────────────────────────────────────────────┐
  │ 1. -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT│ ── packet matches? → ACCEPT. STOP.
  ├───────────────────────────────────────────────────────┤
  │ 2. -i lo -j ACCEPT                                     │ ── no? next rule.
  ├───────────────────────────────────────────────────────┤
  │ 3. -p tcp --dport 22 -j ACCEPT                         │ ── no? next rule.
  ├───────────────────────────────────────────────────────┤
  │ 4. -p tcp --dport 443 -j ACCEPT                        │ ── no? next rule.
  ├───────────────────────────────────────────────────────┤
  │ 5. -s 1.2.3.4 -j DROP    ← DEAD CODE if 1.2.3.4 was    │
  │                            hitting port 443 (rule 4    │
  │                            already accepted it)        │
  └───────────────────────────────────────────────────────┘
              │
              ▼  no rule matched
        ┌───────────────┐
        │ CHAIN POLICY  │  the default verdict. `-P INPUT DROP`
        │  ACCEPT/DROP  │  = "if nothing explicitly allowed it, drop it"
        └───────────────┘
```

**Two consequences that will bite you:**

1. **A permissive rule above a restrictive one makes the restrictive one dead code.** Rule 5 above never fires.
2. **A `-j DROP` with no match criteria at the top of the chain kills everything below it.** `iptables -I INPUT 1 -j DROP` means "insert at position 1: drop everything." Your SSH session freezes the instant you press Enter.

### Stateful Filtering — Conntrack, and Why It's Non-Negotiable

Here's the trap. Your Node app calls `https://api.stripe.com`.

```
YOUR BOX                                          STRIPE
   │
   │  packet OUT (src :51234, dst :443) ──────────▶│   passes OUTPUT ✓ (OUTPUT policy is ACCEPT)
   │                                                │
   │◀────────── packet IN (src :443, dst :51234)   │   this is the RESPONSE
   │                                                │
   │   This inbound packet hits your INPUT chain.
   │   Your rules say: "allow 22, allow 443". This
   │   packet's DESTINATION port is 51234 — a random
   │   ephemeral port. It matches NOTHING.
   │
   │   INPUT policy is DROP.
   │
   │   ☠️  THE RESPONSE IS DROPPED.
   │
   │   Your app hangs. `apt update` hangs. `curl` times out.
   │   You swear the box "has no internet" — but OUTPUT is
   │   wide open. You blocked the RETURN traffic.
```

The kernel's **connection tracking** subsystem (conntrack) solves this. It remembers every connection your box initiates, and it can classify each incoming packet:

| ctstate | Meaning |
|---|---|
| `NEW` | First packet of a connection nobody has seen before (a TCP SYN) |
| `ESTABLISHED` | Part of a connection conntrack already knows about — **including the replies to connections YOU started** |
| `RELATED` | A new connection that is logically part of an existing one (an ICMP error, an FTP data channel) |
| `INVALID` | Doesn't fit any known connection. Usually junk. Drop it. |

**Therefore, the single most important rule in any Linux firewall:**

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

"If this packet belongs to a conversation we're already having, let it in." It must be near the **top** of the INPUT chain (both for correctness with a DROP policy and for performance — it short-circuits the vast majority of packets).

**Stateless vs stateful:**

| | Stateless | Stateful |
|---|---|---|
| What it knows | Only this packet's headers | This packet + the connection it belongs to |
| Cost | Zero memory | A conntrack table entry per connection |
| Can it allow return traffic? | Only by opening all high ports (terrible) | Yes, precisely |
| What Linux does | Almost never | Always, via `nf_conntrack` |

Look at the actual state table:

```bash
sudo conntrack -L | head -3
# tcp 6 431997 ESTABLISHED src=10.0.1.5 dst=52.9.1.1 sport=51234 dport=443 \
#   src=52.9.1.1 dst=10.0.1.5 sport=443 dport=51234 [ASSURED] mark=0 use=1

cat /proc/sys/net/netfilter/nf_conntrack_count   # current tracked connections
cat /proc/sys/net/netfilter/nf_conntrack_max     # the ceiling — 262144 typically
```

If `nf_conntrack_count` hits `nf_conntrack_max`, the kernel starts dropping new connections and logs `nf_conntrack: table full, dropping packet` to dmesg. That's a real production outage on a high-traffic proxy.

### DROP vs REJECT — A Real Distinction

```
DROP:
  Client ──SYN──▶  [ packet enters a black hole. no reply. ever. ]
  Client waits... 1s... 3s... 7s... 15s... TCP retransmit backoff...
  Client: "connect ETIMEDOUT" after ~30-130 seconds.

REJECT:
  Client ──SYN──▶  [ kernel sends back ICMP port-unreachable or TCP RST ]
  Client ◀─RST──
  Client: "connect ECONNREFUSED" in ~1 millisecond.
```

| | DROP | REJECT |
|---|---|---|
| Client experience | **Hangs**, then times out | **Fails instantly**: "connection refused" |
| Reveals the host exists? | No — looks like nothing is there | Yes — something answered |
| Good for | The public internet edge. Port scanners waste time and get no info. | Internal networks. Your own debugging. |
| Bad for | Debugging ("why is my app hanging for 30s instead of erroring?") | The public edge (confirms the host is alive) |

**The backend-dev pain:** your Node app calls an internal service. The firewall DROPs it. Your app's HTTP request doesn't error — it *hangs* until your client timeout (or forever, if you didn't set one). Requests pile up, the event loop fills with pending sockets, and the service falls over. With REJECT it would have failed in 1ms with `ECONNREFUSED` and your retry logic would have handled it cleanly.

**Rule of thumb:** DROP facing the internet. REJECT facing your own VPC.

ufw's default is actually to REJECT for `deny` and DROP for... well, ufw uses `reject` and `deny` as separate verbs — `ufw deny 3000` DROPs, `ufw reject 3000` sends the RST.

---

## Exact Syntax Breakdown

### The anatomy of an iptables rule — every token

```
iptables -A INPUT -i eth0 -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
│        │  │     │       │      │          │                    │      │  │
│        │  │     │       │      │          │                    │      │  └── the TARGET: what to
│        │  │     │       │      │          │                    │      │      DO. ACCEPT / DROP /
│        │  │     │       │      │          │                    │      │      REJECT / LOG / a chain
│        │  │     │       │      │          │                    │      └── -j = "jump to" this target
│        │  │     │       │      │          │                    └── the state to match: only the
│        │  │     │       │      │          │                        FIRST packet of a new connection
│        │  │     │       │      │          └── --ctstate: an option of the conntrack module
│        │  │     │       │      └── -m = load a MATCH MODULE. "conntrack" gives us state awareness.
│        │  │     │       │              (other modules: limit, multiport, string, recent)
│        │  │     │       └── --dport = DESTINATION port. (--sport = source port.)
│        │  │     │                     Only valid after -p tcp or -p udp.
│        │  │     └── -p = PROTOCOL. tcp / udp / icmp / all
│        │  └── -i = INPUT interface. Which NIC did it arrive on? (eth0, lo, docker0)
│        │          -o = OUTPUT interface. Only valid in OUTPUT/FORWARD/POSTROUTING.
│        └── the CHAIN to add this rule to
└── -A = APPEND to the end of the chain
```

### The commands that manipulate rules

```
iptables -A INPUT ...      APPEND to the bottom of INPUT
iptables -I INPUT ...      INSERT at position 1 (the TOP)
iptables -I INPUT 3 ...    INSERT at position 3
iptables -D INPUT 3        DELETE rule number 3
iptables -D INPUT -p tcp --dport 22 -j ACCEPT    DELETE by exact match
iptables -R INPUT 3 ...    REPLACE rule 3
iptables -F INPUT          FLUSH — delete ALL rules in INPUT
iptables -F                FLUSH every chain in the filter table
iptables -P INPUT DROP     set the chain POLICY (the default verdict)
iptables -N MYCHAIN        create a NEW custom chain
iptables -Z                ZERO the packet/byte counters
```

**`-A` vs `-I` is the difference that bites you.** You write your rules top-down mentally, but `-A` appends to the *bottom* — below your DROP-all. Your new ACCEPT rule is dead code. Use `-I` when you need it first, `-A` when order doesn't matter.

### Reading the rules

```
iptables -L INPUT -n -v --line-numbers
│        │        │  │  │
│        │        │  │  └── number each rule — you NEED these numbers to -D a rule
│        │        │  └── VERBOSE: show the packet and byte COUNTERS, and the interfaces.
│        │        │      This is the flag that makes iptables a debugging tool.
│        │        └── NUMERIC: do NOT resolve IPs to hostnames or ports to service names.
│        │            ⚠️  WITHOUT -n, iptables does a reverse-DNS lookup on EVERY IP in
│        │                every rule. On a box with broken DNS this HANGS for minutes.
│        │                Always use -n. Always.
│        └── the chain (omit to list all chains in the table)
└── -L = LIST
```

### ufw commands

```
ufw allow from 10.0.0.0/8 to any port 5432 proto tcp
│   │     │    │           │   │    │    │    │
│   │     │    │           │   │    │    │    └── restrict to TCP
│   │     │    │           │   │    │    └── the port (Postgres)
│   │     │    │           │   │    └── "port" keyword
│   │     │    │           │   └── "any" = any destination IP on this host
│   │     │    │           └── "to" = destination
│   │     │    └── the SOURCE network in CIDR — your private subnet ONLY
│   │     └── "from" = source
│   └── the verb: allow / deny / reject / limit
└── the ufw binary

Translation into iptables (roughly what ufw actually installs):
  -A ufw-user-input -p tcp -s 10.0.0.0/8 --dport 5432 -j ACCEPT
```

---

## Example 1 — Basic: A Sane Firewall From Scratch with ufw

You have a fresh Ubuntu box. You want: SSH in, HTTP/HTTPS in, everything else denied, and outbound traffic unrestricted.

```bash
# ═══ STEP 0 — THE SAFETY NET. DO THIS FIRST, EVERY TIME, ON A REMOTE BOX. ═══
# If you lock yourself out, this un-does it in 5 minutes.
echo "ufw disable" | sudo at now + 5 minutes
# job 3 at Fri Jul 12 14:35:00 2026
# (needs `apt install at`. Alternative: sudo sh -c 'sleep 300 && ufw disable' &)

# ═══ STEP 1 — Look before you leap ═══
sudo ufw status verbose
# Status: inactive          ← nothing is on yet. Good.

# ═══ STEP 2 — Set the DEFAULT POLICIES ═══
sudo ufw default deny incoming    # deny-by-default. The only sane posture.
sudo ufw default allow outgoing   # let the box reach the internet (apt, npm, APIs)
# Default incoming policy changed to 'deny'
# Default outgoing policy changed to 'allow'

# ═══ STEP 3 — ⚠️⚠️⚠️ ALLOW SSH *BEFORE* YOU ENABLE ⚠️⚠️⚠️ ═══
sudo ufw allow OpenSSH            # uses the app profile (see `ufw app list`)
# — or, equivalently —
sudo ufw allow 22/tcp
# Rules updated
# Rules updated (v6)

# ═══ STEP 4 — Allow the web ports ═══
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
# or in one shot:
sudo ufw allow proto tcp from any to any port 80,443

# ═══ STEP 5 — NOW enable ═══
sudo ufw enable
# Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
#   ^^^ this prompt is ufw literally warning you about the lockout
# Firewall is active and enabled on system startup

# ═══ STEP 6 — Verify, and CONFIRM YOUR SSH SESSION STILL WORKS ═══
# Open a SECOND terminal and ssh in again. Do NOT close the first one until it works.
sudo ufw status verbose
# Status: active
# Logging: on (low)
# Default: deny (incoming), allow (outgoing), disabled (routed)
#          ^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^
#          the sane posture   outbound is free  FORWARD is off (until Docker turns it on)
# New profiles: skip
#
# To                         Action      From
# --                         ------      ----
# 22/tcp                     ALLOW IN    Anywhere
# 80/tcp                     ALLOW IN    Anywhere
# 443/tcp                    ALLOW IN    Anywhere
# 22/tcp (v6)                ALLOW IN    Anywhere (v6)
# ...

# ═══ STEP 7 — Cancel the safety net, since you're not locked out ═══
sudo atq            # list scheduled jobs
sudo atrm 3         # remove job 3
```

**What ufw actually built for you.** Look under the hood:

```bash
sudo iptables -L INPUT -n -v --line-numbers
# Chain INPUT (policy DROP 0 packets, 0 bytes)
# num pkts bytes target     prot opt in  out source     destination
# 1   1420  118K ufw-before-logging-input  all -- *  *  0.0.0.0/0  0.0.0.0/0
# 2   1420  118K ufw-before-input          all -- *  *  0.0.0.0/0  0.0.0.0/0
# 3     12   720 ufw-after-input           all -- *  *  0.0.0.0/0  0.0.0.0/0
# ...
#           ^^^ ufw doesn't put rules in INPUT directly. It creates its own chains
#               and JUMPS to them. Your `ufw allow` rules land in `ufw-user-input`.

sudo iptables -L ufw-before-input -n -v | head -5
# Chain ufw-before-input (1 references)
#  pkts bytes target     prot opt in  out  source     destination
#   340 28560 ACCEPT     all  --  lo  *    0.0.0.0/0  0.0.0.0/0
#  1080 89K   ACCEPT     all  --  *   *    0.0.0.0/0  0.0.0.0/0  ctstate RELATED,ESTABLISHED
#                                                                ^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                                            THERE IT IS. ufw installs the conntrack rule
#                                            for you, before any of your rules. This is
#                                            why the box still has internet after you
#                                            set `default deny incoming`.
```

---

## Example 2 — Production Scenario

**2:14am. PagerDuty.** Your Node API at `api.yourco.com` is returning connection timeouts from the outside world. You SSH in (SSH works, so it's not a total network failure).

```bash
# ═══ 1. Is the app even alive and LISTENING? (Topic 28 — ports and sockets) ═══
sudo ss -tulpn | grep -E ':(3000|443|80)'
# tcp  LISTEN 0  511  0.0.0.0:443   0.0.0.0:*  users:(("nginx",pid=812,fd=6))
# tcp  LISTEN 0  511  0.0.0.0:80    0.0.0.0:*  users:(("nginx",pid=812,fd=6))
# tcp  LISTEN 0  511  127.0.0.1:3000 0.0.0.0:* users:(("node",pid=1455,fd=20))
#
# ✓ nginx is listening on 443. node is listening on 3000 (localhost-only — correct).
#   So it's NOT "the app is down."

# ═══ 2. Does the app work FROM ON THE BOX? ═══
curl -sS -o /dev/null -w '%{http_code}\n' https://localhost/health -k
# 200
# ✓ The app is healthy. nginx is proxying fine. The problem is between the
#   internet and this box.

# ═══ 3. Are packets even ARRIVING? ═══
sudo tcpdump -i any -n port 443 -c 5
# tcpdump: listening on any...
# 02:16:41.221 IP 203.0.113.9.51422 > 10.0.1.5.443: Flags [S], seq 91882...
# 02:16:42.223 IP 203.0.113.9.51422 > 10.0.1.5.443: Flags [S], seq 91882...  ← RETRANSMIT
# 02:16:44.227 IP 203.0.113.9.51422 > 10.0.1.5.443: Flags [S], seq 91882...  ← RETRANSMIT
#
# ✓ Packets ARE arriving. So it's not the cloud security group, and not DNS.
# ✗ But we only see SYN, SYN, SYN — no SYN-ACK going back.
#   The kernel is receiving them and NOT answering.
#   → Something between the NIC and the socket is eating them. That's netfilter.

# ═══ 4. Watch the COUNTERS. This is the definitive proof. ═══
sudo iptables -L INPUT -n -v --line-numbers
# Chain INPUT (policy DROP 8842 packets, 512K bytes)
#                              ^^^^ the POLICY counter is climbing
# num  pkts  bytes target             prot opt in  out source     destination
# 1   58221  4.1M  ufw-before-input   all  --  *   *   0.0.0.0/0  0.0.0.0/0
# ...

sudo iptables -L ufw-user-input -n -v --line-numbers
# Chain ufw-user-input (1 references)
# num  pkts bytes target  prot opt in  out source     destination
# 1    1204 72240 ACCEPT  tcp  --  *   *   0.0.0.0/0  0.0.0.0/0  tcp dpt:22
# 2       0     0 ACCEPT  tcp  --  *   *   0.0.0.0/0  0.0.0.0/0  tcp dpt:80
#         ^^^ ZERO packets on port 80
# ← AND THERE IS NO RULE FOR 443 AT ALL.
```

**Found it.** The 443 rule is gone. Check what happened:

```bash
sudo ufw status numbered
# Status: active
#      To                         Action      From
#      --                         ------      ----
# [ 1] 22/tcp                     ALLOW IN    Anywhere
# [ 2] 80/tcp                     ALLOW IN    Anywhere
# [ 3] 22/tcp (v6)                ALLOW IN    Anywhere (v6)
# [ 4] 80/tcp (v6)                ALLOW IN    Anywhere (v6)

# Who touched the firewall?
sudo grep -i ufw /var/log/auth.log | tail -3
# Jul 12 01:47:02 api-01 sudo: deploy : TTY=pts/1 ; PWD=/home/deploy ;
#   USER=root ; COMMAND=/usr/sbin/ufw delete 3
# ← someone ran `ufw delete 3` at 1:47am. Rule 3 WAS the 443 rule.
#   They meant to delete a different rule and the numbers had shifted.
#   (THIS is why `ufw status numbered` before `ufw delete` matters — the
#    numbers renumber after every delete.)

# ═══ 5. THE FIX ═══
sudo ufw allow 443/tcp
# Rule added
# Rule added (v6)

# ═══ 6. VERIFY — from ANOTHER machine, not this one ═══
# (on your laptop)
curl -sS -o /dev/null -w 'HTTP %{http_code} in %{time_total}s\n' https://api.yourco.com/health
# HTTP 200 in 0.184s

# ═══ 7. Confirm the counters are now moving on the ACCEPT rule ═══
sudo iptables -L ufw-user-input -n -v | grep 443
#    88  5280 ACCEPT  tcp  --  *  *  0.0.0.0/0  0.0.0.0/0  tcp dpt:443
#    ^^ climbing. Traffic is flowing.
```

**The five-minute lesson from this incident:** `tcpdump` showed packets arriving but no reply → that proves the kernel ate them → the packet counters told you exactly which chain → `ufw status numbered` + `auth.log` told you who and when.

---

## The Docker + ufw Disaster

**This section alone justifies the doc. It burns real teams, and it is silently catastrophic.**

### The setup

```bash
sudo ufw status
# Status: active
# To            Action      From
# --            ------      ----
# 22/tcp        ALLOW IN    Anywhere
# 443/tcp       ALLOW IN    Anywhere
#
# ✓ "Great. Deny by default. Only 22 and 443 are open. We're secure."

docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=hunter2 postgres:16
# ✓ "It's just for the app to talk to. ufw denies 5432, so it's internal."
```

**From your laptop, right now:**

```bash
psql -h 203.0.113.50 -p 5432 -U postgres
# Password for user postgres:
# psql (16.2)
# postgres=#
```

**Your production database is on the public internet.** `ufw status` says deny. It is lying to you — or rather, you are misreading it.

### Why — the packet path

```
Packet: dst = 203.0.113.50 : 5432     (your public IP, the published port)

  ┌─────────────┐
  │ PREROUTING  │  ← Docker installed a rule in the nat table:
  │  nat table  │      -A DOCKER ! -i docker0 -p tcp --dport 5432 \
  │             │         -j DNAT --to-destination 172.17.0.2:5432
  │             │
  │  DNAT!      │  The destination is REWRITTEN from 203.0.113.50:5432
  └──────┬──────┘  to 172.17.0.2:5432 — the container's IP.
         │
  ┌──────▼──────────┐
  │ ROUTING DECISION│  "Is 172.17.0.2 one of MY addresses?"
  │                 │   NO. It's on the docker0 bridge. I must ROUTE it.
  └──────┬──────────┘
         │
         ▼
  ┌──────────────┐         ┌──────────────┐
  │   FORWARD    │  NOT ──▶│    INPUT     │   ufw's rules live here.
  │              │         │              │   ✗ NEVER EVALUATED.
  │  Docker put  │         │  ufw-user-   │   ✗ Your "deny incoming" is
  │  ACCEPT rules│         │  input       │     IRRELEVANT to this packet.
  │  HERE, in the│         └──────────────┘
  │  DOCKER chain│
  │              │
  │  ✓ ACCEPTED  │
  └──────┬───────┘
         │
         ▼
   CONTAINER :5432   ← the internet is now talking to your database
```

**In one sentence: ufw filters the INPUT chain. Docker's DNAT'd traffic never enters the INPUT chain — it goes through FORWARD, where Docker has already inserted its own ACCEPT rules.** ufw and Docker are two tools writing into the same netfilter, and Docker wins because it plays in a different chain.

**See it yourself:**

```bash
sudo iptables -t nat -L DOCKER -n
# Chain DOCKER (2 references)
# target  prot opt source     destination
# RETURN  all  --  0.0.0.0/0  0.0.0.0/0
# DNAT    tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:5432 to:172.17.0.2:5432
#                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                                        there is your "closed" port

sudo iptables -L DOCKER -n -v
# Chain DOCKER (1 references)
#  pkts bytes target  prot opt in       out      source     destination
#   142  8520 ACCEPT  tcp  --  !docker0 docker0  0.0.0.0/0  172.17.0.2  tcp dpt:5432
#   ^^^ and 142 packets have already gone through it
```

### The fixes, best to worst

**FIX 1 — Bind the published port to localhost. The simplest and best answer.**

```bash
docker run -d -p 127.0.0.1:5432:5432 postgres:16
#              ^^^^^^^^^ bind the host side of the publish to loopback ONLY

# In docker-compose.yml:
services:
  db:
    image: postgres:16
    ports:
      - "127.0.0.1:5432:5432"   # ✓ not "5432:5432"
```

Now Docker's DNAT rule only matches traffic arriving on `lo`. The internet cannot reach it. Your app on the host can. Your `psql` over an SSH tunnel can. **Make this a code-review rule in your team: any `ports:` entry without an IP prefix is a bug unless it's genuinely meant to be public.**

Verify:
```bash
sudo iptables -t nat -L DOCKER -n | grep 5432
# DNAT tcp -- 0.0.0.0/0  127.0.0.1  tcp dpt:5432 to:172.17.0.2:5432
#                        ^^^^^^^^^ destination is now constrained to loopback
```

**FIX 2 — Don't publish the port at all.** If only *other containers* need the DB, put them on a shared Docker network and let DNS-by-service-name do the work. Publishing a port is only for host/internet access.

```yaml
services:
  db:
    image: postgres:16
    networks: [backend]
    # NO ports: section at all. ✓
  api:
    image: myapi
    networks: [backend]
    environment:
      DATABASE_URL: postgres://db:5432/app   # "db" resolves via Docker's DNS
    ports:
      - "127.0.0.1:3000:3000"
networks:
  backend:
```

**FIX 3 — Tell Docker not to manage iptables at all.**

```json
// /etc/docker/daemon.json
{ "iptables": false }
```
Then `systemctl restart docker`. **Now YOU own all the NAT/FORWARD rules — including the MASQUERADE rule that gives containers outbound internet.** If you don't write it, your containers cannot reach npm or your APIs. This is a real, advanced, easy-to-get-wrong option. Only do this if you know exactly what you're replacing.

**FIX 4 — ufw-docker style rules.** Add a `DOCKER-USER` chain (Docker guarantees it is evaluated *before* its own rules, and it never touches it):

```bash
# Block everything into containers except from your private subnet
sudo iptables -I DOCKER-USER -i eth0 ! -s 10.0.0.0/8 -j DROP
# Then persist it (see below), because DOCKER-USER rules are not ufw-managed.
```

`DOCKER-USER` is the officially sanctioned hook. It is the right place for hand-written container firewall rules.

---

## Persistence — Rules Live in the Kernel and Die on Reboot

```
   RAM (kernel netfilter tables)          DISK
   ┌────────────────────────┐            ┌──────────────────────────┐
   │  INPUT: 12 rules       │            │  (nothing)               │
   │  policy DROP           │  reboot ▶  │                          │
   │  FORWARD: 8 rules      │  ═══════▶  │  netfilter comes back    │
   │  nat: 4 rules          │            │  EMPTY. Policies reset   │
   └────────────────────────┘            │  to ACCEPT.              │
                                          └──────────────────────────┘
```

**Raw `iptables` commands are NOT persistent.** Reboot and they are gone. "The firewall was fine until we rebooted" is a genuine, common, embarrassing incident.

```bash
# Dump the current ruleset to a text file
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6

# Restore from that file
sudo iptables-restore < /etc/iptables/rules.v4

# The package that does this automatically at boot:
sudo apt install iptables-persistent
# (the install prompts: "Save current IPv4 rules? <Yes>")
# Later, to re-save after changes:
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

**ufw handles persistence for you.** `ufw enable` sets up a systemd unit (`ufw.service`) that re-applies your rules from `/etc/ufw/` at every boot. This is a real reason to prefer ufw for a single host: one less thing to get wrong.

```bash
systemctl is-enabled ufw
# enabled
ls /etc/ufw/
# after.rules  before.rules  sysctl.conf  ufw.conf  user.rules  user6.rules
#                                                     ^^^^^^^^^^ your `ufw allow` rules
```

---

## The Second Firewall — Cloud Security Groups

On AWS/GCP/Azure, there are **two** firewalls in the path, and both must allow the packet:

```
   INTERNET
      │
      ▼
 ┌─────────────────────────────────────┐
 │  CLOUD SECURITY GROUP / VPC FIREWALL│  ← runs OUTSIDE your VM, in the
 │  (AWS SG, GCP firewall rule)        │    hypervisor/network fabric.
 │                                     │    Your box CANNOT SEE IT.
 │  Blocks here: packet never          │    `tcpdump` on the box shows NOTHING.
 │  reaches the VM at all              │    Stateful by default.
 └──────────────┬──────────────────────┘
                │
                ▼
 ┌─────────────────────────────────────┐
 │  YOUR VM's NIC                      │
 │  └── netfilter / ufw                │  ← `tcpdump` sees the packet arrive.
 │      Blocks here: tcpdump SEES the  │    Counters increment.
 │      packet, counters increment     │
 └──────────────┬──────────────────────┘
                ▼
             your app
```

**The diagnostic that separates them:** run `tcpdump -i any port 443`.
- **Nothing shows up at all** → the packet never reached your box. It's the **cloud security group** (or a route/NACL). Go to the console.
- **SYNs show up, no SYN-ACK back** → the packet reached the box and the kernel ate it. It's **ufw/iptables**. Fix it here.

Nothing has burned more collective engineer-hours than opening a port in ufw, seeing it still time out, and not checking the security group (or vice versa). **Check both. Always.**

---

## nftables — The Successor

nftables replaces iptables/ip6tables/arptables/ebtables with one unified tool and a better syntax. On Ubuntu 22.04 it is already the kernel backend; the `iptables` command is a shim over it.

```bash
sudo nft list ruleset          # THE command. Shows everything, all tables, all families.
sudo nft list tables
# table ip filter
# table ip nat
# table inet ufw-... (on newer ufw)

# nftables syntax, for comparison:
# table inet myfilter {
#   chain input {
#     type filter hook input priority 0; policy drop;
#     ct state established,related accept
#     iif lo accept
#     tcp dport { 22, 80, 443 } accept     ← native sets. iptables needs 3 rules.
#   }
# }
```

**Practical advice for now:** learn iptables concepts (chains, tables, targets, conntrack) — they map 1:1 onto nftables. Use `ufw` day-to-day. Use `nft list ruleset` when you need to see the *real* rules on a modern box, especially when iptables and nft seem to disagree.

---

## Common Mistakes

### Mistake 1: `ufw enable` before allowing SSH — instant, total lockout

**Wrong:**
```bash
sudo ufw default deny incoming
sudo ufw enable
# Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
# ← your SSH session freezes mid-keystroke. Forever.
```

**Root cause (at the kernel level):** ufw set `-P INPUT DROP` and installed no rule for port 22. Your existing SSH connection's packets are ESTABLISHED, so *technically* ufw's conntrack rule should keep it alive — but ufw also flushes and rebuilds the chains, and the conntrack table can be reset. In practice, you lose the session. And you can *never reconnect*, because a new SSH connection is `NEW` state, and nothing accepts it.

**What the broken state looks like:** SSH hangs on connect and eventually times out. The box is up (it responds to ping, if you allowed ICMP). You cannot get in. Your only path is the cloud provider's **serial console / VNC console**.

**Right:**
```bash
sudo ufw allow OpenSSH        # ◀── FIRST. ALWAYS.
sudo ufw default deny incoming
sudo ufw enable
```

**Prevention — the professional habit.** Before you touch firewall rules on a remote box, schedule an escape hatch:
```bash
# Option A (needs `at`):
echo "ufw disable" | sudo at now + 5 minutes

# Option B (works anywhere):
sudo sh -c 'nohup sh -c "sleep 300 && ufw disable" >/dev/null 2>&1 &'

# Option C — for raw iptables, save a known-good ruleset first:
sudo iptables-save > /root/fw-good.rules
sudo sh -c 'nohup sh -c "sleep 300 && iptables-restore < /root/fw-good.rules" &'

# ... now make your changes, verify from a SECOND ssh session, and only then:
sudo atq && sudo atrm <jobid>     # cancel the rollback
```
Do this every single time. It costs 10 seconds and has saved careers.

---

### Mistake 2: Setting `-P INPUT DROP` without the conntrack rule — "the box has no internet"

**Wrong:**
```bash
sudo iptables -P INPUT DROP
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
# "OUTPUT is still ACCEPT, so outbound is fine, right?"

sudo apt update
# 0% [Connecting to archive.ubuntu.com]   ← hangs forever
```

**Root cause:** Your DNS query goes OUT (fine — OUTPUT policy is ACCEPT). The DNS *response* comes back IN, destined for your ephemeral source port (e.g. UDP :44821). It hits INPUT. No rule matches. Policy = DROP. **You blocked the return traffic of your own outbound connections.** The box appears to have no network, even though nothing about OUTPUT changed.

**Right:**
```bash
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT  # ◀── FIRST
sudo iptables -A INPUT -i lo -j ACCEPT                                       # ◀── SECOND
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -P INPUT DROP
```

**And don't forget `-i lo`.** Your Node app connecting to Postgres on `127.0.0.1:5432` goes through the INPUT chain too. With a DROP policy and no loopback rule, **your app cannot talk to its own database.** This one produces the most baffling symptom of all.

**Prevention:** ufw installs both of these for you in `ufw-before-input`. Use ufw.

---

### Mistake 3: `-A` when you meant `-I` — the dead rule

**Wrong:**
```bash
# Your chain currently ends with: -A INPUT -j DROP
sudo iptables -A INPUT -p tcp --dport 3000 -j ACCEPT   # appends BELOW the DROP
sudo iptables -L INPUT -n --line-numbers
# 1  ACCEPT  all  ctstate RELATED,ESTABLISHED
# 2  ACCEPT  tcp  dpt:22
# 3  DROP    all  0.0.0.0/0    ← everything stops here
# 4  ACCEPT  tcp  dpt:3000     ← DEAD CODE. Never reached.
```

**Root cause:** First match wins. The DROP at rule 3 has no match criteria, so it matches everything, so rule 4 is unreachable. The packet counter on rule 4 will stay at 0 forever — **which is exactly how you diagnose this.**

**Right:**
```bash
sudo iptables -I INPUT 3 -p tcp --dport 3000 -j ACCEPT   # insert ABOVE the DROP
# Verify: rule 3 is now the ACCEPT, and the DROP moved to 4.
```

**Prevention:** always `iptables -L -n -v --line-numbers` before and after. If a rule's packet counter never moves, it's either dead code or genuinely unused — investigate which.

---

### Mistake 4: `iptables -F` with a DROP policy — locking yourself out with a "reset"

**Wrong:**
```bash
sudo iptables -F     # "let me just start clean"
# ← your SSH session dies instantly.
```

**Root cause:** `-F` flushes the *rules*. It does **not** reset the *policy*. Your INPUT policy is still DROP. You have just deleted the rule that allowed SSH, and the rule that allowed ESTABLISHED. Every packet now falls through to the DROP policy. Including yours.

**Right — the safe reset order:**
```bash
sudo iptables -P INPUT ACCEPT      # ◀── policy FIRST
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -F                   # now flush
sudo iptables -t nat -F
sudo iptables -X                   # delete custom chains
```

**With ufw:** `sudo ufw reset` does this correctly and safely (it disables the firewall first, and backs up your rules to `/etc/ufw/*.rules.<timestamp>`).

---

### Mistake 5: Trusting `ufw status` while running Docker

**Wrong:** "`ufw status` says deny incoming, so my `-p 5432:5432` Postgres container is not exposed."

**Root cause:** Covered in full above. ufw filters INPUT. Docker's published ports are DNAT'd in PREROUTING and traverse FORWARD, where Docker inserted its own ACCEPT. ufw's rules are simply not in the path.

**Diagnosis — check from OUTSIDE the box, always:**
```bash
# From your laptop, NOT from the server:
nc -zv -w3 203.0.113.50 5432
# Connection to 203.0.113.50 5432 port [tcp/postgresql] succeeded!
# ← if this succeeds, the port is open to the internet, whatever ufw claims.
```

**Right:** `-p 127.0.0.1:5432:5432`, or no published port at all.

**Prevention:** After every deploy, run an external port scan of your own host and diff it against the ports you *intended* to expose. Ten lines of CI. Automate the paranoia.

---

## Hands-On Proof

```bash
# PROVE IT: netfilter is in the KERNEL, not in the iptables binary
lsmod | grep -E '^(nf_|iptable|nft)' | head
# nft_chain_nat  16384  3
# nf_nat         49152  2 nft_chain_nat,xt_MASQUERADE
# nf_conntrack   172032 4 xt_conntrack,nf_nat,...
# ^^^ these are KERNEL MODULES. The rules live inside them.

# PROVE IT: rules are in kernel memory, not in a file
sudo iptables -L INPUT -n | md5sum     # snapshot
sudo iptables -A INPUT -p tcp --dport 9999 -j ACCEPT
sudo iptables -L INPUT -n | md5sum     # changed — instantly, no file was written
sudo find /etc /var -newermt '-1 minute' -type f 2>/dev/null | grep -i -E 'ipt|fire'
# (nothing — no file changed. The rule went straight into the kernel.)
sudo iptables -D INPUT -p tcp --dport 9999 -j ACCEPT    # clean up

# PROVE IT: iptables on Ubuntu 22.04 is really nftables
readlink -f $(which iptables)
# /usr/sbin/xtables-nft-multi     ← the nft-backed shim
sudo nft list ruleset | head -20
# (you will see your iptables rules, rendered as nftables)

# PROVE IT: the packet counters move when traffic arrives
sudo iptables -Z INPUT                      # zero the counters
curl -s -o /dev/null http://localhost/       # generate traffic
sudo iptables -L INPUT -n -v | head -3       # watch pkts/bytes climb

# PROVE IT: DROP hangs, REJECT fails instantly
sudo iptables -I INPUT 1 -p tcp --dport 9999 -j DROP
time curl --connect-timeout 5 http://localhost:9999
# curl: (28) Connection timed out after 5001 ms      ← HANGS
sudo iptables -D INPUT -p tcp --dport 9999 -j DROP

sudo iptables -I INPUT 1 -p tcp --dport 9999 -j REJECT
time curl --connect-timeout 5 http://localhost:9999
# curl: (7) Failed to connect ... Connection refused  ← INSTANT (real 0m0.008s)
sudo iptables -D INPUT -p tcp --dport 9999 -j REJECT

# PROVE IT: conntrack is tracking your connections right now
curl -s https://example.com > /dev/null &
sudo conntrack -L 2>/dev/null | grep 443 | head -2
cat /proc/sys/net/netfilter/nf_conntrack_count

# PROVE IT: Docker bypasses ufw
docker run -d --name proof -p 9999:80 nginx:alpine
sudo ufw status | grep 9999          # (nothing — ufw doesn't know about it)
sudo iptables -t nat -L DOCKER -n | grep 9999
# DNAT tcp -- 0.0.0.0/0  0.0.0.0/0  tcp dpt:9999 to:172.17.0.2:80
# ← THERE it is, in the nat table, invisible to `ufw status`
docker rm -f proof

# PROVE IT: ufw rate-limiting exists and is a real iptables `recent` match
sudo ufw limit ssh
sudo iptables -L ufw-user-input -n -v | grep -A2 'dpt:22'
#  ... recent: UPDATE seconds: 30 hit_count: 6 name: DEFAULT side: source
#  ^^^ 6 connections in 30 seconds from one source → dropped
```

---

## Practice Exercises

### Exercise 1 — Easy

On any Linux box (a Docker container with `--cap-add=NET_ADMIN` works, or a throwaway VM):

```bash
sudo iptables -L -n -v --line-numbers
sudo iptables -t nat -L -n -v
sudo ufw status verbose
```

**Do:** For each of the three default filter chains (INPUT, FORWARD, OUTPUT), write down (a) its policy, (b) how many rules it has, (c) the total packet count that has hit the policy. Then explain, in one sentence each, what kind of packet reaches each chain.

---

### Exercise 2 — Medium

Build a correct firewall by hand, with raw iptables, on a *local* VM (never do this first on a remote box).

```bash
# Requirements:
#  - INPUT policy DROP
#  - loopback allowed
#  - ESTABLISHED,RELATED allowed
#  - SSH (22) allowed, but rate-limited to 4 new connections per minute per IP
#  - HTTP (80) and HTTPS (443) allowed from anywhere
#  - Postgres (5432) allowed ONLY from 10.0.0.0/8
#  - everything else logged before it's dropped (hint: a LOG target with --limit)
#  - saved so it survives a reboot
```

**Then:** verify it with `nc -zv` from another machine, and confirm the packet counters on each rule. Paste `iptables -L -n -v --line-numbers` and your `nc` output.

---

### Exercise 3 — Hard (Production Simulation)

**The Docker exposure audit.**

```bash
# 1. Set up the vulnerable state:
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw allow OpenSSH
sudo ufw enable
docker run -d --name db -e POSTGRES_PASSWORD=x -p 5432:5432 postgres:16

# 2. From ANOTHER machine, prove the port is open despite ufw saying deny.
#    (nc -zv <ip> 5432)

# 3. Explain the packet path in writing — which chains, which tables,
#    where the DNAT happened, why INPUT was skipped. Draw it.

# 4. Find the exact iptables rules Docker installed. Show them.
#    (hint: -t nat -L DOCKER, and -L DOCKER, and -L FORWARD)

# 5. Fix it FOUR different ways, and after each one, re-verify from outside
#    that the port is now closed AND that a container on the host can still
#    reach the DB:
#      a) -p 127.0.0.1:5432:5432
#      b) no published port + a Docker network
#      c) a DOCKER-USER chain rule
#      d) (read-only — don't actually break your box) explain what
#         {"iptables": false} would require you to write by hand

# 6. Write a 5-line shell script that a CI job could run to detect this
#    class of bug: enumerate all published Docker ports and fail if any
#    is bound to 0.0.0.0.
#    (hint: docker ps --format '{{.Ports}}')
```

---

## Mental Model Checkpoint

Answer from memory.

1. **What is netfilter, and how is it different from iptables and ufw?** Which one actually stops a packet?
2. **Draw the packet path.** Which chain does a packet destined for *this host* traverse? Which chain does a packet destined for a *Docker container* traverse? Why does that difference make ufw useless for container ports?
3. **You set `-P INPUT DROP` and allowed only port 22. `apt update` now hangs forever. Why?** What single rule fixes it, and where must it go?
4. **DROP vs REJECT:** what does the client experience with each, and which do you use on an internet-facing host vs an internal VPC — and why?
5. **You add `iptables -A INPUT -p tcp --dport 3000 -j ACCEPT` and port 3000 is still closed.** Give two possible reasons, and the exact command that would tell you which one it is.
6. **Your carefully-built iptables ruleset vanished after a reboot.** Why, and what are the two ways to fix it?
7. **A port times out from the internet. `tcpdump -i any port 443` on the box shows nothing at all.** Where is the packet being blocked — and is looking at `iptables -L` going to help?

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `ufw status` | Show ufw rules | `verbose` (policies + logging), `numbered` (for deleting) |
| `ufw default deny incoming` | Set the deny-by-default policy | pair with `default allow outgoing` |
| `ufw allow 22/tcp` | Open a port | `allow OpenSSH` (app profile), `allow 80,443/tcp` |
| `ufw allow from 10.0.0.0/8 to any port 5432` | Open a port to a subnet only | the canonical production DB rule |
| `ufw limit ssh` | Rate-limit (6 hits / 30s → drop) | brute-force mitigation |
| `ufw delete 3` | Delete rule #3 | **run `ufw status numbered` first — numbers shift** |
| `ufw enable` / `disable` / `reload` / `reset` | Lifecycle | ⚠️ allow SSH BEFORE `enable` |
| `ufw app list` | List app profiles | `ufw app info OpenSSH` |
| `ufw logging on` | Log blocked packets to `/var/log/ufw.log` | `low`/`medium`/`high`/`full` |
| `iptables -L -n -v --line-numbers` | **Read** the rules + counters | `-n` (no DNS — mandatory), `-v` (counters), `-t nat` |
| `iptables -A CHAIN ...` | Append a rule (to the bottom) | vs `-I` (insert at top) |
| `iptables -I CHAIN [n] ...` | Insert a rule at position n (default 1) | the one you usually want |
| `iptables -D CHAIN n` | Delete rule n | or `-D` with the full rule spec |
| `iptables -P INPUT DROP` | Set the chain default policy | **the deny-by-default posture** |
| `iptables -F` | Flush all rules in a chain | ⚠️ set policy to ACCEPT FIRST or you lock yourself out |
| `iptables -Z` | Zero the counters | perfect before a debugging test |
| `iptables-save` / `iptables-restore` | Dump / load rules as text | rules are **NOT** persistent without this |
| `netfilter-persistent save` | Persist (from `iptables-persistent`) | |
| `nft list ruleset` | Show the real modern ruleset | the successor to iptables |
| `conntrack -L` | Show the connection tracking table | needs `conntrack` package |
| `tcpdump -i any port 443` | Are packets even arriving? | separates cloud SG from local firewall |

**The rule anatomy, one more time:**
`-A INPUT` (chain) `-i eth0` (in-interface) `-p tcp` (protocol) `--dport 22` (dest port) `-s 10.0.0.0/8` (source) `-m conntrack --ctstate NEW` (match module) `-j ACCEPT` (target)

---

## When Would I Use This at Work?

### Scenario 1: Locking down your Postgres so it's not on the public internet

Your cloud DB is expensive, so you run Postgres in a container on the app server. You do the obvious thing: `-p 5432:5432`. Six months later a security audit finds your database is reachable from anywhere on the internet, with `ufw status` cheerfully reporting "deny (incoming)." You now know exactly why: ufw filters INPUT; Docker DNAT's into FORWARD. You change one line to `-p 127.0.0.1:5432:5432`, add `ufw allow from 10.0.0.0/8 to any port 5432` for the private-network case, and add a CI check that fails any compose file publishing a port on `0.0.0.0`.

### Scenario 2: "The API is timing out from the internet but works on the box"

A deploy went out and external traffic stopped. You run the ladder: `ss -tulpn` (app is listening ✓) → `curl localhost` on the box (200 ✓) → `curl` from your laptop (timeout ✗) → `tcpdump -i any port 443` (SYNs arriving, no SYN-ACK ✗) → `iptables -L -n -v` (the DROP policy counter is climbing, and there's no 443 rule) → someone ran `ufw delete 3` and the numbers had shifted. You restore the rule in 30 seconds. Without the packet-counter step, this is a two-hour outage of guesswork.

### Scenario 3: Hardening a new production box on day one

You provision a VM. Before anything else: schedule the `at now + 5 minutes; ufw disable` rollback, `ufw default deny incoming`, `ufw default allow outgoing`, `ufw allow OpenSSH`, `ufw limit ssh` (kills the constant SSH brute-force noise from the internet), `ufw allow proto tcp from any to any port 80,443`, `ufw enable`, verify from a second terminal, cancel the rollback. Then check the AWS security group matches — 22 restricted to the office/VPN CIDR, 80/443 open, everything else closed. Two firewalls, both configured, both verified. Then you layer fail2ban (topic 35) on top for dynamic blocking of the attackers who *are* allowed to reach port 22.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 30 — curl and wget in Depth | Once the port is open, you need to actually *speak HTTP* to it — and `curl -v` is how you prove the firewall is fixed |
| **Builds on** | 26 — Network Interfaces | `-i eth0`, `-i docker0`, `-i lo` are exactly the interfaces from topic 26. The routing decision between INPUT and FORWARD is a routing-table lookup |
| **Builds on** | 28 — Ports and Sockets | Before you blame the firewall, `ss -tulpn` proves whether anything is even LISTENING. Half of "the firewall is blocking me" is really "the app isn't running" |
| **Builds on** | 16/17 — Processes | The packet's final destination is a socket owned by a *process*. If the process is dead, the firewall is irrelevant |
| **Builds on** | 23 — Logs | `/var/log/ufw.log` (after `ufw logging on`) and `/var/log/auth.log` (who ran the sudo ufw command) |
| **Used by** | 33 — Node in Production | Your Node app should listen on `127.0.0.1:3000` and be fronted by nginx on 443. The firewall enforces that even if you get the bind address wrong |
| **Used by** | 35 — Security Basics | fail2ban is a *dynamic* firewall layer: it reads `/var/log/auth.log`, and when it sees brute-force attempts it inserts `iptables -I` DROP rules for that IP, then removes them after a bantime. It is netfilter, driven by log parsing |
