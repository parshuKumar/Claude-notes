# 32 — Firewalls and Security Groups

> **Phase 5 — Topic 5 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `04-ip-addresses.md`, `05-ports.md`

---

## ELI5 — The Simple Analogy

Imagine your server is a nightclub, and packets are people trying to get in.

- **The bouncer at the door** is the firewall. Every person (packet) who walks up gets checked against a list of rules before they're allowed in.
- The rules are simple: **"Who are you (source IP), which door are you using (port), and are you on the guest list (allowed)?"**
- A **default-deny** bouncer turns away everyone who isn't explicitly on the list. A **default-allow** bouncer lets everyone in unless they're on a banned list. Good security uses default-deny.

Now here's the important twist. There are two kinds of bouncers:

- A **stateful bouncer** remembers who he let in. When you walk back *out* to grab something from your car, he waves you through — he knows you're already a guest. You only had to show ID once, on the way *in*.
- A **stateless bouncer** has no memory. Every single time you cross the door — in OR out — he re-checks your ID against the list. If the list doesn't explicitly say "this person may exit," you get stuck inside, unable to leave, even though you were allowed in.

That one difference — *does the bouncer remember your connection?* — is the single most important idea in this entire topic. Get it wrong and you'll spend hours debugging a server that accepts requests but can never reply.

---

## Where This Lives in the Network Stack

Firewalls operate primarily at **Layer 3 (IP)** and **Layer 4 (TCP/UDP ports)** — they filter on source/destination IP, port, and protocol. Some firewalls reach up to **Layer 7** (Web Application Firewalls, which inspect HTTP paths and payloads).

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP)          ← WAF filters here      │
│  Layer 6 — Presentation  (TLS)                                  │
│  Layer 5 — Session                                              │
│  Layer 4 — Transport     (TCP/UDP ports) ← SG/NACL/iptables ★   │  ← port + protocol rules
│  Layer 3 — Network       (IP addresses)  ← SG/NACL/iptables ★   │  ← source/dest IP + CIDR rules
│  Layer 2 — Data Link     (Ethernet/MAC)                         │
│  Layer 1 — Physical                                             │
└─────────────────────────────────────────────────────────────────┘
```

★ The firewalls a backend developer touches every day — AWS Security Groups, AWS Network ACLs, `iptables`/`nftables`, and `ufw` — all live at **L3/L4**. They make decisions using the exact fields you learned in Topic 04 (IP addresses / CIDR) and Topic 05 (ports).

---

## What Is This?

A **firewall** is a system that inspects network packets and decides whether to **allow** them through or **block** them, based on a set of **rules**. Each rule matches on some combination of:

- **Source IP** (or CIDR range) — where the packet came from
- **Destination IP** — where it's going
- **Protocol** — TCP, UDP, ICMP, etc.
- **Port** — which service (22 = SSH, 443 = HTTPS, 5432 = Postgres)
- **Direction** — is this packet coming *in* (ingress) or going *out* (egress)?

There are several categories a backend developer meets:

| Firewall type | Where it runs | Example |
|---------------|---------------|---------|
| **Host firewall** | On the machine itself | `iptables`, `nftables`, `ufw`, Windows Firewall |
| **Cloud instance firewall** | Attached to a VM's network interface | **AWS Security Group** |
| **Subnet firewall** | At the edge of a subnet | **AWS Network ACL (NACL)** |
| **Network appliance** | Dedicated hardware/virtual box | Palo Alto, `pfSense`, cloud NAT gateway |
| **Web Application Firewall (WAF)** | In front of your app, inspects HTTP | AWS WAF, Cloudflare, ModSecurity |

**Two philosophies** govern every firewall:

```
DEFAULT-DENY  (whitelist)  →  Block everything. Allow only what you list.
                              ✅ Secure by default. The correct choice.

DEFAULT-ALLOW (blacklist)  →  Allow everything. Block only what you list.
                              ⚠️  One forgotten rule = a hole. Avoid.
