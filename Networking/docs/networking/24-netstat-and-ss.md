# 24 — netstat and ss

> **Phase 4 — Topic 4 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `05-ports.md`, `08-tcp-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine your computer is a giant office building. Every phone line coming in or going out of the building is a **network connection**. Some phones are just sitting there waiting for someone to call (those are **listening**). Some are in the middle of an active conversation (**established**). Some just hung up and the line is cooling down for a minute before it can be reused (**time-wait**). And one phone has a caller who hung up minutes ago, but nobody in your office has put the receiver back down — the line is stuck open forever (**close-wait**, a bug).

There is a receptionist with a clipboard who can, at any instant, list **every single phone line in the building**: which are waiting, which are talking, who they're talking to, and which employee is holding each one.

That clipboard is the **kernel's socket table**. `ss` and `netstat` are the two tools that print the clipboard for you. `netstat` is the retired old receptionist; `ss` is the fast new one who reads the master ledger directly.

---

## Where This Lives in the Network Stack

`ss` and `netstat` do not sit *on* a layer — they are **observation tools** that read the kernel's view of Layer 4 (Transport). They show you the state of TCP and UDP sockets: the connection tables the kernel maintains.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (your Node.js / Postgres / nginx)      │  ← owns the socket (Process column)
│  Layer 4 — Transport     (TCP state machine, ports, queues)     │  ← ss/netstat READ this
│  Layer 3 — Network        (IP — Local/Peer addresses)           │  ← shown as addresses
│  Layer 2 — Data Link      (Ethernet)                            │
│  Layer 1 — Physical       (fiber, copper)                       │
└─────────────────────────────────────────────────────────────────┘

              ┌──────────────────────────┐
              │   KERNEL SOCKET TABLE     │   ← the source of truth
              │  (one row per socket)     │
              └────────────┬─────────────┘
                           │  reads /proc/net/tcp, /proc/net/udp
                           │  (ss uses the faster netlink sock_diag API)
                  ┌────────┴────────┐
                  ▼                 ▼
               ss                netstat
          (modern, fast)     (deprecated, slow)
```

`tcpdump` (Topic 23) shows you **packets on the wire**. `ss` shows you the **kernel's connection state** — the end result of all those packets. They are complementary: tcpdump answers "what bytes moved?", ss answers "what connections exist right now and what state are they in?"

---

## What Is This?

`ss` (**socket statistics**) and `netstat` (**network statistics**) both dump the kernel's table of open sockets. A **socket** is the kernel's endpoint for a network conversation, identified by the 4-tuple `(local IP, local port, remote IP, remote port)` plus a protocol.

- **`netstat`** is the classic Unix tool. On Linux it lives in the `net-tools` package, which has been **deprecated since ~2011** and is no longer installed by default on modern distros. It works by parsing `/proc/net/tcp`, `/proc/net/udp`, etc. — reading and reformatting text files, which is slow on a box with tens of thousands of connections.

- **`ss`** is part of `iproute2` (the same package as `ip`). It is the **modern replacement**. It queries the kernel directly through the **netlink `sock_diag`** interface, which is dramatically faster and can filter server-side (the kernel does the filtering, not your shell). On a host with 100k sockets, `ss` finishes in milliseconds where `netstat` crawls.

**The one-line takeaway:** anything you used to type as `netstat`, type as `ss` instead. The flags are almost identical.

### netstat → ss translation table

| Goal | Old (`netstat`) | Modern (`ss`) |
|------|-----------------|---------------|
| All TCP sockets, numeric | `netstat -tan` | `ss -tan` |
| Listening TCP + process | `netstat -tlnp` | `ss -tlnp` |
| Listening TCP+UDP + PID, numeric | `netstat -tulpn` | `ss -tulpn` |
| Everything with PIDs | `netstat -antp` | `ss -antp` |
| Protocol summary stats | `netstat -s` | `ss -s` |
| UDP sockets | `netstat -uan` | `ss -uan` |
| Routing table | `netstat -r` | `ip route` |
| Interface stats | `netstat -i` | `ip -s link` |

Note the last two rows: `ss` deliberately does **not** do routing or interface stats — those moved to `ip`. `ss` is only about sockets.

---

## Why Does It Matter for a Backend Developer?

This is the single most useful "is my service actually alive?" tool you have on a box. You reach for `ss` when:

