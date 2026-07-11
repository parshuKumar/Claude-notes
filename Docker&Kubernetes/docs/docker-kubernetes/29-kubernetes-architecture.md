# 29 — Kubernetes Architecture

## Section: Kubernetes Foundations

---

## ELI5 — The Simple Analogy

Think of Kubernetes as a **large restaurant chain run from a head office**.

- The **head office** never cooks a single meal. It holds the master plan, takes
  orders, decides which kitchen makes what, and checks that every branch is doing
  what it was told. This is the **control plane** — the brain.
- The **kitchens** are the branches where food is actually cooked. Each kitchen
  has a **manager** who reads the day's orders from head office and makes sure the
  cooks produce exactly those dishes. These are the **worker nodes** — the muscle.

Inside the head office there are four specialists:
- The **front desk** (**API server**) — the only door. Everyone, inside and out,
  must talk through the front desk. Nobody wanders into the back rooms.
- The **filing cabinet** (**etcd**) — the single source of truth. Every order,
  every plan, written down here. Only the front desk is allowed to open it.
- The **table planner** (**scheduler**) — when a new order comes in with no
  kitchen assigned, it decides *which kitchen* should make it, based on who has
  space and the right equipment.
- The **floor supervisors** (**controller manager**) — a room full of people each
  obsessed with one rule: "there must always be 3 taco plates." They constantly
  compare the plan to reality and fix any difference.

In each kitchen:
- The **kitchen manager** (**kubelet**) reads which dishes this kitchen must make
  and tells the cooks to start.
- The **cooks** (**container runtime / containerd**) actually make the food.
- The **waiter/traffic director** (**kube-proxy**) makes sure customers are routed
  to whichever kitchen currently has their dish ready.

Nobody talks to the filing cabinet except the front desk. Every action flows
through that one door. Hold that image — it explains almost everything about how
Kubernetes behaves.

---

## The Linux kernel feature underneath

