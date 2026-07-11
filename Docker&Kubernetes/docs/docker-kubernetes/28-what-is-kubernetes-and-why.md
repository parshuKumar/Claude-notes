# 28 — What Is Kubernetes and Why

## Section: Kubernetes Foundations

---

## ELI5 — The Simple Analogy

Imagine you own one food truck. You park it, you cook, you serve. If the truck
breaks down, you notice instantly because you are standing right there. You fix
it. This is **Docker Compose** — one machine, a handful of containers, you
watching over them.

Now imagine you are asked to run **200 food trucks across the whole city**,
24 hours a day. You cannot personally stand next to every truck. So you hire a
**dispatch office**. You walk in and tell the dispatcher:

> "I always want **10 taco trucks** and **4 burger trucks** running in the
> downtown zone. Keep it that way no matter what."

You do **not** say *"drive truck #47 to 5th street and turn on the grill."*
You describe the **end result you want**, and the dispatcher figures out the how.
If a truck's engine dies at 3 a.m., the dispatcher automatically sends a
replacement so there are still 10 taco trucks. If lunchtime traffic spikes, the
dispatcher rolls out more trucks. You went home and slept. The office kept your
promise.

**Kubernetes is that dispatch office for containers.** You tell it the desired
state ("I want 5 copies of `orders-api` running"), and it relentlessly makes
reality match that description across a whole fleet of machines — replacing dead
containers, spreading them across servers, and healing the system without you.

---

## The Linux kernel feature underneath

Kubernetes is **not a kernel feature**. This is the first big mental shift as you
cross from Docker into Kubernetes. Everything you learned in Topics 01–02 —
namespaces, cgroups, the overlay filesystem — still runs the actual containers.
Kubernetes does **not** replace any of that. On each machine, when a container
finally starts, it is still `runc` calling `clone()` with namespace flags and
writing cgroup files, exactly as in Topic 03.

So what *is* Kubernetes underneath? It is a **distributed control system** built
on three OS/network mechanisms, not one kernel primitive:

