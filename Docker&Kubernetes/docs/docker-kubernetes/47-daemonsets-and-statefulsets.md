# 47 — DaemonSets and StatefulSets
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Think about a big office building with many floors (each floor = a **node** in your cluster).

A **Deployment** is like the manager saying "I need 4 receptionists somewhere in the building — I don't care which floors." Kubernetes drops them wherever there's room. They're interchangeable; if one quits, hire any replacement and put them anywhere.

A **DaemonSet** is the manager saying "I need **one** fire warden on **every single floor**, and if we add a new floor, that floor automatically gets a fire warden too." It's not about *how many* — it's about *one per floor*, guaranteed, everywhere. That's what you want for things that must watch each machine: log collectors, monitoring agents.

A **StatefulSet** is the manager hiring **named, ranked employees who are not interchangeable**: "Alice is Database-0, Bob is Database-1, Carol is Database-2." Alice always keeps her same desk (her same disk), her same phone number (her same network name), and she must be hired *first*, before Bob, who comes before Carol. If Alice leaves, her *replacement* still gets called "Database-0," sits at Alice's exact desk, and answers Alice's exact phone. That identity is permanent and ordered. That's what databases like Postgres need.

Three tools, three completely different jobs: *place many anywhere* (Deployment), *one on every node* (DaemonSet), *named/ordered/with-its-own-storage* (StatefulSet).

---

## The underlying mechanism (controllers + etcd + DNS, no direct kernel primitive)

Like Deployments, neither of these is a kernel feature — they're **controllers** running in `kube-controller-manager` (Topic 29), each running the reconciliation loop (Topic 30) with a different notion of "desired state." Let's be precise about what makes each one different at the object level.

**DaemonSet controller.** Its desired state is not "N pods" — it's **"exactly one pod per eligible node."** It watches the Node objects in etcd. For every Node that matches the DaemonSet's `nodeSelector`/tolerations and doesn't already have one of its pods, it creates a pod **bound to that specific node** (it sets `spec.nodeName` directly, largely bypassing the normal scheduler). Add a node → the controller notices the new Node object appear in etcd → it immediately creates a pod there. Remove a node → the pod goes with it. The count is *derived from the number of nodes*, not configured.

**StatefulSet controller.** Its desired state is "N pods **with stable identities `0..N-1`.**" The magic is three guarantees etcd/DNS make real:
- **Stable name:** pods are named `<statefulset>-<ordinal>` — `postgres-0`, `postgres-1` — not random hashes. This name survives restarts.
- **Stable storage:** each pod gets its **own** PersistentVolumeClaim (Topic 39) created from a `volumeClaimTemplate`, named `data-postgres-0`, `data-postgres-1`. When `postgres-0` is rescheduled, it **re-attaches the exact same PVC** — the same disk with the same data.
- **Stable network:** paired with a **headless Service** (`clusterIP: None`), DNS creates a per-pod A record: `postgres-0.postgres.orders.svc.cluster.local` always resolves to that specific pod.

And crucially, the StatefulSet controller enforces **ordering**: it won't create `postgres-1` until `postgres-0` is Running and Ready; on scale-down it removes the highest ordinal first. This is why the underlying mechanism matters — a Deployment's ReplicaSet would happily start all pods at once with random names and shared/ephemeral identity, which would corrupt a database cluster.

---

## What is this?

A **DaemonSet** runs exactly one copy of a pod on every (or every selected) node in the cluster — perfect for per-node agents like log shippers and metrics collectors that need to see *that machine's* logs or hardware.

A **StatefulSet** runs a set of pods with **stable, ordered, individually-named identities**, each with its **own persistent disk** and its **own DNS name** — perfect for stateful systems like Postgres and Redis where "which replica am I" and "which disk is mine" actually matter.

---

## Why does it matter for a backend developer?

You'll reach for the wrong tool and get burned in very specific ways:

**If you run Postgres as a Deployment** (the classic beginner mistake): all replicas share one PVC or get random ephemeral storage, they all start simultaneously with names like `postgres-6c9f8b7d4-x8k2p`, and there's no "primary." Two pods write to the same volume → **data corruption**. Or a pod restarts and gets a *fresh empty disk* → **your database is gone.** Or replication can't work because no pod has a stable address to replicate *from*.