Like Topic 28, there is **no single kernel primitive** that "is" Kubernetes
architecture. But each component sits on a very concrete OS/network mechanism.
Knowing them turns the diagram from a cartoon into an engineering system:

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Component            │ The real OS / network mechanism underneath            │
├──────────────────────┼───────────────────────────────────────────────────────┤
│ API server           │ An HTTPS server (Go net/http) listening on TCP :6443. │
│                      │ TLS mutual auth via certs in /etc/kubernetes/pki.     │
│                      │ Handles thousands of long-lived "watch" connections   │
│                      │ using epoll on sockets (one goroutine per stream).    │
├──────────────────────┼───────────────────────────────────────────────────────┤
│ etcd                 │ A key-value store using the RAFT consensus protocol    │
│                      │ over gRPC/TCP. Data persisted to a boltdb mmap'd file  │
│                      │ at /var/lib/etcd/member/snap/db, fsync'd on write so   │
│                      │ a power loss cannot lose committed state.             │
├──────────────────────┼───────────────────────────────────────────────────────┤
│ scheduler            │ A plain client of the API server. Pure userspace      │
│                      │ decision code; it changes nothing on the host — it    │
│                      │ only writes spec.nodeName back via an HTTP PATCH.     │
├──────────────────────┼───────────────────────────────────────────────────────┤
│ controller-manager   │ Many control loops (goroutines), each an HTTP client  │
│                      │ that watches objects and PATCHes them. No kernel calls.│
├──────────────────────┼───────────────────────────────────────────────────────┤
│ kubelet              │ A local agent that talks to the container runtime over │
│                      │ a UNIX domain socket (/run/containerd/containerd.sock)│
│                      │ using the CRI gRPC API. THIS is where kernel action    │
│                      │ finally happens (indirectly).                         │
├──────────────────────┼───────────────────────────────────────────────────────┤
│ container runtime    │ containerd → runc → clone() with namespace flags      │
│ (containerd/CRI)     │ (CLONE_NEWNS/NET/PID/…) + cgroup writes. THE actual    │
│                      │ Linux kernel container from Topics 01–02.             │
├──────────────────────┼───────────────────────────────────────────────────────┤
│ kube-proxy           │ Programs the kernel's netfilter subsystem: writes     │
│                      │ iptables rules (or IPVS via the kernel's IP Virtual   │
│                      │ Server) so a virtual Service IP DNATs to a real pod.  │
└──────────────────────┴───────────────────────────────────────────────────────┘
```

The single most important architectural fact, expressed as a rule:

> **Only the API server reads/writes etcd. Everything else is an HTTP client of
> the API server.** The scheduler, controllers, kubelets — none of them open the
> database. They all watch and patch objects through one guarded door.

That design is *why* Kubernetes is extensible and observable: every change is an
HTTP object mutation you can watch, audit, and reason about. The kernel does the
container work at the very end of the chain; the rest is disciplined HTTP over a
consistent database.

---

## What is this?

Kubernetes architecture is the set of components that together turn "desired
state in a database" into "containers running on machines." It splits into two
planes: the **control plane** (API server, etcd, scheduler, controller-manager —
the decision-makers) and the **worker nodes** (kubelet, kube-proxy, container
runtime — the doers). A request travels from the API server, into etcd, out to
controllers and the scheduler that decide, and finally to a kubelet that makes it
real.

---

## Why does it matter for a backend developer?

When something goes wrong in production — and it will — you cannot debug a black
box. Knowing the architecture turns cryptic symptoms into a checklist:

- **"My pod is stuck in `Pending` forever."** With the architecture in your head,
  you instantly know: the API server accepted it (it exists in etcd), but the
  **scheduler** couldn't place it. So you look at scheduler-level reasons —
  insufficient CPU/memory, node taints, affinity. You debug the *right component*.
- **"`kubectl` commands hang or error."** You know every command hits the **API
  server** on :6443. So the problem is API-server reachability or auth
  (`~/.kube/config`), not your YAML.
- **"A pod is `Running` but gets no traffic."** The container runtime + kubelet
  did their job (it's Running), so the fault is in **kube-proxy**/Service
  networking, not in scheduling or the image.
- **"The whole cluster is acting weird / read-only."** You know the one stateful
  component is **etcd**. If etcd loses quorum, the API server can't write, and
  the cluster freezes. You check etcd health first.

Without the architecture, every incident is "Kubernetes is broken, help." With
it, every symptom points at exactly one component to inspect. That is the
difference between a 4-hour outage and a 15-minute fix.

---

## The physical reality

On a real cluster, the control plane components themselves run as **static pods**
— containers the kubelet starts directly from manifest files on disk, before the
cluster is even "up." This is the bootstrapping trick that lets Kubernetes run
Kubernetes.

```
CONTROL-PLANE MACHINE
  /etc/kubernetes/manifests/           ← kubelet watches this dir on boot
      ├── kube-apiserver.yaml          ← starts the API server as a pod
      ├── etcd.yaml                    ← starts etcd as a pod
      ├── kube-scheduler.yaml
      └── kube-controller-manager.yaml
  /etc/kubernetes/pki/                 ← the TLS trust store for everything
      ├── ca.crt / ca.key              ← the cluster's root Certificate Authority
      ├── apiserver.crt / .key
      └── etcd/                        ← separate CA + certs for etcd
  /var/lib/etcd/member/snap/db         ← THE database file. Your whole cluster's
                                         desired+actual state is in this bolt file.

EVERY WORKER MACHINE
  /var/lib/kubelet/
      ├── config.yaml                  ← kubelet's own config
      ├── pods/<pod-uid>/              ← per-pod working dir: mounted volumes,
      │       ├── volumes/             │  projected secrets/configmaps,
      │       └── containers/          │  container metadata
      /var/lib/kubelet/pki/            ← kubelet's client cert to the API server
  /var/lib/containerd/                 ← image layers + container rootfs (Topic 04)
  /run/containerd/containerd.sock      ← CRI socket: kubelet → containerd
  /var/log/pods/<ns>_<pod>_<uid>/      ← stdout/stderr log files kubectl logs reads
  /var/log/containers/                 ← symlinks into /var/log/pods (for agents)
