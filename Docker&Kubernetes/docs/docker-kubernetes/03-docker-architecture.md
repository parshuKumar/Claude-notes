# 03 — Docker Architecture
## Section: Docker Foundations

---

## ELI5 — The Simple Analogy

Imagine ordering food at a big restaurant.

1. **You** speak to the **waiter**. You don't walk into the kitchen; you just say "one pizza." The waiter is the **Docker CLI** (`docker`) — the friendly front desk you talk to.

2. The waiter carries your order to the **head chef**, who runs the whole kitchen: takes orders, manages inventory, decides what to cook, keeps the pantry stocked. The head chef is the **Docker daemon** (`dockerd`) — the long-running brain.

3. The head chef doesn't personally fry every egg. He hands the actual cooking to a **line cook** who manages the day-to-day cooking stations and keeps dishes running. The line cook is **containerd** — it manages the lifecycle of every container.

4. For the single act of *lighting the stove and putting the pan on it*, the line cook uses a **specific tool** — a lighter. It does one tiny job then walks away. That tool is **runc** — it does the raw kernel work (the `clone()`, cgroups, `pivot_root` from Topic 02) to *spawn* the container, then exits.

You (CLI) → head chef (dockerd) → line cook (containerd) → lighter (runc). Four components, each doing less and less but more and more precisely, ending at the kernel. That chain is the entire Docker architecture.

---

## The Linux kernel feature underneath

Two kernel mechanisms make this chain *communicate*:

1. **Unix domain sockets.** The CLI talks to `dockerd` over a socket file at `/var/run/docker.sock`. A Unix socket is a file on disk that two processes read/write to talk — like a phone line that exists as a `/dev`-style special file. `dockerd` talks to `containerd` over another socket at `/run/containerd/containerd.sock`. No network needed; it's kernel-mediated inter-process communication (the `ipc`/socket machinery).

2. **The `clone()`/`execve()` syscalls + namespaces + cgroups from Topic 02.** At the very bottom, `runc` is the *only* component that actually touches those kernel features. Everything above it is management and bookkeeping; `runc` is where the abstraction meets the metal.

The protocols layered on the sockets: the CLI↔dockerd link speaks a **REST/HTTP API** (JSON over the socket). The dockerd↔containerd link speaks **gRPC** (a fast binary RPC protocol). And runc is invoked per the **OCI Runtime Spec** — a JSON file (`config.json`) plus a standard CLI contract. Sockets carry the bytes; HTTP/gRPC/OCI give them meaning.

---

## What is this?

Docker is **not one program**. It's a stack of cooperating components: the **`docker` CLI** (what you type), the **`dockerd` daemon** (the manager that builds images and exposes the API), **`containerd`** (a lower-level daemon that supervises container lifecycles and images), and **`runc`** (a tiny OCI runtime that actually creates the container using kernel features, then exits).

They talk over **Unix sockets** using **HTTP** (CLI→dockerd), **gRPC** (dockerd→containerd), and the **OCI spec** (containerd→runc).

---

## Why does it matter for a backend developer?

Because when something breaks, the error message points at a *specific link in the chain*, and you need to know which:

- `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` → the **CLI↔dockerd** socket link is down. You now know to check `systemctl status docker`, not your Dockerfile.
- Your container is running but `docker` commands hang → **dockerd** may be wedged while **containerd** still supervises the container (which is why containers can keep running even if you restart `dockerd`!).
- Kubernetes announces it's "removing dockershim" and you panic → understanding this chain tells you K8s just talks to **containerd** directly and skips `dockerd`; your images and containers are unaffected (Topic 29).
- You want rootless/daemonless containers (Podman) → you understand it's the *same bottom half* (`runc` + OCI) without the `dockerd` middleman.

Without this map, every Docker error is a mystery. With it, each error names its own culprit.

---

## The physical reality

On a Linux host with Docker installed and `orders-api` running, these processes and files literally exist:

**The process tree (`pstree`-style):**

```
systemd(1)
├─ dockerd(1200)                         ← the Docker daemon, a long-lived process
│                                          listens on /var/run/docker.sock
├─ containerd(1150)                       ← separate long-lived daemon (NOT a child of dockerd!)
│   └─ containerd-shim-runc-v2(4800)      ← one shim per container; the container's real parent
│       └─ node(4820) server.js           ← YOUR orders-api process = the container's PID 1
```

Notice two things: `containerd` is a **sibling** of `dockerd`, not a child — it's managed by systemd independently. And `runc` is **not** in this tree, because it already **exited** (more on that below).

