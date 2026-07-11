# 13 — Networking in Docker
## Section: Docker in Practice

## ELI5 — The Simple Analogy

Think of your computer (the **host**) as a big apartment building. Each container is an apartment. Every apartment needs a way to talk to the others and to the outside street.

- **bridge network** = the building's internal hallway with a receptionist. Every apartment has a door onto the hallway (a private phone extension). To call an apartment from the *street*, you go through the front desk, who forwards your call to the right extension. That "front desk forwarding" is exactly what `-p` sets up.
- **host network** = the apartment has no walls between it and the street — it *is* the street-facing front of the building. Fast, but no privacy.
- **none network** = a sealed room with no phone at all. It can only talk to itself.
- **custom network** = your own private hallway *with a phone book*, so apartments can dial each other **by name** ("call `redis`") instead of memorizing extension numbers.

The magic that connects an apartment door to the hallway is a **veth pair**: a tunnel with two ends, one inside the apartment (`eth0`) and one plugged into the hallway switchboard (`docker0`).

## The Linux kernel feature underneath

Docker networking is built on **four** kernel features working together:

```
1. NETWORK NAMESPACES  → each container gets its own private network stack
                         (own interfaces, own routing table, own iptables, own ports)

2. VIRTUAL ETHERNET (veth) PAIRS → a "cable" with two ends; packets in one end
                                    come out the other. One end in the container's
                                    netns, the other on the host.

3. LINUX BRIDGE (docker0) → a virtual layer-2 switch in the kernel. All the host-side
                            veth ends plug into it, so containers can reach each other.

4. NETFILTER / iptables (nat table) → rewrites packet addresses. This is how -p works
                                       (DNAT) and how containers reach the internet (MASQUERADE).
```

A **network namespace** is the key. Normally your host has one network stack: one set of interfaces, one routing table, one list of listening ports. `clone(CLONE_NEWNET)` creates a *brand new, empty* network stack — just a loopback. A container placed in it sees only *its own* interfaces. Two containers can both `listen(3000)` without colliding, because each `3000` lives in a separate namespace.

But an empty namespace can't talk to anything. So Docker builds a **veth pair** — think of it as a virtual Ethernet cable. One end goes inside the container (named `eth0`), the other stays on the host (named `vethXXXXXX`) and is plugged into the **`docker0` bridge**. The bridge is a software switch: any packet arriving on one port is forwarded toward the right destination, so all containers on `docker0` can reach each other at layer 2.