**If you run your log agent as a Deployment with `replicas: 3`** on a 10-node cluster: 3 random nodes get a log collector, **7 nodes' logs are silently lost.** You won't notice until an incident on one of the 7 dark nodes and you have zero logs to debug with.

Without understanding these:
- You'll lose database data on every pod reschedule.
- You'll have blind spots in logging/monitoring coverage.
- You won't understand why Postgres needs a **headless Service** and what `postgres-0.postgres` DNS names are.
- You'll fight the scheduler trying to force "one pod per node" manually with anti-affinity when a DaemonSet does it for free.

These are the two most important non-Deployment workload types, and picking the right one is an architecture decision you make constantly.

---

## The physical reality

Let's look at what **actually exists** for each.

**DaemonSet — one pod per node, provably.** On a 3-node cluster running a Fluent Bit log agent:

```
$ kubectl get pods -n orders -o wide -l app=log-agent
NAME              READY   STATUS    NODE
log-agent-4x9kd   1/1     Running   node-1     ← exactly one per node
log-agent-7bn2m   1/1     Running   node-2     ← one
log-agent-p8vqz   1/1     Running   node-3     ← one
```

```
$ kubectl get daemonset log-agent -n orders
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
log-agent   3         3         3       3            3           <none>
#           │
#           └─ DESIRED = number of eligible NODES, not a number you typed
```

The agent pod typically **bind-mounts the host's log directory** (this is the whole point — it reads the node's files):

```
$ kubectl get pod log-agent-4x9kd -n orders -o jsonpath='{.spec.volumes}'
[{"name":"varlog","hostPath":{"path":"/var/log"}}]   ← reads THIS node's /var/log
```

**StatefulSet — named pods, each with its own PVC.** For a 3-replica Postgres:

```
$ kubectl get pods -n orders -l app=postgres
NAME         READY   STATUS    AGE
postgres-0   1/1     Running   10m    ← stable ordinal name, NOT a random hash
postgres-1   1/1     Running   9m     ← came up AFTER postgres-0 was Ready
postgres-2   1/1     Running   8m     ← after postgres-1

$ kubectl get pvc -n orders
NAME               STATUS   VOLUME       CAPACITY   AGE
data-postgres-0    Bound    pvc-a1b2...  10Gi       10m   ← postgres-0's OWN disk
data-postgres-1    Bound    pvc-c3d4...  10Gi       9m    ← postgres-1's OWN disk
data-postgres-2    Bound    pvc-e5f6...  10Gi       8m    ← postgres-2's OWN disk
```

In etcd, the identity is durable:

```
/registry/pods/orders/postgres-0
/registry/persistentvolumeclaims/orders/data-postgres-0
```

And DNS (via the headless Service) gives each pod a permanent name:

```
$ kubectl exec -n orders postgres-1 -- nslookup postgres-0.postgres.orders.svc.cluster.local
Name: postgres-0.postgres.orders.svc.cluster.local
Address: 10.244.1.7     ← postgres-0 is always reachable at this stable DNS name
```

So the physical reality: a **DaemonSet's pod count equals your node count and its pods read host files**; a **StatefulSet's pods have permanent names, permanent disks (PVCs that survive reschedule), and permanent DNS records.**

---

## How it works — step by step

### DaemonSet: what happens when you add a node

1. **You apply the DaemonSet.** The API server writes it to etcd. There's no `replicas` field.
2. **The DaemonSet controller lists Nodes.** It reads all Node objects and filters by `nodeSelector`, taints/tolerations, and affinity. Say 3 nodes qualify.
3. **It creates one pod per qualifying node, pre-assigned.** For each node it creates a pod with `spec.nodeName: node-1` (etc.) already set — the pod skips the scheduler's "where should this go?" decision because the answer is fixed.
4. **kubelet on each node runs its pod.** The pod bind-mounts `hostPath: /var/log` and starts reading that node's logs.
5. **You add `node-4` to the cluster.** kubelet on node-4 registers a new Node object in etcd.
6. **The DaemonSet controller's watch fires.** It sees a new eligible Node with no matching pod → it creates a pod pinned to `node-4`. Automatically. You did nothing.
7. **You drain and remove `node-2`.** Its DaemonSet pod is terminated with the node. DESIRED drops from 4 to 3. The count always tracks the node count.