**The sockets:**

```
$ ls -l /var/run/docker.sock
srw-rw---- 1 root docker /var/run/docker.sock     ← 's' = socket. CLI↔dockerd (HTTP/REST)

$ ls -l /run/containerd/containerd.sock
srw-rw---- 1 root root  /run/containerd/containerd.sock   ← dockerd↔containerd (gRPC)
```

**The binaries on disk:**

```
/usr/bin/docker             ← the CLI (a Go binary)
/usr/bin/dockerd            ← the daemon
/usr/bin/containerd         ← the lifecycle manager
/usr/bin/containerd-shim-runc-v2  ← the per-container shim
/usr/bin/runc               ← the OCI runtime (does the kernel work, then exits)
```

**The OCI bundle for a running container** (what `runc` is handed):

```
/run/containerd/io.containerd.runtime.v2.task/moby/<container-id>/
├── config.json     ← the OCI runtime spec: namespaces, cgroups, mounts, the command to run
└── rootfs/         ← the overlay 'merged' dir (Topic 02) = the container's /
```

`config.json` is the *contract*. It's a JSON file that says exactly which namespaces to create, which cgroup limits to set, which capabilities to drop, and what process to `execve`. `runc` reads it and does precisely that.

---

## How it works — step by step

Full trace of `docker run -d --name orders-api -p 3000:3000 orders-api:1.4.0`:

1. **You type the command.** The `docker` CLI binary parses your flags into a request. It does **no container work itself** — it's a thin client.

2. **CLI → dockerd over the socket (HTTP).** The CLI opens `/var/run/docker.sock` and sends an HTTP request, roughly `POST /v1.44/containers/create` with a JSON body describing the image, ports, env, limits. Then `POST /containers/<id>/start`. This is a real REST API — you could `curl` it (see Hands-on proof).

3. **dockerd does the high-level work.** The daemon:
   - Checks whether image `orders-api:1.4.0` exists locally; if not, it pulls it from a registry (Topic 04/24).
   - Prepares the **overlay filesystem** (Topic 02): stacks the image's read-only layers as lowerdirs and creates the container's writable upperdir under `/var/lib/docker/overlay2/`.
   - Sets up **networking**: allocates an IP, creates the `veth` pair, wires it to `docker0`, and installs the `-p 3000:3000` iptables DNAT rule (Topic 13).
   - Then hands the container off to `containerd`.

4. **dockerd → containerd over gRPC.** dockerd calls containerd's gRPC API (over `/run/containerd/containerd.sock`) to create a **container object** and a **task** (a task = an actual running instance of a container). gRPC is a binary, strongly-typed RPC protocol — much faster and stricter than JSON/HTTP, appropriate for this internal, high-frequency link.