```

AWS Security Groups are **default-deny** for inbound (nothing gets in until you add an allow rule).

---

## Why Does It Matter for a Backend Developer?

Firewalls are where a huge fraction of "it works on my machine but not in prod" bugs live. You need this to:

- **Diagnose `ETIMEDOUT`** — a hung connection that never gets a reply is *the* signature of a firewall silently dropping your packets (Topic 01). Distinguishing it from `ECONNREFUSED` tells you instantly whether it's a firewall problem or an app problem.
- **Not get breached** — a database open to `0.0.0.0/0` on port 5432 is one `nmap` scan away from being ransomwared (Topic 26). Firewall misconfiguration is one of the top cloud breach causes.
- **Build the standard 3-tier architecture** — ALB → app → database, where each tier only accepts traffic from the tier in front of it. This is the single most common AWS network pattern and you'll set it up on nearly every project.
- **Understand the stateful/stateless trap** — why adding a Security Group rule "just works" but a Network ACL needs a second rule for return traffic. This one distinction causes mysterious one-way connectivity failures that eat entire afternoons.
- **Debug egress problems** — your app can't reach an external API or `npm install` hangs? Over-restrictive **outbound** rules are a classic culprit that most people forget to check because they only ever think about inbound.

---

## The Packet/Protocol Anatomy

A firewall doesn't invent new packet fields — it reads the ones already there. Here's exactly what it inspects on an incoming TCP packet:

```
INCOMING PACKET the firewall evaluates:

┌──────────────────────────────────────────────────────────────┐
│ IP HEADER (Layer 3)                                           │
│   Source IP:      203.0.113.7      ← rule matches this        │
│   Destination IP: 10.0.1.20        ← (your server)            │
│   Protocol:       6 (TCP)          ← rule matches this        │
├──────────────────────────────────────────────────────────────┤
│ TCP HEADER (Layer 4)                                          │
│   Source Port:      51844          ← client's ephemeral port  │
│   Destination Port: 443            ← rule matches this        │
│   Flags:  SYN                      ← stateful FW tracks this ★ │
│           SYN-ACK                                             │
│           ACK ... FIN ... RST                                 │
├──────────────────────────────────────────────────────────────┤
│ PAYLOAD (encrypted TLS/HTTP — L4 firewalls IGNORE this)       │
│   (only a Layer-7 WAF looks in here)                          │
└──────────────────────────────────────────────────────────────┘
```

★ The **TCP flags** are what make stateful firewalls possible. A brand-new connection starts with a lone `SYN` packet. A stateful firewall sees that `SYN`, and if it allows it, records an entry in its **connection tracking table** (Linux calls this `conntrack`). Every subsequent packet of that conversation — the `SYN-ACK` reply, the `ACK`, the data, the `FIN` teardown — is recognized as belonging to an **ESTABLISHED** connection and passed automatically.

```
CONNECTION TRACKING TABLE (what a stateful firewall remembers):

┌────────────────┬───────┬────────────────┬──────┬─────────────┐
│ Source         │ SPort │ Destination    │ DPort│ State       │
├────────────────┼───────┼────────────────┼──────┼─────────────┤
│ 203.0.113.7    │ 51844 │ 10.0.1.20      │ 443  │ ESTABLISHED │  ← reply auto-allowed
│ 198.51.100.2   │ 40122 │ 10.0.1.20      │ 22   │ ESTABLISHED │
└────────────────┴───────┴────────────────┴──────┴─────────────┘
```

A **stateless** firewall keeps no such table. It has no idea a packet is a "reply." To it, every packet is a stranger to be judged on its own.

---

## How It Works — Step by Step

### The core idea: stateful vs stateless

This is the heart of the topic. Watch what happens when a client makes an HTTPS request to your server, under each type of firewall.

**Recall from Topic 05:** the client connects *from* a random high-numbered **ephemeral port** (Linux: 32768–60999; many other stacks / the IANA range: 49152–65535) *to* your server's port 443. The reply travels the opposite way: from your 443 back to the client's ephemeral port.

```
                    CLIENT                          SERVER
              203.0.113.7:51844   ─────────►   10.0.1.20:443
              (ephemeral port)     request      (well-known port)

              203.0.113.7:51844   ◄─────────   10.0.1.20:443
                                   response
```

#### Stateful firewall (AWS Security Group, iptables+conntrack)

```
STEP 1  Inbound SYN arrives:  src 203.0.113.7:51844 → dst 10.0.1.20:443
        Firewall checks INBOUND rules → "allow TCP 443 from anywhere" ✅
        Firewall RECORDS this connection in its state table.

STEP 2  Server replies:       src 10.0.1.20:443 → dst 203.0.113.7:51844
        Firewall checks state table → "this belongs to an ESTABLISHED
        connection I already approved" → AUTO-ALLOWED ✅
        ⭐ You did NOT have to write any outbound rule for the reply.

RESULT: You write ONE rule (inbound allow 443). Replies flow automatically.
```

#### Stateless firewall (AWS Network ACL)

```
STEP 1  Inbound SYN arrives:  src 203.0.113.7:51844 → dst 10.0.1.20:443
        NACL checks INBOUND rules → "allow TCP 443 from anywhere" ✅