### StatefulSet: what happens on first creation and on a pod restart

1. **You apply the StatefulSet + a headless Service.** Both go to etcd. The StatefulSet references the headless Service by name in `serviceName`.
2. **The controller creates `postgres-0` first — and only `postgres-0`.** It also creates its PVC `data-postgres-0` from the `volumeClaimTemplate`. A PersistentVolume is dynamically provisioned (Topic 39) and bound.
3. **It waits.** It will **not** create `postgres-1` until `postgres-0` is `Running` and passes its readiness probe. This ordered startup lets `postgres-0` become the primary and be ready to accept replicas.
4. **`postgres-0` Ready → create `postgres-1`.** New PVC `data-postgres-1`, its own disk. `postgres-1` boots, and via DNS finds `postgres-0.postgres.orders.svc.cluster.local` to start streaming replication from the primary.
5. **`postgres-1` Ready → create `postgres-2`.** Same pattern. Now the cluster is 0←1, 0←2 replication.
6. **Later, `postgres-1`'s node dies.** The pod is lost. The controller creates a **replacement pod, still named `postgres-1`**, and — critically — **re-attaches the same PVC `data-postgres-1`.** The replacement mounts the exact same disk with all its data, and answers to the same DNS name `postgres-1.postgres`. From every other pod's perspective, `postgres-1` "came back," identity intact.
7. **Scale down from 3 to 2.** The controller removes the **highest ordinal first**: `postgres-2` is deleted. Its PVC `data-postgres-2` is **retained by default** (not deleted) — Kubernetes assumes your data is precious and makes you delete it manually.

**The key insight:** a Deployment's pods are cattle (nameless, interchangeable, disposable storage). A StatefulSet's pods are more like *seats with permanent name plates and lockers* — the person can be replaced but the seat, name, and locker persist.

---

## Exact syntax breakdown

### DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  namespace: orders
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      tolerations:
        - operator: Exists      # tolerate ALL taints so agents run even on control-plane/tainted nodes
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

```
kind: DaemonSet
│                              ← NO `replicas` field exists. Count = number of eligible nodes.
spec:
  selector.matchLabels:        ← which pods this DaemonSet owns (Topic 38); must match template labels
  template:                    ← the pod to run on every node
    spec:
      tolerations:
        - operator: Exists     ← "I tolerate any taint." Without this, tainted nodes (e.g. control-plane)
          │                       get NO agent — a common coverage gap. Exists = match every taint.
      volumes:
        - name: varlog
          hostPath:            ← bind-mount a path from the NODE's filesystem into the pod.
            path: /var/log        THIS is why DaemonSets exist: to read per-node host data.
```

### StatefulSet + headless Service

```yaml
# The headless Service — REQUIRED for stable per-pod DNS.
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: orders
spec:
  clusterIP: None            # <-- "headless": no virtual IP, no load balancing
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: orders
spec:
  serviceName: postgres      # must reference the headless Service above (for DNS)
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:      # each pod gets its OWN PVC minted from this template
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

```
kind: Service
  clusterIP: None            ← makes it HEADLESS. Instead of one virtual IP that load-balances,
  │                             DNS returns each pod's OWN IP + gives each a name:
  │                             postgres-0.postgres, postgres-1.postgres, ...
  │
kind: StatefulSet
  serviceName: postgres      ← the headless Service that governs the pods' DNS domain.
  │                             Pod FQDN = <pod>.<serviceName>.<namespace>.svc.cluster.local
  replicas: 3                ← creates postgres-0, -1, -2 IN ORDER (0 Ready before 1 starts)
  volumeClaimTemplates:      ← NOT a shared volume. A TEMPLATE. Each pod gets its own PVC:
    - metadata:                data-postgres-0, data-postgres-1, data-postgres-2
        name: data              A reschedule of postgres-1 RE-ATTACHES data-postgres-1 (same data).
      spec:
        accessModes:
          - ReadWriteOnce    ← this disk is mounted by exactly ONE pod at a time (Topic 39)
        resources.requests.storage: 10Gi   ← size of EACH pod's own disk