```

You can literally see the control plane running as containers:

```bash
# On a control-plane node, the runtime shows the "brain" as containers:
crictl ps | grep -E "apiserver|etcd|scheduler|controller"
#   CONTAINER    IMAGE                    STATE    NAME
#   a1b2...      kube-apiserver            Running  kube-apiserver
#   c3d4...      etcd                      Running  etcd
#   e5f6...      kube-scheduler            Running  kube-scheduler
#   ...          kube-controller-manager   Running  kube-controller-manager
```

The profound bit: **the control plane is just more containers**, managed by the
same kubelet that manages your app pods. Kubernetes is turtles (containers) all
the way down.

---

## How it works — step by step

Let's trace one full request: `kubectl apply -f orders-api-deployment.yaml`
asking for **3 replicas of `orders-api`**, and follow every component it touches.
This is the canonical "how a request flows" walkthrough.

```
   YOU                                                       (numbers = steps)
    │  kubectl apply -f orders-api-deployment.yaml
    ▼
 ┌─────────────┐  (1) HTTPS POST/PATCH to :6443, TLS client cert from kubeconfig
 │ API SERVER  │─────────────────────────────────────────────────────────────┐
 └─────────────┘                                                              │
    │ (2) authenticate → authorize (RBAC) → admission control → validate      │
    │ (3) write the Deployment object                                          │
    ▼                                                                          │
 ┌─────────────┐                                                              │
 │    etcd     │  now stores: Deployment{replicas:3}. Nothing runs yet.        │
 └─────────────┘                                                              │
    ▲ (only the API server ever touches etcd)                                 │
    │                                                                          │
    │ (4) Deployment controller (in controller-manager) is WATCHING via a     │
    │     long-lived HTTP stream on the API server. It sees desired=3, has=0. │
    │     It creates a ReplicaSet object → API server → etcd.                 │
    │ (5) ReplicaSet controller sees RS wants 3, sees 0 Pods. Creates 3 Pod   │
    │     objects (status: Pending, spec.nodeName: "" ) → API server → etcd.  │
    │                                                                          │
    │ (6) SCHEDULER is watching for Pods with empty nodeName. For each of the │
    │     3 pods: filter nodes (enough CPU/RAM? tolerations? affinity?) then  │
    │     score them, pick the best, and PATCH spec.nodeName=worker-2 back    │
    │     through the API server → etcd. (Scheduler runs NOTHING itself.)     │
    │                                                                          │
    ▼                                                                          │
 ┌───────────────────────────── WORKER NODE (worker-2) ──────────────────────┐│
 │  (7) kubelet is watching for Pods where spec.nodeName == its own name.    ││
 │      It sees "orders-api-xyz is mine." Time to make it real:              ││
 │                                                                            ││
 │      kubelet ──CRI gRPC over /run/containerd/containerd.sock──► containerd ││
 │      (8) containerd pulls the image (if absent), sets up the pod sandbox   ││
 │          (the "pause" container that holds the network namespace), then   ││
 │          calls runc → clone() with namespace flags + cgroup writes.       ││
 │      (9) CNI plugin gives the pod an IP (Topic 40).                        ││
 │      (10) container is RUNNING. kubelet reports status="Running" back UP   ││
 │           to the API server → etcd. Now desired=actual=3.                  ││
 │                                                                            ││
 │  (11) kube-proxy (on every node) has been watching Services + Endpoints.  ││
 │       When a Service selects these pods, kube-proxy writes iptables/IPVS  ││
 │       rules so the Service's virtual IP load-balances to the 3 pod IPs.   ││
 └────────────────────────────────────────────────────────────────────────────┘
