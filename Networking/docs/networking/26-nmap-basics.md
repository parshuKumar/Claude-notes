# 26 — nmap Basics

> **Phase 4 — Topic 6 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `05-ports.md`

---

> ⚠️ **READ THIS FIRST — Authorization and legality.**
> nmap sends unsolicited probe packets to other machines. Doing that to hosts you do **not** own or have **explicit written permission** to test is illegal in many jurisdictions (e.g. the US Computer Fraud and Abuse Act, the UK Computer Misuse Act) and violates the terms of service of every cloud provider (AWS, GCP, Azure all forbid scanning without prior authorization).
>
> This doc is about **legitimate backend and DevOps work**: scanning **your own** servers, verifying **your own** firewall and security-group rules, and auditing what **your own** infrastructure exposes to the internet. Every example here targets one of exactly three things:
> - **`localhost` / `127.0.0.1`** — your own machine.
> - **A server you own** (your EC2 instance, your VPS, your VPC).
> - **`scanme.nmap.org`** — a host the Nmap Project explicitly runs so people can practice legally. (Their rule: a few scans are fine, don't hammer it.)
>
> If you can't put a target in one of those three buckets, don't scan it.

---

## ELI5 — The Simple Analogy

Imagine a big office building. You want to know which doors are usable.

You walk down the hallway and, for each numbered door, you gently try the handle:

- The handle turns and the door opens → **the room is in use, someone's there** (port **open**).
- The handle turns but the door is locked and a voice says "nobody here, go away" → **the room exists but nothing's running** (port **closed**).
- You try the handle and… nothing. No answer, no "go away", just silence. You can't even tell if there's a door → **a security guard is silently blocking you** (port **filtered**).

nmap is you, walking the hallway, trying every handle, and writing down what happened at each door. Then it goes one step further: for the open doors, it peeks inside and reads the name badge — "Postgres 15.3", "nginx 1.24", "OpenSSH 9.6" — so you know *what* is running, not just *that* something is.

That's the whole tool: **knock on ports, record how each one answers, and identify what's behind the open ones.**

---

## Where This Lives in the Network Stack

nmap operates by crafting and reading packets at **Layer 3 (IP)** and **Layer 4 (TCP/UDP)**, then making educated guesses about **Layer 7** (which service, which version).

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, SSH, Postgres)                   │  ← -sV reads banners here to GUESS the service
│  Layer 6 — Presentation  (TLS)                                  │  ← -sV / ssl scripts probe here
│  Layer 5 — Session                                              │
│  Layer 4 — Transport     (TCP flags, UDP — PORTS live here)     │  ← THE MAIN EVENT: nmap sets SYN/ACK/etc.
│  Layer 3 — Network       (IP — host discovery, -Pn, TTL)        │  ← -sn ping sweep, routing decisions
│  Layer 2 — Data Link                                            │
│  Layer 1 — Physical                                             │
└─────────────────────────────────────────────────────────────────┘
```

nmap is fundamentally a **Layer 4 tool**. Its core job is manipulating the TCP flags (`SYN`, `ACK`, `RST`, `FIN`) covered in Topic 08, aimed at the port numbers covered in Topic 05. The service/version detection (`-sV`) reaches up to Layer 7 by reading response banners, but everything starts with "what happens when I send a packet to `host:port`?"

---

## What Is This?

**nmap** ("Network Mapper") is a free, open-source command-line tool that discovers hosts and services on a network by sending them crafted packets and analyzing the responses.

For a backend developer, three of its capabilities matter:

1. **Port scanning** — for a given host, which of its 65,535 TCP/UDP ports are open, closed, or filtered?
2. **Service & version detection** — for each open port, what software is listening, and which version?
3. **Host discovery** — which IPs in a range are actually alive?

It also does OS fingerprinting, scriptable checks (NSE — the Nmap Scripting Engine), and more, but the three above are the bread and butter.

**What nmap does NOT do:** it does not exploit anything, log in, or change state on the target. It knocks and listens. It's a reconnaissance/inventory tool. Think of it as the network equivalent of `ls` — it tells you what's *there*, not what's *wrong* (though "port 5432 open to the whole internet" is usually plenty wrong).

The single most valuable question nmap answers for a backend engineer is:

> *"Is my firewall / security group actually doing what I think it's doing?"*

You configure a rule. You *believe* only 443 is open. nmap tells you whether that belief is true — from an outside vantage point, which is the only vantage point that counts.

---

## Why Does It Matter for a Backend Developer?

You are not a penetration tester. You still need nmap constantly:

- **Verify a firewall / security-group change actually took effect.** You edited a rule to close the Postgres port. Did it work? The AWS console *says* the rule is right, but a scan from outside the VPC is ground truth. (This is the killer use case — see Example 2 and Topic 32.)
- **Confirm a service is — or isn't — publicly exposed.** After a deploy, is your admin dashboard on `:8080` reachable from the internet? Is Redis (`:6379`) accidentally bound to `0.0.0.0`?
- **Post-deploy validation / smoke test.** "The load balancer should expose 80 and 443 and nothing else." One scan confirms it.
- **Inventory open ports on your own hosts.** You inherited a legacy server. What's even listening on it? A scan is faster than reading a decade of config files.
- **Debug "connection refused" vs "connection timed out."** nmap's `closed` vs `filtered` distinction maps directly onto those two errors (see Common Mistakes) and tells you whether the problem is *"nothing's listening"* or *"a firewall is dropping my packets."*
- **Catch the classic disaster before an attacker does:** a database, cache, or admin panel exposed to `0.0.0.0/0`. Every few months there's a headline about an open Elasticsearch / MongoDB / Redis leaking millions of records. Almost always: a security group with an accidental `0.0.0.0/0` on the wrong port. A one-line nmap scan catches it.

The mental upgrade: instead of *trusting* your cloud console's rule list, you **empirically test** what the internet can actually reach. Consoles show intent; nmap shows reality.

---

## The Packet/Protocol Anatomy

Everything nmap infers comes from **which TCP flags come back** (or whether anything comes back at all). Recall from Topic 08 that a normal TCP connection opens with a 3-way handshake:

```
   CLIENT                          SERVER
     │   ── SYN ──────────────────►  │     "I'd like to connect"
     │   ◄──────────── SYN, ACK ──   │     "Sure, go ahead"
     │   ── ACK ──────────────────►  │     "Connected." (ESTABLISHED)