Finally, **iptables** (the userspace command for the kernel's netfilter framework) rewrites addresses so (a) the outside world can reach a container (`-p` → DNAT) and (b) containers can reach the internet (source-NAT/MASQUERADE, so replies find their way back).

## What is this?

Docker networking is the system that gives each container an IP address, connects containers to each other and to the host, and controls which container ports are reachable from outside. It is implemented entirely with standard Linux kernel networking (namespaces, veth, bridges, iptables) — Docker just automates the setup.

## Why does it matter for a backend developer?

Your `orders-api` is useless in isolation. It must reach **Postgres** and **Redis**, and the **outside world** (or nginx) must reach it. Networking decisions determine:

- Whether `orders-api` can resolve `db` and `redis` by name (custom network) or must hardcode fragile IPs (default bridge).
- Whether your API is reachable at all (`-p` published or not).
- Whether a `Connection refused` is a code bug or a missing iptables rule.
- Whether containers are properly isolated or leaking traffic to the host (`host` mode).

Nearly every "my services can't talk to each other" bug traces back to *not understanding these four kernel pieces*.

## The physical reality

After `docker network create orders-net` and running `orders-api`, `db`, `redis` on it, here is what physically exists:

```
HOST KERNEL
│
├── Interfaces (ip link show):
│   ├── docker0            ← default bridge (172.17.0.0/16), used if no --network
│   ├── br-a1b2c3d4        ← the "orders-net" custom bridge (172.18.0.0/16)
│   ├── veth9f2a@if7  ─────┐  host end of API's cable, plugged into br-a1b2c3d4
│   ├── veth3c1d@if7  ─────┤  host end of db's cable
│   └── veth7e88@if7  ─────┘  host end of redis's cable
│
├── iptables NAT table:
│   ├── DOCKER chain        ← DNAT rules from every -p (host port → container ip:port)
│   ├── POSTROUTING         ← MASQUERADE rule (containers → internet)
│   └── DOCKER-USER chain   ← your custom rules run first (for firewalls)
│
└── /var/lib/docker/network/files/local-kv.db   ← Docker's network metadata store

INSIDE orders-api's network namespace (/proc/<PID>/ns/net):
├── lo                     ← loopback (127.0.0.1)
├── eth0 (172.18.0.2/16)   ← container end of the veth cable
├── routing table:  default via 172.18.0.1 dev eth0   ← gateway = the bridge
└── /etc/resolv.conf → nameserver 127.0.0.11          ← Docker's embedded DNS
```

That `127.0.0.11` is the crucial DNS detail: on a **custom** network Docker runs a tiny embedded DNS server at `127.0.0.11` inside each container's namespace. When `orders-api` looks up `db`, the query goes to `127.0.0.11`, which answers with `db`'s current container IP. That is how name-based service discovery works.

## How it works — step by step

Trace of a request: your laptop's browser hits `http://<host-ip>:8080`, which reaches `orders-api` (published with `-p 8080:3000`), which then queries Postgres at `db:5432`.

**Setup (when the container started):**

1. Docker created the container's **network namespace** via `clone(CLONE_NEWNET)`.
2. Docker created a **veth pair**: `veth9f2a` (host) and a peer.
3. The peer was **moved into the container's netns** (`setns`) and **renamed `eth0`**.
4. `veth9f2a` was **attached to the bridge** (`br-a1b2c3d4` for custom, or `docker0` for default).
5. Docker assigned the container an **IP** from the network's subnet (e.g. `172.18.0.2`) and set the default route via the bridge IP (`172.18.0.1`).
6. `-p 8080:3000` inserted a **DNAT rule** in the `nat` table's `DOCKER` chain.

**Inbound request (browser → API):**

7. Packet arrives at host: `dst=<host-ip>:8080`.
8. Kernel's `nat` PREROUTING → jumps to `DOCKER` chain → matches the DNAT rule → **rewrites destination** to `172.18.0.2:3000`.
9. Packet is routed to the bridge, out `veth9f2a`, through the cable, arrives on the container's `eth0`.
10. Node's `app.listen(3000)` accepts it. Response flows back; the reverse NAT translation restores the source so the browser sees a reply from `:8080`.

**Container-to-container (API → Postgres):**

11. Node calls `pg.connect('postgres://db:5432/...')`. The DNS lookup for `db` goes to `127.0.0.11` (Docker's embedded resolver in the container's netns).
12. Embedded DNS answers `172.18.0.3` (Postgres's current IP).
13. Node opens a TCP connection to `172.18.0.3:5432`. The packet leaves `eth0`, hits the bridge `br-a1b2c3d4`, which switches it to Postgres's veth → Postgres's `eth0`. **No NAT involved** — this is pure layer-2 bridging between containers on the same network.

**Container-to-internet (API → external API):**

14. Packet with `src=172.18.0.2` heads to a public IP via the default route (bridge gateway).
15. In POSTROUTING, the **MASQUERADE** rule rewrites the source to the host's IP so the reply can route back. Return packets are un-NAT'd back to `172.18.0.2`.

## Exact syntax breakdown

### Creating a custom network

```
docker network create --driver bridge --subnet 172.18.0.0/16 orders-net
│              │       │        │      │        │              │
│              │       │        │      │        │              └─ network name (becomes DNS domain)
│              │       │        │      │        └─ optional explicit subnet (CIDR)
│              │       │        │      └─ specify the address range
│              │       │        └─ "bridge" (default), "overlay" (swarm), "macvlan"...
│              │       └─ choose the network driver
│              └─ make a new user-defined network
└─ the network subcommand group
```

### Attaching at run time

```
docker run --network orders-net --name db postgres:16
           │         │          │      │
           │         │          │      └─ this name becomes resolvable by other containers
           │         │          └─ set container name = DNS hostname
           │         └─ join this network (gets IP + embedded DNS at 127.0.0.11)
           └─ network selection flag
```

### Publishing a port (the `-p` breakdown, kernel view)

```
-p 8080:3000
   │    │
   │    └─ CONTAINER port → becomes the DNAT *target* port
   └─ HOST port → becomes the iptables match "dpt:8080"
```

The rule this creates (visible via `iptables -t nat -L DOCKER -n`):

```
DNAT   tcp  --  0.0.0.0/0   0.0.0.0/0   tcp dpt:8080  to:172.18.0.2:3000
│      │                                │             │
│      │                                │             └─ rewrite dst to container ip:port
│      │                                └─ match packets whose dst port is 8080
│      └─ protocol tcp
└─ Destination NAT: change where the packet is going
```

### The four built-in network modes

```
--network bridge   → private IP on docker0, isolated, needs -p to be reachable   (DEFAULT)
--network host     → shares the HOST's netns; no veth, no NAT; app binds host port directly
--network none     → only loopback; totally isolated, no external connectivity
--network <name>   → user-defined bridge: same as bridge BUT with automatic DNS by name
          │
          └─ this is what you almost always want for multi-container apps
```

## Example 1 — basic

Prove container-to-container DNS on a custom network with two tiny containers:

```bash
# 1. Create a user-defined bridge network
docker network create orders-net

# 2. Start Redis on it, named "redis"
docker run -d --name redis --network orders-net redis:7-alpine

# 3. Start a throwaway Alpine on the SAME network and ping "redis" BY NAME
docker run --rm -it --network orders-net alpine sh
```

Inside the Alpine shell:

```sh
# /etc/resolv.conf points at Docker's embedded DNS
cat /etc/resolv.conf          # → nameserver 127.0.0.11

# Name resolution works because we're on a custom network
nslookup redis                # → resolves to 172.18.0.2 (redis's IP)
ping -c1 redis                # → replies from 172.18.0.2

# Prove it: try the SAME on the default bridge later and the name WON'T resolve
```

The name `redis` resolved to an IP with zero configuration — that is the embedded DNS at `127.0.0.11` doing its job. On the **default** `bridge` network this would fail (default bridge has no automatic DNS, only legacy `--link`).

## Example 2 — production scenario

**Scenario:** A new engineer wired up `orders-api`, `db`, and `redis` using the **default** bridge network and hardcoded IPs like `172.17.0.3`. It worked in the demo. Then Redis was restarted, got a **new IP** `172.17.0.5`, and the API started throwing `ECONNREFUSED 172.17.0.3:6379` in production. Every deploy shuffled IPs and broke connections.

**Root cause:** the default bridge has **no DNS**, so they hardcoded IPs — which Docker reassigns on every container recreation. The fix is a **custom network** so containers resolve each other by stable *name*.

Correct setup:

```bash
# One custom network for the whole app — gives stable name-based DNS
docker network create orders-net

# Postgres: reachable as "db"
docker run -d --name db --network orders-net \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

# Redis: reachable as "redis"
docker run -d --name redis --network orders-net redis:7-alpine

# API: connects using NAMES, not IPs; published to the world on 8080
docker run -d --name orders-api --network orders-net \
  -p 8080:3000 \
  -e DATABASE_URL="postgres://postgres:secret@db:5432/orders" \
  -e REDIS_URL="redis://redis:6379" \
  orders-api:1.4.2
```

Now `db` and `redis` in the connection strings resolve via embedded DNS to *whatever* IP those containers currently hold. Restarts, redeploys, IP changes — all invisible to the app. The connection string never changes.

Verify the wiring:

```bash
docker network inspect orders-net --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
# db          172.18.0.2/16
# redis       172.18.0.3/16
# orders-api  172.18.0.4/16
```

## Common mistakes

### Mistake 1 — Using the default bridge and expecting DNS

```bash
docker run -d --name redis redis:7-alpine          # default bridge
docker run --rm -it alpine ping redis
```
```
ping: bad address 'redis'
```
- **Root cause:** the **default** `bridge` network deliberately has no embedded DNS resolver — only user-defined networks do. Name resolution simply isn't available there.
- **Wrong:** rely on names on the default bridge.
- **Right:** `docker network create orders-net` and attach both containers with `--network orders-net`. Then `ping redis` works.

### Mistake 2 — Publishing a port but the app binds `127.0.0.1` inside the container

```bash
# Node: app.listen(3000, '127.0.0.1')  ← bound to loopback INSIDE the container
docker run -d -p 8080:3000 orders-api:1.0
curl http://localhost:8080/health
```
```
curl: (52) Empty reply from server
```
- **Root cause:** DNAT delivers the packet to the container's `eth0` (`172.18.0.2:3000`), but the app only listens on `127.0.0.1:3000` *inside the container's namespace*. `eth0` traffic never reaches a listener.
- **Right:** bind to `0.0.0.0` inside the container: `app.listen(3000, '0.0.0.0')`. Inside a container, `0.0.0.0` is safe — the namespace *is* the isolation boundary.

### Mistake 3 — Two containers on different networks trying to talk

```bash
docker network create net-a
docker network create net-b
docker run -d --name db --network net-a postgres:16
docker run -d --name api --network net-b -e DATABASE_URL=postgres://db:5432 orders-api:1.0
# api logs: getaddrinfo ENOTFOUND db
```
- **Root cause:** DNS and bridging only span a *single* network. `db` on `net-a` is invisible to `api` on `net-b` — different bridges, different DNS scope.
- **Right:** put both on the same network, or attach one container to both: `docker network connect net-a api`.

### Mistake 4 — `--network host` and then also `-p`

```bash
docker run -d --network host -p 8080:3000 orders-api:1.0
```
```
WARNING: Published ports are discarded when using host network mode
```
- **Root cause:** in `host` mode the container shares the host's netns — `app.listen(3000)` binds the host's real `:3000` directly. There is no separate namespace to NAT into, so `-p` is meaningless and ignored.
- **Right:** with `host` mode, drop `-p` and connect to the port the app binds. Use `host` only when you truly want no isolation (e.g. high-throughput or when you need the host's exact network view).

### Mistake 5 — Port already in use on the host

```bash
docker run -d -p 8080:3000 orders-api:1.0
docker run -d -p 8080:3000 orders-api:1.0     # second one, same host port
```
```
Error: driver failed programming external connectivity: Bind for 0.0.0.0:8080 failed: port is already allocated
```
- **Root cause:** two DNAT rules can't both own host port `8080` — the host port is a single scarce resource. The container ports can be identical, but host ports must be unique.
- **Right:** map to a different host port for the second: `-p 8081:3000`.

## Hands-on proof

See every kernel piece with your own eyes:

```bash
# Setup
docker network create orders-net
docker run -d --name redis --network orders-net redis:7-alpine
docker run -d --name orders-api --network orders-net -p 8080:3000 nginx:alpine

# 1. See the custom bridge interface on the host
ip link show type bridge | grep -A1 br-
#   → br-xxxxxxxx  (this is orders-net)

# 2. See the veth pairs (host ends) plugged into the bridge
ip link show | grep veth
#   → vethXXXX@ifN  — one per container

# 3. Look INSIDE the container's network namespace
docker exec orders-api ip addr show eth0
#   → eth0: inet 172.18.0.x/16  (the container end of the veth cable)

# 4. See the embedded DNS resolver config
docker exec orders-api cat /etc/resolv.conf
#   → nameserver 127.0.0.11

# 5. Prove name-based DNS resolves to a real container IP
docker exec orders-api getent hosts redis
#   → 172.18.0.y  redis

# 6. See the DNAT rule that -p created
sudo iptables -t nat -L DOCKER -n | grep 8080
#   → DNAT tcp dpt:8080 to:172.18.0.x:80

# 7. See the MASQUERADE rule that lets containers reach the internet
sudo iptables -t nat -L POSTROUTING -n | grep MASQUERADE
#   → MASQUERADE  all  --  172.18.0.0/16  0.0.0.0/0

# 8. Full network topology
docker network inspect orders-net

# Cleanup
docker rm -f redis orders-api && docker network rm orders-net
```

## Practice exercises

### Exercise 1 — easy
Create a custom network `demo-net`. Run two containers on it (`redis` and an Alpine shell). From the Alpine shell, resolve and ping `redis` by name. Then run a second Alpine on the **default** bridge and confirm `ping redis` fails there. Explain the difference in one sentence.

### Exercise 2 — medium
Run `nginx:alpine` with `-p 8080:80`. Use `sudo iptables -t nat -L DOCKER -n` to find the exact DNAT rule. Write down the rule and annotate: which part is the host-port match, which is the rewrite target, and what happens to a packet arriving at `<host>:8080`.

### Exercise 3 — hard (production simulation)
Reproduce the Mistake 3 cross-network bug: put `db` on `net-a` and `orders-api` (or an Alpine that does `nc -zv db 5432`) on `net-b`, and observe `ENOTFOUND`. Then fix it **two different ways** — (a) move both onto one network, and (b) keep them separate but use `docker network connect` to dual-home the API. For each fix, run `docker inspect` to show which networks and IPs each container ended up with, and explain the trade-off between the two approaches.

## Mental model checkpoint

1. What are the four kernel features Docker networking is built on?
2. What is a veth pair, and where do its two ends live?
3. Why can two containers both `listen(3000)` without colliding?
4. What is special about the address `127.0.0.11` inside a container?
5. Why does name-based DNS work on a custom network but not the default bridge?
6. Exactly what does `-p 8080:3000` insert, and in which iptables table/chain?
7. When two containers on the same network talk, is NAT involved? Why or why not?
8. Why must an in-container app bind `0.0.0.0` rather than `127.0.0.1` to be reachable via `-p`?

## Quick reference card

| Command / concept | What it does | Key detail |
|-------------------|--------------|------------|
| `docker network create X` | Make a user-defined bridge | Enables DNS by container name |
| `docker network ls` | List networks | bridge, host, none are built-in |
| `docker network inspect X` | Show subnet, containers, IPs | JSON with gateway + members |
| `docker network connect X c` | Attach running container to net | Dual-homes a container |
| `--network bridge` | Default isolated bridge | No DNS; needs `-p`; on `docker0` |
| `--network host` | Share host netns | No veth, no NAT, `-p` ignored |
| `--network none` | Loopback only | Full network isolation |
| `--network <custom>` | User-defined bridge | Automatic name-based DNS |
| `-p H:C` | Publish port | iptables DNAT: host H → container C |
| `docker0` | Default bridge interface | 172.17.0.0/16 |
| `127.0.0.11` | Embedded DNS server | Resolves container names → IPs |
| veth pair | Virtual cable | `eth0` inside ↔ `vethXXXX` on host |

## When would I use this at work?

1. **Wiring `orders-api` + Postgres + Redis locally:** create one custom network so the API's connection strings use stable names (`db`, `redis`) that survive restarts — the exact pattern Docker Compose automates for you (Topic 18).
2. **Debugging "service can't reach the database":** check whether both containers are on the *same* network (`docker network inspect`), whether DNS resolves the name (`docker exec api getent hosts db`), and whether the app binds `0.0.0.0` — a systematic checklist grounded in the four kernel pieces.
3. **Deciding host vs bridge for a latency-sensitive service:** choose `--network host` on a Linux box to skip the NAT/bridge hop for a high-throughput gateway, accepting the loss of isolation — a deliberate, informed trade-off rather than a guess.

## Connected topics

- **Study before:** Topic 02 (Linux Foundations) — network namespaces; Topic 12 (docker run in Depth) — where `-p` and `--network` are introduced.
- **Study after:** Topic 15 (Env Vars and Secrets) — how connection strings are injected; Topic 18–20 (Docker Compose) — Compose auto-creates a custom network so services reach each other by name; Topic 35 (K8s Services) and Topic 40 (K8s Networking) — the same veth/bridge/iptables ideas scaled across nodes with kube-proxy and CNI.
