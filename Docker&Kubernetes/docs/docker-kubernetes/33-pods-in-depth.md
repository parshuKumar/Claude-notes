# 33 — Pods in Depth
## Section: Kubernetes in Practice

## ELI5 — The Simple Analogy

Imagine you rent an apartment. Inside that apartment there might be one person living alone, or a couple of roommates. But here's the rule of this building: **you rent the apartment, not the individual bedrooms.** When the landlord moves someone in or out, they move the *whole apartment* at once. The roommates inside share the same address, the same front door, the same kitchen, and the same shared closet.

A **Pod** is that apartment. The **containers** inside it are the roommates. Kubernetes never rents you a bare container — it always rents you the apartment (the Pod) with the container(s) living inside. The roommates share one address (one IP), one front door (the network), and one shared closet (some volumes). If the landlord tears down the apartment, everyone inside goes with it.

That's the single most important idea in Kubernetes: **the Pod, not the container, is the unit you deploy, schedule, and throw away.**

## The Linux kernel feature underneath

A Pod is not a Linux kernel object. There is no `pod()` syscall. A Pod is a *convention built on top of Linux namespaces* (Remember from Topic 02 — the kernel trinity of namespaces, cgroups, overlay filesystem).

Here is the trick. Remember that a container is just a process wrapped in its own set of namespaces: its own network namespace (its own IP, its own `lo`, its own routing table), its own IPC namespace, its own PID namespace, its own mount namespace. Normally each container gets a *fresh* set of all of these, so two containers cannot see each other's network or shared memory.

A Pod bends one rule: **the containers in a Pod share some namespaces but not others.**

```
         Container A                 Container B
   ┌───────────────────┐      ┌───────────────────┐
   │  own MOUNT ns     │      │  own MOUNT ns     │   ← NOT shared (own filesystem)
   │  own PID ns*      │      │  own PID ns*      │   ← usually NOT shared
   ├───────────────────┴──────┴───────────────────┤
   │        SHARED network namespace                │   ← same IP, same ports, same lo
   │        SHARED IPC namespace                    │   ← same POSIX shared memory
   │        SHARED UTS namespace                    │   ← same hostname
   └────────────────────────────────────────────────┘
```

So who *owns* those shared namespaces? If container A owned the network namespace and container A crashed and restarted, the IP would vanish and come back — chaos. So Kubernetes creates one tiny extra container whose only job is to *hold the shared namespaces open*. It is called the **pause container** (also called the "infra container" or "sandbox").

The pause container does literally nothing. Its entire program is: create the namespaces, then call `pause()` and sleep forever. Every real container in the Pod is then started with `--net=container:<pause-id>` and `--ipc=container:<pause-id>`, meaning "don't make a new network namespace, *join the pause container's* one." Because the pause container never crashes and never restarts, the IP and the shared namespaces stay stable even while your real app container crashes and restarts fifty times.

You can literally see the pause image on any Kubernetes node:

```
registry.k8s.io/pause:3.9     ← ~700 KB, does nothing but hold namespaces
```

That is the physical secret of a Pod: **one invisible pause process anchoring a shared network + IPC namespace, and your real containers joining it.**

## What is this?

A **Pod** is the smallest thing Kubernetes can create and schedule. It is a group of one or more containers that share a network namespace (one IP address, one set of ports), can share storage volumes, and are always scheduled together onto the same node and live and die as a unit.

You almost never create Pods by hand in production. Higher-level objects (Deployments — Topic 34) create Pods for you. But you *must* understand Pods deeply, because everything above them just makes copies of a Pod template.

## Why does it matter for a backend developer?

If you come from Docker, your instinct is "I run a container." In Kubernetes that instinct is wrong and it will bite you:

- **Networking surprises.** Your `orders-api` container and a logging sidecar in the same Pod both see `localhost:3000`. That is not a coincidence — they share one network namespace. If you try to run two containers that both bind port 3000 in the same Pod, the second one crashes with "address already in use." Understanding the shared namespace explains *why* instantly.
- **Crash behavior.** When people say "the pod restarted," they usually mean a *container* inside the pod restarted while the pod (and its IP) stayed the same. Confusing pod-level and container-level restarts leads to hours of wrong debugging.
- **Storage that magically appears in two containers.** Init containers and sidecars pass data through a shared volume. If you don't know volumes are Pod-scoped, this looks like magic.
- **Startup ordering.** Your API must not start before the database migration runs. Init containers solve this — but only if you know they exist.