```
┌──────────────────────────────────────────────────────────────────┐
│  What Kubernetes is actually built on                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. A consistent distributed database  →  etcd                     │
│     Uses the Raft consensus algorithm over TCP so that many        │
│     machines agree on ONE version of "what should be true."        │
│     This is where your desired state physically lives.             │
│                                                                    │
│  2. HTTP + a control loop            →  the API server + watches    │
│     Every component talks to one HTTP/JSON (protobuf) API over      │
│     TLS. Components "watch" for changes using long-lived HTTP        │
│     streams (kernel: epoll on TCP sockets holding open GETs).      │
│                                                                    │
│  3. The same Linux container kernel  →  namespaces + cgroups        │
│     features you already know          (Topics 01–02) via the CRI. │
│     Kubernetes NEVER touches clone()/cgroups directly — it asks    │
│     the container runtime (containerd) to do it.                   │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

The "magic" of self-healing is not a kernel trap or a syscall. It is a
**software loop**: components read the desired state from etcd, look at the
actual state of the world, and act to close the gap. We study that loop in depth
in Topic 30. For now, hold this line in your head:

> Kubernetes is a giant, distributed **while-loop** that keeps comparing
> "what you asked for" to "what is actually running" and fixing the difference.

The kernel makes one container real. Kubernetes makes **thousands of containers
across many machines match a written promise**.

---

## What is this?

Kubernetes (often written **k8s** — "k", 8 letters, "s") is an open-source
**container orchestrator**: a system that runs and manages containers across a
group of machines (a *cluster*) for you. You describe the desired state of your
app in YAML, hand it to Kubernetes, and Kubernetes continuously works to make
the cluster match that description — scheduling containers onto machines,
restarting them when they crash, scaling them up and down, and networking them
together.

In one sentence: **Docker runs a container; Kubernetes keeps a fleet of
containers running the way you promised, on many machines, without you
watching.**

---

## Why does it matter for a backend developer?

You already got `orders-api` + Postgres + Redis running with Docker Compose
(Topics 18–22). Compose is fantastic — for **one machine**. The moment your
service becomes something a company depends on, Compose runs out of road. Here is
what breaks, and why you feel it as a backend developer:

**1. One machine is a single point of failure.**
Your Compose host reboots for a kernel patch at 2 a.m. Every container on it dies
at once. `orders-api` is down until the box comes back and something runs
`docker compose up`. There is no second machine ready to take the load.

**2. `restart: always` only heals crashes, not the machine.**
Compose's `restart` policy (Topic 19) restarts a container if the *process*
dies. It does nothing if the whole *host* dies, runs out of memory, or loses
disk. There is no way for a dead machine to restart its own containers.

**3. Scaling is manual and dumb.**
Black Friday hits. You need 20 copies of `orders-api`. Compose can do
`--scale orders-api=20`, but only on that one machine — until its CPU and RAM are
exhausted. There is no notion of "spread these 20 copies across 5 servers and
add more servers if needed."

**4. Deploys cause downtime.**
`docker compose up -d` with a new image stops the old container and starts the
new one. For a few seconds, requests fail. No built-in rolling update, no
"bring the new version up, health-check it, then retire the old one."

**5. No load balancing across replicas.**
Even if you run 3 copies, Compose gives you no stable virtual address that
spreads traffic across them and automatically stops sending traffic to a copy
that just died.

Kubernetes exists precisely to solve these five things: **spread work across many
machines, self-heal, scale horizontally, deploy without downtime, and load
balance** — all from a declarative description. As a backend engineer, this is
the difference between "my code works on my laptop" and "my service survives a
dead server during a traffic spike at 3 a.m. while I sleep."

---

## The physical reality

When Compose runs, the whole world is one machine. When Kubernetes runs, the
world is a **cluster**: a set of machines with defined roles.

```
                        THE CLUSTER (physical reality)

   ┌───────────────────────── CONTROL PLANE (the "brain") ─────────────────────┐
   │  machine: control-plane-1                                                  │
   │                                                                            │
   │   ┌────────────┐   ┌───────┐   ┌───────────┐   ┌────────────────────┐     │
   │   │ API server │   │ etcd  │   │ scheduler │   │ controller-manager │     │
   │   │ (HTTPS)    │   │ (DB)  │   │           │   │                    │     │
   │   └────────────┘   └───────┘   └───────────┘   └────────────────────┘     │
   │        ▲   the ONLY component that talks to etcd is the API server        │
   └────────┼───────────────────────────────────────────────────────────────── ┘
            │  HTTPS (TLS, port 6443)
   ┌────────┼─────────────────┐  ┌───────────────────────┐  ┌──────────────────┐
   │  WORKER NODE 1           │  │  WORKER NODE 2        │  │  WORKER NODE 3   │
   │  machine: worker-1       │  │  machine: worker-2    │  │  machine: worker-3│
   │                          │  │                       │  │                  │
   │  ┌────────┐ ┌──────────┐ │  │ ┌────────┐            │  │ ┌────────┐        │
   │  │kubelet │ │kube-proxy│ │  │ │kubelet │  ...       │  │ │kubelet │  ...   │
   │  └────────┘ └──────────┘ │  │ └────────┘            │  │ └────────┘        │
   │  ┌────────────────────┐  │  │ ┌──────────────────┐  │  │ ┌──────────────┐  │
   │  │ containerd (CRI)   │  │  │ │ containerd       │  │  │ │ containerd   │  │
   │  │  ├─ orders-api pod │  │  │ │  ├─ postgres pod │  │  │ │  ├─ redis pod│  │
   │  │  └─ orders-api pod │  │  │ │  └─ orders-api   │  │  │ │  └─ orders-api│ │
   │  └────────────────────┘  │  │ └──────────────────┘  │  │ └──────────────┘  │
   └──────────────────────────┘  └───────────────────────┘  └──────────────────┘