```

Pod DNS name breakdown:

```
postgres-0.postgres.orders.svc.cluster.local
│          │        │      │
│          │        │      └─ cluster domain
│          │        └─ namespace (Topic 37)
│          └─ the headless Service name (serviceName)
└─ the specific pod's stable hostname (ordinal 0)
```

---

## Example 1 — basic

A DaemonSet log agent for the whole cluster, and a single-replica caching Redis as a simple StatefulSet. Every line commented.

```yaml
# log-agent.yaml — one collector on every node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  namespace: orders
spec:
  selector:
    matchLabels: { app: log-agent }
  template:
    metadata:
      labels: { app: log-agent }
    spec:
      tolerations:
        - operator: Exists              # run even on tainted/control-plane nodes → full coverage
      containers:
        - name: agent
          image: busybox:1.36           # stand-in; real world: fluent-bit / vector / datadog-agent
          # tail every container log line on THIS node:
          command: ["sh","-c","tail -f /var/log/containers/*.log 2>/dev/null || sleep 3600"]
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true            # agents read, never write, host logs
      volumes:
        - name: varlog
          hostPath: { path: /var/log }  # the node's real /var/log
```

```yaml
# redis.yaml — Redis with a stable identity + its own disk
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: orders
spec:
  clusterIP: None                       # headless: gives redis-0 a stable DNS name
  selector: { app: redis }
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: orders
spec:
  serviceName: redis
  replicas: 1                           # even a single instance benefits from stable disk+name
  selector:
    matchLabels: { app: redis }
  template:
    metadata:
      labels: { app: redis }
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          args: ["--appendonly","yes"]  # persist to disk (AOF) — needs stable storage
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: data
              mountPath: /data          # Redis writes appendonly.aof here
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 5Gi } }
```

```bash
kubectl apply -f log-agent.yaml
kubectl apply -f redis.yaml

# Prove one log-agent exists per node:
kubectl get pods -n orders -o wide -l app=log-agent

# Prove redis got a stable name and its own PVC:
kubectl get pods -n orders -l app=redis          # -> redis-0 (not a random hash)
kubectl get pvc  -n orders                        # -> data-redis-0