```

nmap sends the first `SYN` and then reads the reply to classify the port. Here's the decision tree for a TCP SYN probe:

```
        nmap sends:  SYN  ──────────────►  host:port
                                              │
              ┌───────────────────────────────┼───────────────────────────────┐
              │                               │                                │
       reply = SYN,ACK               reply = RST,ACK                    NO reply at all
     (something accepted)          (actively rejected)               (packet vanished)
              │                               │                                │
              ▼                               ▼                                ▼
        ┌──────────┐                    ┌──────────┐                    ┌────────────┐
        │  OPEN    │                    │  CLOSED  │                    │  FILTERED  │
        └──────────┘                    └──────────┘                    └────────────┘
   A service is listening.        Host is up and reachable,        A firewall silently DROPPED
   nmap tears the half-open       but NOTHING is listening         the packet. Host may be up or
   connection down with a RST.    on that port. (TCP RST =         down — you can't tell. nmap
                                  "go away, nobody home.")         retries, then gives up.
```

### The six port states nmap can report

| State | What nmap saw | What it means for you |
|-------|---------------|-----------------------|
| **open** | Got `SYN,ACK` (TCP) or a service reply | A service is actively listening and accepting connections. |
| **closed** | Got `RST` | Host is reachable, but no service on this port. Maps to `ECONNREFUSED`. |
| **filtered** | Nothing came back (or ICMP unreachable) | A firewall/security group is **dropping** your packets. Maps to `ETIMEDOUT`. You can't tell if a service is behind it. |
| **unfiltered** | Reachable (via ACK scan) but state undetermined | Firewall allows the packet through, but nmap can't tell open vs closed. Rare; mostly from `-sA` ACK scans. |
| **open\|filtered** | No response, can't disambiguate | Common in **UDP** scans and stealth scans — no reply could mean "open service that stays quiet" OR "firewall dropped it." |
| **closed\|filtered** | Can't tell closed from filtered | Rare; only from certain idle scans. |

**The distinction that matters most: `closed` vs `filtered`.**

```
closed   = "I reached the host. It said 'no service here' (RST)."     → app not running / wrong port
filtered = "I reached NOTHING back. A firewall ate my packet."       → firewall / security group blocking
```

Confusing these two is the #1 nmap mistake for backend devs (see Common Mistakes). They point at completely different fixes: `closed` → start your process / fix the port; `filtered` → open the firewall rule.

---

## How It Works — Step by Step

Let's trace a real scan end to end: `nmap -sV scanme.nmap.org`.

### Step 1 — Host discovery ("is it even up?")

```
Before scanning ports, nmap checks the host is alive so it doesn't waste
time probing 1000 ports on a dead IP.

By default (unprivileged, remote host) nmap sends a mix of:
  • TCP SYN to port 443
  • TCP ACK to port 80
  • ICMP echo request (a "ping")
  • ICMP timestamp request