- **"Is my server even listening?"** — You deployed, the app started, but clients get `connection refused`. One `ss -tlnp` tells you whether your process bound the port at all, and on which interface (`0.0.0.0` = all, `127.0.0.1` = localhost only — a classic "works locally, refuses in prod" bug).
- **"Which process is hogging port 8080?"** — `Address already in use` on startup. `ss -tlnp | grep :8080` names the PID holding it.
- **"Why did my server run out of file descriptors?"** — Every socket is a file descriptor. Thousands of stuck `CLOSE-WAIT` sockets means your app is leaking connections. `ss` finds them.
- **"Why can't I make new outbound connections?"** — Thousands of `TIME-WAIT` sockets from short-lived outbound calls exhaust your ephemeral port range. `ss` counts them.
- **"Is my accept queue overflowing?"** — Under load, clients time out even though the server looks healthy. A non-zero `Recv-Q` on a `LISTEN` socket means connections are arriving faster than your app calls `accept()`.

Every one of these is a real production incident. `ss` turns "the service is weird" into a specific, named root cause in under a minute.

---

## The Packet/Protocol Anatomy

`ss` doesn't produce packets — it reports the TCP **state machine** for each socket. Every TCP connection is, at any moment, in exactly one of 11 states. This is the machine from Topic 08, and `ss` is the window into it:

```
                          CLIENT SIDE                    SERVER SIDE
                          (initiator)                    (responder)

  server boots:                                          ┌──────────┐
                                                         │  LISTEN  │  waiting for SYN
                                                         └────┬─────┘
  ── 3-WAY HANDSHAKE ──                                       │
  ┌──────────┐   ─── SYN ──────────────────────────────►      │
  │ SYN-SENT │                                          ┌──────▼──────┐
  └────┬─────┘   ◄──────────────── SYN,ACK ──────────── │  SYN-RECV   │
       │         ─── ACK ──────────────────────────────►└──────┬──────┘
  ┌────▼─────┐                                          ┌───────▼──────┐
  │  ESTAB   │◄════════ data flows both ways ══════════►│    ESTAB     │
  └────┬─────┘                                          └───────┬──────┘
       │                                                        │
  ── 4-WAY TEARDOWN (whoever calls close() first is "active") ──
  ┌────▼──────┐  ─── FIN ──────────────────────────────►┌───────▼──────┐
  │ FIN-WAIT-1│  ◄──────────────── ACK ──────────────── │  CLOSE-WAIT  │  ← app must call close()
  └────┬──────┘                                         └───────┬──────┘
  ┌────▼──────┐                                         ┌───────▼──────┐
  │ FIN-WAIT-2│  ◄──────────────── FIN ──────────────── │   LAST-ACK   │
  └────┬──────┘  ─── ACK ──────────────────────────────►└──────────────┘
  ┌────▼──────┐                                                 │
  │ TIME-WAIT │  wait 2×MSL (~60s), then ──► CLOSED             ▼
  └───────────┘                                              CLOSED
```

Key insight from this diagram: **`TIME-WAIT` lives on whoever closed the connection first** (usually the client/initiator). **`CLOSE-WAIT` lives on whoever received the FIN** and is waiting for its own application to call `close()`. If that `close()` never comes, the socket is stuck in `CLOSE-WAIT` forever. Remember these two — they are 90% of your `ss`-driven debugging.

### The columns `ss` prints

```
State    Recv-Q  Send-Q   Local Address:Port    Peer Address:Port    Process
  │        │       │             │                      │                │
  │        │       │             │                      │                └─ which process/PID/fd owns it (-p)
  │        │       │             │                      └─ the OTHER end (remote IP:port)
  │        │       │             └─ YOUR end (local IP:port)
  │        │       └─ Send queue  (meaning depends on state — see below)
  │        └─ Receive queue       (meaning depends on state — see below)
  └─ TCP state (LISTEN, ESTAB, TIME-WAIT, CLOSE-WAIT, ...)
```

**Recv-Q and Send-Q are the columns people misread.** Their meaning changes based on socket state:

| State | Recv-Q means | Send-Q means |
|-------|--------------|--------------|
| `LISTEN` | current **accept queue** depth — established conns waiting for `accept()` | the **backlog limit** (max accept queue size) |
| `ESTAB` (and others) | bytes **received** by kernel, not yet `read()` by your app | bytes **sent** by your app, not yet **ACKed** by the peer |