```

Where things physically live on disk:

```
On the control-plane machine:
  /var/lib/etcd/                 ← etcd's database files (the desired state)
                                   member/snap/db  ← the actual boltdb file
  /etc/kubernetes/manifests/     ← static pod manifests that boot the control
                                   plane itself (api-server.yaml, etcd.yaml…)
  /etc/kubernetes/pki/           ← all the TLS certs components use to trust
                                   each other

On every worker node:
  /var/lib/kubelet/              ← kubelet's state: which pods it should run
      /pods/<pod-uid>/           ← per-pod dirs (volumes, mounts, logs)
  /var/lib/containerd/           ← the actual container/image storage
                                   (the overlay2-style layers from Topic 04)
  /var/log/pods/                 ← stdout/stderr log files per container
  /run/containerd/containerd.sock← the CRI socket kubelet talks to
```

Notice the key fact: **your desired state is a set of rows in etcd on the control
plane.** The running containers are on the workers. Kubernetes' whole job is to
keep those two in sync. If you `kubectl get pods`, you are reading etcd via the
API server. If a container is running on `worker-2`, that is because etcd said it
should be, and a chain of components made it happen.

---

## How it works — step by step

Let's trace exactly what happens when you tell Kubernetes: *"Run 3 copies of
`orders-api`."* You will use a **Deployment** (studied fully in Topic 34); here we
care about the flow, not the object.

```
$ kubectl apply -f orders-api-deployment.yaml   # asks for 3 replicas
```

1. **kubectl reads the YAML** and sends it as an HTTP `POST`/`PATCH` to the
   **API server** at `https://<control-plane>:6443`, over TLS, authenticated
   with your client certificate/token from `~/.kube/config`.

2. **The API server validates** the object (is the YAML well-formed? do you have
   permission via RBAC — Topic 53?) and then **writes it to etcd**. Nothing runs
   yet. All that exists is a **record of intent**: "desired: 3 orders-api pods."

3. **The Deployment controller** (inside controller-manager) is *watching* etcd
   through the API server. It sees a Deployment wanting 3 replicas but 0 pods
   exist. It creates a **ReplicaSet**, which in turn creates **3 Pod objects** in
   etcd — still just records, still not running. Each Pod is now "Pending" with
   **no node assigned**.

4. **The scheduler** is watching for Pods with no node. It sees 3 unscheduled
   Pods. For each one it runs a scoring algorithm — which node has enough
   CPU/RAM? which spreads load? — and **writes the chosen node name back into the
   Pod object** in etcd (`spec.nodeName: worker-2`). Still no container running;
   the scheduler only *decides*, it does not launch.

5. **The kubelet on each chosen node** is watching for Pods assigned to *its*
   node. `worker-1`'s kubelet sees "Pod orders-api-abc is yours." It now does the
   real work: it calls **containerd via the CRI** (Container Runtime Interface)
   to pull the image and start the container. containerd calls `runc`, which
   calls `clone()` with namespace flags and writes cgroup files — the exact
   kernel mechanics from Topics 01–02.

6. **The container is now running.** The kubelet reports the Pod's status
   ("Running") back to the API server, which writes it to etcd. Now
   **desired = actual = 3 running pods.**

7. **kube-proxy** on each node has been watching Services (Topic 35). It programs
   **iptables/IPVS rules** so a single stable virtual IP load-balances across all
   3 pods. Traffic to `orders-api` now spreads across the replicas.

Now the self-healing part — the reason Kubernetes exists:

8. **A worker node dies.** `worker-1` loses power. Its kubelet stops reporting.
   After a timeout, the node controller marks the node "NotReady" and its pods as
   gone. Now etcd says **desired = 3, actual = 2.** The ReplicaSet controller
   sees the gap and creates **1 new Pod object**. The scheduler places it on a
   healthy node. That node's kubelet starts it. **Back to 3.** You did nothing.

That loop — *see the gap between desired and actual, act to close it* — is the
heartbeat of Kubernetes. Topic 30 is entirely about it.

---

## Exact syntax breakdown