If ANY of these gets a reply → host is UP → proceed to port scan.
If nothing replies → "Host seems down" → nmap SKIPS it.

  ⚠️ Many firewalls block ping. A host that's actually up can look "down".
     That's what -Pn is for (Step 1b).
```

### Step 1b — `-Pn`: skip discovery, assume up

```
nmap -Pn scanme.nmap.org
       │
       └─► "Don't bother pinging. Treat the host as UP and scan the ports anyway."

Use -Pn when a host blocks ping but you KNOW it's alive (very common for
cloud servers and hardened firewalls). Without it, nmap may wrongly declare
your live server "down" and scan nothing.
```

### Step 2 — Port scan (the core loop)

```
For each target port (default: the top 1000 most common ports):

  nmap ──SYN──► host:port
       ◄─reply─
       classify → open / closed / filtered   (per the decision tree above)

nmap does this for MANY ports in parallel (hundreds of probes in flight),
which is why a 1000-port scan finishes in seconds, not minutes.
```

### Step 3 — Service/version detection (`-sV` only)

```
For each port found OPEN, nmap now asks "what is this?":

  3a. Connect fully to the open port.
  3b. Read whatever the service sends first (the "banner").
        e.g. SSH sends:  "SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13"
  3c. If the service is quiet, nmap sends known probe strings from its
      nmap-service-probes database and matches the responses against
      thousands of signatures.
  3d. Report: service name (ssh) + product (OpenSSH) + version (9.6p1).
```

### Step 4 — Report

```
nmap prints the PORT / STATE / SERVICE / VERSION table (see the syntax
section), plus timing and how many ports were closed/filtered but hidden.
```

### The full picture

```
┌──────────────┐                                  ┌──────────────────────┐
│   You (nmap) │                                  │  scanme.nmap.org     │
│              │──── host discovery (ping) ───────►│                      │
│              │◄─── reply → "host is up" ─────────│                      │
│              │                                  │                      │
│              │──── SYN → :22 ───────────────────►│  sshd listening      │
│              │◄─── SYN,ACK ─────────────────────│  → OPEN              │
│              │──── SYN → :80 ───────────────────►│  nginx listening     │
│              │◄─── SYN,ACK ─────────────────────│  → OPEN              │
│              │──── SYN → :3306 ─────────────────►│  (nothing there)     │
│              │◄─── RST ─────────────────────────│  → CLOSED            │
│              │──── SYN → :23 ───────────────────►│  (firewall drops it) │
│              │      ...no reply... (retry x2)    │  → FILTERED          │
│              │                                  │                      │
│              │──── banner grab :22 (-sV) ───────►│                      │
│              │◄─── "SSH-2.0-OpenSSH_9.6" ────────│                      │
└──────────────┘                                  └──────────────────────┘
```

---

## Exact Syntax Breakdown

### `nmap -sV -p 22,80,443 scanme.nmap.org`

```
nmap   -sV        -p 22,80,443          scanme.nmap.org
│      │          │                     │
│      │          │                     └── TARGET: hostname, IP, CIDR (10.0.0.0/24),
│      │          │                         or range (10.0.0.1-50). Here: the practice host.
│      │          └── PORTS to scan: only 22, 80, and 443 (comma-separated list).
│      │              Restricting ports = much faster + politer than scanning 1000.
│      └── SERVICE/VERSION detection: for each open port, identify the software + version.
└── the program itself.
```

This is the everyday backend command: "check these specific ports and tell me what's running." No root required (`-sV` uses full TCP connects when unprivileged).

### `sudo nmap -sS -p- <your-own-server>`

```
sudo   nmap   -sS         -p-              <your-own-server>
│      │      │           │                │
│      │      │           │                └── TARGET: an IP/host YOU OWN. Never anything else.
│      │      │           └── ALL 65,535 ports (shorthand for -p 1-65535). Thorough but slower.
│      │      └── SYN / "half-open" scan: send SYN, read reply, then RST before the
│      │          handshake completes. Fast and the nmap default WHEN run as root.
│      └── the program.
└── ROOT is required: crafting raw SYN packets needs privileged socket access.
    Without sudo, nmap silently falls back to -sT (full connect).