So on a `LISTEN` socket, `Recv-Q 0 / Send-Q 4096` means "0 connections waiting, queue can hold 4096." If `Recv-Q` climbs toward `Send-Q`, your app is too slow to `accept()` and new connections are being dropped. On an `ESTAB` socket, a stuck non-zero `Send-Q` means the peer isn't ACKing (slow/dead peer, or network stall); a stuck `Recv-Q` means **your own app** isn't reading fast enough.

---

## How It Works — Step by Step

What actually happens when you run `ss -tlnp`:

```
Step 1 — Parse flags
   -t  → filter to TCP sockets
   -l  → filter to LISTEN state only
   -n  → numeric: do NOT resolve ports→names or IPs→hostnames
   -p  → include the owning process (requires root for other users' procs)

Step 2 — Build a request
   ss constructs a netlink SOCK_DIAG_BY_FAMILY message describing
   "give me all AF_INET/AF_INET6 TCP sockets in state LISTEN"

Step 3 — Kernel does the filtering
   The request goes to the kernel via a netlink socket. The kernel
   walks its internal socket hash tables and returns ONLY matching
   sockets. (This is why ss is fast — filtering happens kernel-side,
   not by grepping a giant text dump.)

Step 4 — Resolve the process (-p)
   For each socket, the kernel returns the owning inode. ss scans
   /proc/<pid>/fd/* to find which process holds that inode → PID + fd.
   (This scan needs privileges to see other users' processes.)

Step 5 — Format & print
   ss prints the columns. Without -n it would now do a DNS/service
   lookup per row (SLOW, can hang). With -n it skips that entirely.
```

`netstat` does the same job but at Step 3 it reads and re-parses `/proc/net/tcp` line by line in userspace — no kernel-side filtering. On a busy host that file can be megabytes, and `netstat` re-reads it fully for every invocation.

**The `-n` step is the one that bites people.** Without `-n`, every socket triggers a reverse-DNS lookup (`142.250.80.46` → hostname) and a `/etc/services` lookup (`443` → `https`). If DNS is slow or unreachable, `ss` / `netstat` **hangs for seconds per row**. Always use `-n` when you want speed and raw numbers.

---

## Exact Syntax Breakdown

### `ss -tlnp` — "what is listening on this box?"

```
ss   -t      -l        -n          -p
│    │       │         │           │
│    │       │         │           └── -p = show Process (name, pid, fd) owning each socket
│    │       │         └── -n = Numeric — no DNS, no /etc/services lookup (fast, raw)
│    │       └── -l = Listening sockets only (State = LISTEN)
│    └── -t = TCP sockets only (add -u for UDP)
└── socket statistics
```

Reads as: *"Show me the TCP (`-t`) sockets that are listening (`-l`), as raw numbers (`-n`), and tell me which process owns each (`-p`)."* This is the single most-typed `ss` command. Memorize `ss -tlnp`.

### `ss -tan state established '( dport = :443 )'` — filtered query

```
ss  -t   -a   -n   state established  '( dport = :443 )'
│   │    │    │    │                  │
│   │    │    │    │                  └── FILTER EXPRESSION: only conns whose
│   │    │    │    │                      destination (peer) port is 443
│   │    │    │    └── STATE FILTER: only ESTABLISHED sockets
│   │    │    └── -n = numeric
│   │    └── -a = All sockets (listening AND non-listening)
│   └── -t = TCP only
└── ss
```

Reads as: *"Show all established TCP connections whose remote port is 443"* — i.e., every outbound HTTPS connection this box currently has open. The `state <name>` clause and the `'( ... )'` expression are **kernel-side filters** — the kernel returns only matching rows, so this is fast even with 100k sockets.