# Your orders-api connects to Redis by the stable name:
#   REDIS_URL=redis://redis-0.redis.orders.svc.cluster.local:6379
```

Even at `replicas: 1`, the StatefulSet gives you two things a Deployment wouldn't: the pod is always `redis-0` (not a changing hash) and it always re-attaches `data-redis-0` on restart, so your `appendonly.aof` cache survives.

---

## Example 2 — production scenario

**The situation.** `orders-api` (a Deployment, Topic 34) needs a **highly-available Postgres**: one primary that takes writes, two replicas that stream from it for read scaling and failover. You also need centralized logging so you can debug `orders-api` across all nodes.

**The wrong instinct** is to `kubectl create deployment postgres --replicas=3`. Here's the disaster that causes:

```
$ kubectl get pods -n orders
postgres-59d8c7b6f-2k4mp   1/1   Running   # random name — who's the primary?
postgres-59d8c7b6f-9xqlt   1/1   Running   # random name
postgres-59d8c7b6f-hh7vz   1/1   Running   # all started at once, all racing to init the same DB
```

All three have random, changing names, so nothing can be configured as "the primary to replicate from." If they share a PVC (ReadWriteOnce won't even allow it) or each gets ephemeral storage, you either can't schedule them or you lose data on restart. It simply cannot form a real database cluster.

**The right architecture:**

```yaml
# postgres StatefulSet (abbreviated — full volumeClaimTemplates as in the syntax section)
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: postgres, namespace: orders }
spec:
  serviceName: postgres          # headless Service → postgres-0/1/2.postgres DNS names
  replicas: 3
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            # postgres-0 is ALWAYS the primary because it's created first and has a stable name.
            # Replicas point at it by DNS:
            - name: PRIMARY_HOST
              value: "postgres-0.postgres.orders.svc.cluster.local"
          volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data }]
      # (an init/entrypoint script uses ordinal 0 => run as primary, else => pg_basebackup from PRIMARY_HOST)
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 50Gi } }
```

Now everything works because of stable identity:

- **`postgres-0` is the primary** — deterministically, because it's created first and its name never changes. `orders-api` writes to `postgres-0.postgres.orders.svc.cluster.local:5432`.
- **`postgres-1` and `postgres-2` are read replicas** that `pg_basebackup` + stream from `postgres-0` by its stable DNS name.
- **Each has its own 50Gi disk.** When `postgres-1`'s node crashes, the replacement `postgres-1` re-attaches `data-postgres-1` — same data, no re-sync from scratch, no data loss.
- **Ordered startup** guarantees the primary is up and Ready before replicas try to connect, so no boot-time race.

Meanwhile the **DaemonSet** log agent runs on all 3 nodes, so wherever an `orders-api` pod or a `postgres` pod lands, its logs are collected. No dark nodes.

**The payoff:** your data tier is HA and durable, `orders-api` has a single stable write endpoint and stable read endpoints, and you have full-cluster log coverage — all because you matched the workload type to the job. (In real life you'd use an operator like CloudNativePG or the Bitnami chart, but they build on *exactly* this StatefulSet + headless Service foundation.)

---

## Common mistakes

**Mistake 1 — Running a database as a Deployment.**

```
$ kubectl scale deployment postgres --replicas=3 -n orders
# Two pods try to mount/init the same data → corruption, OR each gets ephemeral storage → data lost on restart.
$ kubectl delete pod postgres-59d8c7b6f-2k4mp -n orders
# New pod gets a FRESH empty volume → "database system was not properly shut down" or simply empty DB.
```

Root cause: Deployment pods have random names and no per-pod stable storage; a ReplicaSet gives no ordering or identity. Right: use a **StatefulSet** with `volumeClaimTemplates` so each pod keeps its own disk across reschedules.

**Mistake 2 — Forgetting the headless Service (or giving it a clusterIP).**

```
$ kubectl exec -n orders postgres-1 -- nslookup postgres-0.postgres.orders.svc.cluster.local
** server can't find postgres-0.postgres.orders.svc.cluster.local: NXDOMAIN
```

Root cause: per-pod DNS names (`postgres-0.postgres...`) only exist when the governing Service is **headless** (`clusterIP: None`) and named in `serviceName`. A normal Service gives one virtual IP and no per-pod records. Right: create the headless Service and set `spec.serviceName` to it.

**Mistake 3 — Expecting a DaemonSet to run on control-plane/tainted nodes without tolerations.**

```
$ kubectl get pods -n orders -o wide -l app=log-agent
# 3 pods on 3 worker nodes... but the control-plane node has NO agent → its logs are lost.
```

Root cause: control-plane and some special nodes carry **taints** (e.g. `node-role.kubernetes.io/control-plane:NoSchedule`); a DaemonSet without matching **tolerations** skips them. Right: add `tolerations: [{ operator: Exists }]` if you need agents everywhere.

**Mistake 4 — Deleting a StatefulSet and expecting the data to vanish (or expecting scale-down to reclaim disks).**

```
$ kubectl delete statefulset postgres -n orders
$ kubectl get pvc -n orders
NAME               STATUS   CAPACITY
data-postgres-0    Bound    50Gi        ← STILL HERE. PVCs are NOT deleted with the StatefulSet.
data-postgres-1    Bound    50Gi        ← STILL HERE (and still billing you for the disk).
```

Root cause: Kubernetes deliberately **retains PVCs** from `volumeClaimTemplates` so you don't lose data by accident. Scale-down also retains them. Right: know this is intentional; delete PVCs manually (`kubectl delete pvc -l app=postgres -n orders`) when you truly want the data gone. (Newer clusters offer `persistentVolumeClaimRetentionPolicy` to automate this.)

**Mistake 5 — Assuming StatefulSet pods start/stop in parallel like a Deployment.**

You scale `postgres` from 1 to 5 and expect all 5 up fast, but:

```
$ kubectl get pods -n orders -w
postgres-1   0/1   Running   # postgres-2 won't even be CREATED until postgres-1 is Ready
```

Root cause: default `podManagementPolicy: OrderedReady` starts pods one at a time in order and stops them in reverse. If `postgres-1` never becomes Ready, `postgres-2..4` never start. Right: that ordering is usually what you want for databases; if you genuinely need parallel start (e.g. a leaderless system), set `podManagementPolicy: Parallel`.

---

## Hands-on proof

Run on any cluster (a multi-node `kind` cluster shows DaemonSets best; single-node still works).

```bash
kubectl create namespace orders 2>/dev/null

