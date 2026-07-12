# 28 — Ports and Sockets

## ELI5 — The Simple Analogy

Think of a big apartment building that receives mail.

- **The building's street address** is the machine's IP address (topic 26). It gets the mail to the right building.
- **The apartment number** is the **port**. `203.0.113.10` gets you to the building; `:3000` gets you to the specific apartment where your Node app lives.
- **A mailbox slot bolted to the wall** is a **socket** — the physical thing that actually holds mail. You (the app) install a slot, and the kernel is the postal service that stuffs letters into it.
- **The apartment manager's front desk** is a **listening socket**. One desk. Every visitor for the whole building checks in there.
- **A private meeting room the manager opens for each visitor** is a **connected socket**. The front desk (`accept()`) hands each visitor their own room so the desk stays free for the next person. One desk, thousands of private rooms.
- **The visitor's log entry** — *"visitor from 198.51.100.5, checked in via the north entrance, assigned room 12"* — is the **4-tuple**. It's how the manager keeps a thousand simultaneous visitors from getting each other's mail.
- **Apartments 1–1023** are reserved for the building's official staff — only the superintendent (root) may occupy them. Your app, a mere tenant, gets apartment 3000.

The whole doc is: sockets are mailbox slots, ports are apartment numbers, and the 4-tuple is how the kernel never mixes up whose mail is whose.

---

## Where This Lives in the Linux Stack

```
Hardware (NIC)
  └── KERNEL
       │   • TCP/IP state machine (LISTEN, ESTABLISHED, TIME_WAIT ...) ◀◀◀ THIS TOPIC
       │   • the socket table — every socket, keyed by the 4-tuple    ◀◀◀ THIS TOPIC
       │   • the accept queue + the SYN queue (backlog)               ◀◀◀ THIS TOPIC
       │   • ephemeral port allocator                                 ◀◀◀ THIS TOPIC
       │
       └── System Calls: socket() bind() listen() accept() connect()  ◀◀◀ THIS TOPIC
       │                 read() write() setsockopt() close()
       │                 → each returns/uses a FILE DESCRIPTOR (topic 19)
       │
       └── C Library (glibc — thin wrappers over the syscalls)
            │
            └── iproute2 / lsof / procps: ss, lsof -i, netstat, fuser  ◀◀◀ THIS TOPIC
                 │
                 └── Your Node app: server.listen(3000)
                                    = socket() + bind() + listen()
```

**The load-bearing idea:** a socket is a **file descriptor** (topic 19). Everything you learned about fds — that they're small integers, that they live in `/proc/PID/fd`, that `lsof` lists them, that `ulimit` caps them, that "everything is a file" (topic 03) — **applies to network sockets without exception.** That's why a connection leak shows up as an fd leak, and why a busy server dies with `EMFILE: too many open files`.

---

## What Is This?

A **socket** is the kernel's abstraction for a communication endpoint. You create one with the `socket()` syscall and it hands you back a **file descriptor** — the same kind of integer you get from `open()`. You then read and write it like a file, but the bytes travel over the network (or, for a Unix socket, over the filesystem) instead of to disk.

A **port** is a 16-bit number (0–65535) that lets one IP address host many independent services. `:3000` is your Node app; `:5432` is Postgres; `:443` is nginx. The kernel uses the port — as part of a larger **4-tuple** — to decide which socket a given packet belongs to.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `ss -tulpn` | You won't be able to answer "what's running on port 3000?" — the most common server question there is |
| The 4-tuple | You'll think one port = one connection, and be baffled how a server handles 10,000 clients |
| Sockets are fds | A connection leak will look like a memory leak, and `EMFILE` will be a mystery |
| `TIME_WAIT` | You'll see thousands of them, panic, and apply a `sysctl` "fix" that corrupts connections |
| `CLOSE_WAIT` | You'll miss the single clearest signal that **your code forgot to close something** |
| Privileged ports | You won't know why your app can't bind :80, or why running it as root to fix that is a security mistake |
| `Recv-Q` on a listen socket | You'll miss a gorgeous, direct signal that **your event loop is blocked** |
| `EADDRINUSE` triage | You'll `kill -9` the wrong PID and take down the wrong service at 3 a.m. |
| Loopback vs 0.0.0.0 bind | Your app will be up and unreachable, and you'll blame the firewall (topic 26 — again) |

---

## The Physical Reality

A socket is an entry in the kernel, pointed to by a file descriptor in your process:

```
  YOUR PROCESS (node, PID 8231)              THE KERNEL
  ┌─────────────────────────┐                ┌───────────────────────────────────┐
  │ fd table                │                │ open file table → socket structs  │
  │ ┌────┬──────────────┐   │                │                                   │
  │ │ 0  │ stdin        │   │                │  ┌─ LISTENING socket ───────────┐ │
  │ │ 1  │ stdout       │   │                │  │ 0.0.0.0:3000                 │ │
  │ │ 2  │ stderr       │   │                │  │ state: LISTEN                │ │
  │ │ 23 │ ●────────────┼───┼───────────────▶│  │ accept queue: [c1, c2]       │ │
  │ │    │  (listen fd) │   │                │  │ backlog: 511                 │ │
  │ │ 24 │ ●────────────┼───┼──────┐         │  └──────────────────────────────┘ │
  │ │ 25 │ ●────────────┼───┼───┐  │         │  ┌─ CONNECTED socket (fd 24) ───┐ │
  │ └────┴──────────────┘   │   │  └────────▶│  │ 10.0.0.15:3000 ↔ 198.51..:5510│ │
  └─────────────────────────┘   │            │  │ state: ESTABLISHED           │ │
                                │            │  └──────────────────────────────┘ │
                                │            │  ┌─ CONNECTED socket (fd 25) ───┐ │
                                └───────────▶│  │ 10.0.0.15:3000 ↔ 203.0..:4720 │ │
                                             │  │ state: ESTABLISHED           │ │
                                             │  └──────────────────────────────┘ │
                                             └───────────────────────────────────┘
```

**Notice:** the listening socket (fd 23) and the two connected sockets (fd 24, 25) are **different objects**, even though all three involve port 3000. The listener's only job is to *produce* new connected sockets via `accept()`. Every established connection is its own socket, its own fd, its own kernel struct.

