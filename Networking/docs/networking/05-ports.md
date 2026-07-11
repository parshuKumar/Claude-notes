# 05 — Ports

> **Phase 1 — Topic 5 of 5**
> Prerequisites: `01-how-the-internet-works.md`, `02-osi-model.md`, `03-tcp-ip-model.md`, `04-ip-addresses.md`

---

## ELI5 — The Simple Analogy

In Topic 04 you learned: **the IP address is the street address of the building.** `142.250.80.46` gets a letter to the right building, and nothing more.

But a building isn't one occupant — it's an apartment complex. Hundreds of apartments, each with its own life going on, all sharing one street address. If a letter just says "142.250.80.46" and nothing else, the mail carrier drops it in the lobby and... now what? Which apartment opens it?

That's what a **port number** solves. The port is the **apartment/mailbox number** inside the building.

```
Street address (IP):        142.250.80.46          ← gets mail to the building
Apartment number (Port):    :443                    ← gets mail to the right door

Full address = "142.250.80.46, Apartment 443"      ← this is a SOCKET
```

One building (one server, one IP address) can run dozens of independent "apartments" (services) at the same time, completely unaware of each other:

```
142.250.80.46
├── Apartment 22   → sshd (you SSH in to manage the server)
├── Apartment 80   → nginx (serving your website over plain HTTP)
├── Apartment 443  → nginx (serving your website over HTTPS)
├── Apartment 3000 → your Node.js API (dev server)
├── Apartment 5432 → PostgreSQL (your database)
└── Apartment 6379 → Redis (your cache)
```

Six completely separate services, one IP address, zero confusion — because every packet carries both a street address (IP) *and* an apartment number (port). The building's front desk (your OS kernel) reads the apartment number and delivers the mail to exactly the right door.

---