You interact with Kubernetes mostly through **YAML manifests** and **kubectl**.
Here is the smallest meaningful manifest that captures "keep 3 of my API
running." (Full Deployment anatomy is Topic 34; this is the shape.)

```yaml
apiVersion: apps/v1          # which API group/version defines this object
kind: Deployment             # the TYPE of object we are declaring
metadata:
  name: orders-api           # the object's name in the "orders" namespace
  namespace: orders
spec:                        # the DESIRED STATE — the whole point
  replicas: 3                # "I always want 3 copies running"
  selector:
    matchLabels:
      app: orders-api        # which pods this Deployment owns
  template:                  # the blueprint for each pod it creates
    metadata:
      labels:
        app: orders-api
    spec:
      containers:
        - name: orders-api
          image: myregistry/orders-api:1.4.0
          ports:
            - containerPort: 3000
```

Field-by-field, with pointers:

```
apiVersion: apps/v1
│           │
│           └── group "apps", version "v1". Deployments live in the "apps"
│               group; a bare Pod would be just "v1" (the core group).
└── EVERY manifest starts with this. It tells the API server which schema to
    use to validate the rest of the file.

kind: Deployment
│     │
│     └── The object type. Determines which controller will act on it.
│         (Deployment → Deployment controller → ReplicaSet → Pods.)
└── Together apiVersion+kind uniquely identify the object's schema.

spec:
│
└── "Specification" = the state YOU want. Kubernetes objects are split into
    spec (what you want) and status (what actually is). You write spec;
    Kubernetes writes status. This split IS the declarative model.

  replicas: 3
  │         │
  │         └── The number Kubernetes will relentlessly maintain. Delete a
  │             pod and it comes back. This one integer is "self-healing."
  └── Lives under spec, so it is DESIRED. The matching status.replicas is
      the ACTUAL count Kubernetes reports back.
```

The command that submits it:

```
kubectl apply -f orders-api-deployment.yaml
│       │     │  │
│       │     │  └── the manifest file to send
│       │     └── "-f" = from this file
│       └── "apply" = declarative: "make the cluster match this file."
│           Re-running it is safe; it computes the diff and only changes what
│           differs. (Contrast "kubectl create", which errors if it exists.)
└── the Kubernetes CLI; reads ~/.kube/config to find the API server + creds
```

---

## Example 1 — basic

The clearest way to *feel* the difference is to see the same app in both worlds.
Here is `orders-api` in **Docker Compose** (what you know), then in
**Kubernetes** (where we are going).

**Docker Compose — one machine, imperative-ish (Topic 20):**

```yaml
# docker-compose.yml — runs on ONE host only
services:
  orders-api:
    image: myregistry/orders-api:1.4.0
    ports:
      - "3000:3000"
    restart: always          # restarts the PROCESS if it crashes...
                             # ...but cannot survive the host dying
```

```bash
docker compose up -d --scale orders-api=3   # 3 copies, all on THIS box
```

If this machine dies, all 3 die and nothing brings them back.

**Kubernetes — a cluster, declarative:**

```yaml
# orders-api-deployment.yaml — runs across the whole cluster
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: orders
spec:
  replicas: 3                # desired state; enforced across ALL nodes
  selector:
    matchLabels: { app: orders-api }
  template:
    metadata:
      labels: { app: orders-api }
    spec:
      containers:
        - name: orders-api
          image: myregistry/orders-api:1.4.0
          ports:
            - containerPort: 3000
```

```bash
kubectl apply -f orders-api-deployment.yaml   # tell the "dispatch office"
kubectl get pods -n orders                    # watch 3 pods appear, possibly
                                              # on 3 DIFFERENT machines
```

Kill any pod (`kubectl delete pod <name>`) and a replacement appears within
seconds — because you *promised* 3, and Kubernetes keeps promises. Kill a whole
node and the pods it held are rescheduled onto surviving nodes. That is the
entire pitch of this topic, made concrete.

---

## Example 2 — production scenario