```bash
# PROVE IT: a socket really is a file descriptor (topic 19)
NODE_PID=$(pgrep -n node)
ls -l /proc/$NODE_PID/fd/ | grep socket
# lrwx------ 1 app app 64 Jul 12 09:40 23 -> 'socket:[ł4821773]'
# lrwx------ 1 app app 64 Jul 12 09:40 24 -> 'socket:[54821999]'
#                                              ^^^^^^^^^^^^^^^^^^ inode number of the
#                                              socket, in the kernel's socket table
ls /proc/$NODE_PID/fd | wc -l          # total fds — sockets + files + pipes
```

### The 4-tuple — the one fact that explains everything

A TCP connection is uniquely identified by **four** values, not one:

```
        ┌──────────────┬────────────┬──────────────┬────────────┐
        │ SOURCE IP    │ SOURCE PORT│ DEST IP      │ DEST PORT  │
        ├──────────────┼────────────┼──────────────┼────────────┤
  conn1 │ 198.51.100.5 │   51000    │ 10.0.0.15    │   3000     │
  conn2 │ 198.51.100.5 │   51001    │ 10.0.0.15    │   3000     │  ← same client!
  conn3 │ 203.0.113.9  │   44000    │ 10.0.0.15    │   3000     │
        └──────────────┴────────────┴──────────────┴────────────┘
              differ ────────┘              same ────────┘  same
```

All three connect to **the same** `10.0.0.15:3000`. They don't collide because the *source* side differs. The kernel hashes the full 4-tuple to find the right socket for each incoming packet.

This single fact answers three questions at once:

- **"How does one server on one port handle 10,000 clients?"** Each client is a distinct 4-tuple → a distinct socket → a distinct fd. The listening socket stays on `:3000`; the 10,000 *connected* sockets each have their own entry. The port is not "used up."
- **"Why is the listening socket different from the connected ones?"** The listener has a half-empty tuple (`*:3000`) and only makes new sockets. The connected ones have full tuples and carry data.
- **"Why do OUTBOUND connections run out of ports?"** When *you* are the client, the kernel must pick a **source** port for each connection. To the same destination (`db:5432`), the only thing that can vary is your source port — and there are only ~28,000 of them by default. Open 28,000 connections to one destination without closing them and you get `EADDRNOTAVAIL`. (More below.)

---

## How It Works — The Socket Lifecycle, Syscall by Syscall

```
        SERVER SIDE                              CLIENT SIDE
  ────────────────────────────           ────────────────────────────
  fd = socket(AF_INET,                    fd = socket(AF_INET,
              SOCK_STREAM, 0)                         SOCK_STREAM, 0)
      │ "give me a TCP endpoint,              │ same — an unconnected
      │  return an fd"                        │ TCP endpoint
      ▼                                       │
  setsockopt(fd, SO_REUSEADDR, 1)             │
      │ "let me re-bind this port             │
      │  even if it's in TIME_WAIT"           │
      ▼                                       │
  bind(fd, 0.0.0.0:3000)                      │
      │ "claim port 3000 on all               │
      │  interfaces"  ← EADDRINUSE here       │
      ▼                                       │
  listen(fd, backlog=511)                     │
      │ "start queuing incoming               │
      │  connections; hold up to 511          │
      │  that I haven't accepted yet"         │
      ▼                                       ▼
  accept(fd)  ──────── 3-way handshake ────  connect(fd, 10.0.0.15:3000)
      │ BLOCKS until a client arrives.        │ SYN ──────────▶
      │                          ◀── SYN-ACK ─┤
      │                                    ────┤ ACK ──────────▶
      │ returns a NEW fd (say 24) for         │ connect() returns: connected
      │ THIS ONE connection. The              │
      │ listening fd (23) is untouched        │
      │ and immediately accepts the next.     │
      ▼                                       ▼
  read(24) / write(24)   ◀══ data ══▶    write(fd) / read(fd)
      │                                       │
      ▼                                       ▼
  close(24)   ──────── 4-way close ──────  close(fd)
   (the listener, fd 23, keeps running)
```

### Map it directly onto Node

```javascript
const server = http.createServer(handler);

server.listen(3000, '0.0.0.0');
//     └────────────────────────► socket() + bind(0.0.0.0:3000) + listen(511)
//         Node sets SO_REUSEADDR for you. The default backlog is 511.

server.on('connection', (socket) => {
//         └──────────────────────► each event = ONE accept() that returned a NEW fd.
//                                   `socket` wraps that fd. ★ Every concurrent
//                                   connection is one more open fd.
});
```

**The `EMFILE` consequence (topic 19):** every live connection holds an fd. The default soft limit is often 1024. A server under a connection storm — or with a leak — hits the ceiling and `accept()` starts failing with `EMFILE: too many open files`. New clients can't connect while old fds pile up. The fix is `ulimit -n` / the systemd unit's `LimitNOFILE=` (topic 22), *and* making sure you actually close connections.

---

## TCP Socket States — What Each One Means for Debugging

Every TCP socket is in exactly one state of a state machine. You'll meet these in `ss` output constantly. Learn what each **tells you**:

```
                    CONNECTION SETUP
      CLOSED ──socket()+bind()+listen()──▶ LISTEN
                                             │ a server, waiting. Steady state.
                                             │ Recv-Q here = accept-queue depth (!)
        ┌─── passive open ──────────────────┤
        │                                    │
   (client) SYN_SENT ──▶ SYN_RECV ──▶ ESTABLISHED ◀── active open (client)
        connect() sent,     got SYN,      │  A LIVE connection. Data flows.
        awaiting SYN-ACK    sent SYN-ACK  │  Thousands here = a busy server. Normal.
                                          │
                    CONNECTION TEARDOWN — WHO CALLS close() FIRST MATTERS
        ┌─────────────────────────────────┴──────────────────────────────┐
        │ ACTIVE close (you closed first)      PASSIVE close (peer first) │
        ▼                                                                 ▼
   FIN_WAIT_1 ── sent FIN, await ack                              CLOSE_WAIT
        │                                          ★ THE PEER CLOSED. The kernel is
        ▼                                            waiting for YOUR APP to call
   FIN_WAIT_2 ── peer's ack got, await peer FIN      close(). Until you do, it sits
        │                                            here. A PILE of CLOSE_WAIT =
        ▼                                            YOUR CODE isn't closing sockets.
   TIME_WAIT ── got peer FIN, sent ack                          │ (app finally close())
        │  ★ You wait here 2×MSL (~60s) so late/                ▼
        │    duplicate packets from this connection        LAST_ACK ── sent FIN,
        │    can't corrupt a NEW one reusing the tuple.        │        await final ack
        │    THOUSANDS on a busy client/proxy = NORMAL.        ▼
        ▼                                                    CLOSED
     CLOSED
```