**Expression operators:** `sport` (source/local port), `dport` (destination/peer port), `src` (local address), `dst` (peer address). Combine with `and` / `or`. Note the `:` before a port and the shell-quoting of the parentheses (they're shell metacharacters).

```
ss -tn '( dport = :443 or dport = :80 )'      # all outbound web traffic
ss -tn dst 10.0.2.7                           # all conns to that peer host
ss -tln sport = :8080                          # is anything listening on 8080?
ss -tan state time-wait                        # every socket cooling down
```

---

## Example 1 — Basic

**Goal: see everything your machine is listening on, and who owns it.**

```bash
sudo ss -tlnp
```

Annotated output on a typical backend box:

```
State   Recv-Q  Send-Q   Local Address:Port   Peer Address:Port  Process
LISTEN  0       4096     0.0.0.0:5432         0.0.0.0:*          users:(("postgres",pid=812,fd=6))
LISTEN  0       511      0.0.0.0:80           0.0.0.0:*          users:(("nginx",pid=1024,fd=8))
LISTEN  0       128      127.0.0.1:6379       0.0.0.0:*          users:(("redis-server",pid=640,fd=7))
LISTEN  0       4096     *:8080               *:*               users:(("node",pid=2140,fd=18))
LISTEN  0       128      [::]:22              [::]:*            users:(("sshd",pid=701,fd=4))
        │       │        │                    │                 │
        │       │        │                    │                 └─ process name, PID, file descriptor
        │       │        │                    └─ "*" = accepts from any peer
        │       │        └─ where it's bound: see the three cases below
        │       └─ backlog LIMIT (max accept-queue size for this listener)
        └─ 0 connections currently waiting to be accepted
```

**Read the Local Address column carefully — this is where the bugs hide:**

```
0.0.0.0:5432   → Postgres listens on ALL IPv4 interfaces. Reachable from the network.
                 (In prod this had better be firewalled — see Topic 32.)
127.0.0.1:6379 → Redis listens on LOCALHOST ONLY. A remote client gets "connection
                 refused." This is the #1 "works on my laptop, refuses in prod" cause.
*:8080         → Node on all interfaces (ss prints "*" for 0.0.0.0 in some builds).
[::]:22        → SSH on all IPv6 interfaces ([::] is the IPv6 "any" address).
```

If your teammate says *"the API is refusing connections from the load balancer,"* and `ss -tlnp` shows `127.0.0.1:8080`, you've found it in five seconds: the app bound to localhost, not `0.0.0.0`.

---

## Example 2 — Production Scenario

**The situation:** Your Node.js payment service starts throwing `Error: connect EMFILE` and then `Too many open files`. New requests fail. Restarting fixes it for ~2 hours, then it happens again. It's a leak. Let's find it with `ss`.

### Step 1 — How many sockets, and in what states?

```bash
ss -s
```

```
Total: 8402
TCP:   8231 (estab 47, closed 33, orphaned 0, timewait 21, ports 68)

Transport Total     IP        IPv6
RAW       0         0         0
UDP       4         2         2
TCP       8177      8170      7
INET      8181      8172      9
FRAG      0         0         0
```

8177 open TCP sockets but only **47 established** and only **21 timewait**. So where are the other ~8100? Summary doesn't break out `CLOSE-WAIT`, so ask directly.

### Step 2 — Group every TCP socket by state

```bash
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
```

```
   8109 CLOSE-WAIT      ← 🚨 there it is
     47 ESTAB
     21 TIME-WAIT
      5 LISTEN
```

**8109 sockets stuck in `CLOSE-WAIT`.** Recall the state machine: `CLOSE-WAIT` means *the peer sent us a FIN, the kernel ACKed it, and we are waiting for OUR application to call `close()`* — which it never does. Every one of those is a leaked file descriptor. This is a **code bug**, not a network problem.

### Step 3 — Which process is leaking?

```bash
sudo ss -tanp state close-wait | head
```

```
State       Recv-Q Send-Q  Local Address:Port  Peer Address:Port     Process
CLOSE-WAIT  1      0       10.0.1.5:52210      34.223.10.4:443       users:(("node",pid=2140,fd=814))
CLOSE-WAIT  1      0       10.0.1.5:52244      34.223.10.4:443       users:(("node",pid=2140,fd=815))
CLOSE-WAIT  1      0       10.0.1.5:52301      34.223.10.4:443       users:(("node",pid=2140,fd=816))
...
```

Every socket is: our `node` process (pid 2140), talking to `34.223.10.4:443` — a remote **HTTPS API**. The `Recv-Q 1` even tells us there's a byte waiting to be read that we never read. The fd numbers (814, 815, 816…) climb without bound — the leak made visible.

`34.223.10.4:443` is the third-party payment provider. Diagnosis: **we make outbound HTTPS calls to the provider and never close the response.** In Node that's an `http.request` / `fetch` whose response body is never consumed or whose client isn't ending the socket.

### Step 4 — The fix (in code, not the network)

```javascript
// BEFORE — the leak: response never fully consumed/closed
const res = await fetch(providerUrl);
if (res.status === 200) return true;   // ← body never read on non-200; socket stuck in CLOSE-WAIT

// AFTER — always drain/close the response, and reuse connections via a pooled agent
import { Agent } from 'undici';
const agent = new Agent({ connections: 50, keepAliveTimeout: 30_000 });

const res = await fetch(providerUrl, { dispatcher: agent });
await res.body.cancel();   // or await res.text() — consume it so the socket can close/return to pool
```

### Step 5 — Confirm the fix

```bash
watch -n2 "ss -tan state close-wait | wc -l"
# Should hover near 0 after deploy, instead of climbing into the thousands.
```

---

**The TIME-WAIT sibling incident.** Same symptom class (`cannot assign requested address`, i.e. `EADDRNOTAVAIL`), opposite state. If `ss -s` showed `timewait 27000` and `ss -tan state time-wait | wc -l` returned ~27,000, the story is different:

```bash
ss -tan state time-wait | wc -l
# 27411

# Where are they going? Group by peer:
ss -tan state time-wait | awk 'NR>1 {print $5}' | sed 's/:[0-9]*$//' | sort | uniq -c | sort -rn | head
#  27400 10.0.2.7          ← all TIME-WAIT toward one host: the database
```

27k TIME-WAIT toward the DB means we open a **fresh connection per query and close it**, churning through ephemeral ports (default range `32768–60999` ≈ 28k ports). Once they're all in TIME-WAIT you cannot allocate a new source port → outbound connections fail. `TIME-WAIT` itself is **normal and correct** (it's the 2×MSL cool-down protecting against stray old packets), so you don't "fix" it — you **stop creating so many short-lived connections**:

- **Primary fix:** use a **connection pool / keep-alive** (PgBouncer for Postgres — Topic 39; a keep-alive HTTP agent for APIs). Reuse connections instead of open-close-open-close.
- **Kernel mitigation (initiator only):** `net.ipv4.tcp_tw_reuse=1` lets the kernel safely reuse a TIME-WAIT socket for a **new outbound** connection when TCP timestamps confirm it's safe. (Do **not** use the old `tcp_tw_recycle` — it broke clients behind NAT and was **removed in Linux 4.12**.)
- Widen the ephemeral range: `net.ipv4.ip_local_port_range = 1024 65535`.

The lesson: **CLOSE-WAIT is your bug (you didn't close); TIME-WAIT is you opening too many connections (pool them).**

---

## Common Mistakes

### Mistake 1: Still reaching for `netstat` when `ss` is right there

```bash
# Old habit:
netstat -tulpn        # slow, and often "command not found" on modern distros
# net-tools is deprecated; many containers/cloud images ship without it.

# Do this instead — same flags, faster, always present with iproute2:
ss -tulpn
```
`netstat` re-parses `/proc/net/*` in userspace with no kernel-side filtering; on a box with 50k+ sockets it can take seconds. `ss` filters in the kernel and returns in milliseconds. There is no scenario in 2026 where you *should* prefer `netstat` on Linux.

---

### Mistake 2: Confusing TIME-WAIT (fine) with CLOSE-WAIT (bug)

```
TIME-WAIT   → NORMAL. The side that closed FIRST holds it for 2×MSL (~60s on Linux)
              to absorb stray retransmitted packets. Thousands of them = you're
              opening too many short-lived connections → POOL them. Not a code bug.

CLOSE-WAIT  → BUG. The peer closed; you ACKed; your app never called close().
              The socket (a file descriptor) leaks forever. Thousands of them =
              your code forgets to close sockets/responses/connections. FIX THE CODE.
```
People panic about `TIME-WAIT` (harmless, self-clearing) and ignore `CLOSE-WAIT` (the actual leak). Reverse your instincts: **CLOSE-WAIT is the one that means you have a bug.**

---

### Mistake 3: Forgetting `-n`, and watching the command hang

```bash
ss -tlp        # ← no -n: tries reverse-DNS on every peer IP and looks up
               #   every port in /etc/services. If DNS is slow/broken, this
               #   HANGS for seconds, printing "https" instead of "443",
               #   "ip-10-0-1-5.ec2.internal" instead of "10.0.1.5".

ss -tlnp       # ← with -n: raw IPs and port numbers, returns instantly.
```
When you're debugging a *networking* problem, DNS is often exactly what's broken — the last thing you want is your diagnostic tool blocking on a DNS lookup. Always add `-n`.

---

### Mistake 4: Not using `-p`, so you never learn which process owns the socket

```bash
ss -tln            # shows the port is listening... but WHICH process? Unknown.
sudo ss -tlnp      # shows users:(("node",pid=2140,fd=18)) — now you can kill/inspect it.
```
Without `-p` you know a port is busy but not *who* is busy on it — useless for `Address already in use`. Note `-p` needs **root** to see processes owned by other users (otherwise you get the socket but a blank Process column).

---

### Mistake 5: Misreading Recv-Q / Send-Q on a LISTEN socket

```bash
ss -tlnp
# LISTEN  128   128   0.0.0.0:8080  ...
#         │     │
#         │     └─ Send-Q on a LISTEN row = the backlog LIMIT (not "unsent bytes"!)
#         └─ Recv-Q on a LISTEN row = conns waiting to be accept()ed (not "unread bytes"!)
```
On a `LISTEN` socket these columns do **not** mean bytes. `Recv-Q` = current accept-queue depth; `Send-Q` = the max backlog. If `Recv-Q` is pinned at the `Send-Q` value, your accept queue is **full** and the kernel is dropping/refusing new connections. On any *non*-LISTEN socket the same columns mean unread and unacked **bytes**. Same columns, different meaning — check the State first.

---

### Mistake 6: Blaming the network when the accept queue is full

```bash
# Clients report timeouts / "connection reset". You assume it's the network.
# But:
ss -tlnp | grep :8080
# LISTEN  511   511   0.0.0.0:8080   users:(("node",pid=2140,fd=18))
#         └────┴─ Recv-Q maxed at the backlog: the queue is FULL.
```
A full accept queue means connections arrive faster than your app calls `accept()` — the app's event loop is blocked or the worker pool is saturated. The packets arrived fine; **your application** is the bottleneck. The fix is more workers / a non-blocking accept loop / a bigger backlog, not touching the network.

---

## Hands-On Proof

```bash
# 1. Everything listening, with owning process (the command to memorize):
sudo ss -tlnp

# 2. Every TCP socket in every state:
ss -tan

# 3. Protocol summary — quick health snapshot:
ss -s

# 4. Count connections by state (find leaks / churn at a glance):
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# 5. Just the TIME-WAIT pile-up count:
ss -tan state time-wait | wc -l

# 6. Just the CLOSE-WAIT leak count (the one that means a bug):
ss -tan state close-wait | wc -l

# 7. Who owns a specific port?
sudo ss -tlnp 'sport = :8080'          # or: sudo ss -tlnp | grep :8080

# 8. All ESTABLISHED outbound HTTPS connections:
ss -tn state established '( dport = :443 )'

# 9. All connections to one peer host (e.g. your database):
ss -tn dst 10.0.2.7

# 10. Watch CLOSE-WAIT live while reproducing a bug:
watch -n2 "ss -tan state close-wait | wc -l"

# --- macOS / no ss available? Use the BSD netstat and lsof: ---
netstat -an -p tcp | grep LISTEN          # listeners (BSD netstat has no -p PID flag)
lsof -nP -iTCP -sTCP:LISTEN               # listeners WITH the owning process
lsof -nP -iTCP -sTCP:CLOSE_WAIT           # find CLOSE_WAIT leaks on macOS
```

---

## Practice Exercises

### Exercise 1 — Easy: Inventory your listeners

```bash
sudo ss -tlnp        # (macOS: sudo lsof -nP -iTCP -sTCP:LISTEN)

# Answer from the output:
# 1. How many ports is this machine listening on?
# 2. Which listeners are on 127.0.0.1 (localhost-only) vs 0.0.0.0 / [::] (all interfaces)?
# 3. Pick one listener — what process and PID owns it?
# 4. For each 0.0.0.0 listener, ask: SHOULD this be exposed to the network, or is it
#    a database/cache that ought to be localhost-only or firewalled?
```

### Exercise 2 — Medium: Watch a connection walk the state machine

```bash
# Terminal A — start a throwaway TCP listener on port 9999:
nc -l 9999            # (Linux/macOS; on some systems: nc -l -p 9999)

# Terminal B — connect to it:
nc 127.0.0.1 9999

# Terminal C — observe, twice:
ss -tan '( sport = :9999 or dport = :9999 )'
#   → You should see a LISTEN row plus TWO ESTAB rows (both ends are on this host).

# Now press Ctrl-C in Terminal B (the client) and immediately re-run Terminal C.
# Questions:
# 1. Which side shows TIME-WAIT — the one you Ctrl-C'd, or the other? Why?
# 2. How long does the TIME-WAIT row persist before it disappears? (~60s)
# 3. Ctrl-C the LISTENER (Terminal A) instead next time. Does the client side now
#    show CLOSE-WAIT? What would clear it?
```

### Exercise 3 — Hard: Reproduce and diagnose a CLOSE-WAIT leak

```bash
# Goal: create sockets stuck in CLOSE-WAIT, then find them purely via ss.

# 1. Write a tiny server that ACCEPTS connections but NEVER calls close():
cat > leak.js <<'EOF'
const net = require('net');
const held = [];
net.createServer(sock => {
  held.push(sock);            // keep a reference; never sock.end()/destroy()
  sock.on('end', () => {});   // peer's FIN arrives → we go to CLOSE-WAIT and stay
}).listen(9000, () => console.log('leaky server on :9000, pid', process.pid));
EOF
node leak.js &

# 2. Open then immediately close many client connections:
for i in $(seq 1 200); do (exec 3<>/dev/tcp/127.0.0.1/9000; exec 3<&-; exec 3>&-); done

# 3. Now find the leak with ss ONLY (don't peek at the code):
ss -tan state close-wait | wc -l                 # how many are stuck?
sudo ss -tanp state close-wait | head            # which PID / process owns them?

# Questions:
# 1. How many CLOSE-WAIT sockets did you create? Which process holds them?
# 2. In the state machine, what single function call would move them out of CLOSE-WAIT?
# 3. If this were production and you couldn't fix the code immediately, what's the
#    ONLY reliable way to release those leaked file descriptors right now?
#    (Hint: the FDs die with the process.)
# Cleanup: kill %1
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the linked section.

1. **`ss` and `netstat` both read what single kernel structure?** Why is `ss` faster at reading it than `netstat`?

2. **On a `LISTEN` socket, what do the `Recv-Q` and `Send-Q` columns mean?** How is that different from their meaning on an `ESTABLISHED` socket?

3. **A connection is in `TIME-WAIT`. Which side of the connection is it — the one that called `close()` first, or the other?** How long does it last, and why does the state exist at all?

4. **You see 9,000 sockets in `CLOSE-WAIT` owned by your app. Is this a network problem or a code problem? What exact thing has your code failed to do?**

5. **You see 30,000 sockets in `TIME-WAIT`, all toward your database's IP:port. What does this tell you about how your app talks to the DB, and what is the correct fix?** (Name one code fix and one kernel setting.)

6. **Write the `ss` command to list all ESTABLISHED TCP connections whose remote port is 5432 (Postgres), showing the owning process.**

7. **Why does `ss -tlp` (no `-n`) sometimes hang for several seconds, and what one flag prevents it?**

---

## Quick Reference Card

### Core flags

| Flag | Meaning |
|------|---------|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets only |
| `-a` | All sockets (listening + non-listening) |
| `-n` | Numeric — no DNS / no `/etc/services` (fast; always use it) |
| `-p` | Show owning **p**rocess (name/PID/fd); needs root for other users |
| `-s` | Summary statistics (counts per protocol & state) |
| `-4` / `-6` | IPv4 only / IPv6 only |
| `-o` | Show timers (retransmit, keepalive, TIME-WAIT countdown) |
| `-m` | Socket memory usage |

### The commands to memorize

```bash
ss -tlnp                       # what's listening + which process   ← #1 command
ss -tanp                       # ALL TCP sockets + process
ss -s                          # summary: estab / timewait / total counts
ss -tan state established      # only established
ss -tan state time-wait        # only TIME-WAIT (churn / port pressure)
ss -tan state close-wait       # only CLOSE-WAIT (LEAK / bug)
ss -tn '( dport = :443 )'      # outbound HTTPS
ss -tln 'sport = :8080'        # is anything listening on 8080?
ss -tn dst 10.0.2.7            # all conns to that peer
```

### TCP states — who's in it & what it means

| State | Who / when | Operational meaning |
|-------|-----------|---------------------|
| `LISTEN` | server socket awaiting SYN | Normal & permanent for a server. Absent = your app didn't bind. |
| `SYN-SENT` | client sent SYN, awaits SYN-ACK | Transient. Piling up = peer unreachable / firewall dropping SYNs. |
| `SYN-RECV` | server got SYN, sent SYN-ACK, awaits ACK | In the SYN queue. Piling up = SYN flood / half-open attack. |
| `ESTABLISHED` | both ends, connection open | The working state. Data flows. |
| `FIN-WAIT-1` | we sent FIN, await its ACK | Transient teardown step (active closer). |
| `FIN-WAIT-2` | our FIN ACKed, await peer's FIN | Can linger if peer never closes (half-closed); `tcp_fin_timeout` caps it. |
| `CLOSE-WAIT` | peer sent FIN, we ACKed, await **our** `close()` | 🚨 **BUG if it piles up** — app leaking sockets/FDs. |
| `LAST-ACK` | we sent our FIN (from CLOSE-WAIT), await final ACK | Transient (passive closer). |
| `TIME-WAIT` | we closed first, waiting 2×MSL (~60s) | Normal. Pile-up = too many short-lived conns → pool them. |
| `CLOSING` | both sent FIN simultaneously | Rare (simultaneous close). |
| `CLOSED` | no connection | Not usually shown; the socket is gone. |

### TIME-WAIT vs CLOSE-WAIT cheat block

```
TIME-WAIT   = you closed a lot of connections   → normal; POOL / keep-alive
              (fix: connection reuse; kernel: net.ipv4.tcp_tw_reuse=1;
               NEVER tcp_tw_recycle — removed in Linux 4.12)

CLOSE-WAIT  = you FORGOT to close a connection   → BUG; fix the CODE
              (the peer already sent FIN; your app must call close()/end()/
               consume the response; each leaked socket = one leaked FD)
```

### netstat → ss cheat block

```
netstat -tan      → ss -tan
netstat -tlnp     → ss -tlnp
netstat -tulpn    → ss -tulpn
netstat -s        → ss -s
netstat -r        → ip route          (routing moved to `ip`)
netstat -i        → ip -s link        (interface stats moved to `ip`)
```

---

## When Would I Use This at Work?

### Scenario 1: "Deploy succeeded but the load balancer says the target is unhealthy"
`sudo ss -tlnp | grep :8080`. If it shows `127.0.0.1:8080`, the app bound to localhost — the LB (a different host) can't reach it. Change the bind address to `0.0.0.0`. If it shows **nothing**, the app crashed on startup before binding. Five-second diagnosis either way.

### Scenario 2: "`Error: bind EADDRINUSE :::3000` on startup"
Something already holds port 3000. `sudo ss -tlnp 'sport = :3000'` names the PID (maybe a zombie copy of your own app from a bad restart). `kill` it, then start clean.

### Scenario 3: "The service slowly runs out of file descriptors and dies every few hours"
`ss -tan | awk 'NR>1{print $1}' | sort | uniq -c | sort -rn`. A mountain of `CLOSE-WAIT` = your app isn't closing sockets (usually outbound API/DB calls whose responses aren't consumed). It's a **code** fix, and `ss` points straight at the offending peer IP and PID.

### Scenario 4: "Outbound calls suddenly fail with `cannot assign requested address`"
`ss -s` shows a huge `timewait` count. `ss -tan state time-wait | awk 'NR>1{print $5}' | sed 's/:[0-9]*$//' | sort | uniq -c | sort -rn` shows they're all toward one host — you're churning short-lived connections and exhausting ephemeral ports. Introduce connection pooling / keep-alive (see Topic 39 for the DB case).

### Scenario 5: "Under load, clients time out but CPU is low and the network looks fine"
`ss -tlnp` shows `Recv-Q` pinned at the `Send-Q` backlog on your listener — the accept queue is full because the app can't `accept()` fast enough. Scale workers or raise the backlog; the network is innocent.

---

## Connected Topics

**Study before this:**
- `05-ports.md` — Local/Peer `Address:Port` columns and the ephemeral port range that TIME-WAIT exhausts.
- `08-tcp-in-depth.md` — The 3-way handshake and 4-way teardown that *is* the state machine `ss` reports.

**Study alongside / next:**
- `23-tcpdump-and-wireshark.md` — Packets on the wire. Pair it with `ss`: tcpdump shows the bytes moving, `ss` shows the resulting connection state.
- `25-ping-traceroute-mtr.md` — When `ss` says the socket is fine but the peer is unreachable, these find *where* on the path it dies.
- `39-database-connections-over-network.md` — The definitive fix for TIME-WAIT churn toward a database: connection pooling and PgBouncer at the TCP level.
- `32-firewalls-and-security-groups.md` — Why a `0.0.0.0` listener you found with `ss -tlnp` still needs a firewall in front of it.

---

*Doc saved: `/docs/networking/24-netstat-and-ss.md`*