**The situation.** You are the on-call backend engineer for `orders-api`. It runs
on a single beefy VM with Docker Compose: `orders-api` (3 processes), Postgres,
Redis. It has served you well for a year.

**Friday, 6 p.m.** A marketing email goes out. Traffic 8×. The VM's CPU pins at
100%. `orders-api` response times blow past your 500 ms SLA. You SSH in and run
`docker compose up --scale orders-api=6`. It helps for two minutes — then the VM
runs out of RAM, the kernel **OOM-killer** (Topic 25) starts killing containers,
**including Postgres**. Now the API is fully down. You have no second machine to
move load to. You spend the evening firefighting on one box.

**What Kubernetes would have done differently:**

1. **Horizontal scaling across machines.** With a `HorizontalPodAutoscaler`
   (Topic 45), Kubernetes watches CPU and, seeing 8× load, scales `orders-api`
   from 3 to (say) 15 pods — *spread across 5 worker nodes*, not crammed onto
   one. No single machine is overwhelmed.

2. **The scheduler respects resource requests.** Each pod declares it needs, say,
   `250m` CPU and `256Mi` RAM (Topic 44). The scheduler only places a pod on a
   node with room. Postgres, sitting in its own pod with a guaranteed memory
   request, is **never** starved by a stampede of API pods — they physically
   cannot be scheduled onto its node once it is full.

3. **Self-healing.** If a pod does get OOM-killed, the ReplicaSet notices
   `desired > actual` and starts a replacement on a node with capacity — within
   seconds, automatically, while you are still reading the alert.

4. **A stable Service address** (Topic 35) load-balances across whatever number
   of healthy `orders-api` pods currently exist, adding and removing them from
   rotation as they come and go. Clients never point at a specific pod.

**The lesson for a backend dev:** Compose gives you *"my containers run."*
Kubernetes gives you *"my service stays within SLA during an 8× spike, survives a
dead node, and heals itself while I sleep."* Topic 28's job is only to make you
want that. Topics 29–54 teach you how to build it.

---

## Common mistakes

**Mistake 1 — "Kubernetes replaces Docker."**

```
Belief:  "If I move to Kubernetes, I delete my Dockerfiles."
```
Wrong. Kubernetes **runs** container images — the exact images you built in
Topics 06–11. It does not build them and it does not replace the container
runtime. On each node, containerd + runc still start containers using the same
namespaces/cgroups. **Root cause of the confusion:** conflating "how a container
is built and run on one host" (Docker's job) with "how many containers are kept
running across many hosts" (Kubernetes' job). Right mental model: *Docker builds
and runs one container; Kubernetes orchestrates many.*

**Mistake 2 — Treating Kubernetes imperatively, like Compose.**

```
$ kubectl run orders-api --image=myregistry/orders-api:1.4.0
$ # later, pod is gone after a node reboot, and you're confused
```
Running a bare pod imperatively gives you a pod with **no controller watching
it**. Nothing restores it if it dies with the node. **Root cause:** you skipped
the declarative desired-state object (Deployment) that owns and heals the pod.
Right: declare a Deployment and `kubectl apply` it, so a controller guarantees
the replica count.

**Mistake 3 — Expecting scaling to work on one node.**

```
Error you eventually hit:
  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.
```
You built a "cluster" of exactly one node and asked for 20 replicas. The
scheduler cannot place them — there is no capacity, and no other machine.
**Root cause:** Kubernetes scales *horizontally across nodes*; with one node you
have the same ceiling as Compose. Right: run multiple worker nodes so the
scheduler has somewhere to place pods.

**Mistake 4 — Thinking pods have stable identities/IPs like VMs.**

```
Belief:  "I'll hardcode the pod's IP 10.244.1.7 in my Redis client."
Reality: the pod gets rescheduled, comes back as 10.244.3.2, your app breaks.
```
Pods are **cattle, not pets** — they are created and destroyed constantly, each
time with a new IP. **Root cause:** carrying a single-server habit into a fleet.
Right: talk to a **Service** (Topic 35), which gives a stable virtual IP/DNS name
that always routes to the current healthy pods.