### The two states you must be able to read instantly

```
┌──────────────┬───────────────────────────────────────────────────────────────┐
│ CLOSE_WAIT   │ The REMOTE end sent FIN (it closed). Your app has NOT yet       │
│  (a bug      │ called close() on its fd. The socket is stuck waiting for YOU.  │
│  smell)      │                                                                │
│              │ A GROWING pile of CLOSE_WAIT on YOUR side = a near-certain      │
│              │ CONNECTION/FD LEAK IN YOUR CODE. You opened a client (to a DB,  │
│              │ an upstream API, redis), the other end hung up, and you never   │
│              │ closed your handle. Each one wastes an fd → march toward EMFILE.│
│              │ FIX: find the code path that doesn't close(). It is YOURS.      │
├──────────────┼───────────────────────────────────────────────────────────────┤
│ TIME_WAIT    │ YOU closed first (active close). The kernel holds the tuple for │
│  (usually    │ 2×MSL ≈ 60s so a delayed packet from the old connection can't   │
│  NORMAL)     │ be mistaken for data on a NEW connection that reuses the same   │
│              │ 4-tuple. It is CORRECTNESS, not a leak.                         │
│              │                                                                │
│              │ Thousands on a busy PROXY or HTTP CLIENT (nginx, a service that │
│              │ makes many short outbound calls) is EXPECTED and FINE.          │
│              │ It only becomes a problem if you EXHAUST ephemeral ports.       │
│              │ FIX (if truly needed): connection pooling / keep-alive so you   │
│              │ close FEWER connections; SO_REUSEADDR; net.ipv4.tcp_tw_reuse=1  │
│              │ (safe, for OUTBOUND). ★ NEVER tcp_tw_recycle — it was REMOVED   │
│              │ from Linux 4.12 because it silently broke clients behind NAT.   │
└──────────────┴───────────────────────────────────────────────────────────────┘
```