## Where This Lives in the Network Stack

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, WebSocket, DNS, SMTP)           │
│  Layer 6 — Presentation  (TLS/SSL, encoding, compression)       │
│  Layer 5 — Session       (session management)                   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4 — Transport     (TCP, UDP — PORTS LIVE HERE)     ◄──── │  ← you are here
├─────────────────────────────────────────────────────────────────┤
│  Layer 3 — Network       (IP — addresses live here)             │
│  Layer 2 — Data Link     (Ethernet, Wi-Fi — MAC addresses)      │
│  Layer 1 — Physical      (cables, fiber, radio waves, photons)  │
└─────────────────────────────────────────────────────────────────┘
```

Ports are a **Layer 4 (Transport)** concept. Both transport protocols carry a port number:

- **TCP** (Transmission Control Protocol) — reliable, ordered, connection-based. Uses ports. Covered fully in Topic 08.
- **UDP** (User Datagram Protocol) — unreliable, unordered, connectionless. *Also* uses ports. Covered fully in Topic 07.

This matters: **TCP port 443 and UDP port 443 are two completely different, independent "mailboxes."** A process listening on TCP/443 knows nothing about traffic arriving on UDP/443, even though the number is identical. (This is exactly why HTTP/3 — which runs over UDP via QUIC — can use port 443 alongside HTTPS over TCP without conflict. More in Topic 12.)

Layer 3 (IP) gets the packet to the right *building*. Layer 4 (ports) gets it to the right *apartment*. Neither layer knows or cares what's inside the envelope — that's Layer 7's job.

---

## What Is This?

A **port** is a **16-bit unsigned integer** — meaning it can be any value from **0 to 65535** (2¹⁶ = 65,536 possible values) — that identifies a specific process or service on a machine.

Why does a single IP address need multiple ports? Because a server usually runs many independent network services simultaneously (web server, database, cache, SSH daemon), and without ports, an incoming packet addressed to an IP would have no way to know which of those services it's meant for.

The 65,536 possible port numbers are split into three official ranges, defined by IANA (Internet Assigned Numbers Authority):

```
┌───────────────────────────────────────────────────────────────────────┐
│ 0 ─────────────── 1023   WELL-KNOWN / SYSTEM PORTS                    │
│   Reserved for well-established protocols (HTTP, SSH, DNS...).       │
│   On Unix/Linux, binding to one of these requires root privilege     │
│   or the CAP_NET_BIND_SERVICE capability.                            │
├───────────────────────────────────────────────────────────────────────┤
│ 1024 ────────── 49151   REGISTERED / USER PORTS                       │
│   Assigned by IANA to specific applications on request               │
│   (3306 = MySQL, 5432 = PostgreSQL, 6379 = Redis...).                │
│   Anyone can bind to these — no special privilege required.          │
├───────────────────────────────────────────────────────────────────────┤
│ 49152 ───────── 65535   DYNAMIC / PRIVATE / EPHEMERAL PORTS (IANA)    │
│   Not assigned to anyone. The OS hands these out temporarily         │
│   as SOURCE ports for outbound client connections.                   │
└───────────────────────────────────────────────────────────────────────┘
```

**The catch:** that bottom range is the *official* IANA recommendation, but Linux does not actually follow it by default. Check your own machine:

```bash
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768   60999
```

Linux's actual default ephemeral range is roughly **32768–60999**, not 49152–65535. This is a historical BSD-derived default that most distros never changed. It means on a stock Linux box, the ephemeral range actually *overlaps* part of the "registered" range (32768–49151). This rarely causes real problems, but it explains why you sometimes see a client-side connection using a source port like `34521` — that's a perfectly normal ephemeral port on Linux, even though IANA says ephemeral should start at 49152. (macOS/BSD tend to track the IANA range more closely.)

---

## Why Does It Matter for a Backend Developer?

You touch ports every single day, often without thinking about it:

- **Choosing a port for your service.** `app.listen(3000)` — why 3000 and not 80? (Because 80 is privileged, see below.)
- **`EADDRINUSE` errors.** The single most common local-dev annoyance: "why won't my server start?" — because something else already owns that port.
- **Privileged-port permission failures.** Trying to bind to port 80 or 443 directly as a non-root process fails with `EACCES`, and this bites people constantly in Docker and production deployments.
- **Understanding the client's ephemeral port.** Every outbound connection your Node.js process makes (to a database, to an external API) uses a *temporary* source port picked by the OS — understanding this is essential for reading `ss`/`netstat` output and diagnosing connection issues.
- **Port exhaustion under load.** A service that opens thousands of short-lived outbound connections per second (no connection pooling, no keep-alive) can run out of ephemeral ports for a given destination, causing intermittent `EADDRNOTAVAIL` or connection failures — a classic production incident.
- **Docker and Kubernetes port mapping.** `-p 8080:3000` in Docker, or `port` / `targetPort` / `nodePort` in a Kubernetes Service — these are three *different* port concepts that map onto each other, and mixing them up is one of the most common causes of "my service isn't reachable" tickets.

If you don't understand ports precisely, you will misdiagnose connection errors, misconfigure container networking, and eventually cause a production outage that traces back to "we ran out of ports."

---

## The Packet/Protocol Anatomy

Ports live inside the **TCP** and **UDP** headers — the first thing in both, before anything else.

### TCP Header (simplified)

```
┌─────────────────────────────┬─────────────────────────────┐
│ Source Port (16 bits)       │ Destination Port (16 bits)  │  ← THE PORTS
├─────────────────────────────┴─────────────────────────────┤
│ Sequence Number (32 bits)                                  │
├──────────────────────────────────────────────────────────┤
│ Acknowledgment Number (32 bits)                            │
├───────┬─────────┬────────────────────┬─────────────────────┤
│ Offset│ Reserved│ Flags (URG ACK PSH  │ Window Size (16 b)  │
│ (4b)  │  (6b)   │  RST SYN FIN)       │                     │
├─────────────────────────────┬─────────────────────────────┤
│ Checksum (16 bits)          │ Urgent Pointer (16 bits)     │
├──────────────────────────────────────────────────────────┤
│ Options (variable) │ Data (payload — e.g. your HTTP bytes) │
└──────────────────────────────────────────────────────────┘
```

### UDP Header (the whole thing — it's tiny)

```
┌─────────────────────────────┬─────────────────────────────┐
│ Source Port (16 bits)       │ Destination Port (16 bits)  │  ← THE PORTS
├─────────────────────────────┼─────────────────────────────┤
│ Length (16 bits)            │ Checksum (16 bits)          │
├─────────────────────────────┴─────────────────────────────┤
│ Data (payload)                                              │
└──────────────────────────────────────────────────────────┘
```

Notice: **every TCP segment and every UDP datagram carries exactly two port numbers** — a source port and a destination port — regardless of what's inside. This is true for every packet, in both directions, for the entire life of a connection.

### Socket = IP + Port

A **socket** is the combination of an IP address and a port number, written `IP:PORT`:

```
203.0.113.10:443     ← a socket: "this specific service on this specific machine"
```

A socket by itself identifies *one side* of a conversation. A full TCP connection needs **both** sides — which is why a connection is uniquely identified by a **4-tuple** (sometimes called a 5-tuple when you include the protocol):

```
4-TUPLE (uniquely identifies one TCP connection):
┌────────────────┬────────────┬─────────────────┬─────────────┐
│  Source IP      │ Source Port│  Destination IP  │ Dest. Port  │
├────────────────┼────────────┼─────────────────┼─────────────┤
│  203.0.113.5    │   54231    │  142.250.80.46   │    443      │
└────────────────┴────────────┴─────────────────┴─────────────┘