**Mistake 5 — Believing Kubernetes makes a single app "highly available" by
magic.**

```
Belief:  "I run one replica in Kubernetes, so it's HA now."
```
One replica on one node is still a single point of failure — if that pod's node
dies, you have an outage until reschedule. **Root cause:** confusing "Kubernetes
*can* provide HA" with "Kubernetes *automatically* makes everything HA." Right:
you must *ask* for it — multiple replicas, spread across nodes (anti-affinity),
plus readiness probes so traffic only hits healthy pods.

---

## Hands-on proof

You do not need a 200-machine data center. Install a local single-node cluster
and watch self-healing with your own eyes. (We use `kind` — Kubernetes IN Docker
— which runs a whole cluster as containers on your laptop.)

```bash
# 1. Install a local cluster tool (macOS example; see kind.sigs.k8s.io)
brew install kind kubectl

# 2. Create a real Kubernetes cluster (this spins up control plane + node
#    as Docker containers — proving K8s rides ON containers, not vice versa)
kind create cluster --name orders-lab

# 3. Prove the cluster exists and see its one node
kubectl get nodes
#   NAME                        STATUS   ROLES           AGE   VERSION
#   orders-lab-control-plane    Ready    control-plane   40s   v1.31.x

# 4. Prove the control plane itself runs as pods (the physical reality!)
kubectl get pods -n kube-system
#   you'll see: etcd-..., kube-apiserver-..., kube-scheduler-...,
#               kube-controller-manager-... — the "brain" from the diagram

# 5. Declare desired state: 3 replicas of a simple web server
kubectl create deployment web --image=nginx --replicas=3
kubectl get pods -w        # watch 3 pods go Pending -> Running

# 6. THE MONEY SHOT — self-healing. Delete a pod and watch it return.
kubectl delete pod <one-pod-name-from-above>
kubectl get pods           # still 3! A new pod (new name) took its place.

# 7. Prove desired vs actual is stored, not imperative:
kubectl get deployment web -o wide
#   READY   UP-TO-DATE   AVAILABLE   ← Kubernetes maintaining desired=3

# 8. See where the desired state physically lives (etcd), via the API:
kubectl get deployment web -o yaml | grep -A2 "replicas:"
#   spec.replicas: 3   ← what you asked for (in etcd)
#   status.replicas: 3 ← what actually is (reported by controllers)

# 9. Clean up
kind delete cluster --name orders-lab
```

Step 6 is the whole topic in one command: you deleted a running thing and
Kubernetes *put it back*, because reality no longer matched your promise.

---

## Practice exercises

### Exercise 1 — easy
Using `kind`, create a cluster and a Deployment named `orders-api` with
`--image=nginx --replicas=2`. Run `kubectl get pods`, note the two pod names,
then delete one. Run `kubectl get pods` again. **Write down:** how many pods now
exist, and whether the new pod has the same name as the one you deleted. Explain
in one sentence *why* the count returned to 2.

### Exercise 2 — medium
Scale your `orders-api` Deployment to 5 replicas two different ways and observe
the difference in mindset:
1. Imperatively: `kubectl scale deployment orders-api --replicas=5`.
2. Declaratively: `kubectl edit deployment orders-api` and change
   `spec.replicas` to 5.
Then run `kubectl get deployment orders-api -o yaml` and find both
`spec.replicas` and `status.replicas`. **Explain** what each number means and why
they are stored separately. Which approach mirrors how you'd manage a real
cluster from Git, and why?

### Exercise 3 — hard (production simulation)
Simulate a node failure. With a multi-node kind cluster
(`kind create cluster --config` with 3 nodes — look up the config format),
deploy `orders-api` with 6 replicas. Run
`kubectl get pods -o wide` and record **which node** each pod is on. Now
"kill" a worker node by stopping its Docker container:
`docker stop orders-lab-worker`. Watch `kubectl get pods -o wide` over the next
few minutes. **Document:** how long until Kubernetes notices the node is
NotReady, what happens to the pods that were on it, and where the replacements
land. Then explain, referencing the reconciliation loop, *which component
detected the gap and which component fixed it.*