5. **containerd builds the OCI bundle.** containerd writes the `config.json` (the OCI runtime spec) and points `rootfs/` at the overlay merged dir. This bundle is the standardized handoff format — *any* OCI runtime (runc, crun, gVisor's runsc, Kata) could consume it. That standard is why the ecosystem is pluggable.

6. **containerd starts a shim.** containerd spawns `containerd-shim-runc-v2`, one shim **per container**. The shim is the container's long-lived parent. Why a shim? So the container's lifetime is **decoupled** from containerd and dockerd — you can restart or upgrade `dockerd`/`containerd` and your `orders-api` keeps running, because the shim (not the daemon) is holding it open, keeping its stdio, and watching for exit.

7. **shim → runc (OCI CLI contract).** The shim invokes `runc create` then `runc start`, pointing at the bundle directory. The interface here is the **OCI Runtime CLI spec**: a documented set of subcommands (`create`, `start`, `kill`, `delete`) operating on a bundle. This is the seam where you could swap `runc` for another runtime by changing one setting.

8. **runc does the actual kernel work (Topic 02).** `runc` reads `config.json` and performs the real syscalls: `clone()` with the namespace flags, writes the cgroup files (`memory.max`, `cpu.max`), sets up mounts, `pivot_root` into `rootfs/`, drops capabilities, applies seccomp, and finally `execve("/usr/local/bin/node", ["node","server.js"])`.

9. **runc EXITS.** This is the surprising, important part: once the container process is running, `runc` **exits immediately**. It's a one-shot tool — spawn and leave, like a lighter that lights the stove then is put away. The now-orphaned container process is re-parented to the **shim**. This is why you never see `runc` in `pstree` for a running container.

10. **The container runs, supervised by the shim.** The shim holds the container's stdout/stderr (so `docker logs` works — Topic 17), reports the exit code back up to containerd (and dockerd) when the process dies, and keeps the container alive independent of the daemons.

11. **Response flows back up.** containerd returns success to dockerd (gRPC), dockerd returns the container ID to the CLI (HTTP), and your terminal prints the container ID. Done.

**The elegant summary:** each layer does less. dockerd = features & build & network. containerd = lifecycle & images & tasks. shim = per-container supervision. runc = one-shot kernel setup. Kernel = the actual isolation.

---

## Exact syntax breakdown

The command that traverses the whole chain:

```
docker run -d -p 3000:3000 --name orders-api orders-api:1.4.0
│      │   │  │             │            │
│      │   │  │             │            └─ image; dockerd resolves & stacks its layers into rootfs
│      │   │  │             └─ --name: label stored in dockerd's metadata (/var/lib/docker)
│      │   │  └─ -p 3000:3000: dockerd installs the iptables DNAT rule (not runc)
│      │   └─ -d: detached; CLI returns immediately after dockerd confirms start
│      └─ run: CLI sends POST /containers/create + /start over /var/run/docker.sock (HTTP)
└─ docker: the thin CLI client — talks to dockerd, does no container work itself
```

Talking to the daemon's REST API directly (proving the CLI is just an HTTP client):

```
curl --unix-socket /var/run/docker.sock http://localhost/v1.44/containers/json
│    │            │                     │                │      │
│    │            │                     │                │      └─ the endpoint: list running containers
│    │            │                     │                └─ API version dockerd exposes
│    │            │                     └─ host is ignored for unix sockets; any value works
│    │            └─ the socket file the CLI normally uses
│    └─ --unix-socket: send HTTP over a Unix domain socket instead of TCP
└─ curl: we are BEING the docker CLI by hand
```

Inspecting which low-level runtime is used:

```
docker info --format '{{ .DefaultRuntime }}'
│           │                  │
│           │                  └─ .DefaultRuntime: prints "runc" (the OCI runtime containerd calls)
│           └─ --format: Go-template output
└─ docker info: asks dockerd to report its own configuration
```

---

## Example 1 — basic

See the whole chain with your own eyes on a Linux host.

```bash
# 1. Start orders-api-style container.
docker run -d --name orders-api alpine sleep 600

# 2. See the daemons — they are SEPARATE processes, siblings under systemd.
ps -ef | grep -E 'dockerd|containerd' | grep -v grep
# root  1200  ... /usr/bin/dockerd                          <- the daemon
# root  1150  ... /usr/bin/containerd                       <- lifecycle manager (NOT a child of dockerd)
# root  4800  ... containerd-shim-runc-v2 -namespace moby   <- one shim for OUR container

# 3. See the container's real parent is the SHIM, not dockerd and not runc.
PID=$(docker inspect --format '{{.State.Pid}}' orders-api)
ps -o ppid= -p $PID | xargs ps -o pid,comm= -p
#  4800 containerd-shim   <- the shim is the parent. runc is GONE (it exited after spawning).

# 4. Prove the CLI is just an HTTP client over a socket — bypass it with curl.
curl -s --unix-socket /var/run/docker.sock \
  http://localhost/v1.44/containers/json | head -c 300
# You just got the SAME data `docker ps` shows, by hitting dockerd's REST API directly.

# 5. See the OCI bundle handed to runc.
sudo ls /run/containerd/io.containerd.runtime.v2.task/moby/*/
# config.json  rootfs/     <- config.json IS the OCI runtime spec runc executed

docker rm -f orders-api
```

The lesson: `docker ps` and a `curl` to the socket return the same thing (step 4), and your app's real parent is the shim while runc has already vanished (step 3).

---

## Example 2 — production scenario

**The situation.** Your `orders` platform needs a **zero-downtime Docker upgrade** on a busy node running `orders-api`, `postgres`, and `redis` containers. A colleague says "if we restart the Docker daemon to upgrade it, won't all the containers die and drop customer traffic?" You need to answer correctly — and the architecture gives you the answer.

**The reasoning, straight from the chain:**

Recall step 6 and step 9: running containers are supervised by their **`containerd-shim`**, and `runc` already exited. The containers are **not children of `dockerd`**. So restarting `dockerd` does **not** kill them — the shims keep them alive. Prove it live:

```bash
# Note the container's PID and its parent shim BEFORE the upgrade.
PID=$(docker inspect --format '{{.State.Pid}}' orders-api)
echo "orders-api host PID: $PID"
ps -o ppid= -p $PID          # -> the shim's PID, e.g. 4800

# Restart ONLY the Docker daemon (simulating an upgrade of dockerd).
sudo systemctl restart docker

# The container process is STILL ALIVE with the SAME PID — traffic never dropped.
ps -o pid,comm= -p $PID       # -> 4820 node   (unchanged!)
docker ps                     # orders-api still Up, same container
```

Why it works: `systemctl restart docker` restarts `dockerd` (and, depending on config, keeps `containerd` running via `live-restore`/independent unit). The **shim** owns the container's lifetime and stdio, so a daemon bounce is invisible to `orders-api`. Contrast: if Docker were one monolithic process that was the direct parent of every container, restarting it *would* kill them all. The **deliberate split** (dockerd / containerd / shim / runc) exists precisely to enable daemon upgrades without downtime.

**The production takeaway you give your colleague:** "Upgrading `dockerd` is safe for running containers because they're held by per-container shims, not by the daemon. Enable `live-restore` in `/etc/docker/daemon.json` to be certain, and we can roll the Docker upgrade node-by-node with zero customer impact." That answer comes *directly* from knowing the four-component architecture.

---

## Common mistakes

**Mistake 1 — "The daemon is down" panics.**

```
$ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

Root cause: the **CLI↔dockerd** socket link failed — either `dockerd` isn't running or your user isn't in the `docker` group so you can't open `/var/run/docker.sock`. Wrong fix: reinstalling Docker. Right fix: `sudo systemctl status docker` to check the daemon, and `sudo usermod -aG docker $USER` (then re-login) for the permission case.

**Mistake 2 — Assuming restarting Docker kills your containers.**

Belief: "don't touch `dockerd` in prod, it'll drop everything." Root cause: thinking Docker is monolithic. Reality (Example 2): the shim keeps containers alive across a daemon restart. Wrong action: draining the whole node just to patch dockerd. Right: enable `live-restore` and restart the daemon safely.

**Mistake 3 — Mounting `/var/run/docker.sock` into a container without realizing it's root-equivalent.**

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock   # gives the container FULL control of the host's Docker
```

Root cause: that socket is the daemon's REST API with no extra auth — whoever can write to it can start a privileged container and own the host. Right: avoid it; if a container truly needs Docker access, use a scoped proxy or rootless setup (Topic 23).

**Mistake 4 — "Kubernetes removed Docker, so my images won't work."**

Panic when K8s deprecated `dockershim`. Root cause: conflating **Docker (dockerd)** with the **image format** and **containerd**. Reality: K8s now talks to **containerd** (or CRI-O) directly, skipping `dockerd`, but images built by Docker are **OCI images** that containerd runs unchanged. Right understanding: your `orders-api:1.4.0` image and your Dockerfiles are completely unaffected (Topic 29).

**Mistake 5 — Expecting to see `runc` running for a live container.**

```
$ ps -ef | grep runc     # nothing for a running container!
```

Root cause: `runc` is a **one-shot** tool — it sets up the container and **exits** (step 9). Seeing no `runc` is *correct*. You'll only catch it during the brief create/start moment. Right mental model: the persistent supervisor is the **shim**, not runc.

---

## Hands-on proof

```bash
# 1. All four components exist as binaries.
for b in docker dockerd containerd containerd-shim-runc-v2 runc; do
  echo -n "$b -> "; command -v $b || which $b
done

# 2. Daemons are separate, long-lived processes.
ps -ef | grep -E '/usr/bin/(dockerd|containerd)$' | grep -v grep

# 3. The sockets that carry the protocols.
ls -l /var/run/docker.sock /run/containerd/containerd.sock   # both 's' (sockets)

# 4. Start a container and find its supervisor.
docker run -d --name proof alpine sleep 300
PID=$(docker inspect --format '{{.State.Pid}}' proof)
PPID=$(ps -o ppid= -p $PID | tr -d ' ')
echo "container PID $PID, parent PID $PPID ->"; ps -o comm= -p $PPID   # -> containerd-shim...

# 5. Be the CLI: hit dockerd's REST API directly (same result as `docker ps`).
curl -s --unix-socket /var/run/docker.sock http://x/v1.44/containers/json \
  | tr ',' '\n' | grep -m1 Names

# 6. See the OCI bundle runc was handed.
sudo find /run/containerd -name config.json -path '*proof*' 2>/dev/null | head -1

# 7. Which OCI runtime does containerd use?
docker info --format 'Default runtime: {{ .DefaultRuntime }}'   # -> runc

docker rm -f proof
```

You verified: four distinct binaries exist, two daemons run independently, two sockets carry the traffic, the container's parent is the **shim** (not runc), the CLI is replaceable by `curl`, and `runc` is the configured OCI runtime.

---

## Practice exercises

### Exercise 1 — easy
Run `docker run -d --name a alpine sleep 300`. Using `ps -ef`, draw the process chain from `systemd` down to your `sleep` process, labeling each component (dockerd, containerd, shim, the app). Note explicitly why `runc` does **not** appear.

### Exercise 2 — medium
Without using the `docker` command at all, use `curl --unix-socket /var/run/docker.sock` to (a) list running containers and (b) fetch the details of one container by ID (`GET /v1.44/containers/<id>/json`). Explain what this proves about where the "intelligence" lives (CLI vs daemon).

### Exercise 3 — hard (production simulation)
Simulate the zero-downtime upgrade from Example 2. Start `orders-api` (`docker run -d --name orders-api alpine sleep 900`). Record its host PID and parent shim PID. Add `{"live-restore": true}` to `/etc/docker/daemon.json`, `sudo systemctl restart docker`, and then prove the container survived with the **same PID** and the **same shim**. Write 3 sentences explaining, in terms of the component chain, exactly why the container survived a daemon restart.

---

## Mental model checkpoint

1. Name the four components in order from what you type to what touches the kernel.
2. What protocol runs on each link: CLI↔dockerd, dockerd↔containerd, containerd/shim↔runc?
3. Which two files are the physical sockets, and which directory holds container binaries?
4. Why does `runc` not appear in `pstree` for a running container?
5. What is the shim's job, and why does its existence allow you to restart `dockerd` without downtime?
6. Is `containerd` a child process of `dockerd`? What is it instead?
7. Why did "Kubernetes removing dockershim" NOT break Docker-built images?

---

## Quick reference card

| Component | Role | Talks to next via | Persists? |
|---|---|---|---|
| `docker` (CLI) | Thin client you type into | HTTP/REST over `/var/run/docker.sock` | Exits after command |
| `dockerd` (daemon) | Build, network, volumes, image mgmt, exposes API | gRPC over `/run/containerd/containerd.sock` | Long-lived |
| `containerd` | Container lifecycle & image management, tasks | OCI CLI to runc, spawns shim | Long-lived |
| `containerd-shim-runc-v2` | Per-container supervisor; holds stdio, reports exit | Invokes `runc` | One per container, long-lived |
| `runc` | OCI runtime: does the actual namespaces/cgroups/pivot_root, then exits | Kernel syscalls | **One-shot, exits** |
| `/var/run/docker.sock` | CLI↔dockerd channel | HTTP/REST/JSON | — |
| `/run/containerd/containerd.sock` | dockerd↔containerd channel | gRPC | — |
| `config.json` (OCI bundle) | The spec runc executes | OCI Runtime Spec | Per container |

---

## When would I use this at work?

1. **Reading Docker errors correctly.** "Cannot connect to the Docker daemon" instantly means the CLI↔dockerd link, so you check the daemon/socket permissions — not your Dockerfile or app. You fix in seconds instead of guessing.

2. **Planning safe host maintenance.** When patching Docker on production nodes, you know running containers are held by shims and survive a `dockerd` restart with `live-restore`, so you schedule rolling daemon upgrades with zero customer downtime.

3. **Understanding the Kubernetes transition.** When your platform moves K8s to containerd directly (bypassing dockerd), you can reassure the team that images and behavior are unchanged because everything below dockerd — containerd, runc, OCI — is identical (Topic 29).

---

## Connected topics

- **Study before:**
  - **Topic 01 — What Is a Container**: established that a container is a process wearing kernel costumes; this topic shows *which programs* dress it.
  - **Topic 02 — Linux Foundations of Containers**: the namespaces/cgroups/overlayfs that **runc** actually creates in step 8.
- **Study after:**
  - **Topic 04 — Images and Layers**: what dockerd/containerd pull and stack into the `rootfs/` handed to runc.
  - **Topic 05 — Your First Container**: a full `docker run` lifecycle trace building on this chain.
  - **Topic 29 — Kubernetes Architecture**: how the kubelet uses the **CRI** to talk to containerd directly, skipping `dockerd`.