## The physical reality

When a Pod named `orders-api-7d9f` is running on a node, here is what actually exists on that machine.

**In etcd (the cluster's database, Topic 29):** a key holding the Pod's desired + observed state.

```
/registry/pods/orders/orders-api-7d9f
   → a serialized object: which node, which containers, which image,
     phase=Running, podIP=10.244.1.37, and more
```

**On the node, processes (real PIDs):**

```
$ ps aux | grep -E 'pause|node'
65001  /pause                              ← the pause container, holds namespaces
65014  node /app/server.js                 ← your orders-api container
65090  /bin/log-shipper --tail /var/log    ← a sidecar container (if you have one)
```

**On the node, the shared network namespace (one file the containers all point at):**

```
$ ls -l /proc/65001/ns/net /proc/65014/ns/net
lrwxrwxrwx net -> net:[4026532567]          ← pause container's net namespace
lrwxrwxrwx net -> net:[4026532567]          ← SAME inode! node container joined it
```

Notice `net:[4026532567]` is **identical** for both PIDs. That single number is the proof that the two processes share one network namespace — one IP, one `lo`, one set of ports.

**On the node, the container root filesystems (Topic 04, overlay2):**

```
/var/lib/docker/overlay2/<hashA>/merged   ← orders-api container's own filesystem
/var/lib/docker/overlay2/<hashB>/merged   ← sidecar container's own filesystem
```

These are *separate* — mount namespaces are NOT shared. Each container sees its own files.

**On the node, any shared Pod volume (emptyDir):**

```
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~empty-dir/shared-data/
   → this one directory is bind-mounted into BOTH containers,
     so init container writes here and the app container reads here
```

## How it works — step by step

Let's trace a single Pod from "you typed apply" to "it's Running." Assume this manifest asks for one `orders-api` container.

1. **You run** `kubectl apply -f pod.yaml`. kubectl POSTs the JSON to the API server (Topic 29).
2. **API server validates** the object (is `image` set? is the name legal?) and writes it to etcd under `/registry/pods/orders/orders-api`. At this moment the Pod's `phase` is `Pending` and `nodeName` is empty.
3. **The scheduler is watching** for Pods with no `nodeName`. It sees ours. It runs its filtering + scoring (does a node have enough CPU/memory? any affinity rules?) and picks, say, `node-2`. It writes `nodeName: node-2` back to etcd. Still `Pending`.
4. **The kubelet on node-2 is watching** for Pods assigned to it. It sees ours. Now the real work starts on the node.
5. **kubelet asks the container runtime (containerd via CRI, Topic 03)** to create the *Pod sandbox*. The runtime:
   - creates a fresh network namespace,
   - starts the **pause container** inside it,
   - calls the **CNI plugin** (Topic 40) which assigns the Pod an IP (e.g. `10.244.1.37`) and wires a veth pair into the namespace.
6. **kubelet pulls the image** `orders-api:1.4.0` if not cached (Topic 04 pull mechanics), unpacking layers into overlay2.
7. **kubelet starts your container** and tells the runtime "put it in the pause container's network + IPC namespace." Your `node server.js` process now shares the IP `10.244.1.37`.
8. **kubelet runs any probes** (startup/readiness/liveness — Topic 43). Once the app is up, the Pod `phase` becomes `Running` and it is marked Ready.
9. **kubelet continuously reports status** back to the API server, which stores it in etcd. This is the reconciliation loop (Topic 30) in action: kubelet keeps the actual node state matching the desired state in etcd forever.

If your container crashes, kubelet restarts *just that container* (per `restartPolicy`) — the pause container and the IP stay. That is why the IP is stable across container crashes.

## Exact syntax breakdown

Here is a complete, real single-container Pod. Every field annotated.

```yaml
apiVersion: v1                    # Pod is a "core" object → group is empty → just "v1"
kind: Pod                         # the object type we are creating
metadata:
  name: orders-api               # Pod name, unique within the namespace
  namespace: orders              # which namespace this lives in (our running example)
  labels:
    app: orders-api              # key=value tags used by Services/Deployments to find it
spec:
  containers:                    # a LIST — one or more containers in this Pod
    - name: orders-api           # container name, unique within the Pod
      image: orders-api:1.4.0    # the image to run (Topic 04)
      ports:
        - containerPort: 3000    # documentation + tooling hint; the port your app listens on
      env:
        - name: PORT             # environment variable name
          value: "3000"          # its value (must be a string, hence the quotes)
      resources:                 # cgroup limits (Topic 25 / 44)
        requests:                # what the scheduler reserves for this container
          cpu: "100m"            # 100 millicores = 0.1 of one CPU
          memory: "128Mi"        # 128 mebibytes
        limits:                  # hard ceiling; exceed memory → OOMKilled
          cpu: "500m"
          memory: "256Mi"
  restartPolicy: Always          # Always | OnFailure | Never — how kubelet handles exits
```

Now the char-by-char breakdown of the two most confusing values.

```
          cpu: "100m"
               │  │└──── "m" = milli = 1/1000. So this is 1000ths of a CPU.
               │  └───── 100 of those thousandths.
               └──────── => 0.1 of one logical CPU core reserved.
```

```
          memory: "128Mi"
                   │  │└── "i" makes it BINARY. Mi = mebibyte = 1024*1024 bytes.
                   │  └─── "M" without the i would be 1000*1000 (decimal). Almost always use Mi.
                   └────── 128 of them = 134,217,728 bytes.
```

```
      - containerPort: 3000
        │              └──── the TCP port your process inside the container listens on.
        └───────────────────  Note: this does NOT publish the port. It is informational.
                              To reach it from outside you need a Service (Topic 35).
```

## Example 1 — basic

A minimal `orders-api` Pod you can run and inspect. Every line commented.

```yaml
# file: pod-basic.yaml
apiVersion: v1
kind: Pod
metadata:
  name: orders-api          # we'll refer to it as "orders-api"
  namespace: orders         # our example namespace (create it first)
  labels:
    app: orders-api         # a tag; not required for a bare Pod but good habit
spec:
  containers:
    - name: web             # the single container's name
      image: node:20-alpine # base image; we'll run a one-liner server
      command: ["node", "-e",                         # override the image's CMD
        "require('http').createServer((_,r)=>r.end('orders ok')).listen(3000)"]
      ports:
        - containerPort: 3000
  restartPolicy: Always     # if node process exits, kubelet restarts the container
```

Run it:

```bash
kubectl create namespace orders            # make the namespace once
kubectl apply -f pod-basic.yaml            # create the Pod
kubectl get pod orders-api -n orders -w    # watch it go Pending → ContainerCreating → Running
```

Prove the app answers from *inside the cluster* (a Pod has no external route yet — that needs a Service, Topic 35):

```bash
kubectl exec -it orders-api -n orders -- wget -qO- localhost:3000
# → orders ok
```

## Example 2 — production scenario: init container + sidecar

**The situation.** Your team's `orders-api` must not start serving traffic until the database schema is migrated. Migrating from inside the API container is racy (five replicas all running migrations at once). Also, security wants request logs shipped to a central collector, but the app only writes logs to a local file. You solve both *inside one Pod*:

- an **init container** runs the DB migration to completion *before* the app container even starts,
- a **sidecar container** tails the app's log file and ships it out, sharing a volume with the app.

```yaml
# file: pod-prod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: orders-api
  namespace: orders
  labels:
    app: orders-api
spec:
  volumes:
    - name: logs                       # a Pod-scoped scratch volume...
      emptyDir: {}                     # ...created empty on the node, gone when Pod dies
  initContainers:                      # run to completion, IN ORDER, before app containers
    - name: db-migrate
      image: orders-api:1.4.0          # same image, but run the migration command
      command: ["node", "migrate.js"]  # exits 0 on success; Pod stays Pending until it does
      env:
        - name: DATABASE_URL
          value: "postgres://postgres:5432/orders"
  containers:                          # the long-running containers
    - name: web                        # your actual API
      image: orders-api:1.4.0
      ports:
        - containerPort: 3000
      volumeMounts:
        - name: logs                   # mount the shared volume...
          mountPath: /var/log/app      # ...at this path; app writes access.log here
    - name: log-shipper                # the SIDECAR
      image: busybox:1.36
      command: ["sh", "-c", "tail -F /var/log/app/access.log"]  # follow the same file
      volumeMounts:
        - name: logs                   # SAME volume, so it sees what "web" writes
          mountPath: /var/log/app
```

What actually happens on the node, in order:

```
1. pause container starts, Pod gets IP 10.244.1.37
2. init container db-migrate runs → applies schema → exits 0     [Pod phase: Pending]
3. ONLY NOW do web + log-shipper start together                  [Pod phase: Running]
4. web binds localhost:3000 ; log-shipper CANNOT bind 3000 too   (shared network ns!)
5. web writes /var/log/app/access.log ; log-shipper tails the SAME file (shared volume)
```

This is the **sidecar pattern**: a helper container that augments the main container (logging, proxy, metrics) by sharing the Pod's network and volumes. The main container stays simple and unaware.

Watch the ordering yourself:

```bash
kubectl apply -f pod-prod.yaml
kubectl get pod orders-api -n orders -o wide
# STATUS column shows: Init:0/1  → PodInitializing → Running
kubectl logs orders-api -n orders -c db-migrate     # -c picks the container
kubectl logs orders-api -n orders -c log-shipper -f # follow the shipped logs
```

## Common mistakes

**Mistake 1 — Two containers binding the same port in one Pod.**

```
Wrong: container "web" listens on 3000, container "metrics" also listens on 3000.
```
Error you'll see in `kubectl logs`:
```
Error: listen EADDRINUSE: address already in use 0.0.0.0:3000
```
**Root cause:** both containers share ONE network namespace (the pause container's). They are effectively two processes on the same host trying to bind the same TCP port. Only one can.
**Right:** give each a different port (`3000` and `9090`), exactly like two processes on one machine.

**Mistake 2 — Expecting a bare Pod to come back after a node dies.**

```
You created a Pod directly (kind: Pod). The node crashes. The Pod is gone forever.
```
`kubectl get pods` shows nothing, or the Pod stuck in `Terminating`/`Unknown`.
**Root cause:** a bare Pod has no controller watching over it. Nothing reconciles it back. Pods are *cattle, not pets* — mortal by design.
**Right:** never run bare Pods in production. Use a Deployment (Topic 34) so a ReplicaSet recreates the Pod elsewhere.

**Mistake 3 — Putting the main app in `initContainers`.**

```
You put "node server.js" (a long-running server) as an init container.
```
The Pod is stuck forever:
```
STATUS: Init:0/1     (never progresses)
```
**Root cause:** init containers must **run to completion** (exit 0). A web server never exits, so the init phase never finishes, so the app containers never start.
**Right:** only put finite tasks (migrations, waiting for a dependency, fetching config) in `initContainers`. Long-running processes go in `containers`.

**Mistake 4 — Assuming containers in a Pod share a filesystem by default.**

```
Init container writes /data/config.json. App container reads /data/config.json → ENOENT.
```
```
Error: ENOENT: no such file or directory, open '/data/config.json'
```
**Root cause:** mount namespaces are NOT shared between containers. Each has its own filesystem. They only share directories you explicitly declare as a `volume` and `volumeMount` in both.
**Right:** declare an `emptyDir` volume and mount it into both containers at `/data`.

**Mistake 5 — Confusing Pod restart with container restart.**

```
"My pod restarted 200 times!" — actually RESTARTS column counts CONTAINER restarts.
```
```
NAME         READY   STATUS             RESTARTS   AGE
orders-api   1/1     Running            213        2d
```
**Root cause:** the Pod (and its IP) has lived for 2 days; the *container inside* has crash-looped 213 times. kubelet restarts the container per `restartPolicy`; the Pod object stays.
**Right:** read the RESTARTS column as *container* restarts. Investigate the container's `kubectl logs --previous`.

## Hands-on proof

Run these right now on any cluster (minikube/kind/real) to *see* the shared namespace and the pause container.

```bash
# 1. Create the namespace and a 2-container Pod.
kubectl create namespace orders
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: proof, namespace: orders }
spec:
  containers:
    - name: a
      image: busybox:1.36
      command: ["sh","-c","sleep 3600"]
    - name: b
      image: busybox:1.36
      command: ["sh","-c","sleep 3600"]
EOF

# 2. Both containers see the SAME IP — proof of a shared network namespace.
kubectl exec proof -n orders -c a -- ip addr show eth0 | grep inet
kubectl exec proof -n orders -c b -- ip addr show eth0 | grep inet
#   → identical inet address in both. One IP, two containers.

# 3. Container a starts a listener; container b reaches it over localhost.
kubectl exec proof -n orders -c a -- sh -c 'nc -lk -p 8080 -e echo hi &' 
kubectl exec proof -n orders -c b -- sh -c 'echo | nc localhost 8080'
#   → "hi". Container b talked to container a via localhost = shared net namespace.
```

To see the pause container itself, SSH/`minikube ssh` onto the node:

```bash
minikube ssh
docker ps | grep pause          # or: crictl ps | grep pause
#   → you'll see registry.k8s.io/pause:3.x running for your Pod
ls -l /proc/$(pgrep -f 'sleep 3600' | head -1)/ns/net    # your container's net ns inode
ls -l /proc/$(pgrep -f '/pause'      | head -1)/ns/net    # pause's net ns inode
#   → the inode numbers MATCH. That is the whole trick.
```

## Practice exercises

### Exercise 1 — easy
Create a single-container `orders-api` Pod in namespace `orders` using `node:20-alpine`. Confirm it reaches `Running`, then use `kubectl describe pod` to find (a) the node it landed on, (b) its Pod IP, (c) the events showing "Scheduled", "Pulling", "Started".

### Exercise 2 — medium
Build a Pod with one **init container** that runs `sh -c 'until nc -z postgres 5432; do sleep 1; done'` and a main `orders-api` container. Deploy it *without* a Postgres service present. Observe the Pod stuck in `Init:0/1`. Then create a dummy Pod/Service named `postgres` on port 5432 and watch the init container complete and the app start. Explain in one sentence why the app never started earlier.

### Exercise 3 — hard (production simulation)
Reproduce the "logging sidecar" pattern: main container writes a timestamp every second to `/var/log/app/out.log` on a shared `emptyDir`; sidecar `tail -F`s the same file. Then, in a second terminal, `kubectl exec` into the main container and `rm` the log file. Explain what happens to the sidecar's `tail -F` (hint: `-F` vs `-f`), and what happens to the shared volume's data if you `kubectl delete pod` (hint: `emptyDir` lifecycle). Bonus: add a second port-binding conflict on purpose and capture the exact `EADDRINUSE` error.

## Mental model checkpoint

Answer from memory:

1. What is the pause container and why does it exist?
2. Which namespaces do containers in a Pod share, and which do they not?
3. Why does a Pod's IP survive a container crash?
4. What must be true for an init container to let the app containers start?
5. How do two containers in a Pod exchange files, given they don't share a filesystem?
6. Why should you never run a bare `kind: Pod` in production?
7. In `RESTARTS: 213`, what exactly restarted 213 times — the Pod or a container?

## Quick reference card

| Field / command | What it does | Key detail |
|---|---|---|
| `kind: Pod` | Declares the smallest deployable unit | You rarely write these directly; Deployments generate them |
| `spec.containers[]` | Long-running containers in the Pod | Share one network + IPC namespace via the pause container |
| `spec.initContainers[]` | Run-to-completion setup containers | Run in order, must exit 0, block app containers until done |
| `spec.volumes[]` + `volumeMounts` | Pod-scoped shared storage | Only way two containers share files (mount ns aren't shared) |
| `restartPolicy` | How kubelet handles container exit | `Always` (default) / `OnFailure` / `Never` — Pod-level, not per-container |
| `containerPort` | Documents the listening port | Informational only; does NOT expose the port externally |
| `kubectl logs <pod> -c <container>` | Logs of a specific container | `-c` is required for multi-container Pods; `--previous` for the last crash |
| pause container | Holds the shared namespaces | `registry.k8s.io/pause`, ~700KB, calls `pause()` forever |
| Pod phase | `Pending`/`Running`/`Succeeded`/`Failed`/`Unknown` | Phase is Pod-level; container states are separate |

## When would I use this at work?

1. **Database migration before rollout.** Put the migration as an init container so all replicas wait for one clean schema change before serving — no racing migrations.
2. **Log/metrics sidecar.** Ship `orders-api` access logs or expose a `/metrics` endpoint via a sidecar (e.g. a Prometheus exporter) without touching the app code — the sidecar shares the Pod's network and log volume.
3. **Debugging a crash-looping service.** Read the RESTARTS column and `kubectl logs --previous` correctly, knowing it's the container crashing while the Pod/IP stays — so you look at app logs, not at scheduling.

## Connected topics

- **Study before:** Topic 02 (namespaces & cgroups — the kernel basis), Topic 29 (K8s architecture — scheduler/kubelet), Topic 30 (reconciliation loop), Topic 32 (YAML & manifests).
- **Study after:** Topic 34 (Deployments — what actually creates Pods in production), Topic 35 (Services — how to reach a Pod's IP), Topic 43 (probes — liveness/readiness that gate a Pod's phase), Topic 44 (resource requests/limits — the cgroups that shape scheduling).