---

## Mental model checkpoint

Answer these from memory before moving on:

1. In one sentence, what is the difference between what **Docker** does and what
   **Kubernetes** does?
2. Where does your **desired state** physically live in a Kubernetes cluster?
3. Name the five things Docker Compose does *not* give you that pushed the
   industry to Kubernetes.
4. When a pod dies and comes back, is it the *same* pod? What stays stable so
   clients aren't affected?
5. Is Kubernetes a Linux kernel feature? If not, what three mechanisms is it
   actually built on?
6. What does "declarative" mean here, and how does the `spec` vs `status` split
   embody it?
7. Why does running 1 replica in Kubernetes *not* automatically make your app
   highly available?

---

## Quick reference card

| Concept / Command | What it does | Key detail |
|---|---|---|
| Kubernetes (k8s) | Orchestrates containers across many machines | Not a kernel feature; a distributed control loop |
| Cluster | The set of machines K8s manages | Control plane (brain) + worker nodes (muscle) |
| Desired state | What you *want* running | Written to etcd via the API server; the `spec` |
| Actual state | What *is* running | Reported by kubelet; the `status` |
| Self-healing | Auto-replace dead pods | Controller sees `desired > actual`, creates pods |
| Horizontal scaling | Add more *copies* across nodes | Contrast vertical (bigger machine) |
| Declarative | Describe the end result, not steps | `kubectl apply -f` is safe to re-run |
| `kubectl apply -f file` | Make cluster match this manifest | Computes a diff; idempotent |
| `kubectl get pods` | List pods = read etcd via API server | Add `-o wide` to see nodes, `-w` to watch |
| Deployment | Object that maintains a replica count | Self-heals via a ReplicaSet (Topic 34) |
| Service | Stable virtual IP over changing pods | Solves "pods have no stable IP" (Topic 35) |

---

## When would I use this at work?

**1. Your side-project graduates to a real service.**
`orders-api` on one Compose VM was fine at 100 users. At 100,000 users with a
paying SLA, you need it to survive a dead host and a traffic spike without you
babysitting it. You move to Kubernetes so the platform self-heals and
autoscales — turning "the box is down" from a 2 a.m. page into a non-event.

**2. Your team ships multiple times a day.**
Ten engineers merging to main need deploys that don't cause downtime. Kubernetes
Deployments (Topic 34, 46) give rolling updates: bring the new version up,
health-check it, drain the old — zero dropped requests. Compose's stop-then-start
can't.

**3. You run many services, not one.**
Once you have `orders-api`, `payments-api`, `inventory-api`, plus their Postgres
and Redis, packing them efficiently across a shared fleet — and letting the
scheduler bin-pack them by resource request — is exactly the problem Kubernetes
was built for. You describe what each service needs; the cluster figures out
where everything runs.

---

## Connected topics

**Study before this:**
- **Topic 01–02** — What a container is; namespaces/cgroups. Kubernetes runs
  these; it does not replace them.
- **Topic 03** — Docker architecture (CLI → daemon → containerd → runc).
  Kubernetes' kubelet talks to that same containerd via the CRI.
- **Topics 18–22** — Docker Compose. This is the thing Kubernetes scales beyond;
  you must feel Compose's limits to appreciate why K8s exists.
- **Topic 25** — Resource limits and the OOM-killer, referenced in the
  production scenario.

**Study after this:**
- **Topic 29** — Kubernetes Architecture. We opened the boxes here; next we look
  inside each control-plane and worker component and trace a request end-to-end.
- **Topic 30** — The Reconciliation Loop. The "keep desired = actual" heartbeat,
  studied as the single idea behind *all* Kubernetes behavior.
- **Topic 34 / 35 / 45** — Deployments, Services, and the HorizontalPodAutoscaler
  — the concrete objects that deliver the self-healing, stable-networking, and
  autoscaling promises made in this topic.
```