```

Key properties this trace reveals:

- **The API server is the hub.** Every arrow passes through it. Components never
  talk to each other directly; they communicate by *changing objects* that others
  are watching. This is called the **hub-and-spoke** or **level-triggered**
  design (more in Topic 30).
- **Each component does one job and hands off.** Scheduler decides *where*, never
  *runs*. Kubelet runs, never *decides where*. Clean separation.
- **Everything is a watch, not a poll-loop hammering the DB.** Components open a
  long-lived HTTP "watch" and the API server pushes changes to them — efficient
  and near-real-time.

If any step's component is unhealthy you get a *specific* stuck state: no
scheduler → pods stuck `Pending`; no kubelet on a node → pods assigned there
never start; no kube-proxy → running pods get no traffic. The architecture is a
debugging map.

---

## Exact syntax breakdown

You mostly *observe* architecture rather than configure it. Here are the exact
commands to inspect each component, annotated.

```
kubectl get componentstatuses
│       │   │
│       │   └── legacy check of scheduler/controller-manager/etcd health
│       └── "get" = read objects from the API server
└── prints "Healthy" per control-plane component (deprecated but illustrative)
```

```
kubectl get pods -n kube-system -o wide
│                 │              │
│                 │             └── "-o wide" adds NODE + IP columns so you see
│                 │                 WHERE each control-plane pod runs
│                 └── the namespace the control plane lives in
└── lists the control plane running AS PODS (apiserver, etcd, scheduler, ...)
```

```
kubectl describe node worker-2
│        │       │    │
│        │       │   └── the node to inspect
│        │       └── the object kind
│        └── "describe" = human-readable dump incl. Conditions, Capacity,
│            Allocatable, and which pods the kubelet placed here
└── shows the worker's view: how much CPU/RAM the scheduler can still use
```

The one field that *drives* the whole scheduler→kubelet handoff:

```yaml
# Inside a Pod object, written by different components:
spec:
  nodeName: worker-2       # ← written by the SCHEDULER (step 6). Empty until
  │                        #   scheduled. When kubelet on worker-2 sees its own
  │                        #   name here, it starts the pod (step 7).
  └── This single field is the baton passed from "decide where" to "make it run."

status:
  phase: Running           # ← written by the KUBELET (step 10) after the
  │                        #   container is actually up. This is the ACTUAL state.
  hostIP: 10.0.0.12        # the node's IP
  podIP: 10.244.2.5        # the IP the CNI plugin assigned (step 9)
```

And the CRI socket — the exact boundary where "Kubernetes" ends and "container
runtime" begins:

```
/run/containerd/containerd.sock
│                │
│                └── a UNIX domain socket file; kubelet dials it and speaks the
│                    CRI gRPC protocol (RunPodSandbox, CreateContainer, ...)
└── on-disk socket. Swap containerd for CRI-O here and Kubernetes doesn't care —
    that's the whole point of the CRI abstraction.
```

---

## Example 1 — basic

Let's inspect a real (local) cluster's architecture piece by piece with
`orders-api` deployed, so each component becomes tangible.

```bash
# Spin up a 3-node cluster (1 control-plane + 2 workers) with kind
cat > kind-3node.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane    # runs API server, etcd, scheduler, controller-mgr
  - role: worker           # runs kubelet, kube-proxy, containerd
  - role: worker
EOF
kind create cluster --name arch-lab --config kind-3node.yaml

# 1) SEE THE CONTROL PLANE (as pods) — the "brain"
kubectl get pods -n kube-system -o wide
#   etcd-arch-lab-control-plane                 Running   <- filing cabinet
#   kube-apiserver-arch-lab-control-plane       Running   <- front desk
#   kube-scheduler-arch-lab-control-plane       Running   <- table planner
#   kube-controller-manager-...                 Running   <- floor supervisors
#   kube-proxy-... (one per node)               Running   <- traffic director
#   coredns-...                                 Running   <- cluster DNS

# 2) SEE THE WORKERS — the "muscle"
kubectl get nodes
#   arch-lab-control-plane   Ready   control-plane
#   arch-lab-worker          Ready   <none>
#   arch-lab-worker2         Ready   <none>