# ---------- DaemonSet: one per node ----------
cat <<'EOF' | kubectl apply -n orders -f -
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: log-agent }
spec:
  selector: { matchLabels: { app: log-agent } }
  template:
    metadata: { labels: { app: log-agent } }
    spec:
      tolerations: [{ operator: Exists }]
      containers:
        - name: agent
          image: busybox:1.36
          command: ["sh","-c","sleep 3600"]
          volumeMounts: [{ name: varlog, mountPath: /var/log, readOnly: true }]
      volumes: [{ name: varlog, hostPath: { path: /var/log } }]
EOF

# 1. DESIRED equals your node count — you never typed a replica number.
kubectl get daemonset log-agent -n orders
kubectl get nodes            # compare: DESIRED should equal number of Ready nodes
kubectl get pods -n orders -o wide -l app=log-agent   # one pod per node

# ---------- StatefulSet: stable names + own PVCs ----------
cat <<'EOF' | kubectl apply -n orders -f -
apiVersion: v1
kind: Service
metadata: { name: redis }
spec:
  clusterIP: None
  selector: { app: redis }
  ports: [{ port: 6379 }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: redis }
spec:
  serviceName: redis
  replicas: 2
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          volumeMounts: [{ name: data, mountPath: /data }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 1Gi } }
EOF

# 2. Pods have ORDINAL names, not random hashes; they came up IN ORDER.
kubectl get pods -n orders -l app=redis -w   # watch redis-0 become Ready BEFORE redis-1 starts; Ctrl-C when both Ready

# 3. Each pod got its OWN PVC.
kubectl get pvc -n orders                     # data-redis-0, data-redis-1

# 4. Prove stable identity survives deletion: write a key, delete the pod, key is still there.
kubectl exec -n orders redis-0 -- redis-cli set proof "identity-persists"
kubectl delete pod redis-0 -n orders          # controller recreates it AS redis-0 with the SAME PVC
kubectl wait --for=condition=Ready pod/redis-0 -n orders --timeout=60s
kubectl exec -n orders redis-0 -- redis-cli get proof   # -> "identity-persists"  (same disk!)

# 5. Prove per-pod DNS via the headless Service.
kubectl exec -n orders redis-1 -- getent hosts redis-0.redis.orders.svc.cluster.local