5-TUPLE adds the protocol (TCP vs UDP), since TCP/443 and UDP/443
are entirely separate, independent connections/sockets:
(TCP, 203.0.113.5:54231, 142.250.80.46:443)
```

This 4-tuple is the single most important idea in this whole topic — it's *how* one server port can serve thousands of simultaneous clients. Keep reading.

---

## How It Works — Step by Step

### Step 1 — A server binds and listens

```
Your Node.js process calls: server.listen(443)

1a. OS allocates a LISTENING socket bound to 0.0.0.0:443
    (0.0.0.0 = "accept on any network interface I have")
1b. This socket does NOT represent a connection.
    It's a doorbell — it just waits for incoming SYN packets.
1c. The kernel adds an entry to its socket table:
    Proto  Local Address     State
    tcp    0.0.0.0:443       LISTEN
```

### Step 2 — A client picks an ephemeral source port

```
Your browser (or another service's HTTP client) wants to connect.

2a. OS looks at its ephemeral range (e.g. Linux: 32768–60999)
2b. OS picks a currently-unused port for THIS destination, e.g. 51422
2c. Outbound SYN is built:
    [src: 198.51.100.5:51422]  →  [dst: 142.250.80.46:443]
```
The client almost never picks port 443 as its *own* port — it picks a random-ish ephemeral one. The server's port (443) is fixed and well-known; the client's port is temporary and essentially disposable.

### Step 3 — The kernel routes the incoming SYN to the listening socket

```
SYN arrives at 142.250.80.46:443.
The kernel checks: "Do I have a LISTEN socket on port 443?" → yes.
It hands the SYN to that listening socket to begin the 3-way handshake.
(Full handshake mechanics are Topic 08 — for now, just: connection begins.)
```

### Step 4 — A brand-new socket is created for THIS connection

```
Once the handshake completes, the kernel creates a NEW socket entry —
separate from the LISTEN socket — keyed by the full 4-tuple:

Proto  Local Address:Port   Remote Address:Port    State
tcp    142.250.80.46:443    198.51.100.5:51422     ESTABLISHED

The original LISTEN socket on :443 is untouched and keeps listening
for the NEXT client's SYN.
```

### Step 5 — Multiplexing: one port, thousands of simultaneous clients

```
Ten seconds later, your server on port 443 might have a socket table like:

┌────────────────┬───────┬──────────────────┬────────┬─────────────┐
│ Local IP        │ Lport │ Remote IP         │ Rport  │ State       │
├────────────────┼───────┼──────────────────┼────────┼─────────────┤
│ 142.250.80.46   │  443  │ 198.51.100.5      │ 51422  │ ESTABLISHED │
│ 142.250.80.46   │  443  │ 198.51.100.5      │ 51423  │ ESTABLISHED │ ← same client, 2nd tab
│ 142.250.80.46   │  443  │ 203.0.113.200     │ 33110  │ ESTABLISHED │
│ 142.250.80.46   │  443  │ 8.8.4.4           │ 60213  │ ESTABLISHED │
│ 142.250.80.46   │  443  │ 0.0.0.0           │   *    │ LISTEN      │
└────────────────┴───────┴──────────────────┴────────┴─────────────┘

All four active connections share the SAME local port (443).
The kernel distinguishes them ONLY by the full 4-tuple.
This is how a single port serves an unlimited number of clients —
the port doesn't identify the client; the 4-tuple does.
```

### Step 6 — The client side avoids collisions

```
If your Node.js API opens two outbound connections to the SAME
destination (e.g. two requests to the same downstream API at
203.0.113.9:443), the OS must give them DIFFERENT ephemeral
source ports — because the 4-tuple must be unique:

(you:51422 → 203.0.113.9:443)   ✓ ok
(you:51423 → 203.0.113.9:443)   ✓ ok — different source port, valid

But your OS CAN reuse the SAME ephemeral port for connections to
DIFFERENT destinations at the same time, because the 4-tuple is
still unique overall:

(you:51422 → 203.0.113.9:443)    ✓
(you:51422 → 8.8.4.4:443)        ✓ — same source port, different
                                     destination, still a unique 4-tuple
```
This last point is the key to understanding ephemeral port exhaustion (see Common Mistakes below): you don't run out of "ports" globally — you run out of *available 4-tuples for a specific destination*.

---

## Exact Syntax Breakdown

### `ss -tlnp`

```
$ ss -tlnp
State    Recv-Q  Send-Q  Local Address:Port    Peer Address:Port   Process
LISTEN   0       128     0.0.0.0:22            0.0.0.0:*           users:(("sshd",pid=612,fd=3))
LISTEN   0       511     127.0.0.1:5432         0.0.0.0:*           users:(("postgres",pid=1201,fd=5))
LISTEN   0       511     0.0.0.0:3000           0.0.0.0:*           users:(("node",pid=2044,fd=19))
```

```
ss   -t         -l          -n         -p
│    │          │           │          └── show the process (PID/name) that owns the socket
│    │          │           └── numeric: don't resolve hostnames or /etc/services port names
│    │          └── listening sockets only (not established connections)
│    └── tcp sockets only (add -u for udp)
```

Column by column:
| Column | Meaning |
|---|---|
| `State` | Socket state — `LISTEN` here. (Full state machine: TIME_WAIT, ESTABLISHED, etc. is Topic 24/08.) |
| `Recv-Q` | For a LISTEN socket: current length of the SYN backlog queue (pending connections not yet accepted) |
| `Send-Q` | For a LISTEN socket: the maximum backlog size configured (e.g. `511` is Node's default `listen()` backlog) |
| `Local Address:Port` | The IP and port this socket is bound to. `0.0.0.0` = all interfaces; `127.0.0.1` = loopback only |
| `Peer Address:Port` | `0.0.0.0:*` for a listening socket — there is no peer yet |
| `Process` | `users:(("name",pid=NNNN,fd=NN))` — which process owns this socket and which file descriptor |

**The line to memorize as a red flag:** `127.0.0.1:5432` means Postgres is *only* reachable from the same machine. `0.0.0.0:3000` means the Node app is reachable from any network interface, including the outside world. This exact distinction is what breaks Docker port mapping (see Example 2).

### `lsof -i :3000`

```
$ lsof -i :3000
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
node    2044  backend   19u  IPv6  0x3af9      0t0  TCP *:3000 (LISTEN)
```

| Column | Meaning |
|---|---|
| `COMMAND` | Process name (truncated to ~9 chars) |
| `PID` | Process ID — use this to `kill` it |
| `USER` | Which OS user owns the process |
| `FD` | File descriptor number and mode (`19u` = fd 19, read/write) |
| `TYPE` | Socket family — `IPv4` or `IPv6` |
| `NODE` | Protocol — `TCP` or `UDP` |
| `NAME` | `*:3000 (LISTEN)` — bound to all interfaces, port 3000, currently listening |

### `/etc/services`

This file is where `ss`, `lsof`, and other tools look up human-readable names for well-known ports (unless you pass `-n` for numeric output):

```
# service-name  port/protocol   aliases...
ssh             22/tcp
ssh             22/udp
smtp            25/tcp          mail
domain          53/tcp          # DNS
domain          53/udp
http            80/tcp          www www-http
https           443/tcp
postgresql      5432/tcp
```
It's just a static text mapping — editing it doesn't open or close anything, it only changes how tools *display* a port number.

---

## Example 1 — Basic: Start a Server and Find It

Start a trivial local server:

```bash
python3 -m http.server 8000
# Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

In another terminal, prove it's listening — two different ways:

```bash
$ ss -tlnp | grep 8000
LISTEN   0   5   0.0.0.0:8000   0.0.0.0:*   users:(("python3",pid=4821,fd=3))

$ lsof -i :8000
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
python3 4821  backend    3u  IPv4  0x91a2      0t0  TCP *:8000 (LISTEN)
```

Now hit it:

```bash
$ curl -v localhost:8000
*   Trying 127.0.0.1:8000...
* Connected to localhost (127.0.0.1) port 8000
> GET / HTTP/1.1
> Host: localhost:8000
>
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/3.11.4
<
<!DOCTYPE HTML>
<html>...directory listing...</html>
```

The full picture, end to end: `curl` asked the OS to connect to `127.0.0.1:8000`. The OS picked an ephemeral source port for `curl` (say `54876`), completed the TCP handshake against the process bound to `0.0.0.0:8000`, and the kernel matched the incoming SYN to that LISTEN socket purely by destination port number.

---

## Example 2 — Production Scenario: "Connection Refused" from a Docker Container

**The setup:** a Node.js API is meant to listen on port 3000 inside a container. You run:

```bash
docker run -p 8080:3000 my-api:latest
```

`-p 8080:3000` means: **map host port 8080 → container port 3000.** Requests to `localhost:8080` on your machine should be forwarded into the container's port 3000.

```bash
$ curl localhost:8080
curl: (7) Failed to connect to localhost port 8080: Connection refused
```

**Diagnosis — check what's actually listening *inside* the container:**

```bash
$ docker exec -it <container_id> sh
/app # ss -tlnp
State   Local Address:Port     Peer Address:Port   Process
LISTEN  127.0.0.1:3000          0.0.0.0:*           users:(("node",pid=1,fd=19))
```

There it is: `127.0.0.1:3000`, not `0.0.0.0:3000`. The app's code looked like this:

```js
const http = require('http');
const server = http.createServer((req, res) => res.end('ok\n'));

// WRONG inside a container:
server.listen(3000, '127.0.0.1', () => console.log('listening'));
```

**Why this breaks:** `127.0.0.1` inside the container is the container's *own private loopback interface* — it is not reachable from outside the container, not even from Docker's own port-forwarding proxy on the host. Docker's `-p` mapping connects to the container's *external* network interface, not its loopback. If your app only listens on loopback, the mapping has nothing to attach to, and every external connection attempt gets refused at the container's network boundary.

**The fix:**

```js
// RIGHT inside a container:
server.listen(3000, '0.0.0.0', () => console.log('listening'));
// or simply omit the host argument — Node defaults to 0.0.0.0 for TCP servers
server.listen(3000);
```

```bash
$ docker exec -it <container_id> sh
/app # ss -tlnp
State   Local Address:Port     Peer Address:Port   Process
LISTEN  0.0.0.0:3000            0.0.0.0:*           users:(("node",pid=1,fd=19))

$ curl localhost:8080
ok
```

**The mental model to keep:** `docker run -p HOST_PORT:CONTAINER_PORT` only creates a *path* from the host into the container's network namespace. Whether that path actually reaches your app depends entirely on which interface your app bound to *inside* that namespace. `0.0.0.0` = "any interface, including the one Docker's proxy talks to." `127.0.0.1` = "only myself" — which, inside a container, effectively means "only requests already inside the container," i.e. unreachable from the host.

---

## Common Mistakes

### Mistake 1 — Binding to `127.0.0.1` instead of `0.0.0.0` in a container

Covered in full above. The single most common cause of "it works locally but not in Docker/Kubernetes." If `curl` from *inside* the container works but `curl` from the host (or another pod) doesn't, this is almost always the cause. Check with `ss -tlnp` inside the container — `127.0.0.1` in the Local Address column is the smoking gun.

### Mistake 2 — Trying to bind to a port below 1024 without privilege

```bash
$ node -e "require('http').createServer((q,r)=>r.end('hi')).listen(80)"
node:events:496
      throw er; // Unhandled 'error' event
Error: listen EACCES: permission denied 0.0.0.0:80
    at Server.setupListenHandle [as _listen2] (node:net:1901:21)
```
Ports 0–1023 require root or, on Linux, the `CAP_NET_BIND_SERVICE` capability. Running your whole Node process as root just to bind port 80 is a security anti-pattern. Correct fixes: run your app on an unprivileged port (e.g. 3000) and put a reverse proxy (nginx, a Docker/k8s port mapping, or a cloud load balancer) in front that does the actual port-80/443 binding; or grant just the capability: `setcap 'cap_net_bind_service=+ep' $(which node)`.

### Mistake 3 — Reversing `HOST:CONTAINER` order in `docker run -p`

```bash
docker run -p 3000:8080 my-api:latest
```
It is dangerously easy to write this backwards. The syntax is always `-p HOST_PORT:CONTAINER_PORT`. If your app inside the container actually listens on 3000 (not 8080), the command above maps *host* port 3000 to a *container* port (8080) that nothing is listening on — you'll get connection refused despite everything otherwise being correct. Always double-check by reading it aloud as "traffic hits my **host** on the **first** number and gets forwarded to the **container** on the **second** number."

### Mistake 4 — Ephemeral port exhaustion from unpooled outbound connections

```
Symptom: a service making rapid HTTP calls to a downstream API starts
throwing intermittent errors under load:

Error: connect EADDRNOTAVAIL 203.0.113.9:443 - Local (0.0.0.0:0)
```
**Root cause:** every outbound HTTP request without connection reuse (`Connection: close`, no keep-alive, no pooling) opens a *brand-new* TCP connection — which consumes a fresh ephemeral source port for the life of that connection (and it lingers in `TIME_WAIT` for ~60 seconds afterward — see Topic 08/24). If your service fires thousands of short-lived requests per second at the *same destination IP:port*, you can exhaust the pool of currently-available 4-tuples for that specific destination faster than `TIME_WAIT` sockets free up.

```js
// BAD — every call opens a brand-new socket, burns an ephemeral port
const res = await fetch(url); // default Node fetch/undici does pool, but
                               // a naive axios/http.request without an Agent does not

// GOOD — reuse a keep-alive connection pool
const https = require('https');
const agent = new https.Agent({ keepAlive: true, maxSockets: 50 });
const res = await fetch(url, { agent });
```
**Fix:** use HTTP keep-alive and a bounded connection pool (an `http.Agent`/`https.Agent` with `keepAlive: true`, or your HTTP client's built-in pooling) so a handful of long-lived sockets are reused across thousands of requests, instead of opening and tearing down a new socket — and burning a new ephemeral port — every single time.

---

## Hands-On Proof

```bash
# 1. See every listening TCP socket on your machine, with owning process
ss -tlnp

# 2. Find exactly what's bound to one specific port
lsof -i :3000

# 3. The legacy equivalent (netstat) — still common in older docs/scripts.
#    `ss` is the modern replacement; netstat and ss are covered in full in Topic 24.
netstat -an | grep LISTEN

# 4. Start a throwaway local server and prove it's alive
python3 -m http.server 8000 &
ss -tlnp | grep 8000
curl -s -o /dev/null -w "%{http_code}\n" localhost:8000
kill %1   # stop the background server

# 5. See your OS's actual ephemeral port range (Linux)
cat /proc/sys/net/ipv4/ip_local_port_range
# e.g.: 32768   60999

# 6. Watch an ephemeral port get picked in real time
curl -s ifconfig.me & 
ss -tn | grep -E ':(80|443)\s'
# You'll see a line like:
# ESTAB  0  0  192.168.1.20:52104  34.117.59.81:443
#                          ^^^^^ your OS-assigned ephemeral source port
```

Expected `netstat` output shape (macOS/BSD-style columns shown; Linux differs slightly):

```
$ netstat -an | grep LISTEN
tcp4       0      0  127.0.0.1.5432         *.*                    LISTEN
tcp46      0      0  *.3000                 *.*                    LISTEN
tcp4       0      0  *.22                   *.*                    LISTEN
```

---

## Practice Exercises

### Exercise 1 — Easy: Inventory the ports on your machine

```bash
ss -tlnp
# or if ss requires elevated permission for process names:
sudo ss -tlnp
```
Answer:
1. List every port currently in `LISTEN` state on your machine and the process that owns it.
2. Which of these are bound to `0.0.0.0` (reachable externally) vs `127.0.0.1` (loopback only)?
3. Pick one unfamiliar port from the list and look it up — is it a well-known port, a registered port, or something in a higher range?

### Exercise 2 — Medium: Cause and fix `EADDRINUSE`

```bash
# Terminal 1
python3 -m http.server 5050

# Terminal 2 — try to start ANOTHER server on the same port
python3 -m http.server 5050
```
1. Read the exact error message. What port and address does it mention?
2. Use `ss -tlnp` or `lsof -i :5050` to identify the PID already holding the port.
3. Kill that process (`kill <PID>`) and confirm with `ss`/`lsof` that the port is now free before restarting the second server successfully.
4. Bonus: what would `lsof -i :5050` show for that PID one second *before* you killed it vs one second *after*?

### Exercise 3 — Hard (Production Simulation): Reason about ephemeral port exhaustion

A payment-processing service makes a **new** outbound HTTPS connection (no keep-alive, no pooling) to the same downstream fraud-check API (`fraud.internal:8443`) for every incoming request. Under normal load (50 req/s) it's fine. During a flash sale, load jumps to 3,000 req/s and the service starts intermittently failing with `EADDRNOTAVAIL`.

Work through this without running anything — pure reasoning:

1. Given Linux's default ephemeral range (`32768–60999`, ~28,000 ports) and a `TIME_WAIT` duration of roughly 60 seconds, roughly how many outbound connections-per-second to the *same* destination can this service sustain indefinitely before it starts running out of available 4-tuples? (Hint: ports_available ÷ seconds_held_in_TIME_WAIT ≈ sustainable rate.)
2. At 3,000 req/s to a single destination, is that rate above or below your answer to #1?
3. Explain concretely why switching to a pooled, keep-alive HTTP client (say, 50 persistent connections reused across all 3,000 req/s) resolves this, in terms of how many *distinct* ephemeral ports it now needs.
4. Would this same exhaustion happen if those 3,000 req/s were spread across 3,000 *different* destination IPs instead of one? Why or why not?

---

## Mental Model Checkpoint

Answer these from memory. If you can't, re-read the relevant section.

1. **What is a port, in one sentence, and how many bits is it?**
2. **Name the three IANA port ranges and their numeric boundaries. Which one requires root/CAP_NET_BIND_SERVICE to bind on Unix?**
3. **What is a "socket"? What is the 4-tuple, and why is it the 4-tuple — not just the destination port — that uniquely identifies a TCP connection?**
4. **A server has one process listening on port 443 but 10,000 clients connected right now. How is that possible with only 65,536 total port numbers?**
5. **Why does `server.listen(3000, '127.0.0.1')` break port mapping inside a Docker container, while `server.listen(3000, '0.0.0.0')` works?**
6. **What's the difference between what you'd see in `ss -tlnp` for a socket bound to `0.0.0.0:5432` vs `127.0.0.1:5432`, and which one is safer for a database?**
7. **Explain, in your own words, how ephemeral port exhaustion happens, and why connection pooling/keep-alive fixes it.**

---

## Quick Reference Card

**The three port ranges:**

| Range | Bounds | Also called | Notes |
|---|---|---|---|
| Well-known / System | 0 – 1023 | "privileged ports" | Requires root / `CAP_NET_BIND_SERVICE` to bind on Unix |
| Registered / User | 1024 – 49151 | — | Assigned by IANA per-application; no special privilege needed |
| Dynamic / Private / Ephemeral (IANA) | 49152 – 65535 | "ephemeral ports" | Used as temporary client-side source ports |
| **Linux's actual default ephemeral range** | **32768 – 60999** | — | Check via `cat /proc/sys/net/ipv4/ip_local_port_range` |

**Well-known ports every backend developer must memorize:**

| Port | Protocol/Service | Notes |
|---|---|---|
| 20 / 21 | FTP (data / control) | File transfer — legacy, plaintext |
| 22 | SSH | Remote shell, also used for `git` over SSH and SFTP |
| 23 | Telnet | Legacy remote shell — plaintext, avoid |
| 25 | SMTP | Mail transfer between mail servers |
| 53 | DNS | Name resolution — UDP mostly, TCP for large responses/zone transfers |
| 67 / 68 | DHCP (server / client) | Automatic IP address assignment |
| 80 | HTTP | Unencrypted web traffic |
| 110 | POP3 | Legacy mail retrieval |
| 123 | NTP | Network Time Protocol — clock sync |
| 143 | IMAP | Mail retrieval, keeps mail on server |
| 443 | HTTPS | Encrypted web traffic (TCP for HTTP/1.1 & 2, UDP for HTTP/3-QUIC) |
| 465 / 587 | SMTPS / SMTP submission | 465 = implicit TLS, 587 = STARTTLS (modern standard for sending mail) |
| 993 | IMAPS | IMAP over TLS |
| 995 | POP3S | POP3 over TLS |
| 3000 | — | Common Node.js/dev-server default (not officially reserved) |
| 3306 | MySQL | Default MySQL/MariaDB port |
| 5432 | PostgreSQL | Default Postgres port |
| 5672 | RabbitMQ (AMQP) | Message broker |
| 6379 | Redis | Default Redis port |
| 8080 | — | Common HTTP alternate/proxy port (not officially reserved) |
| 9092 | Kafka | Default Kafka broker port |
| 9200 | Elasticsearch | REST API port |
| 27017 | MongoDB | Default MongoDB port |
| 2181 | Zookeeper | Coordination service (often paired with Kafka) |

**Key commands:**
```bash
ss -tlnp                        # list all listening TCP sockets + owning process
ss -tn                          # list established TCP connections
lsof -i :PORT                   # find exactly what owns a given port
netstat -an | grep LISTEN       # legacy equivalent of `ss -tlnp` (see Topic 24)
cat /proc/sys/net/ipv4/ip_local_port_range   # Linux ephemeral port range
kill <PID>                      # free up a port held by a stuck process
```

---

## When Would I Use This at Work?

### Scenario 1: Local dev server won't start — `EADDRINUSE`
You run `npm run dev` and get `Error: listen EADDRINUSE: address already in use :::3000`. You now know exactly what to do: `lsof -i :3000` (or `ss -tlnp | grep 3000`) to find the PID squatting on the port — usually a zombie process from a crashed previous run — and kill it, instead of randomly changing your app's port and forgetting why.

### Scenario 2: Designing which ports a service exposes in Kubernetes
You're writing a Deployment + Service manifest. You now understand the three distinct port concepts involved: `containerPort`/`targetPort` (the port your app actually listens on inside the pod — must match what your code binds to, and it must be `0.0.0.0`, not `127.0.0.1`), `port` (the port the Service itself is reachable on for other pods), and `nodePort` (if using `NodePort`, the port exposed on every cluster node's external interface, from the 30000–32767 range Kubernetes reserves for this). Confusing these three is one of the most common causes of "why can't my other pod reach this service."

### Scenario 3: Diagnosing intermittent connection failures in a high-throughput service
A payment service starts throwing sporadic connection errors only during traffic spikes, never at normal load. Instead of guessing, you check whether it's opening a fresh outbound TCP connection per request to the same downstream host (no keep-alive/pooling) — the classic ephemeral port exhaustion pattern from this topic. The fix is almost never "add more ports"; it's "reuse the ones you have" via a persistent connection pool.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the big picture this all fits into
- `02-osi-model.md` — where Layer 4 sits relative to everything else
- `03-tcp-ip-model.md` — the practical model backend developers actually use
- `04-ip-addresses.md` — the "street address" half of every socket

**Phase 1 is now complete.** You understand, end to end, how a request physically travels the internet, how the layered models describe it, how addresses identify machines, and how ports identify services on those machines. That's the entire foundation.

**Study next:**
- `06-dns-in-depth.md` — Phase 2 begins here. DNS famously runs on **port 53** — a direct callback to this topic, and your first deep dive into a specific Layer 7 protocol built on everything you now know about addressing and ports.
- `08-tcp-in-depth.md` — the 3-way handshake and connection lifecycle referenced throughout this doc, covered in full.
- `24-netstat-and-ss.md` *(forward reference, Phase 4)* — goes far deeper on the connection-table tools (`ss`, `netstat`, `lsof`) introduced here, including every TCP state (`ESTABLISHED`, `TIME_WAIT`, `CLOSE_WAIT`) only briefly mentioned in this doc.

---

*Doc saved: `/docs/networking/05-ports.md`*