STEP 2  Server replies:       src 10.0.1.20:443 → dst 203.0.113.7:51844
        NACL has NO memory of step 1. It checks OUTBOUND rules from scratch.
        Destination port is 51844 (the client's EPHEMERAL port).
        Is there an outbound rule allowing TCP to ports 1024–65535?
        ❌ If not → the reply is DROPPED. Client hangs. ETIMEDOUT.

RESULT: You must write TWO rules — inbound allow 443 AND
        outbound allow the EPHEMERAL PORT RANGE (1024–65535).
```

That ephemeral-return-port rule is the single most infamous gotcha in cloud networking. Your server accepted the request perfectly, processed it, generated a response — and the firewall threw the response in the bin because you never told it replies were allowed to leave.

```
THE STATELESS RETURN-TRAFFIC TRAP (visualized):

   Inbound rule    ✅  allow TCP 443 from 0.0.0.0/0
   ───────────────────────────────────────────────► request gets IN

   Outbound rule   ❓  allow TCP 1024-65535 to 0.0.0.0/0
   ◄─────────────────────────────────────────────── reply tries to get OUT
                       (missing this = one-way black hole)
```

### Inbound (ingress) vs outbound (egress)

```
                        ┌─────────────────┐
     INBOUND / INGRESS  │                 │  OUTBOUND / EGRESS
   ───────────────────► │   YOUR SERVER   │ ───────────────────►
   traffic ARRIVING     │                 │   traffic LEAVING
   at your server       └─────────────────┘   your server

   Controls WHO can       Controls WHERE your
   reach your ports.      server is allowed to connect.

   Most common bug:       Most common bug:
   missing inbound        over-restrictive egress
   allow → nobody can     → app can't reach DB,
   connect to you.        external API, or npm.
```

Key insight: **most breakage is either a missing inbound allow or an over-restrictive outbound rule.** People obsess over inbound (locking the front door) and forget outbound exists at all — then wonder why their app can't call Stripe's API.

### CIDR in rules — who is the "source"?

Firewall rules use CIDR notation from Topic 04 to describe a set of source addresses:

```
0.0.0.0/0        = "the ENTIRE internet / anywhere"    (all 4.3 billion IPv4 addresses)
203.0.113.7/32   = "exactly ONE host"                  (a single IP)
10.0.0.0/16      = "everything in this VPC"            (65,536 addresses)
10.0.1.0/24      = "just this one subnet"              (256 addresses)
::/0             = "anywhere" for IPv6
```

`0.0.0.0/0` is fine (necessary, even) for inbound port 443 on a public load balancer — the whole world *should* be able to reach your website. It is **catastrophic** on port 22 (SSH) or 5432 (Postgres), where it means "anyone on Earth may attempt to log into my database."

---

## Exact Syntax Breakdown

### AWS Security Group rule (annotated)

An SG inbound rule has four parts:

```
Type        Protocol   Port Range    Source
─────       ────────   ──────────    ─────────────────────
HTTPS       TCP        443           0.0.0.0/0
│           │          │             │
│           │          │             └── who may connect: here, anyone
│           │          └── the destination port on YOUR instance
│           └── transport protocol (TCP/UDP/ICMP)
└── a friendly label AWS maps to protocol+port (HTTPS ⇒ TCP/443)
```

The **Source** field is the powerful part. It can be:

```
0.0.0.0/0          ← the entire internet
203.0.113.7/32     ← one specific IP (e.g. your office)
10.0.1.0/24        ← one subnet
sg-0app1234abcd    ← ANOTHER SECURITY GROUP  ⭐ (the AWS superpower)
```

Referencing another SG as the source means: *"allow traffic from any instance that has SG `sg-0app1234abcd` attached."* No hardcoded IPs. When you add or replace app servers, the rule keeps working because it matches on *membership*, not address.

### AWS Network ACL rule table (annotated — note the ephemeral rule)

NACL rules are **numbered and ordered**. Lowest number wins; evaluation stops at the first match. They support both `ALLOW` and `DENY`.

```
INBOUND NACL:
Rule#  Type      Protocol  Port      Source        Allow/Deny
─────  ────      ────────  ────      ──────        ──────────
100    HTTPS     TCP       443       0.0.0.0/0     ALLOW
110    SSH       TCP       22        203.0.113.7/32 ALLOW
*      ALL       ALL       ALL       0.0.0.0/0     DENY   ← implicit final deny

OUTBOUND NACL:
Rule#  Type      Protocol  Port         Dest          Allow/Deny
─────  ────      ────────  ────         ────          ──────────
100    HTTPS     TCP       443          0.0.0.0/0     ALLOW
120    Custom    TCP       1024-65535   0.0.0.0/0     ALLOW  ⭐ ephemeral return ports
*      ALL       ALL       ALL          0.0.0.0/0     DENY
```

⭐ Rule 120 is the return-traffic rule. Without it, inbound HTTPS requests arrive but the responses (which leave *to* the client's ephemeral port) get denied → mysterious hangs.

### `ufw` (Uncomplicated Firewall — the friendly Linux front-end)

```bash
ufw default deny incoming     # default-deny for inbound (secure baseline)
ufw default allow outgoing    # allow the box to reach out
ufw allow 22/tcp              # permit SSH
ufw allow 443/tcp             # permit HTTPS
ufw allow from 203.0.113.7 to any port 22   # SSH only from one IP
ufw enable                    # turn it on
ufw status verbose            # inspect
```

`ufw` is **stateful** — like a Security Group, you only write the inbound allow and replies flow automatically.

### `iptables` (the raw Linux packet filter)

```bash
# The 3 built-in chains: INPUT (inbound), OUTPUT (outbound), FORWARD (routed-through)

# ⭐ The single line that makes iptables stateful — allow established replies:
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

iptables -A INPUT -p tcp --dport 22  -j ACCEPT   # allow SSH in
iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # allow HTTPS in
iptables -A INPUT -i lo -j ACCEPT                # allow loopback
iptables -P INPUT DROP                           # default policy: DROP (default-deny)
```

Breakdown of one rule:
```
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
│        │  │     │  │    │       │  │  │
│        │  │     │  │    │       │  │  └── ACCEPT (or DROP / REJECT)
│        │  │     │  │    │       │  └── jump target = the action
│        │  │     │  │    │       └── destination port 443
│        │  │     │  │    └── --dport = match destination port
│        │  │     │  └── protocol tcp
│        │  │     └── -p = protocol match
│        │  └── the INPUT chain (inbound packets)
│        └── -A = append a rule to the chain
└── the command
```

Note the difference between `DROP` and `REJECT` — it maps directly onto the two errors from Topic 01:

```
DROP    → silently discard the packet, send nothing back  → client sees ETIMEDOUT
REJECT  → send back an ICMP "port unreachable" / TCP RST   → client sees ECONNREFUSED
```

---

## Example 1 — Basic: Lock Down a Cloud Server with a Security Group

**Goal:** You have a web server at private IP `10.0.1.20` that should serve HTTPS to the world but only allow SSH from your office.

**The Security Group (`sg-web`):**

```
INBOUND RULES
Type    Protocol  Port   Source            Purpose
─────   ────────  ────   ──────            ───────
HTTPS   TCP       443    0.0.0.0/0         serve the site to everyone
HTTP    TCP       80     0.0.0.0/0         redirect to HTTPS
SSH     TCP       22     203.0.113.7/32    admin access from office ONLY

OUTBOUND RULES
Type    Protocol  Port   Destination       Purpose
─────   ────────  ────   ──────────        ───────
All     All       All    0.0.0.0/0         (AWS default: allow all egress)
```

**Why this is correct:**

- Port 443 and 80 from `0.0.0.0/0` — correct, a public website *should* be reachable by anyone.
- Port 22 from `203.0.113.7/32` — SSH is restricted to a single IP. **Never** open 22 to `0.0.0.0/0`.
- **No outbound return rules needed for the inbound traffic** — Security Groups are stateful. The HTTPS *responses* leave automatically because AWS remembers each inbound connection.

**Prove it's stateful — there is no inbound rule mentioning ephemeral ports, yet clients get their replies.** That's the whole point. Compare this to Example 2, where the stateless NACL forces you to add exactly that.

Create it with the AWS CLI:

```bash
aws ec2 create-security-group \
  --group-name sg-web --description "web tier" --vpc-id vpc-0abc123

aws ec2 authorize-security-group-ingress \
  --group-id sg-0web456 \
  --ip-permissions \
    IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges='[{CidrIp=0.0.0.0/0}]' \
    IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges='[{CidrIp=203.0.113.7/32,Description="office"}]'
```

---

## Example 2 — Production Scenario: "The Connection Times Out (But Only Sometimes)"

**The situation:** Your team ships a new microservice. The app tier (`sg-app`, instances in subnet `10.0.2.0/24`) needs to talk to a Postgres database (`sg-db`, instance `10.0.3.15:5432`). After deploy, the app logs are full of:

```
Error: connect ETIMEDOUT 10.0.3.15:5432
    at TCPConnectWrap.afterConnect [as oncomplete]
```

Requests to the app itself from the load balancer are fine. But the app→DB call hangs for ~30 seconds then times out. **Nothing is refused — everything just hangs.** That word "timeout" is your biggest clue.

### Step 1 — Timeout vs refused (Topic 01)

```
ECONNREFUSED  → a TCP RST came back instantly. Something is THERE,
                actively saying "no." Usually: wrong port, service down.
                → NOT a firewall drop. Look at the app.

ETIMEDOUT     → packets left, nothing came back at all. A black hole.
                → Classic signature of a firewall DROPPING packets.
                → Look at Security Groups and NACLs. ★
```

We have `ETIMEDOUT`, so we're hunting a dropped packet, not a broken app.

### Step 2 — Confirm the app can reach the DB at all

From an app instance:

```bash
# Try to open the TCP connection directly (nc = netcat)
nc -zv -w 5 10.0.3.15 5432
# HANGS for 5s then:  nc: connect to 10.0.3.15 port 5432 (tcp) timed out

# Compare: is the DB even listening? SSH into the DB box and check:
ss -tlnp | grep 5432
# LISTEN 0 244 0.0.0.0:5432   ← yes, Postgres IS listening. Not an app problem.
```

The DB is listening; the app's packets are being dropped in transit. Firewall confirmed.

### Step 3 — Inspect the DB Security Group

```bash
aws ec2 describe-security-groups --group-ids sg-0db789 \
  --query "SecurityGroups[0].IpPermissions"
```

```json
[
  {
    "IpProtocol": "tcp",
    "FromPort": 5432, "ToPort": 5432,
    "IpRanges": [ { "CidrIp": "10.0.3.0/24" } ]   ← ⚠️ allows the DB's OWN subnet
  }
]
```

**The bug:** the inbound rule on `sg-db` allows `10.0.3.0/24` — the **database's** subnet. But the app runs in `10.0.2.0/24`. The app's packets arrive from a source the DB SG never allowed, so they're dropped → `ETIMEDOUT`.

### Step 4 — Fix it the right way (reference the SG, don't hardcode)

Instead of hardcoding the app subnet CIDR (which breaks the moment the app moves), reference the **app's Security Group** as the source:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0db789 \
  --ip-permissions \
    IpProtocol=tcp,FromPort=5432,ToPort=5432,\
UserIdGroupPairs='[{GroupId=sg-0app456,Description="app tier only"}]'
```

Now `sg-db` reads: **"allow TCP 5432 from any instance carrying `sg-app`."** No IPs to maintain. The app connects instantly:

```bash
nc -zv -w 5 10.0.3.15 5432
# Connection to 10.0.3.15 5432 port [tcp/postgresql] succeeded!
```

### Step 5 — The stateless twist (if there were a NACL too)

If the subnets also had a restrictive **Network ACL**, fixing the SG wouldn't be enough. The NACL is stateless, so even with inbound 5432 allowed, the DB's *reply* leaves to the app's **ephemeral port**. You'd need the outbound ephemeral rule:

```
sg-db (stateful)   → fixed by ONE inbound rule. Replies automatic.
NACL (stateless)   → ALSO needs OUTBOUND allow TCP 1024-65535 to 10.0.2.0/24
                     or the DB's response never gets back to the app.
```

### Bonus — the audit that finds the *opposite* mistake

A week later, security runs an external scan (Topic 26) and finds a horror:

```bash
nmap -Pn -p 5432 db.myapp.com
# PORT     STATE SERVICE
# 5432/tcp open  postgresql     ← ❌ reachable FROM THE PUBLIC INTERNET
```

Someone had earlier added `5432 from 0.0.0.0/0` to "make it work quickly." That's a database exposed to all 4.3 billion IPv4 addresses. The fix is to **remove** that rule and rely on the SG-reference rule from Step 4, so the DB is reachable *only* from the app tier and never from the internet.

---

## Common Mistakes

### Mistake 1: Opening SSH or a database port to `0.0.0.0/0`

```
INBOUND  SSH   TCP  22    0.0.0.0/0    ❌ the entire internet can attempt SSH
INBOUND  PG    TCP  5432  0.0.0.0/0    ❌ your database is exposed to the world
```
Bots scan the entire IPv4 space for open 22 and 5432 constantly. An open DB port is found in minutes and is a top cause of ransomware / data theft.

**Fix:** restrict to a specific source. Reference the app SG for the DB, and a `/32` (your office/VPN) for SSH — or eliminate public SSH entirely with a bastion / SSM Session Manager (Topic 31).
```
INBOUND  SSH   TCP  22    203.0.113.7/32   ✅ one host
INBOUND  PG    TCP  5432  sg-0app456        ✅ only the app tier
```

---

### Mistake 2: Treating a stateful SG like a stateless NACL (redundant return rules)

```
On a Security Group, people add:
  INBOUND   allow TCP 443
  OUTBOUND  allow TCP 1024-65535   ❌ unnecessary — SG is stateful
```
This isn't dangerous, just confused — it reveals a misunderstanding, and worse, people sometimes *tighten* egress this way and accidentally block legitimate outbound calls. On an SG, the reply to an allowed inbound connection is **always** allowed automatically. You never write return rules.

**Fix:** on a Security Group, write rules only for the *initiating* direction. Leave egress at default-allow unless you have a specific reason to restrict it.

---

### Mistake 3: Forgetting the NACL ephemeral return-port rule (one-way failure)

```
On a Network ACL:
  INBOUND   allow TCP 443       ✅ requests arrive
  OUTBOUND  allow TCP 443       ❌ WRONG — replies don't leave on 443!
```
The reply to an inbound HTTPS request leaves *from* your port 443 *to* the client's **ephemeral** port (e.g. 51844). An outbound rule for port 443 does nothing for it. Result: connections succeed one way and hang the other → the maddening "server accepts but never responds."

**Fix:** on a NACL, always add the outbound ephemeral range.
```
  OUTBOUND  allow TCP 1024-65535 to 0.0.0.0/0   ✅ return traffic
```

---

### Mistake 4: Confusing a dropped packet (timeout) with a rejected one (refused)

```
ETIMEDOUT     → packet was DROPPED. → It's a FIREWALL. Check SG/NACL.
ECONNREFUSED  → packet got a RST.   → It's the APP. Wrong port / not running.
```
People burn hours poking at Security Groups when the error was `ECONNREFUSED` (an app problem), or restarting the app when the error was `ETIMEDOUT` (a firewall problem). The error string tells you which half of the stack to look at *before* you touch anything.

**Fix:** read the error first. Refused = app/port; timeout = firewall/route.

---

### Mistake 5: Forgetting egress rules exist

```
"My app can't reach the Stripe API / can't run npm install / DNS hangs."
```
You locked down **outbound** rules (or a NACL denies outbound), and now your server can't make *its own* connections out. Egress is the forgotten half. Note: DNS needs **UDP 53** outbound, and package installs / API calls need **TCP 443** outbound.

**Fix:** ensure egress allows what your app initiates — at minimum TCP 443 and UDP 53 outbound. On a NACL, remember the *inbound* ephemeral rule for the *replies* to your outbound calls.

---

### Mistake 6: Hardcoding IPs instead of referencing Security Groups

```
INBOUND  PG  TCP  5432  10.0.2.14/32, 10.0.2.15/32, 10.0.2.16/32   ❌ brittle
```
Every time autoscaling adds an app instance with a new IP, this rule silently stops covering it → intermittent `ETIMEDOUT` that "randomly" affects some requests.

**Fix:** reference the app's Security Group as the source. It matches on membership, not address, so it survives autoscaling and IP churn.
```
INBOUND  PG  TCP  5432  sg-0app456   ✅ always current
```

---

### Mistake 7: Assuming a Security Group can DENY

```
"I'll add a DENY rule to block that one bad IP on my SG."   ❌ impossible
```
Security Groups are **allow-only**. There is no deny rule. You cannot use an SG to block a specific IP while allowing everyone else.

**Fix:** to explicitly *block* a source, use a **Network ACL** (which supports `DENY`) or a WAF. SGs express "who is allowed"; NACLs can express "who is forbidden."

---

## Hands-On Proof

```bash
# ── On any Linux host: prove your firewall is stateful ──────────────

# 1. See the connection-tracking table (the "memory" of a stateful FW)
sudo conntrack -L 2>/dev/null | head
#   tcp 6 431999 ESTABLISHED src=10.0.1.20 dst=203.0.113.7 sport=443 dport=51844 ...
#   ↑ each line is a remembered connection whose replies flow automatically

# 2. Look at the raw iptables rules and spot the stateful line
sudo iptables -L INPUT -n -v --line-numbers
#   Look for: ctstate RELATED,ESTABLISHED  ACCEPT   ← this makes it stateful

# 3. ufw quick view
sudo ufw status verbose

# ── Prove timeout vs refused (the two failure signatures) ───────────

# 4. REFUSED: connect to a port where nothing listens on localhost
nc -zv 127.0.0.1 9999
#   nc: connect to 127.0.0.1 port 9999 (tcp) failed: Connection refused  ← instant

# 5. TIMEOUT: connect to a routable IP that silently drops (a firewalled host)
nc -zv -w 5 10.255.255.1 5432
#   (hangs ~5s) ... timed out                                            ← dropped

# ── On AWS: inspect your live rules ─────────────────────────────────

# 6. Dump a Security Group's rules
aws ec2 describe-security-groups --group-ids sg-0web456 \
  --query "SecurityGroups[0].IpPermissions"

# 7. Dump the NACL for a subnet and look for the ephemeral outbound rule
aws ec2 describe-network-acls \
  --filters Name=association.subnet-id,Values=subnet-0abc \
  --query "NetworkAcls[0].Entries"

# 8. Check what YOUR box is actually listening on (Topic 24)
ss -tlnp
```

---

## Practice Exercises

### Exercise 1 — Easy: Read a rule set and predict the outcome

```
Given this Security Group on a web server (10.0.1.20):

INBOUND
  HTTPS  TCP  443   0.0.0.0/0
  SSH    TCP  22    203.0.113.7/32
OUTBOUND
  (empty — no rules at all)

Answer:
1. Can a random user on the internet load the HTTPS site? Why?
2. Can that user SSH in? Why?
3. The OUTBOUND rules are EMPTY. Can the site still send HTTPS
   RESPONSES back to visitors? (Careful — think stateful.)
4. Can this server run `npm install` (make an outbound HTTPS call)?

# Hint: the SG is stateful, and the OUTBOUND section being empty only
# matters for connections the SERVER itself initiates — not for replies.
```

---

### Exercise 2 — Medium: Fix the one-way NACL

```
A subnet has this Network ACL. Users report the web server accepts
connections but pages never load (browser spins forever).

INBOUND
  100  TCP  443         0.0.0.0/0   ALLOW
  *    ALL  ALL         0.0.0.0/0   DENY
OUTBOUND
  100  TCP  443         0.0.0.0/0   ALLOW
  *    ALL  ALL         0.0.0.0/0   DENY

Tasks:
1. Explain EXACTLY why pages hang, tracing a single request+reply.
2. Which error will the client eventually see — refused or timeout?
3. Write the ONE rule that fixes it.
4. Would this same bug exist if this were a Security Group instead? Why?

# Hint: trace where the REPLY goes — from port 443 to the client's
# ephemeral port. Does any outbound rule cover that destination port?
```

---

### Exercise 3 — Hard: Design the 3-tier rule set from scratch

```
Design the Security Groups for this architecture. Use SG references,
never hardcoded IPs. State the exact inbound rules for each tier.

  Internet ──443──► ALB (sg-alb) ──8080──► App (sg-app) ──5432──► DB (sg-db)

Requirements:
- The public may reach the ALB on 443 only.
- The app tier accepts traffic ONLY from the ALB, on port 8080.
- The DB accepts Postgres ONLY from the app tier.
- Admins SSH to the app tier only from the corporate VPN (198.51.100.0/24).
- Nothing else may reach the app or DB.

Deliverable: write the INBOUND rules for sg-alb, sg-app, sg-db.
Then answer: why is referencing sg-app on the DB better than
listing the app instances' IPs?

# Hint: each tier's inbound source should be the SG of the tier in
# FRONT of it (sg-alb → sg-app → sg-db). Think about what happens to a
# hardcoded-IP rule the moment autoscaling launches a new app instance.
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **You allow inbound TCP 443 on an AWS Security Group and nothing else. A client connects and gets a response. You wrote no outbound rule. Why did the reply get out?**

2. **The exact same setup on a Network ACL results in the client hanging forever. Why — and what one rule fixes it?**

3. **What is the ephemeral port range, and why does a stateless firewall force you to open it on the *return* direction?** (Tie to Topic 05.)

4. **Your app logs `ECONNREFUSED` connecting to a service. Should you look at the Security Group first, or the app? Why?**

5. **What does `0.0.0.0/0` mean, and name one port where it's appropriate for inbound and one where it's dangerous.**

6. **A teammate says "let's add a DENY rule to our Security Group to block this attacker's IP." What's wrong with that sentence, and what should you use instead?**

7. **Why is referencing another Security Group as a source better than hardcoding the app servers' IP addresses?**

---

## Quick Reference Card

### Security Group vs Network ACL

| Feature | Security Group (SG) | Network ACL (NACL) |
|---------|--------------------|--------------------|
| **State** | **Stateful** (remembers connections) | **Stateless** (each packet judged alone) |
| **Attached to** | ENI / instance | Subnet |
| **Rule actions** | **ALLOW only** | **ALLOW and DENY** |
| **Rule order** | No order — union of all rules | Numbered, first match wins |
| **Return traffic** | Automatic | **You must add ephemeral rules** |
| **Default inbound** | Deny all | (default NACL) allow all; custom = deny all |
| **Evaluation** | Union of all attached SGs | Single NACL per subnet |
| **Source can be** | CIDR **or another SG** | CIDR only |
| **Blocks a specific bad IP?** | ❌ can't (allow-only) | ✅ yes (DENY rule) |

### Direction cheat sheet

| Term | Also called | Controls | Common bug |
|------|-------------|----------|------------|
| **Inbound** | Ingress | Who can reach your ports | Missing allow → nobody connects |
| **Outbound** | Egress | Where your box may connect | Over-restrictive → app can't reach DB/API |

### Failure-signature cheat sheet

```
ECONNREFUSED  → RST received       → APP problem (wrong port / not running)
ETIMEDOUT     → nothing came back  → FIREWALL problem (packet dropped)
ENOTFOUND     → DNS didn't resolve → name resolution, not the firewall
```

### The stateless return-port trap

```
STATEFUL  (SG, iptables+conntrack, ufw): write ONE rule (the inbound allow).
STATELESS (NACL, raw packet filter):     write TWO — inbound allow
          + OUTBOUND allow TCP 1024-65535 (the ephemeral return ports).

Ephemeral ranges:  Linux 32768–60999   |   IANA 49152–65535
NACL rule of thumb: open outbound 1024–65535 to cover all clients.
```

### The canonical 3-tier pattern

```
Internet ─443─► ┌─────────┐ ─8080─► ┌─────────┐ ─5432─► ┌────────┐
   0.0.0.0/0    │ sg-alb  │         │ sg-app  │         │ sg-db  │
                └─────────┘         └─────────┘         └────────┘
  sg-alb  in: 443 from 0.0.0.0/0
  sg-app  in: 8080 from sg-alb        (+ 22 from VPN /24)
  sg-db   in: 5432 from sg-app
  Each tier only trusts the tier in front of it. No hardcoded IPs.
```

### Key commands

```bash
# AWS
aws ec2 describe-security-groups --group-ids sg-xxxx
aws ec2 authorize-security-group-ingress --group-id sg-xxxx --ip-permissions ...
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-xxxx

# Linux host firewall
sudo ufw status verbose
sudo iptables -L -n -v --line-numbers
sudo conntrack -L                          # see the stateful connection table

# Diagnose
nc -zv -w 5 HOST PORT                       # timeout=firewall, refused=app
ss -tlnp                                    # what is actually listening
nmap -Pn -p 22,443,5432 HOST                # what is exposed (Topic 26)
```

---

## When Would I Use This at Work?

### Scenario 1: "The new service can't reach the database"

Logs show `ETIMEDOUT` (not refused), so you know it's a dropped packet, not the app. You check the DB's Security Group, find its inbound rule allows the wrong source, and fix it by referencing the app's SG. Ten-minute fix once you read the error string correctly. Getting `ECONNREFUSED` instead would send you to check whether Postgres was actually running and on port 5432.

### Scenario 2: "Security audit flagged an exposed database"

An `nmap` scan (Topic 26) shows port 5432 `open` from the public internet — someone added `0.0.0.0/0` to unblock themselves during an incident and never reverted it. You remove the rule, replace it with an SG-reference so only the app tier can connect, and add a NACL `DENY` if you need belt-and-suspenders. This is one of the most common and most serious cloud findings.

### Scenario 3: "Standing up a new environment"

Every new service gets the 3-tier treatment: an ALB SG open to the world on 443, an app SG that only trusts the ALB SG, and a DB SG that only trusts the app SG. You wire it with SG references so autoscaling never breaks the rules. This pattern is muscle memory once you've done it a few times — and it's exactly what interviewers expect you to describe.

### Scenario 4: "The outbound call to a payment API keeps hanging"

Your app can serve requests but can't *make* them. You remember egress is the forgotten half, check the outbound rules / NACL, and discover DNS (UDP 53) or HTTPS (TCP 443) egress was locked down. On a NACL you also add the inbound ephemeral rule so the API's *responses* to your outbound call can return.

---

## Connected Topics

**Study before this:**
- `04-ip-addresses.md` — CIDR notation (`/32`, `/24`, `0.0.0.0/0`) is the language of firewall source/destination rules
- `05-ports.md` — well-known ports (22, 443, 5432) and **ephemeral ports** — the return-port range is the crux of stateless firewalls
- `01-how-the-internet-works.md` — `ECONNREFUSED` vs `ETIMEDOUT`, the two failure signatures that tell you app-vs-firewall

**Related / study alongside:**
- `31-vpns-and-private-networks.md` — VPCs, private subnets, and bastions; firewalls enforce the boundaries a private network defines
- `26-nmap-basics.md` — how you *audit* which ports your firewall actually leaves exposed
- `24-netstat-and-ss.md` — confirm what your host is truly listening on before blaming the firewall

**Study next:**
- `33-network-latency.md` — once packets are allowed through, how fast can they possibly travel?

---

*Doc saved: `/docs/networking/32-firewalls-and-security-groups.md`*