# Clean up.
kubectl delete namespace orders
```

What you verified: a DaemonSet's count **tracks node count** (steps 1), a StatefulSet gives **ordinal names in order** (step 2) with **per-pod PVCs** (step 3) that **survive pod deletion carrying the data** (step 4), reachable by **stable per-pod DNS** (step 5).

---

## Practice exercises

### Exercise 1 — easy
Deploy the `log-agent` DaemonSet above. Run `kubectl get daemonset log-agent -n orders` and `kubectl get nodes`. Confirm DESIRED equals your node count. Then explain in one sentence what would happen to the DaemonSet's pod count if you added a fourth node — and who makes that happen.

### Exercise 2 — medium
Deploy the 2-replica `redis` StatefulSet. Write a value into `redis-0` with `redis-cli set`. Delete `redis-0` and wait for it to come back. Verify the value is still there. Then run `kubectl get pvc -n orders` before and after the deletion and explain why the PVC count did not change and why that is what preserved your data.

### Exercise 3 — hard (production simulation)
Model the primary/replica idea. Deploy a 3-replica `postgres` StatefulSet with a headless Service (use `postgres:16` and set `POSTGRES_PASSWORD`). From inside `postgres-2`, resolve `postgres-0.postgres.orders.svc.cluster.local` and connect to it with `psql`. Now delete `postgres-1`, watch it get recreated with the same name and re-attach `data-postgres-1`. Write down: (a) why `postgres-0` is a safe choice for "the primary," (b) what the headless Service provided that a normal ClusterIP Service could not, and (c) what happens to `data-postgres-2` if you scale down to 2 replicas.

---

## Mental model checkpoint

Answer from memory:

1. A DaemonSet has no `replicas` field — what determines how many pods it runs?
2. What single field makes a Service "headless," and why does a StatefulSet need one?
3. What is a `volumeClaimTemplate` and how is it different from a normal `volume`?
4. When `postgres-1` is rescheduled to a new node, what happens to its data and its DNS name?
5. In what order does a StatefulSet create pods, and in what order does it delete them on scale-down?
6. Why must a DaemonSet often include `tolerations: [{operator: Exists}]`?
7. What happens to a StatefulSet's PVCs when you delete the StatefulSet, and why is that the default?

---

## Quick reference card

| Concept / field | What it does | Key detail |
|---|---|---|
| `kind: DaemonSet` | One pod per (eligible) node | No `replicas`; count = node count |
| `hostPath` volume | Mount a node directory into the pod | Why DaemonSets exist (read `/var/log`) |
| `tolerations: [{operator: Exists}]` | Let the DaemonSet run on tainted nodes | Needed for control-plane coverage |
| `kind: StatefulSet` | Ordered, named pods with own storage | Names are `name-0`, `name-1`, ... |
| `serviceName` | Ties StatefulSet to its headless Service | Enables per-pod DNS |
| `clusterIP: None` | Makes a Service **headless** | Gives each pod a DNS A record |
| `volumeClaimTemplates` | Mint one PVC per pod | Re-attached on reschedule (data survives) |
| `podManagementPolicy` | `OrderedReady` (default) vs `Parallel` | Ordering guarantee for start/stop |
| Pod DNS `pod.svc.ns.svc...` | Stable per-pod address | `postgres-0.postgres.orders.svc.cluster.local` |
| PVC on StatefulSet delete | **Retained**, not deleted | Delete manually to reclaim storage |

---

## When would I use this at work?

1. **Deploying stateful infra (Postgres/Redis/Kafka) in-cluster.** Any time data must survive pod restarts and each replica needs a distinct identity, you reach for a StatefulSet — directly, or via an operator/Helm chart that builds one for you.

2. **Rolling out cluster-wide agents.** Installing a logging agent (Fluent Bit/Vector), a metrics agent (node-exporter), a security agent (Falco), or a CNI/CSI component means a DaemonSet, so every node — including new ones added by autoscaling — is automatically covered.

3. **Giving `orders-api` a stable write endpoint.** When your API needs "the primary database" address that never changes, the StatefulSet + headless Service gives you `postgres-0.postgres.orders.svc.cluster.local`, a name you can safely bake into config that survives every reschedule.

---

## Connected topics

- **Study before:**
  - **Topic 34 — Deployments**: the default workload type these two are the specialized alternatives to.
  - **Topic 39 — Persistent Volumes and Claims**: PVs/PVCs/StorageClasses that `volumeClaimTemplates` provision per pod.
  - **Topic 35 — Services**: normal Services vs the headless Service StatefulSets require.
  - **Topic 43 — Health Checks in Kubernetes**: readiness gates the ordered startup of StatefulSet pods.
- **Study after:**
  - **Topic 40 — Kubernetes Networking in Depth**: how per-pod DNS records are created and resolved.
  - **Topic 49 — Helm Basics**: real Postgres/Redis charts that wrap these StatefulSet patterns.
  - **Topic 51 — Observability in Kubernetes**: DaemonSet-based log/metrics agents feeding your monitoring stack.