# 3) DEPLOY orders-api and watch the scheduler spread it across workers
kubectl create namespace orders
kubectl create deployment orders-api -n orders --image=nginx --replicas=4
kubectl get pods -n orders -o wide   # look at the NODE column — pods land on
                                     # arch-lab-worker AND arch-lab-worker2

# 4) PROVE the scheduler wrote nodeName, the kubelet wrote status
kubectl get pod -n orders <pod-name> -o jsonpath='{.spec.nodeName}{"\n"}'
#   arch-lab-worker2          <- scheduler's decision
kubectl get pod -n orders <pod-name> -o jsonpath='{.status.phase}{"\n"}'
#   Running                   <- kubelet's report
```

Every abstract box from the diagram is now a real process you can point at.

---

## Example 2 — production scenario

**The situation.** It's Monday. A teammate deployed a new `orders-api` version
Friday. This morning, `orders-api` pods are stuck: `kubectl get pods -n orders`
shows **4 pods in `Pending`**, none `Running`. Customers can still be served by
the old ReplicaSet, but no new capacity is coming up. You are on call. Use the
architecture as a decision tree.

**Step 1 — Which component is responsible for `Pending`?**
A pod is `Pending` *after* the API server accepted it and *before* a kubelet
started it. That gap is owned by the **scheduler**. So the object exists in etcd
but has no node. Confirm:

```bash
kubectl get pod -n orders orders-api-abc -o jsonpath='{.spec.nodeName}'
#   (empty)   <- confirmed: scheduler has NOT placed it
```

**Step 2 — Ask the scheduler why (it records its reason as an Event).**

```bash
kubectl describe pod -n orders orders-api-abc
#   Events:
#   Warning  FailedScheduling  0/2 nodes are available:
#            2 Insufficient memory.
```

**Step 3 — Read the message with the architecture in mind.** "Insufficient
memory" means the scheduler's *filter* stage rejected every worker: the new
version bumped the pod's memory **request** to `2Gi`, and each worker only has
`1.5Gi` allocatable. The scheduler is doing its job perfectly — it refuses to
place a pod where the kubelet couldn't honor the request. This is **not** a bug;
it's the control plane protecting node stability.

**Step 4 — Confirm at the node level.**

```bash
kubectl describe node arch-lab-worker | grep -A5 Allocatable
#   Allocatable:
#     memory:  1536Mi        <- less than the 2Gi request → no fit
```

**Step 5 — Fix at the right layer.** Two correct options, both architectural:
- Lower the memory **request** in the Deployment to something the nodes can
  satisfy (Topic 44), then `kubectl apply`. The scheduler re-evaluates and places
  the pods.
- Or add worker capacity (bigger/more nodes) so the scheduler has somewhere to
  put a `2Gi` pod.

**The lesson:** because you knew `Pending` = scheduler's domain, you went
straight to `describe pod` → Events → node Allocatable, and found the cause in
minutes. Someone without the architecture would restart pods, re-pull images, and
blame the app — none of which touch the actual (scheduling) problem.

---

## Common mistakes

**Mistake 1 — Thinking components talk directly to each other or to etcd.**

```
Belief:  "The scheduler tells the kubelet to start the pod."
Reality: The scheduler only PATCHes spec.nodeName via the API server. The
         kubelet independently WATCHES the API server and reacts.
```
**Root cause:** imagining Kubernetes as a call graph instead of a shared
database everyone watches. The truth: **all coordination is through objects on
the API server.** Fixing the wrong mental model here misleads every later debug.
Right model: components are decoupled watchers of one source of truth.

**Mistake 2 — Editing etcd directly to "fix" the cluster.**

```
$ ETCDCTL_API=3 etcdctl put /registry/pods/... <hand-edited json>
   # cluster behaves bizarrely, validation/admission bypassed
```
**Root cause:** bypassing the API server means skipping validation, admission
control, and the change-notification machinery. Objects can become corrupt or
inconsistent. Right: **never touch etcd directly**; always go through
`kubectl`/the API server, which is the *only* sanctioned writer.

**Mistake 3 — Believing a `Running` pod means "my app works."**

```
$ kubectl get pods -n orders
   orders-api-abc   1/1   Running   ← but requests still 502