The mnemonic: **CLOSE_WAIT = *you* forgot to close (your bug). TIME_WAIT = *you* closed correctly (the kernel's cleanup).** Confusing them sends you fixing the wrong thing.

---

## Port Classes and the Privileged-Port Rule

```
0 ─────────────────── 1023 ──────────────── 49151 ──────────────── 65535
│  WELL-KNOWN /        │   REGISTERED         │      DYNAMIC /         │
│  PRIVILEGED          │                      │      EPHEMERAL         │
│                      │                      │                        │
│ binding needs root   │ any user may bind    │ the kernel auto-picks  │
│ or CAP_NET_BIND_     │ 3000 (Node), 5432    │ these for OUTBOUND     │
│ SERVICE              │ (Postgres), 6379,    │ connections' SOURCE    │
│ 22 ssh, 80 http,     │ 8080 ...             │ port. Default range:   │
│ 443 https, 25 smtp   │                      │ 32768–60999 (~28k)     │
```

**Why your Node app runs on 3000 and nginx fronts it on 80/443:** binding a port below 1024 requires `CAP_NET_BIND_SERVICE` (historically, root). This is an old anti-spoofing rule: a random user shouldn't be able to impersonate the system's SSH or web server. So:

```
❌ WRONG: run Node as root so it can bind :80.
   A remote-code-execution bug in a single npm dependency now has ROOT on your box.
   You traded a port number for your entire server.

✅ RIGHT — three legitimate ways to serve :80/:443 without running as root:

  1. REVERSE PROXY (the standard). nginx starts as root, binds 80/443, immediately
     drops to the `www-data` user for its workers, and proxies to your Node app on
     127.0.0.1:3000. Node never needs privilege. (You also get TLS termination,
     gzip, caching, static files.) This is what everyone does.

  2. setcap — grant the ONE capability to the node BINARY, nothing else:
        sudo setcap 'cap_net_bind_service=+ep' $(readlink -f $(which node))
     Now node (as any user) can bind low ports. ⚠️ It applies to EVERY node
     process on the box, and is lost when node is upgraded/reinstalled.

  3. iptables REDIRECT — accept on 80, forward to 3000 in the kernel:
        sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3000
     App stays on 3000, unprivileged; the kernel rewrites the destination. (topic 29)

  (systemd socket activation is a 4th way: systemd binds the privileged socket as
   root and passes the ready fd to your unprivileged service — topic 22.)
```

---

## `ss` — The Modern Tool

`netstat` (from net-tools, topic 26) is deprecated and slow: it resolves names by default and it reads `/proc/net/tcp` by scanning text. `ss` ("socket statistics") pulls the socket table straight from the kernel over a netlink socket — **much faster on a busy box**, and it can filter server-side.

### Memorize this one: `ss -tulpn`

```
ss -tulpn
│  ││││││
│  │││││└── n = NUMERIC. Do NOT resolve ports→names or IPs→hostnames.
│  │││││       ★ This is also why it's FAST — no DNS, no /etc/services lookups.
│  ││││└── p = PROCESS. Show the PID/program that owns each socket.
│  ││││       ★ Needs root to see OTHER users' processes (else it's blank).
│  │││└── l = LISTENING sockets only (the servers). Omit l to see ESTABLISHED too.
│  ││└── u = UDP
│  │└── t = TCP
│  └──   (t and u are filters; with neither you get all socket families)
└── ss

Output:
Netid State  Recv-Q Send-Q  Local Address:Port  Peer Address:Port  Process
tcp   LISTEN 0      511      0.0.0.0:3000        0.0.0.0:*          users:(("node",pid=8231,fd=23))
tcp   LISTEN 0      4096     127.0.0.1:5432      0.0.0.0:*          users:(("postgres",pid=1120,fd=6))
tcp   LISTEN 0      511      0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=990,fd=8))
udp   UNCONN 0      0        127.0.0.53%lo:53    0.0.0.0:*          users:(("systemd-resolve"...))
│     │      │      │        │               │  │                  │
│     │      │      │        │               │  │                  └── program, PID, and
│     │      │      │        │               │  │                      the FD (topic 19!)
│     │      │      │        │               │  └── on a LISTEN socket, peer is always *:*
│     │      │      │        │               └── port
│     │      │      │        └── ★ 0.0.0.0 = all interfaces (reachable).
│     │      │      │             127.0.0.1 = loopback ONLY (topic 26 bug!). See Postgres ↑
│     │      │      └── (see the Recv-Q/Send-Q note — it's special on LISTEN sockets)
│     │      └── Send-Q
│     └── Recv-Q
└── socket family
```

### Recv-Q / Send-Q — the meaning FLIPS depending on the socket's state

This is the most under-appreciated diagnostic in the whole toolkit:

```
┌──────────────┬──────────────────────────────┬──────────────────────────────────┐
│              │ On a LISTENING socket         │ On an ESTABLISHED socket          │
├──────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Recv-Q       │ ★ # of established connections│ bytes received by the kernel that │
│              │   sitting in the ACCEPT QUEUE │ your app has NOT yet read()        │
│              │   that your app has NOT yet   │ (app is slow to consume)          │
│              │   accept()-ed.                │                                   │
│              │                               │                                   │
│              │ ★★ A PERSISTENTLY HIGH Recv-Q │                                   │
│              │    HERE MEANS YOUR EVENT LOOP  │                                   │
│              │    IS BLOCKED and not calling  │                                   │
│              │    accept(). A gorgeous, direct│                                   │
│              │    Node diagnostic. ★★         │                                   │
├──────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Send-Q       │ the BACKLOG size (the max     │ bytes your app write()-d that the │
│              │ accept queue length = the     │ peer has NOT yet ACK'd (slow      │
│              │ `listen(backlog)` value, 511) │ client / congested link)          │
└──────────────┴──────────────────────────────┴──────────────────────────────────┘
```

```bash
# Watch the accept queue on your Node listener. If Recv-Q climbs and stays up while
# Send-Q (the backlog cap) is fixed, your event loop is stalled — a CPU-bound handler,
# a sync call, a giant JSON.parse — and connections are piling up unaccepted.
watch -n1 "ss -ltn 'sport = :3000'"
# State  Recv-Q Send-Q Local Address:Port
# LISTEN 128    511    0.0.0.0:3000          ← 128 waiting, event loop is behind!
```

### The other `ss` invocations worth memorizing

```bash
ss -tan                       # all TCP sockets, numeric (LISTEN + ESTABLISHED + TIME_WAIT...)
ss -tp state established      # only established, with owning process
ss -s                         # one-line SUMMARY: totals per state
# Total: 341
# TCP:   1893 (estab 210, closed 1620, orphaned 0, timewait 1601)
#                                                              ^^^^ at a glance

ss -tn state time-wait | wc -l          # count TIME_WAIT (is it actually a problem?)
ss -tn state close-wait                 # ★ any CLOSE_WAIT? whose fds are leaking?
ss -lntp 'sport = :3000'                # who is LISTENING on 3000 (numeric, w/ process)
ss -tnp 'dst 10.0.0.30'                 # all connections TO the DB host
ss -tnp 'dport = :5432'                 # all connections to ANY host's port 5432
ss -o state established '( dport = :443 or sport = :443 )'   # -o shows timers
ss -tie                                 # -i = TCP internals: rtt, cwnd, retrans, mss
```

### `netstat` equivalents (for old boxes with no `ss`)

| `ss` | `netstat` |
|---|---|
| `ss -tulpn` | `netstat -tulpn` |
| `ss -tan` | `netstat -an -t` |
| `ss -tp state established` | `netstat -tp \| grep ESTABLISHED` |
| `ss -s` | `netstat -s` (protocol stats — different, but useful) |

---

## `lsof` and `fuser` — the fd-centric view

`ss` thinks in sockets. `lsof` thinks in **open files**, and a socket is a file (topic 03/19), so `lsof -i` shows you network sockets *with the owning process front and center*. **On macOS there is no `ss`** — `lsof` and `netstat` are what you use locally.

```bash
lsof -i :3000                 # everything touching port 3000 (listeners AND connections)
lsof -i -P -n                 # ALL network sockets. -P = numeric ports, -n = numeric hosts
#                               (both flags = don't do DNS/service lookups = fast)
lsof -iTCP -sTCP:LISTEN       # only TCP sockets in the LISTEN state — "what's serving?"
lsof -iTCP:3000 -sTCP:ESTABLISHED   # established connections on 3000
lsof -p 8231                  # every fd this PID holds (sockets, files, pipes — topic 19)

# fuser — "which process is USING this port?" Terse, scriptable.
fuser -n tcp 3000             # 3000/tcp:  8231
fuser -k -n tcp 3000          # ★ -k = KILL whatever holds it. Powerful and dangerous.
#                               (identify with ps BEFORE you -k. See the runbook below.)
```

```
lsof -iTCP -sTCP:LISTEN -P -n
│    │    │ │           │  │
│    │    │ │           │  └── -n = no hostname resolution
│    │    │ │           └── -P = no port-name resolution (show 3000, not "hbci")
│    │    │ └── -s TCP:LISTEN = filter by TCP state = LISTEN
│    │    └── -i TCP = TCP sockets only
│    └── -i = internet (network) sockets  (compare: no -i = regular files, topic 19)
└── lsof = "LiSt Open Files"
```

---

## UNIX DOMAIN SOCKETS — a socket in the filesystem

Not everything that uses `socket()` goes over the network. A **Unix domain socket** (AF_UNIX) is an endpoint identified by a **filesystem path** instead of an IP:port. Same `socket()`/`bind()`/`listen()`/`accept()` dance — but `bind()` takes a path like `/var/run/app.sock`, no TCP/IP stack is involved, and **its access control is filesystem permissions** (topic 05).

```
   TCP loopback socket                        Unix domain socket
   ───────────────────                        ───────────────────
   nginx ──┐                                  nginx ──┐
           │ connect 127.0.0.1:3000                   │ connect /run/app.sock
           ▼                                          ▼
   [ full TCP/IP stack ]                       [ NOTHING — it's a kernel
   • 3-way handshake                             memory copy between two fds ]
   • checksums                                 • no handshake
   • TCP state machine                         • no checksums
   • routing, port lookup                      • no ports, no routing
   • TIME_WAIT accumulation                    • no TIME_WAIT
           │                                          │
           ▼ (measurably slower)                      ▼ (faster, lower latency)
        your Node app                             your Node app
```

Because the socket is a path, it obeys file permissions — that IS its auth boundary:

```bash
srwxr-xr-x 1 root docker 0 Jul 12 09:00 /var/run/docker.sock
│                    │
│                    └── group `docker` can read/write it → can control the daemon
└── "s" = it's a socket (topic 03). Not a regular file.
```

Real Unix sockets you will meet:

| Path | Service | Note |
|---|---|---|
| `/var/run/docker.sock` | the Docker daemon's API | ★ see the warning below |
| `/var/run/postgresql/.s.PGSQL.5432` | Postgres local connections | `psql` uses this when you omit `-h` — faster than TCP |
| `/run/php/php8.2-fpm.sock` | PHP-FPM ← nginx | |
| `/run/systemd/...`, `/run/dbus/...` | system plumbing | |
| your own `/run/app.sock` | **nginx → Node over a Unix socket** | skip TCP for a real latency win |

**`/var/run/docker.sock` — the security landmine (topic 35):** the Docker daemon runs as **root**. Anyone who can write this socket can tell the daemon to `docker run -v /:/host` and read/modify the entire host filesystem as root. Therefore **mounting `/var/run/docker.sock` into a container is effectively granting that container root on the host.** People do it constantly for CI runners and "Docker-in-Docker" and don't realize they've dissolved the container boundary entirely.

```bash
ss -xlp                       # list LISTENING Unix domain sockets, with process
# u_str LISTEN 0 4096 /run/app.sock 12345 * 0 users:(("node",pid=8231,fd=19))
#       │                │                              └── still just an fd (topic 19)
#       │                └── the filesystem path IS the address
#       └── "u_str" = Unix stream socket
ss -xp                        # all Unix sockets (connected too)
lsof -U                       # Unix domain sockets via lsof
ls -l /var/run/docker.sock    # inspect the PERMISSIONS — that's the whole ACL
```

**Nginx → Node over a Unix socket** (the latency win):

```javascript
// Node: listen on a path instead of a port
const server = http.createServer(handler);
server.listen('/run/app.sock', () => {
  fs.chmodSync('/run/app.sock', 0o660);   // permissions = the access boundary
});
```
```nginx
# nginx: proxy to the socket instead of 127.0.0.1:3000
upstream app { server unix:/run/app.sock; }
server { location / { proxy_pass http://app; } }
```
No port, no TCP overhead, no `TIME_WAIT` churn between nginx and Node, and the socket can't be reached from off-box at all — it's not even on the network.

---

## Example 1 — Basic

```bash
# Start a server on 3000
node -e "require('http').createServer((_,r)=>r.end('ok')).listen(3000,'0.0.0.0')" &

# What's listening? (the command you'll type forever)
ss -tulpn | grep :3000
# tcp LISTEN 0 511 0.0.0.0:3000 0.0.0.0:*  users:(("node",pid=9001,fd=19))

# Make some connections and watch them become ESTABLISHED sockets
for i in $(seq 1 3); do curl -s localhost:3000 >/dev/null & done
ss -tn state established '( sport = :3000 )'
# State  Recv-Q Send-Q  Local Address:Port     Peer Address:Port
# ESTAB  0      0       127.0.0.1:3000         127.0.0.1:51002
# ESTAB  0      0       127.0.0.1:3000         127.0.0.1:51004
#                                    ^^^^ SAME local port, DIFFERENT peer ports.
#                                    Distinct 4-tuples. This is how one port serves many.

# The listening socket is a separate object from the connected ones
ss -tanp '( sport = :3000 )'          # you'll see 1 LISTEN + N ESTAB, all "port 3000"

# Prove the socket is a file descriptor (topic 19)
ls -l /proc/9001/fd | grep socket

# lsof view of the same thing
lsof -i :3000 -P -n

# Summary of everything
ss -s

kill %1
```

---

## Example 2 — Production Scenario (THE #1 one): `EADDRINUSE`

**Deploy fails. CI logs:**

```
Error: listen EADDRINUSE: address already in use 0.0.0.0:3000
    at Server.setupListenHandle [as _listen2] (node:net:1897:16)
    at listenInCluster (node:net:1945:12)
```

`EADDRINUSE` = `bind()` was refused because something already owns `0.0.0.0:3000`. **Do NOT reflexively `kill -9`.** Find out *what* it is first — it might be the very instance you just deployed, or the old one that should die, or something unrelated.

```bash
# ── 1. WHO owns the port? (-l listening, -n numeric, -t tcp, -p process)
ss -tulpn | grep :3000
# tcp LISTEN 0 511 0.0.0.0:3000 0.0.0.0:*  users:(("node",pid=8421,fd=23))
#                                                          ^^^^^^^^ the PID

# ── 2. ★ IDENTIFY IT BEFORE YOU TOUCH IT. This is the step people skip and regret.
ps -p 8421 -o pid,ppid,user,etime,cmd
#   PID  PPID USER   ELAPSED CMD
#  8421     1 deploy 2-04:11 node /srv/app/current/server.js
#            ^^^^                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#            PPID 1 → orphaned; adopted by init. A LEFTOVER from a previous deploy
#            whose parent (pm2? the old shell?) died. ELAPSED 2 days → it's stale.
#            → Safe to stop. If ppid were your CURRENT deploy's pid, you'd have a
#              double-listen bug in your OWN launch script instead.

# ── 3. Stop it GRACEFULLY first — SIGTERM (topic 17), let it close sockets cleanly
kill 8421                # this is SIGTERM (signal 15), NOT -9
sleep 2

# ── 4. VERIFY it's actually gone before restarting
ss -tulpn | grep :3000 || echo "port is free"
# port is free

# ── 5. Only if it REFUSED to die (ignored SIGTERM), escalate — last resort (topic 17)
#      kill -9 8421          # SIGKILL — no cleanup, no graceful shutdown. Avoid unless stuck.

# ── 6. Restart the deploy
systemctl restart shipfast-api      # (topic 22)
```

### The OTHER cause of `EADDRINUSE`: a `TIME_WAIT` ghost

Sometimes the old process is already gone, yet `bind()` STILL fails:

```bash
ss -tan state time-wait '( sport = :3000 )'
# State     Recv-Q Send-Q Local Address:Port  Peer Address:Port
# TIME-WAIT 0      0      0.0.0.0:3000        198.51.100.5:51002
#                         ^^^^^^^^^^^^^^^ the tuple is still reserved for ~60s
```
The previous process closed first, so the socket lingers in `TIME_WAIT`, and the *same* local address is briefly un-bindable. **`SO_REUSEADDR` is the fix — and Node sets it by default**, so pure-Node servers rarely hit this. If you do (e.g. a raw socket without the option), set it. Do **not** reach for `net.ipv4.tcp_tw_recycle` (removed, harmful).

### The Docker version

```bash
# EADDRINUSE on the HOST when publishing a port:
# Error: ... Bind for 0.0.0.0:8080 failed: port is already allocated
docker ps --format '{{.Names}}\t{{.Ports}}' | grep 8080
# old-api   0.0.0.0:8080->3000/tcp        ← a previous container still holds it
docker stop old-api      # or `docker rm -f old-api`
```

---

## Common Mistakes

### Mistake 1 — Panicking over thousands of `TIME_WAIT`

**Wrong:** `ss -s` shows `timewait 24000`. "The server is leaking connections!" You set `net.ipv4.tcp_tw_recycle=1` from a 2011 blog post.

**Root cause:** `TIME_WAIT` is **correct behavior on the side that closes first** — normal and healthy for a busy HTTP client or proxy (nginx, an API gateway, a service making many short outbound calls). Each closed connection parks its tuple for ~60s so a stray delayed packet can't poison a new connection reusing that tuple. And `tcp_tw_recycle` didn't just "clean them up" — it dropped connections from clients behind NAT (many real users share one NAT IP, and it rejected their out-of-order timestamps). **It was removed from the kernel in 4.12.**

**Diagnose:** Is it actually causing harm? Only if you're **exhausting ephemeral ports** (below). Check: `ss -s`, and `cat /proc/sys/net/ipv4/ip_local_port_range`.

**Fix (only if truly exhausting ports):** reduce how many connections you *close* — use **keep-alive / connection pooling** so connections are reused, not opened-and-closed per request. For high-volume outbound, `net.ipv4.tcp_tw_reuse=1` safely lets the kernel reuse `TIME_WAIT` tuples for **new outbound** connections. Never `tcp_tw_recycle`.

**Prevent:** Keep-alive everywhere. `TIME_WAIT` is a symptom of churning connections; pooling removes the churn.

---

### Mistake 2 — Ignoring a growing `CLOSE_WAIT` pile

**Wrong:** Memory and fd count climb over days; you restart the app nightly as a "fix" and never look at socket states.

**Root cause:** `CLOSE_WAIT` means **the remote closed and your app never called `close()`**. The classic culprits: an HTTP client to an upstream where you read the response but never destroy the socket; a DB/redis client created per-request and never released; a stream you `.on('data')` but never `.destroy()`. Each stuck socket holds a file descriptor. It's a straight-line march to `EMFILE: too many open files` (topic 19) and, eventually, a wedged server.

**Diagnose:**
```bash
ss -tan state close-wait | head            # are they piling up?
ss -tanp state close-wait '( dport = :5432 )'   # ★ WHICH upstream? whose fds?
NODE_PID=$(pgrep -n node)
ls /proc/$NODE_PID/fd | wc -l              # fd count trending up? (topic 19)
watch -n5 "ss -tan state close-wait | wc -l"   # is it monotonic? then it's a leak.
```

**Fix:** Find the code path that doesn't close. `CLOSE_WAIT` is on **your** side, so the missing `close()`/`.destroy()`/`.release()` is in **your** code. The remote port in `ss` output tells you which client (the DB? an API? redis?).

**Prevent:** Use pooled clients (`pg.Pool`, keep-alive agents) that manage lifecycle for you. Add an fd-count metric to monitoring and alert on the *trend*.

---

### Mistake 3 — Running Node as root to bind port 80

**Wrong:** `sudo node server.js` so it can `listen(80)`.

**Root cause:** Ports < 1024 need `CAP_NET_BIND_SERVICE`, so people run the whole app as root. Now every dependency in `node_modules` — hundreds of packages you didn't audit — runs with the power to read `/etc/shadow`, rewrite system binaries, and own the box. A single RCE in one transitive dependency = total compromise. You bought a port number with your entire security posture.

**Diagnose:** `ps -o user= -p $(pgrep -n node)` shows `root`. That's the smell.

**Fix:** Put nginx (or a cloud load balancer) on 80/443 and proxy to Node on `127.0.0.1:3000`, running as an unprivileged `deploy` user (topic 32). Or `setcap 'cap_net_bind_service=+ep'` on the node binary. Or an `iptables ... REDIRECT --to-port 3000` (topic 29). Never run the app as root for a port number.

**Prevent:** Bake "app runs as non-root, proxy handles :443" into your deploy template and your Dockerfile (`USER node`).

---

### Mistake 4 — Ephemeral port exhaustion from no connection pooling

**Wrong:** A service opens a fresh outbound HTTP/DB connection **per request** to the same upstream, at high volume. Under load: `connect EADDRNOTAVAIL` or intermittent stalls.

**Root cause:** When your app is the *client*, the kernel assigns a **source** port from the ephemeral range (`/proc/sys/net/ipv4/ip_local_port_range`, ~32768–60999 ≈ 28k). To one destination `(dst IP, dst port)`, the source IP and dest are fixed, so the **only** variable in the 4-tuple is your source port. Open connections faster than they leave `TIME_WAIT` and you run out of source ports for that destination. ~28k connections in a minute, none reused, and you're wedged.

**Diagnose:**
```bash
cat /proc/sys/net/ipv4/ip_local_port_range          # 32768   60999
ss -tan state time-wait '( dport = :5432 )' | wc -l  # burning ports on the DB?
ss -s                                                # huge timewait + estab counts
```

**Fix:** **Connection pooling / keep-alive.** Reuse a small fixed set of connections instead of opening one per request (`pg.Pool`, an HTTP agent with `keepAlive:true`, a persistent redis client). This slashes the port churn at the root. Secondary knobs: widen the range, or `tcp_tw_reuse=1` for outbound.

**Prevent:** Never create a network client per request. Create it once, pool it, reuse it. This is the same lesson as the DNS chapter (topic 27) — Node re-resolves *and* re-connects per request unless you pool.

---

### Mistake 5 — App is "up" but unreachable: bound to loopback (topic 26 callback)

**Wrong:** `pm2 list` says `online`, `curl localhost:3000` works on the box, external clients and the load balancer get connection refused. You open firewall ports for an hour.

**Root cause:** The socket is bound to `127.0.0.1:3000`, not `0.0.0.0:3000`. `127.0.0.1` is the loopback interface — packets to it never leave the kernel (topic 26). A request arriving on the real NIC finds no matching socket and the kernel sends RST → `ECONNREFUSED`. The firewall was never involved. The identical bug bites inside a container, where `-p` forwards to the container's `eth0`, not its loopback.

**Diagnose:** the single most valuable line in this doc:
```bash
ss -tulpn | grep :3000
# tcp LISTEN 0 511 127.0.0.1:3000 0.0.0.0:*  users:(("node",...))
#                  ^^^^^^^^^ ← loopback. Unreachable from outside. Done.
```

**Fix:** `server.listen(3000, '0.0.0.0')` (or `HOST=0.0.0.0`).

**Prevent:** Put `ss -tulpn | grep :3000` in your deploy smoke test and assert on `0.0.0.0`, not `127.0.0.1`.

---

## Hands-On Proof

```bash
# PROVE IT: server.listen() = socket() + bind() + listen(). Watch the syscalls (topic 01)
strace -f -e trace=socket,setsockopt,bind,listen,accept \
  node -e "require('http').createServer(()=>{}).listen(3000,'0.0.0.0')" &
sleep 1
# socket(AF_INET, SOCK_STREAM|SOCK_NONBLOCK|SOCK_CLOEXEC, 0) = 19
# setsockopt(19, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0      ← Node sets it FOR you
# bind(19, {sa_family=AF_INET, sin_port=htons(3000),
#           sin_addr=inet_addr("0.0.0.0")}, 16) = 0
# listen(19, 511) = 0                                       ← default backlog 511
kill %1

# PROVE IT: a socket is a file descriptor (topic 19)
node -e "require('http').createServer((_,r)=>r.end()).listen(3000)" &
P=$!; sleep 1
ls -l /proc/$P/fd | grep 'socket:'
# 19 -> socket:[54821773]       ← same fd machinery as any open file

# PROVE IT: accept() returns a NEW fd per connection
#   Count fds, open a hanging connection, count again — it goes UP by one.
before=$(ls /proc/$P/fd | wc -l)
exec 7<>/dev/tcp/127.0.0.1/3000        # bash opens a raw TCP connection on fd 7
after=$(ls /proc/$P/fd | wc -l)
echo "server fds: $before -> $after"   # the server gained a connected-socket fd
exec 7>&-                              # close it
kill $P

# PROVE IT: one listening port, many distinct 4-tuples
node -e "require('http').createServer((_,r)=>setTimeout(()=>r.end(),5000)).listen(3000)" &
P=$!; sleep 1
for i in 1 2 3 4 5; do curl -s localhost:3000 & done
ss -tn '( sport = :3000 )'
#   1 LISTEN + several ESTAB rows, all Local=...:3000, each a different Peer port
kill $P 2>/dev/null

# PROVE IT: the ephemeral range and privileged boundary are just kernel settings
cat /proc/sys/net/ipv4/ip_local_port_range     # 32768   60999
# and: try to bind :80 as a normal user → EACCES (permission denied) at bind()
node -e "require('http').createServer(()=>{}).listen(80)" 2>&1 | head -1
# Error: listen EACCES: permission denied 0.0.0.0:80        ← the privileged-port rule

# PROVE IT: Unix domain sockets exist and use file permissions
ss -xlp | head
ls -l /var/run/docker.sock 2>/dev/null   # srwxr-xr-x ... "s" = socket, then perms = the ACL

# PROVE IT: Recv-Q on a listener = the accept queue
#   (block the event loop and watch Recv-Q climb while requests queue)
node -e "const h=require('http');h.createServer((_,r)=>{const t=Date.now()+3000;while(Date.now()<t){};r.end()}).listen(3000)" &
P=$!; sleep 1
for i in $(seq 1 20); do curl -s localhost:3000 & done
ss -ltn 'sport = :3000'
# LISTEN 15 511 0.0.0.0:3000     ← Recv-Q=15: fifteen connections waiting, loop is busy
kill $P 2>/dev/null
```

---

## Practice Exercises

### Exercise 1 — Easy

1. Start a Node server on port 3000 bound to `0.0.0.0`. With `ss -tulpn`, find it and read off: the state, the local address, the backlog (Send-Q), the PID, and the fd number.
2. Prove the fd is real: `ls -l /proc/<PID>/fd` and find the `socket:[...]` entry matching it.
3. Rebind the server to `127.0.0.1` instead. Re-run `ss -tulpn`. What one column changed, and what does it mean for reachability (topic 26)?
4. Run `ss -s`. How many sockets are in each state right now?

### Exercise 2 — Medium

Demonstrate the 4-tuple and the difference between listening and connected sockets.

```bash
# 1. Start a server whose handler sleeps 10s (so connections stay ESTABLISHED):
node -e "require('http').createServer((_,r)=>setTimeout(()=>r.end('ok'),10000)).listen(3000,'0.0.0.0')" &
# 2. Fire 5 concurrent curls at it.
# 3. Run: ss -tanp '( sport = :3000 )'
#    - Identify the ONE listening socket and the FIVE connected sockets.
#    - For the connected ones: what is IDENTICAL across all five? What DIFFERS?
#    - Explain, in one sentence, why they don't collide despite sharing port 3000.
# 4. Run: ss -tn state established '( sport = :3000 )' | wc -l   — does it match?
# 5. Count the server's fds before and during the load (ls /proc/<PID>/fd | wc -l).
#    Explain the increase in terms of accept().
```

### Exercise 3 — Hard (Production Simulation)

**Reproduce and triage all three flagship failures.**

```bash
# ── FAILURE A: EADDRINUSE ──────────────────────────────────────────────
#   1. Start server #1 on 3000. Start server #2 on 3000 → capture the EADDRINUSE error.
#   2. Run the FULL runbook: ss -tulpn to get the PID → ps -p PID -o pid,ppid,user,etime,cmd
#      to identify it → kill (SIGTERM) → verify it's gone → restart.
#   3. WITHOUT killing server #1, explain how you'd know from `ps` whether the holder is
#      a stale orphan (ppid 1) or your own just-launched double-listen.

# ── FAILURE B: CLOSE_WAIT leak ─────────────────────────────────────────
#   Write a tiny "leaky" server: on each request it opens an outbound connection to
#   some host (e.g. connects to 127.0.0.1:9999 where a peer immediately closes) and
#   NEVER closes its own end. Hit it 50 times.
#   1. Show the CLOSE_WAIT pile with `ss -tan state close-wait`.
#   2. Show the fd count climbing: `ls /proc/<PID>/fd | wc -l` before and after.
#   3. State, in one sentence, why CLOSE_WAIT proves the bug is in YOUR code and not
#      the peer's. Then fix the server (close the outbound socket) and prove the pile
#      is gone.

# ── FAILURE C: loopback bind ───────────────────────────────────────────
#   1. Bind a server to 127.0.0.1:3000.
#   2. curl localhost:3000 (works) and curl <your-LAN-IP>:3000 (fails).
#   3. Use ONE ss command to explain the failure. Fix by binding 0.0.0.0 and prove both work.

# Write all three as a single annotated script. Every command gets a comment saying
# what you EXPECT to see and what it PROVES.
```

---

## Mental Model Checkpoint

1. **A socket is a ____.** Complete the sentence, and explain why that makes a connection leak, `EMFILE`, `lsof`, and `/proc/PID/fd` all the same underlying story (topics 03/19).
2. **Walk the server-side lifecycle syscall by syscall** (`socket` → … → `close`), and name which Node API call maps to which syscall(s).
3. **How does one server on one port serve 10,000 simultaneous clients without the port "running out"?** Answer using the 4-tuple.
4. **You see 20,000 sockets in `TIME_WAIT` and 300 in `CLOSE_WAIT`.** Which one is (probably) fine, which one is (probably) your bug, and why? What's the fix for each?
5. **Why can't your Node app bind port 80 as a normal user, and what are the three correct ways to serve :80/:443 without running Node as root?**
6. **On a LISTENING socket, what do `Recv-Q` and `Send-Q` mean — and what does a persistently high `Recv-Q` tell you about your Node event loop?**
7. **`EADDRINUSE` on deploy.** Give the exact command sequence you run, and explain why `ps -p PID` comes *before* `kill`, and why you try `kill` before `kill -9`.

---

## Quick Reference Card

| Command | What it does | Key flags |
|---|---|---|
| **`ss -tulpn`** | **What's listening** (the one to memorize) | `t`=TCP `u`=UDP `l`=listen `p`=process(root) `n`=numeric |
| `ss -tan` | All TCP sockets, numeric | add `state established` / `time-wait` / `close-wait` |
| `ss -tp state established` | Established connections + owning process | |
| `ss -s` | Summary counts per state | |
| `ss -ltn 'sport = :3000'` | Who's listening on a specific port | `sport`/`dport`/`dst`/`src` filters |
| `ss -tn state close-wait` | **Find fd/connection leaks (your bug)** | |
| `ss -tn state time-wait \| wc -l` | Count TIME_WAIT (usually fine) | |
| `ss -tie` | TCP internals: rtt, cwnd, retransmits | |
| `ss -xlp` | Listening Unix domain sockets + process | |
| `lsof -i :3000` | Everything touching a port (macOS-friendly) | `-P -n` numeric+fast |
| `lsof -iTCP -sTCP:LISTEN` | All TCP listeners | `-P -n` |
| `lsof -p PID` | All fds a process holds (topic 19) | |
| `fuser -n tcp 3000` | Which PID uses a port | `-k` = **kill it** (careful) |
| `ps -p PID -o pid,ppid,user,etime,cmd` | **Identify a process BEFORE killing** | |
| `netstat -tulpn` | Legacy `ss` (old boxes) | slower; resolves names by default |
| `cat /proc/sys/net/ipv4/ip_local_port_range` | Ephemeral port range | |
| `/proc/PID/fd/` | The socket-as-file proof (topic 19) | |

---

## When Would I Use This at Work?

### Scenario 1: Deploy fails with `EADDRINUSE`
`ss -tulpn | grep :3000` gives you the PID, `ps -p <PID> -o ppid,user,etime,cmd` tells you it's a stale orphan from last night's crashed deploy (parent is init, running 14 hours), you `kill` it gracefully, verify the port is free, and re-run the deploy — all in under a minute, and without `kill -9`-ing something important. Knowing to identify *before* killing is what keeps you from taking down the wrong service.

### Scenario 2: The server is "up" but the load balancer marks it unhealthy
One `ss -tulpn | grep :3000` shows `127.0.0.1:3000` instead of `0.0.0.0:3000`. You've found in 10 seconds what would otherwise be an hour of blaming security groups and firewalls (topic 26). Fix: bind `0.0.0.0` in the systemd unit's environment.

### Scenario 3: fds slowly climbing, nightly restarts "fixing" it
`ss -tan state close-wait` shows a growing pile all pointing at your Postgres host's `:5432`. That tells you your DB client isn't releasing connections — a leak in *your* code, not the database. You switch to a pooled client (`pg.Pool`), watch `ls /proc/<PID>/fd | wc -l` go flat, and delete the nightly-restart cron job. You diagnosed a leak that looked like a memory problem by reading socket *state*.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 29 — Firewalls | You now know what's listening; firewalls decide who's *allowed to reach* it. Docker's `-p` DNAT (topic 26) and iptables `REDIRECT` for privileged ports both live there. |
| **Next** | 30 — curl and wget | The client side of every socket in this doc: `curl` does `socket()`+`connect()` and you can watch the handshake you learned here. |
| **Builds on** | 26 — Network Interfaces | A socket binds to an interface's address. `0.0.0.0` vs `127.0.0.1` is the same loopback-vs-all-interfaces distinction, and each container's own port space is *why* two can bind :3000. |
| **Builds on** | 19 — File Descriptors | ★ A socket **is** an fd. Every connection consumes one; `EMFILE`, `ulimit -n`, `/proc/PID/fd`, and `lsof` all carry straight over. |
| **Builds on** | 03 — Everything Is a File | Network sockets and Unix domain sockets are files under one abstraction; a Unix socket literally has a path and file permissions. |
| **Builds on** | 17 — Process Management | `SIGTERM` before `SIGKILL`, identifying a PID before killing it — the `EADDRINUSE` runbook is applied process management. |
| **Builds on** | 01 — What Linux Actually Is | `socket()`/`bind()`/`listen()`/`accept()` are syscalls across the Ring 3→Ring 0 boundary; the kernel owns the socket table and the TCP state machine. |
| **Used by** | 33 — Node in Production | Bind address, backlog, `LimitNOFILE`, keep-alive/pooling, Unix-socket-to-nginx — every production-Node decision traces back to this doc. |
| **Used by** | 35 — Security Basics | `ss -tulpn` is *the* audit command: "what is this box exposing, and to whom?" Binding to loopback and the docker.sock=root warning are hardening fundamentals. |