```

### The flags you'll actually use

| Flag | Meaning | Root needed? |
|------|---------|--------------|
| `-sT` | **TCP connect** scan — completes the full 3-way handshake using the OS. Works unprivileged; more likely to be logged by the target. | No |
| `-sS` | **SYN / half-open** scan — SYN, read reply, RST before completing. Fast, default *when root*. | Yes |
| `-sU` | **UDP** scan — for DNS(53), SNMP(161), NTP(123), etc. Slow (UDP gives no handshake). | Yes |
| `-sn` | **Ping sweep** — host discovery only, **no port scan**. "Which IPs are alive?" | No* |
| `-Pn` | **Skip discovery** — assume host is up, scan anyway. For hosts that block ping. | No |
| `-p 80,443` | Scan a specific list of ports. |  |
| `-p 1-1024` / `-p-` | A range / **all 65,535** ports (`-p-` = `-p 1-65535`). |  |
| `--top-ports N` | Scan the **N most common** ports (e.g. `--top-ports 20`). |  |
| `-F` | **Fast** — scan the top **100** ports instead of the default top 1000. |  |
| `-sV` | **Version detection** on open ports. Add `--version-intensity 0-9` to tune. | No |
| `-O` | **OS detection** — fingerprint the operating system (needs enough open+closed ports). | Yes |
| `-sC` | Run the **default script set** (equivalent to `--script=default`). | No |
| `--script=<name>` | Run a specific **NSE** script (e.g. `--script=ssl-enum-ciphers`). | No |
| `-A` | **Aggressive**: `-sV` + `-O` + `-sC` + traceroute, all at once. Loud. | Yes |
| `-T0` … `-T5` | **Timing template**, slowest→fastest. `-T3` is default. `-T4` fine for your own LAN/cloud. `-T5` can drop packets and trip IDS. | No |
| `-v` / `-vv` | Verbose / very verbose — show progress and results as they arrive. | No |
| `-oN file` | Save **normal** (human-readable) output to a file. | No |
| `-oG file` | Save **grepable** output (one host per line — pipe to grep/awk). | No |
| `-oX file` | Save **XML** output (for tooling / CI parsing). | No |
| `-oA base` | Save **all three** formats at once (`base.nmap`, `base.gnmap`, `base.xml`). | No |

*`-sn` needs root only if you want ARP-level discovery on a local LAN; over the internet it works unprivileged.

---

## Example 1 — Basic

**Goal:** scan the Nmap Project's legal practice host and identify its services. No root, three ports.

```bash
nmap -sV -p 22,80,443 scanme.nmap.org
```

Annotated output:

```
Starting Nmap 7.94 ( https://nmap.org ) at 2026-07-12 10:14 UTC
Nmap scan report for scanme.nmap.org (45.33.32.156)   ← resolved the hostname to an IP
Host is up (0.087s latency).                           ← discovery succeeded; ~87ms RTT

PORT    STATE    SERVICE  VERSION
22/tcp  open     ssh      OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp  open     http     Apache httpd 2.4.7 ((Ubuntu))
443/tcp filtered https                                 ← no reply → a firewall dropped the probe
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.42 seconds
```

**How to read the table (this is the block you'll read a thousand times):**

```
PORT    STATE    SERVICE  VERSION
 │       │        │        │
 │       │        │        └── what nmap thinks is running (-sV): product + version string
 │       │        └── nmap's GUESS of the service by port number (before -sV: just the /etc/services name)
 │       └── open / closed / filtered  ← the single most important column
 └── port number + protocol (tcp/udp)
```

Reading it out loud:
- **`22/tcp open ssh OpenSSH 6.6.1`** — SSH is listening, and it's an old OpenSSH. Reachable.
- **`80/tcp open http Apache 2.4.7`** — a web server is up on plain HTTP.
- **`443/tcp filtered`** — nmap got **nothing** back. A firewall dropped the packet. This is *not* the same as "closed" — a service might be there, hidden behind the firewall. You simply can't tell from here.

The two lines nmap adds at the bottom about "closed/filtered ports not shown" appear when you scan many ports — nmap collapses the boring ones so the interesting ones stand out.

---

## Example 2 — Production Scenario

**The situation.** You run a Node.js API on an EC2 instance behind an AWS security group. The intended posture:

- **443** (HTTPS) — open to the world (`0.0.0.0/0`). This is your public API.
- **22** (SSH) — open only to the office IP.
- **5432** (Postgres) — open only to the app tier's security group. **Never** the public internet.

A teammate edited the security group last night "to debug something." You need to confirm the database is still private. The AWS console shows the rules, but console rules describe *intent* — you want *reality*, measured from **outside the VPC** (your laptop, over the public internet — the same vantage an attacker has). This ties directly to Topic 32 (Firewalls and Security Groups): a security group is a **stateful firewall**, and `0.0.0.0/0` means "the entire internet."

**Step 1 — Scan the public IP of your own server, from outside.**

```bash
# Your own EC2 instance's public IP. -Pn because AWS SGs block ICMP ping by default,
# so without it nmap would wrongly call the host "down".
nmap -Pn -sS -p 22,443,5432,6379 -T4 <YOUR_EC2_PUBLIC_IP>
```

Output — and here's the problem:

```
Starting Nmap 7.94 ( https://nmap.org ) at 2026-07-12 09:02 UTC
Nmap scan report for ec2-52-14-x-x.us-east-2.compute.amazonaws.com (52.14.x.x)
Host is up (0.031s latency).

PORT     STATE    SERVICE     VERSION
22/tcp   filtered ssh                    ← good: not your office IP, so SG drops it. Expected.
443/tcp  open     https                  ← good: public API, intended to be open.
5432/tcp open     postgresql             ← 🚨 DISASTER: Postgres reachable from the public internet
6379/tcp filtered redis                  ← good: firewall dropping it. Expected.

Nmap done: 1 IP address (1 host up) scanned in 2.10 seconds
```

**Reading it:**

- `22/tcp filtered` — SSH packet was dropped. Correct: your laptop isn't the whitelisted office IP, so the SG silently drops it. `filtered`, not `closed` — a firewall is doing its job.
- `443/tcp open` — intended. Your API is public.
- **`5432/tcp open postgresql`** — the database answered a SYN from the open internet. That accidental rule your teammate added opened `5432` to `0.0.0.0/0`. Anyone on earth can now attempt to connect to your Postgres. This is exactly how databases get ransomed.
- `6379/tcp filtered` — Redis is correctly firewalled off.

**Step 2 — Confirm what's exposed (version detection).**

```bash
nmap -Pn -sV -p 5432 <YOUR_EC2_PUBLIC_IP>
```

```
PORT     STATE SERVICE    VERSION
5432/tcp open  postgresql PostgreSQL 15.3
```

Yep — a real, live PostgreSQL 15.3, wide open. Not a honeypot, not a red herring.

**Step 3 — Fix it (in AWS), then re-scan to prove the fix.**

Remove the bad inbound rule so `5432` is only allowed from the app-tier security group. Then **re-run the exact same scan** — verification is not optional:

```bash
nmap -Pn -sS -p 22,443,5432,6379 -T4 <YOUR_EC2_PUBLIC_IP>
```

```
PORT     STATE    SERVICE
22/tcp   filtered ssh
443/tcp  open     https
5432/tcp filtered postgresql    ← ✅ now DROPPED for the public internet. Fixed and PROVEN.
6379/tcp filtered redis
```

`5432` flipped from `open` to `filtered`. That state change is your empirical proof the security-group fix took effect — from the outside, where it counts. The console said the rule changed; nmap confirmed the packets are actually being dropped.

**The lesson:** cloud consoles show you the rules you *wrote*. nmap shows you the rules the network is *enforcing*. After every firewall/security-group change, scan from outside. Make it a post-deploy CI step if you can (`nmap -oX` → parse → fail the build if an unexpected port is `open`).

---

## Common Mistakes

### Mistake 1: Scanning hosts you don't own or aren't authorized to test

```
nmap somecompany.com          # ❌ ILLEGAL without written permission
nmap 8.8.8.8                   # ❌ that's Google's, not yours
nmap 192.0.2.0/24             # ❌ someone else's /24
```

**Why it's wrong:** Sending probe packets to machines you don't control is a crime under laws like the US CFAA and the UK Computer Misuse Act, and it violates every cloud provider's ToS (AWS/GCP/Azure require prior authorization even to scan *between your own* accounts in some cases). "I was just curious" is not a defense.

**Do this instead:** scan `localhost`, a server you own, or `scanme.nmap.org` (Nmap's explicitly-authorized practice host — a few scans, don't abuse it). If you need to test a third party's system, get **written** authorization first.

---

### Mistake 2: Reading `filtered` as `closed` (they mean opposite things)

```
5432/tcp closed   postgresql   ← host reached you; NOTHING is listening. App down / wrong port.
5432/tcp filtered postgresql   ← a FIREWALL dropped your packet. Can't tell what's behind it.
```

**Why it's wrong:** `closed` and `filtered` point at completely different fixes. `closed` = your process isn't running (or you scanned the wrong port) → start the service. `filtered` = a firewall/security group is dropping packets → change the firewall rule. Treating `filtered` as "nothing there, all good" hides the fact that a service *might* be running behind the firewall — and treating `closed` as "firewall blocked me" sends you editing security groups when the real problem is your app crashed.

**Remember:** `closed` ↔ `ECONNREFUSED` (got a RST). `filtered` ↔ `ETIMEDOUT` (got silence). See Topic 32.

---

### Mistake 3: Running `-sS` without root and not realizing you got `-sT`

```bash
nmap -sS -p- 10.0.0.5
# ⚠️ You must have privileges to send raw packets.  (nmap warns, then silently uses -sT)
```

**Why it's wrong:** SYN scan (`-sS`) crafts raw packets, which needs root. Run it unprivileged and nmap quietly falls back to a full TCP connect scan (`-sT`). That's not fatal — but `-sT` completes the whole handshake, so it's slower and far more likely to be **logged** by the target's application (your app sees a real, if immediately-closed, connection). If you expected the quiet half-open behavior of `-sS`, you didn't get it.

**Fix:** use `sudo nmap -sS ...` when you want a real SYN scan; otherwise consciously use `-sT` and know it'll show up in your app logs.

---

### Mistake 4: Trusting version detection as ground truth

```
80/tcp open http Apache httpd 2.4.7 ((Ubuntu))
```

**Why it's wrong:** `-sV` *guesses* from banners and response patterns. Banners can be **stale, spoofed, or wrong**: a reverse proxy may advertise a fake `Server:` header, a backport may patch the CVEs while keeping the old version string, or a WAF may deliberately lie. "nmap says Apache 2.4.7, and 2.4.7 has CVE-X" does **not** prove the box is vulnerable. It's a strong hint, not a verdict.

**Fix:** treat `-sV` output as a lead to confirm another way (check the actual package version on the host you own, hit the service directly). Never file "server is vulnerable" off an nmap banner alone.

---

### Mistake 5: Aggressively scanning production (`-A`, `-T5`, full `-p-`)

```bash
nmap -A -T5 -p- prod-api.mycompany.com   # ❌ on live production during peak hours
```

**Why it's wrong:** `-A` fires version + OS + default scripts + traceroute; `-T5` blasts probes as fast as possible; `-p-` hits all 65,535 ports. Together they can: (a) trip your **IDS/IPS** and page the security team, (b) get your scanning IP **auto-blocked** by fail2ban / AWS Shield, (c) put real **load** on the box (thousands of connections), and (d) even crash fragile services that mishandle unexpected probes. `-T5` may also drop packets and give you *wrong* results.

**Fix:** on production, be surgical and polite — scan only the ports you care about, use `-T3` (default) or `-T4` at most, skip `-A`, and prefer a maintenance window. Save the heavy artillery for staging or a box you can afford to annoy.

---

### Mistake 6: Forgetting UDP services need `-sU`

```bash
nmap -p 53,123,161 dns-server        # ❌ this scans TCP 53/123/161 — misses the real UDP services
```

**Why it's wrong:** nmap scans **TCP by default**. DNS(53), NTP(123), SNMP(161), DHCP, and many VPN protocols are **UDP**. A default scan will report these `closed`/`filtered` on TCP and you'll wrongly conclude "DNS isn't exposed" when it's wide open on UDP.

**Fix:** use `-sU` for UDP (`sudo nmap -sU -p 53,123,161 dns-server`). Note UDP scans are **slow** — UDP has no handshake, so nmap must wait for timeouts — and often report `open|filtered` because silence is ambiguous. Scan a short UDP port list, not `-p-`.

---

## Hands-On Proof

Everything here targets your own machine or the authorized practice host. Install nmap first (`brew install nmap` on macOS, `sudo apt install nmap` on Debian/Ubuntu).

```bash
# 1. Scan your OWN machine's most common ports. See what YOU are running.
nmap localhost
#    Any 'open' ports are services on your laptop right now (5432? 6379? 3000?).

# 2. Start something, then prove nmap sees it.
python3 -m http.server 8000 &          # start a throwaway web server on :8000
nmap -p 8000 localhost                 # → 8000/tcp open  http-alt
kill %1                                 # stop it
nmap -p 8000 localhost                 # → 8000/tcp closed http-alt   (RST: nothing listening now)
#    Watch the state flip open → closed as you start/stop the service. THIS is the whole tool.

# 3. Identify versions of what's running locally.
nmap -sV -p- localhost                 # every open port + its software/version

# 4. The legal practice host — the exact Example 1 command.
nmap -sV -p 22,80,443 scanme.nmap.org

# 5. See open|filtered ambiguity for yourself with a UDP scan of your own box.
sudo nmap -sU --top-ports 20 localhost

# 6. Save results in all three formats (for tooling / diffing later).
nmap -sV -oA my-localhost-scan localhost
ls my-localhost-scan.*                 # → .nmap (readable) .gnmap (grepable) .xml (machine)

# 7. Grepable output → pull just the open ports with awk.
nmap -oG - localhost | grep -o '[0-9]*/open' | sort -u
```

The `open → closed` flip in step 2 is the single clearest demonstration of what nmap measures: a live service answers with `SYN,ACK` (open); once you kill it, the OS answers with `RST` (closed).

---

## Practice Exercises

### Exercise 1 — Easy: Inventory your own laptop

```bash
# Scan your own machine and list what's listening.
nmap -sV localhost

# Answer from the output:
# 1. How many ports are 'open'? Name the services.
# 2. Are any of them things you didn't expect to be running? (a stray dev server?)
# 3. Cross-check with the OS: does `ss -tlnp` (Linux) or `lsof -iTCP -sTCP:LISTEN`
#    (macOS) show the same listening ports nmap found? They should match.
```

### Exercise 2 — Medium: closed vs filtered, on purpose

```bash
# Goal: produce a 'closed' result and a 'filtered' result and tell them apart.

# (a) CLOSED: scan a port on localhost where nothing is running.
nmap -p 9999 localhost
#     Expected: 9999/tcp closed  (your OS sent a RST — reachable, nothing there)

# (b) FILTERED: add a firewall rule that DROPS a port, then scan it.
#     Linux:   sudo iptables -A INPUT -p tcp --dport 9998 -j DROP
#     Then start a listener on 9998 (python3 -m http.server 9998 &) and scan:
nmap -p 9998 localhost
#     Expected: 9998/tcp filtered  (the DROP rule ate the packet — even though a
#               service IS listening behind it!)
#     Cleanup:  sudo iptables -D INPUT -p tcp --dport 9998 -j DROP

# Answer:
# 1. In case (b), a real server was running — why did nmap say 'filtered', not 'open'?
# 2. Which of these two states maps to ECONNREFUSED, and which to ETIMEDOUT?
# 3. If your production scan showed 'filtered' on your API port, what would you check first?
```

### Exercise 3 — Hard: Simulate the Example 2 security-group audit

```bash
# Goal: build the "database accidentally exposed" scenario locally, catch it, fix it.

# 1. Run a "database" and a "web server" on your own machine:
python3 -m http.server 8443 &          # pretend this is your public API  (:8443)
python3 -m http.server 5432 &          # pretend this is your DB, wrongly exposed (:5432)

# 2. Scan as if from outside — inventory the exposed ports:
nmap -sV -p 8443,5432 -oN before.txt localhost

# 3. "Fix the security group": DROP 5432 with a firewall rule (Linux iptables,
#    or on macOS use pf). Then RE-SCAN and save:
#      sudo iptables -A INPUT -p tcp --dport 5432 -j DROP
nmap -sV -p 8443,5432 -oN after.txt localhost

# 4. Diff the two scans to PROVE the fix:
diff before.txt after.txt

# Answer:
# 1. What state was 5432 in 'before.txt'? What state in 'after.txt'?
# 2. If you only had the AWS console (not nmap), how confident could you be the fix worked?
# 3. Write a one-line nmap command a CI job could run nightly against your prod IP that
#    would FAIL the build if 5432 ever shows up as 'open'. (Hint: -oG output + grep.)
# 4. Cleanup: kill the servers and delete the iptables rule.
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the linked section.

1. **A port scan returns `filtered`. Give the one-sentence definition, and say which connection error (`ECONNREFUSED` or `ETIMEDOUT`) it corresponds to — and why.**

2. **You scan your own EC2 instance from your laptop and Postgres (5432) shows `open`. Walk through what that means and what you'd do in the next five minutes.**

3. **What's the difference between `-sS` and `-sT`? Which needs root, and what silently happens if you ask for `-sS` without it?**

4. **Your DNS server is definitely serving queries, but `nmap -p 53 dns-host` reports 53 as `closed`. What did you forget, and what's the corrected command?**

5. **`nmap -sV` reports `nginx 1.18.0`, and 1.18.0 has a known CVE. Can you conclude the server is vulnerable? Why or why not?**

6. **A colleague wants to run `nmap -A -T5 -p-` against the live production API at 2pm on a weekday. List three concrete things that could go wrong.**

7. **Explain, in terms of TCP flags, exactly how nmap decides a port is `open` vs `closed` on a SYN scan.**

---

## Quick Reference Card

**Port states:**

| State | nmap saw | Backend meaning | Error analog |
|-------|----------|-----------------|--------------|
| `open` | `SYN,ACK` | Service listening & reachable | (connects fine) |
| `closed` | `RST` | Reachable, nothing listening | `ECONNREFUSED` |
| `filtered` | silence / ICMP unreachable | Firewall dropping packets | `ETIMEDOUT` |
| `open\|filtered` | no reply (UDP/stealth) | Can't tell open from firewalled | — |
| `unfiltered` | reachable, state unknown | Firewall lets it through (ACK scan) | — |

**Scan types:**

| Flag | Scan | Root? | Use it for |
|------|------|-------|-----------|
| `-sT` | TCP connect | no | when you're not root; more visible in logs |
| `-sS` | SYN / half-open | yes | fast default; the everyday scan |
| `-sU` | UDP | yes | DNS/NTP/SNMP; slow |
| `-sn` | ping sweep, no ports | no | "which IPs are alive?" |
| `-Pn` | skip discovery | no | hosts that block ping (most cloud servers) |

**Port selection & tuning:**

| Flag | Meaning |
|------|---------|
| `-p 22,80,443` | just these ports |
| `-p 1-1024` / `-p-` | a range / all 65,535 |
| `--top-ports N` | the N most common |
| `-F` | fast — top 100 |
| `-sV` / `-O` / `-sC` / `-A` | version / OS / default scripts / all-aggressive |
| `-T0`…`-T5` | slowest→fastest (default `-T3`; `-T4` fine for your own hosts) |
| `-oN`/`-oG`/`-oX`/`-oA` | normal / grepable / XML / all three |

**Cheat block — copy/paste (own hosts + scanme.nmap.org only):**

```bash
nmap localhost                            # what am I running right now?
nmap -sV -p 22,80,443 scanme.nmap.org     # identify services on specific ports
sudo nmap -sS -p- <your-server>           # full SYN scan, all ports (root)
nmap -Pn -sV -p 443,5432 <your-ec2-ip>    # audit exposure from outside (host blocks ping)
sudo nmap -sU --top-ports 20 <your-host>  # UDP top-20 (DNS/NTP/SNMP)
nmap -sn 10.0.0.0/24                       # which hosts in my subnet are up?
nmap -sV -oA scan-results localhost        # save all 3 output formats
nmap -oG - <host> | grep '/open'           # quick 'just the open ports'
```

```
Golden rule:   Only scan localhost, hosts you own, or scanme.nmap.org.
Golden state:  filtered = FIREWALL dropping (ETIMEDOUT).  closed = nothing listening (ECONNREFUSED).
Golden habit:  after every firewall/SG change → re-scan from OUTSIDE to prove it.
```

---

## When Would I Use This at Work?

### Scenario 1: "I changed the security group — did it actually close the DB port?"
Don't trust the console. From a machine outside the VPC: `nmap -Pn -p 5432 <public-ip>`. If it says `filtered`, the firewall is dropping it (good). If it says `open`, your database is exposed to the internet — fix it now. Re-scan after the fix to prove `open → filtered`. (Topic 32.)

### Scenario 2: "Post-deploy smoke test: is only 443 exposed?"
Your deploy pipeline flips DNS to a new load balancer. Add a step: `nmap -Pn -p 22,80,443,8080 <lb-ip> -oX scan.xml`, parse the XML, and **fail the build** if anything other than `443` (and maybe `80` for the redirect) is `open`. Now an accidental exposed admin port can't reach production silently.

### Scenario 3: "We inherited a legacy box — what's even running on it?"
Before you touch a decade-old server you now own: `sudo nmap -sV -p- <box>`. In one command you get the full inventory of listening services and versions — faster and more honest than grepping through years of config files and cron jobs.

### Scenario 4: "Is Redis / Elasticsearch / Mongo accidentally public?"
The recurring data-breach headline. From outside: `nmap -Pn -p 6379,9200,27017 <your-ip>`. Any `open` here means a datastore with (usually) no auth is reachable by the entire internet. This single habit prevents the most common cloud data leak there is.

### Scenario 5: "A service says it's up but clients can't reach it."
Is it a listening problem or a firewall problem? `nmap -p <port> <host>`: `closed` → the app isn't actually listening (or wrong bind address / wrong port) → check the process. `filtered` → a firewall/security group is in the way → check the rules. The two answers send you to two completely different places.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — packets, IP addresses, the client→server path nmap probes.
- `05-ports.md` — the port numbers nmap scans and what each well-known port means. Essential.
- `08-tcp-in-depth.md` — the SYN/ACK/RST flags that nmap reads to classify every port.

**Study alongside / next:**
- `24-netstat-and-ss.md` — the *inside* view: `ss`/`netstat` show what YOUR host is listening on; nmap shows what's reachable from OUTSIDE. Use them together (Exercise 1 cross-checks one against the other).
- `25-ping-traceroute-mtr.md` — host discovery and reachability, the layer nmap's `-sn`/`-Pn` build on.
- `32-firewalls-and-security-groups.md` — **the payoff.** `filtered` vs `closed`, `0.0.0.0/0`, stateful security groups, inbound rules. nmap is how you *verify* everything in Topic 32 actually works. Example 2 is a direct preview.
- `23-tcpdump-and-wireshark.md` — when you want to see the individual SYN/RST packets nmap is sending and receiving, on the wire.

---

*Doc saved: `/docs/networking/26-nmap-basics.md`*