```
`Running` only means the **kubelet + container runtime** started the process.
It says nothing about whether the app is *ready* to serve, or whether
**kube-proxy**/Service routing points at it. **Root cause:** conflating the
runtime's job (start the container) with readiness + networking (other
components/Topics 35, 43). Right: check readiness probes and the Service's
Endpoints, not just phase.

**Mistake 4 — Assuming losing the control plane kills running apps.**

```
Belief:  "The control-plane VM rebooted, so all my pods are down."
Reality: Existing pods keep running — kubelet + containerd on workers are
         autonomous. You just can't SCHEDULE, SCALE, or SELF-HEAL until the
         control plane returns.
```
**Root cause:** thinking the control plane is in the request path of your app's
traffic. It is not — it's in the *management* path. Right: the data plane
(running pods, kube-proxy) survives a control-plane outage; only changes stall.

**Mistake 5 — Ignoring etcd quorum in HA clusters.**

```
Error:  etcdserver: request timed out  /  the apiserver goes read-only
```
etcd uses Raft and needs a **majority** of members alive (e.g. 2 of 3). Run 2
etcd members and lose 1 → no majority → no writes → the API server can't accept
changes. **Root cause:** not understanding etcd needs an *odd* member count for
quorum. Right: run 3 or 5 etcd members so one failure still leaves a majority.

---

## Hands-on proof

```bash
# 1. Create a cluster and confirm the control plane runs AS containers
kind create cluster --name arch-proof
docker exec arch-proof-control-plane crictl ps | \
  grep -E "kube-apiserver|etcd|scheduler|controller"
#   proves the "brain" is just containers managed by the node's kubelet

# 2. Prove every kubectl call hits the API server on :6443
kubectl config view --minify | grep server
#   server: https://127.0.0.1:PORT   <- the ONE door everything uses

# 3. Watch the API server as the hub: open a watch, then change something
kubectl get pods -A -w &            # long-lived WATCH stream to the API server
kubectl create deployment demo --image=nginx --replicas=2
#   watch stream prints ADDED/MODIFIED events in real time = push, not poll
kill %1

# 4. See etcd hold the desired state (read it via the API server, the RIGHT way)
kubectl get deployment demo -o jsonpath='{.spec.replicas}{"\n"}'   # 2 (desired)
kubectl get deployment demo -o jsonpath='{.status.replicas}{"\n"}' # 2 (actual)

# 5. Prove scheduler → kubelet handoff via spec.nodeName
POD=$(kubectl get pod -l app=demo -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD -o jsonpath='SCHEDULED_TO={.spec.nodeName}  PHASE={.status.phase}{"\n"}'

# 6. Prove the data plane is autonomous: stop the control plane, apps keep running
docker stop arch-proof-control-plane
kubectl get pods            # kubectl FAILS (API server down)...
docker exec arch-proof-control-plane echo 2>/dev/null || \
  echo "control plane down, but the demo containers are STILL running on workers"
docker start arch-proof-control-plane   # bring it back

# 7. Clean up
kind delete cluster --name arch-proof
```

Step 6 is the eye-opener: `kubectl` dies but your workloads don't. Management
plane ≠ data plane.

---

## Practice exercises

### Exercise 1 — easy
On a kind cluster, run `kubectl get pods -n kube-system -o wide`. **List** each
control-plane component you see and, next to each, write its one-line job from
this doc (front desk / filing cabinet / table planner / floor supervisors /
traffic director). Then run `kubectl get nodes` and identify which node the
control-plane pods run on and why the workers don't run them.

### Exercise 2 — medium
Create a Deployment with 3 replicas, then trace the object chain that the
architecture creates: run `kubectl get deployment,replicaset,pods` and draw the
ownership tree (Deployment → ReplicaSet → Pods). For one pod, print both
`spec.nodeName` and `status.phase` with `-o jsonpath`. **Explain** which
component wrote each of those two fields and at which step of the request flow.

### Exercise 3 — hard (production simulation)
Reproduce the Example 2 outage deliberately. Deploy `orders-api` with a memory
**request** larger than any node's allocatable memory (check with
`kubectl describe node`). Observe the pods stuck `Pending`. Using ONLY
`kubectl describe pod` and `kubectl describe node`, write a short incident note
that: (a) names the responsible component, (b) quotes the exact Event message,
(c) explains at the architecture level why the scheduler refused to place the
pod, and (d) gives two correct fixes at different layers. Then apply one fix and
confirm the pods reach `Running`.

---

## Mental model checkpoint

1. Which single component is allowed to read and write **etcd**, and why does
   that rule matter?
2. Walk the full path of `kubectl apply` for a 3-replica Deployment, naming every
   component and what it writes.
3. What is the exact job of the **scheduler**, and what does it *not* do?
4. Which field is the "baton" handed from the scheduler to the kubelet?
5. If the **control plane** goes down, what happens to already-running pods? What
   stops working?
6. A pod is `Running` but serves no traffic. Which components are *not* the
   problem, and which one probably is?
7. Why does etcd need an odd number of members (3 or 5) rather than 2 or 4?

---

## Quick reference card

| Component | Plane | Job | Key detail |
|---|---|---|---|
| API server | Control | The only door; validates + persists all objects | HTTPS :6443; only writer of etcd |
| etcd | Control | Stores desired + actual state | Raft consensus; needs quorum; `/var/lib/etcd` |
| scheduler | Control | Decides *which node* a pod runs on | Writes `spec.nodeName`; runs nothing |
| controller-manager | Control | Runs control loops that reconcile state | Deployment/ReplicaSet/Node controllers |
| kubelet | Worker | Starts/monitors pods assigned to its node | Talks to runtime via CRI socket |
| container runtime (containerd) | Worker | Actually runs containers | containerd → runc → namespaces/cgroups |
| kube-proxy | Worker | Programs Service routing | iptables/IPVS DNAT to pod IPs |
| CRI | Worker | Runtime abstraction boundary | gRPC over `containerd.sock` |
| `kubectl get pods -n kube-system` | — | See the control plane as pods | Add `-o wide` for nodes |
| `kubectl describe pod` | — | Read a pod's Events | Shows scheduler `FailedScheduling` reasons |

---

## When would I use this at work?

**1. On-call debugging.** Every incident starts with "which component owns this
symptom?" `Pending` → scheduler. `ImagePullBackOff` → kubelet/registry.
`CrashLoopBackOff` → your container. No route to a `Running` pod → kube-proxy/
Service. The architecture is your triage map.

**2. Capacity and reliability planning.** Knowing etcd needs quorum and the
control plane is separate from the data plane tells you how to size an HA
cluster (3 control-plane nodes, odd etcd count) and reassure stakeholders that a
control-plane blip won't drop live traffic.

**3. Runtime and platform decisions.** Understanding the CRI boundary
lets you reason about swapping containerd for CRI-O, why Docker-shim was removed,
and how managed platforms (EKS/GKE/AKS) run the control plane for you while you
own the worker nodes.

---

## Connected topics

**Study before this:**
- **Topic 03** — Docker architecture (CLI → daemon → containerd → runc). The
  kubelet plugs into that same containerd via the CRI.
- **Topic 28** — What Kubernetes is and why. This topic opens the boxes that
  topic named.
- **Topics 01–02** — namespaces/cgroups, the kernel work that containerd+runc do
  at the very end of the request flow.

**Study after this:**
- **Topic 30** — The Reconciliation Loop. We saw controllers "watch and act"
  here; next we study that loop as the single idea behind all of Kubernetes.
- **Topic 31–32** — kubectl and manifests: how you actually talk to the API
  server and what an object looks like.
- **Topic 35 / 40** — Services and cluster networking: the kube-proxy and CNI
  parts of this diagram, in full depth.
```
