# 39 — Persistent Volumes and Claims
## Section: Kubernetes in Practice

## ELI5 — The Simple Analogy

Imagine your app is a person renting apartments in a huge building, and the building is Kubernetes.

- A **Pod** is a person. People come and go. They get evicted, they move rooms, they even get replaced by a new person with the same job.
- A person's **stuff inside the room** (files, memory) is thrown away the moment they leave. That's a container's filesystem — gone on restart.
- A **Persistent Volume (PV)** is a **real storage unit in the basement** — a physical locker with a lock. It exists whether or not anyone is renting it. The building manager (the cluster) owns it.
- A **Persistent Volume Claim (PVC)** is a **rental request slip**: "I need a locker, at least 10 GB, that I can read and write." You hand the slip to the manager, and the manager finds a matching locker and gives you the key.
- A **StorageClass** is the **catalogue** the manager orders new lockers from: "Standard SSD locker", "Cheap spinning-disk locker", "Super-fast NVMe locker". When no existing locker matches your slip, the manager phones the supplier and a brand-new locker gets built for you automatically. That's **dynamic provisioning**.

The magic: the person (Pod) never talks to the physical locker directly. They only wave their rental slip (PVC). If the person leaves and a new person takes their job, the new person picks up the **same slip**, and the manager gives them the **same locker with all the stuff still inside**. That is how your Postgres data survives when the Pod dies.

## The Linux kernel feature underneath

Kubernetes storage is not a magic K8s invention. At the bottom, it is the same Linux `mount()` syscall you saw with Docker volumes in Topic 14.

When a Pod using a PVC lands on a node, three layers cooperate:

```
        Kubernetes objects (etcd)                Node kernel reality
        ─────────────────────────                ───────────────────
  PVC "orders-pgdata" ──bound──► PV "pvc-8f3a…"
                                    │ points to
                                    ▼
                          a real disk / NFS export / cloud volume
                                    │
   kubelet on the node  ──────────► attaches it   (cloud API: "attach vol to this VM")
                                    │
                                    ▼
              /dev/nvme1n1  appears as a block device on the node
                                    │  mkfs + mount()
                                    ▼
   /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/pvc-8f3a…/mount
                                    │  bind-mounted into the container's mnt namespace
                                    ▼
              container sees it as  /var/lib/postgresql/data
```

The kernel primitives at work:

- **Block device attachment** — a cloud volume (AWS EBS, GCP PD) or local disk shows up in `/dev/` on the node, exactly like plugging in a USB drive.
- **A filesystem** — `mkfs.ext4` (or xfs) is run once on that block device so it can hold files.
- **`mount()` syscall** — the node mounts that device under `/var/lib/kubelet/pods/.../`.
- **Bind mount into the mount namespace** — the same `mount(src, dst, MS_BIND)` from Topic 02 and Topic 14 injects that path into the container's private mount namespace, so the container sees `/var/lib/postgresql/data`.

The whole PV/PVC/StorageClass machinery is just a **declarative, cluster-wide bookkeeping system** that decides *which* device gets mounted *where*, then hands the real work to the kubelet and the kernel.

The component that talks to the storage backend is a **CSI driver** (Container Storage Interface) — a small program running as a Pod that knows how to "create a volume", "attach a volume", and "mount a volume" for a specific backend (EBS, Ceph, NFS, etc.). CSI is to storage what CRI is to containers (Topic 29).

## What is this?

A **PersistentVolume (PV)** is a piece of storage in the cluster — a real disk somewhere — represented as a Kubernetes object. A **PersistentVolumeClaim (PVC)** is a request for storage made by a user or a Pod. A **StorageClass** describes a *kind* of storage and a **provisioner** that can create PVs on demand. Together they let a Pod ask for durable storage by describing what it needs, without knowing the physical details — and the data survives Pod restarts, rescheduling, and node failures.

## Why does it matter for a backend developer?

Because **containers are ephemeral and databases are not**. Everything a container writes to its own filesystem lives in the overlay writable layer (Topic 04), and that layer is **deleted** when the container is removed. In Kubernetes, Pods get removed *constantly* — on rollout, on node drain, on crash, on scale-down.

Without persistent volumes:

- Your Postgres Pod restarts → **every order, every user, every row is gone.**
- You scale your Deployment → the new Pod starts with an empty database.
- A node dies → your data died with it.

With PV/PVC done right:

- Postgres Pod crashes and reschedules to another node → the **same volume follows it**, all data intact.
- You upgrade the Postgres image → new Pod, same data.
- You can back up, resize, and snapshot storage as first-class operations.

This is the difference between a toy cluster and one you'd trust with real customer data.

## The physical reality

Say `orders-api` uses a Postgres that claims 10 GB. Here is what physically exists across etcd and the node.

**In etcd (the cluster's brain, Topic 29):**

```
/registry/persistentvolumeclaims/orders/orders-pgdata      ← the claim object
/registry/persistentvolumes/pvc-8f3a1c2d-...               ← the auto-created PV object
/registry/storageclasses/standard                          ← the StorageClass
```

**On the node where the Postgres Pod runs:**

```
/dev/nvme1n1                                                ← the attached cloud volume (block device)

/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/pvc-8f3a1c2d-.../
├── mount/                                                  ← the volume, mounted here by kubelet
│   ├── base/                                               ← Postgres data dir contents
│   ├── PG_VERSION
│   ├── pg_wal/
│   └── ...
└── vol_data.json                                           ← CSI bookkeeping (volume handle, driver)
```

You can prove the mount is real:

```bash
# On the node:
mount | grep pvc-8f3a
# /dev/nvme1n1 on /var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~csi/pvc-8f3a…/mount type ext4 (rw,relatime)
```

Inside the container, that same directory appears as `/var/lib/postgresql/data`. Two names, one set of bytes on one physical disk. When the Pod dies, etcd still holds the PVC and PV; the block device still holds the data. A new Pod re-mounts it and Postgres reads its existing files.

## How it works — step by step

Full trace of **dynamic provisioning**: you apply a PVC, and a brand-new disk gets created, attached, and mounted.

1. **You `kubectl apply` a PVC** named `orders-pgdata` requesting `10Gi`, `ReadWriteOnce`, `storageClassName: standard`. The API server validates it and writes it to etcd. State: `Pending`.
2. **The PV controller (in kube-controller-manager) sees an unbound PVC.** It looks for an existing PV that satisfies the request (size ≥ 10Gi, matching access mode, matching class). None exists.
3. **Because the PVC names a StorageClass, dynamic provisioning kicks in.** The controller looks up the StorageClass `standard`, reads its `provisioner` field (e.g. `ebs.csi.aws.com`), and asks that provisioner to **create a volume**.
4. **The CSI provisioner calls the cloud API:** "create a 10 GB gp3 volume". The cloud returns a volume ID like `vol-0abc…`.
5. **The provisioner creates a PV object** in etcd (`pvc-8f3a1c2d-…`) describing that real volume, and **binds** the PVC to it. The PVC state flips `Pending → Bound`. The volume is not attached to any node yet — it's just created.
6. **A Pod that references the PVC gets scheduled** to node `worker-2` (Topic 33 covers scheduling). With `volumeBindingMode: WaitForFirstConsumer`, steps 3–5 are actually *delayed until here* so the volume is created in the same zone as the chosen node.
7. **The AttachDetach controller** tells the cloud to **attach** `vol-0abc…` to `worker-2`'s VM. The block device `/dev/nvme1n1` appears on that node.
8. **The kubelet on `worker-2` runs the CSI `NodeStageVolume`**: if the disk is brand-new, it runs `mkfs.ext4`, then mounts it to a global staging path.
9. **The kubelet runs `NodePublishVolume`**: it bind-mounts the staged path into the Pod's directory `/var/lib/kubelet/pods/<uid>/volumes/.../mount`.
10. **The container starts**, and its `volumeMounts` entry bind-mounts that path to `/var/lib/postgresql/data` inside the container's mount namespace.
11. **Postgres boots**, sees an empty data dir on first run, runs `initdb`, and starts writing. On every later restart it finds its existing files and just recovers.

When the Pod later dies and reschedules to `worker-5`: steps 7–10 repeat on the new node (detach from old, attach to new), and Postgres finds all its data. **The claim is the anchor** — the Pod points at the PVC, the PVC stays bound to the same PV, the PV points at the same disk.

## Exact syntax breakdown

### A PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: orders-pgdata
  namespace: orders
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```

```
apiVersion: v1
│         │
│         └── PVCs are core/v1 objects — no group prefix
└── the "which API" field (Topic 32)

kind: PersistentVolumeClaim
│     │
│     └── this object is a REQUEST for storage, not the storage itself
└── the object type

metadata:
  name: orders-pgdata          ← Pods reference storage by THIS name
  namespace: orders            ← PVCs are namespaced; PVs are NOT (cluster-wide)

spec:
  accessModes:
    - ReadWriteOnce            ← "one node may mount this read-write" (see table below)
      │        │
      │        └── the volume is writable
      └── by a single node at a time

  storageClassName: standard   ← which "catalogue" to order a new disk from.
  │                    │           "" (empty) = no dynamic provisioning, must match a pre-made PV
  │                    └── must be an existing StorageClass name (kubectl get sc)
  └── omit this field entirely → the cluster's DEFAULT StorageClass is used

  resources:
    requests:
      storage: 10Gi            ← how much space you want.
               │  │              Ki, Mi, Gi, Ti = binary (1Gi = 2^30 bytes)
               │  └── binary gibibyte (NOT 10 GB decimal)
               └── the number; the bound PV must be >= this
```

### A StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

```
provisioner: ebs.csi.aws.com          ← the CSI driver that CREATES volumes for this class.
             │                            local kind cluster → rancher.io/local-path
             └── "kubernetes.io/no-provisioner" = static only, no auto-create

parameters:
  type: gp3                           ← passed straight to the provisioner (disk type here)

reclaimPolicy: Delete                 ← what happens to the PV+disk when its PVC is deleted.
               │                         Delete  = destroy the real disk too (default for dynamic)
               └── Retain  = keep the disk & data; PV goes to "Released", needs manual cleanup

allowVolumeExpansion: true            ← may you grow a bound PVC later by editing its request? 

volumeBindingMode: WaitForFirstConsumer
                   │                     provision/bind ONLY once a Pod schedules,
                   │                     so the disk lands in the Pod's zone/node.
                   └── Immediate = bind as soon as the PVC is created (can pick wrong zone)
```

### A Pod/StatefulSet mounting the PVC

```yaml
spec:
  containers:
    - name: postgres
      image: postgres:16-alpine
      volumeMounts:
        - name: data                        # must match a volume name below
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: orders-pgdata            # ← the PVC to attach
```

```
volumeMounts:                     ← WHERE the storage appears INSIDE the container
  - name: data                    ← a label linking to spec.volumes below
    mountPath: /var/lib/postgresql/data
    │                               the path the container process reads/writes.
    │                               For postgres:16 this is $PGDATA's parent.
    └── this dir in the container = the physical disk

volumes:                          ← WHICH storage this Pod uses (Pod-level)
  - name: data                    ← matches volumeMounts[].name
    persistentVolumeClaim:
      claimName: orders-pgdata    ← "give me the disk bound to THIS claim"
      │                              the indirection that lets the disk outlive the Pod
      └── must be a PVC in the SAME namespace as the Pod
```

## Access modes — the exact meanings

| Mode | Short | Meaning | Typical use |
|------|-------|---------|-------------|
| `ReadWriteOnce` | RWO | Mounted read-write by **one node** (multiple Pods on that node can share it) | Postgres, Redis — most block storage |
| `ReadOnlyMany` | ROX | Mounted read-only by **many nodes** | Shared config, static assets |
| `ReadWriteMany` | RWX | Mounted read-write by **many nodes at once** | Shared uploads dir; needs NFS/CephFS, NOT plain EBS |
| `ReadWriteOncePod` | RWOP | Mounted read-write by exactly **one Pod** in the whole cluster | Strict single-writer (K8s 1.22+) |

Critical gotcha: `ReadWriteOnce` is per **node**, not per Pod. A cloud block volume like EBS is `RWO` — it physically cannot attach to two VMs at once. That's why you cannot run two Postgres Pods on two different nodes sharing one EBS PVC. For that you need `RWX` storage (NFS-like).

## Reclaim policies — what happens to the disk when the PVC is deleted

```
                    kubectl delete pvc orders-pgdata
                                 │
         ┌───────────────────────┴───────────────────────┐
         ▼                                                ▼
   reclaimPolicy: Delete                          reclaimPolicy: Retain
         │                                                │
   PV is deleted from etcd                        PV stays, status → "Released"
   CSI destroys the real disk                     the real disk & DATA remain
   ↳ data is GONE forever                         ↳ NOT auto-reusable; admin must
                                                     manually clear claimRef to reuse,
                                                     or manually delete the disk
```

- **`Delete`** (default for dynamically provisioned volumes): deleting the PVC destroys the PV *and the underlying cloud disk*. Convenient, and **dangerous** for databases — a fat-fingered `kubectl delete pvc` erases production data.
- **`Retain`**: deleting the PVC leaves the disk and data untouched; the PV goes to `Released` and won't be reused until an admin intervenes. This is the safe choice for stateful data. You get durability at the cost of manual cleanup.

For anything holding real data (`orders` Postgres), prefer `Retain`, or use a StorageClass whose `reclaimPolicy: Retain`, plus backups.

## What happens when a Pod restarts or reschedules

This is the whole reason PVs exist. Three scenarios:

**1. Container restarts in place (crash, liveness probe kill — Topic 43):**
The Pod object stays, same node, same mounts. The kubelet just restarts the container process. The volume is already mounted — **nothing re-attaches**. Postgres reopens the exact same files. Fastest, zero storage movement.

**2. Pod is deleted and recreated on the SAME node (e.g. `kubectl delete pod`):**
The PVC/PV binding is untouched (they're separate objects). The new Pod references the same `claimName`, kubelet re-runs `NodePublishVolume`, the same disk is re-bind-mounted. Data intact.

**3. Pod reschedules to a DIFFERENT node (node drain, node crash, rollout):**
- Kubelet on the old node unmounts and CSI `NodeUnstage`s the volume.
- AttachDetach controller **detaches** the disk from the old node.
- Scheduler places the new Pod on `worker-5`.
- The disk is **attached** to `worker-5`, `mkfs` is skipped (filesystem already exists), it's mounted, bind-mounted into the new container.
- Postgres starts, runs crash recovery on its WAL, and comes up with all data.

The catch with `ReadWriteOnce`: the volume must fully **detach** from the old node before it can attach to the new one. If the old node is *dead but not confirmed dead* (network partition), Kubernetes waits (default ~6 minutes) before force-detaching, to avoid two nodes writing to one disk and corrupting it. This is why a hard node failure can stall a stateful Pod's recovery — a well-known operational reality.

## Example 1 — basic

A single Postgres Pod with a persistent claim, on any cluster with a default StorageClass (kind, minikube, cloud all work).

```yaml
# pg-basic.yaml
apiVersion: v1
kind: PersistentVolumeClaim          # 1) the claim: "I need 5Gi, single-node RW"
metadata:
  name: orders-pgdata
  namespace: orders
spec:
  accessModes: ["ReadWriteOnce"]     # block storage, one node
  resources:
    requests:
      storage: 5Gi                   # ask for 5 gibibytes
  # storageClassName omitted → uses the cluster's DEFAULT class
---
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  namespace: orders
spec:
  containers:
    - name: postgres
      image: postgres:16-alpine
      env:
        - name: POSTGRES_PASSWORD
          value: devsecret           # (Topic 36 shows the Secret way)
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata  # subdir avoids "lost+found" init errors
      ports:
        - containerPort: 5432
      volumeMounts:
        - name: data                 # link to volumes[].name below
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: orders-pgdata     # attach the disk bound to our claim
```

Run it and prove persistence:

```bash
kubectl create namespace orders
kubectl apply -f pg-basic.yaml

# wait for Bound + Running
kubectl -n orders get pvc,pod

# write a row
kubectl -n orders exec -it postgres -- \
  psql -U postgres -c "CREATE TABLE orders(id int); INSERT INTO orders VALUES (42);"

# DELETE the Pod (simulate a crash / reschedule)
kubectl -n orders delete pod postgres

# recreate ONLY the Pod (PVC still exists!)
kubectl apply -f pg-basic.yaml

# the row is still there — data survived the Pod's death
kubectl -n orders exec -it postgres -- psql -U postgres -c "SELECT * FROM orders;"
#  id
# ----
#  42
```

The Pod died and came back, but the PVC never moved, so the disk (and row 42) survived.

## Example 2 — production scenario

**The situation:** Your team runs `orders-api` in namespace `orders`. Postgres is currently a plain Pod with a PVC (Example 1). Two problems keep biting you:

1. On a node upgrade, the Postgres Pod got rescheduled and its **name/DNS changed**, so `orders-api` connection strings broke.
2. Someone ran `kubectl delete pvc` during cleanup and **wiped staging data**, and you're terrified of that happening in prod.

**The fix:** use a **StatefulSet** with a `volumeClaimTemplate` (stable identity + per-replica storage) and a **Retain** StorageClass. A StatefulSet (full detail in Topic 47) gives each replica a stable name (`postgres-0`) and a stable PVC (`data-postgres-0`) that is **never auto-deleted** when the Pod dies.

First, a safe StorageClass for stateful data:

```yaml
# sc-retain.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pg-retain
provisioner: ebs.csi.aws.com           # use your cluster's real provisioner
parameters:
  type: gp3
reclaimPolicy: Retain                   # deleting the PVC does NOT destroy the disk
allowVolumeExpansion: true              # we can grow it later without downtime-ish
volumeBindingMode: WaitForFirstConsumer # disk lands in the Pod's zone
```

Now the StatefulSet:

```yaml
# pg-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: orders
spec:
  serviceName: postgres                  # headless Service gives stable DNS: postgres-0.postgres
  replicas: 1
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      terminationGracePeriodSeconds: 30  # let Postgres flush + shut down cleanly
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:            # secret, not plaintext (Topic 36)
                  name: pg-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          readinessProbe:                # don't send traffic until Postgres accepts it (Topic 43)
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 10
            periodSeconds: 5
  volumeClaimTemplates:                  # ← the StatefulSet magic
    - metadata:
        name: data                       # → creates PVC "data-postgres-0"
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: pg-retain      # our Retain class
        resources:
          requests:
            storage: 20Gi
```

```
volumeClaimTemplates:
  - metadata:
      name: data          ← PVC name pattern is  <template-name>-<statefulset>-<ordinal>
                             → "data-postgres-0"  (and data-postgres-1 if replicas=2)
    spec:
      storageClassName: pg-retain   ← each replica gets its OWN durable disk
```

Why this fixes both problems:

- **Stable identity:** the pod is always `postgres-0`, reachable at `postgres-0.postgres.orders.svc.cluster.local`. Your `orders-api` connection string never changes across restarts or reschedules. Compare to a Deployment, whose pods get random name suffixes.
- **Safe storage:** the PVC `data-postgres-0` is created from the template and, crucially, a StatefulSet **does not delete its PVCs when a Pod or the StatefulSet is deleted** (unless you opt in via `persistentVolumeClaimRetentionPolicy`). Combined with `reclaimPolicy: Retain`, the disk survives even an accidental `kubectl delete statefulset postgres`.
- **Ordered, clean shutdown:** `terminationGracePeriodSeconds: 30` gives Postgres time to checkpoint and close cleanly before SIGKILL, avoiding crash-recovery on every restart.

Apply and verify:

```bash
kubectl -n orders create secret generic pg-secret --from-literal=password='S0meStr0ng!'
kubectl apply -f sc-retain.yaml -f pg-statefulset.yaml

kubectl -n orders get pvc
# NAME             STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS
# data-postgres-0  Bound    pvc-…         20Gi       RWO            pg-retain

# delete the whole StatefulSet — watch the PVC and disk SURVIVE
kubectl -n orders delete statefulset postgres
kubectl -n orders get pvc      # data-postgres-0 is STILL Bound — data is safe
```

## Common mistakes

**Mistake 1 — PVC stuck `Pending` forever, Pod stuck `Pending`.**

```
kubectl -n orders get pvc
NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
orders-pgdata   Pending                                     slow-ssd

kubectl -n orders describe pvc orders-pgdata
  Warning  ProvisioningFailed   storageclass.storage.k8s.io "slow-ssd" not found
```

Root cause: the PVC names a StorageClass that doesn't exist, **or** the cluster has no default StorageClass and you omitted `storageClassName`, **or** no static PV matches. Nothing can bind, so the Pod can't schedule.
Wrong: guessing a class name. Right: `kubectl get storageclass`, use a real one (the one marked `(default)`), or create a matching PV. On bare kind/minikube the class is often `standard` or `local-path`.

**Mistake 2 — Data disappears on every restart because there's no PVC at all.**

```yaml
# WRONG — emptyDir is wiped when the Pod is removed
volumes:
  - name: data
    emptyDir: {}
```

Root cause: `emptyDir` lives in `/var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~empty-dir/` and is **deleted with the Pod**. It survives a container *restart* but not a Pod *deletion/reschedule*. People use it by accident and think K8s "lost their data".
Right: use `persistentVolumeClaim` for anything that must outlive the Pod.

**Mistake 3 — Two Postgres Pods on two nodes sharing one RWO PVC.**

```
kubectl -n orders describe pod postgres-1
  Warning  FailedAttachVolume   Multi-Attach error for volume "pvc-8f3a…"
           Volume is already exclusively attached to node worker-2 and can't be
           attached to another node
```

Root cause: `ReadWriteOnce` (and the underlying EBS/PD disk) can attach to only one node. You scaled a Deployment to 2 with a shared PVC, and the second Pod can't get the disk.
Right: for a database, don't share one PVC across replicas — use a StatefulSet where each replica gets its own PVC. For a genuinely shared writable dir, use `ReadWriteMany` storage (NFS/CephFS/EFS).

**Mistake 4 — Deleting a Pod's PVC on a `Delete`-policy class and losing prod data.**

```bash
kubectl -n orders delete pvc orders-pgdata   # class reclaimPolicy: Delete
# → PV deleted, cloud disk destroyed, data gone. No undo.
```

Root cause: default dynamic provisioning uses `reclaimPolicy: Delete`. Deleting the PVC cascades to destroying the real disk.
Right: for stateful data use a `Retain` StorageClass, take backups, and restrict who can delete PVCs via RBAC (Topic 53).

**Mistake 5 — Postgres refuses to init because the mount root isn't empty.**

```
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
It contains a lost+found directory ...
```

Root cause: a freshly `mkfs`'d ext4 volume mounted at the data root contains a `lost+found` directory, and Postgres refuses to init into a non-empty dir.
Right: set `PGDATA` to a **subdirectory** of the mount (`/var/lib/postgresql/data/pgdata`), as both examples above do.

## Hands-on proof

Run these to see PV/PVC machinery for real (kind or minikube is enough):

```bash
# 0. see what storage the cluster offers
kubectl get storageclass
#   NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
#   standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer

# 1. create a claim
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: proof-pvc, namespace: default }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 1Gi } }
EOF

# 2. watch it bind and see the AUTO-CREATED PV
kubectl get pvc proof-pvc
kubectl get pv                     # a pvc-<uuid> PV appeared, created dynamically
kubectl describe pv $(kubectl get pvc proof-pvc -o jsonpath='{.spec.volumeName}')
#   look at: Reclaim Policy, Access Modes, Source (the real backend path/handle),
#            Claim: default/proof-pvc  ← the binding

# 3. mount it in a pod and write a file
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: writer, namespace: default }
spec:
  containers:
    - name: c
      image: busybox
      command: ["sh","-c","echo hello-from-pod > /data/note.txt && sleep 3600"]
      volumeMounts: [{ name: v, mountPath: /data }]
  volumes:
    - name: v
      persistentVolumeClaim: { claimName: proof-pvc }
EOF
kubectl wait --for=condition=Ready pod/writer

# 4. DELETE the pod, recreate it, prove the file survived
kubectl delete pod writer
# re-apply the same pod spec (repeat step 3), then:
kubectl exec writer -- cat /data/note.txt        # → hello-from-pod  (survived!)

# 5. on kind, SEE the physical files on the node
docker exec -it kind-control-plane sh -c \
  'ls /var/local-path-provisioner/*/note.txt 2>/dev/null; find / -name note.txt 2>/dev/null | head'

# 6. cleanup
kubectl delete pod writer; kubectl delete pvc proof-pvc
```

You watched: a PVC auto-create a PV, bind to it, mount into a Pod, survive a Pod deletion, and the actual file sitting on the node's disk.

## Practice exercises

### Exercise 1 — easy
Create a PVC `cache-pvc` requesting `2Gi`, `ReadWriteOnce`, using your cluster's default StorageClass. Confirm it becomes `Bound` and identify the name of the PV that was dynamically created and its reclaim policy (`kubectl get pv`).

### Exercise 2 — medium
Mount `cache-pvc` into a Redis Pod at `/data` (Redis persists its `dump.rdb` there). Write a key, force a save (`redis-cli SAVE`), delete the Pod, recreate it, and prove the key survived. Then explain in one sentence why deleting the *Pod* kept the data but deleting the *PVC* would not.

### Exercise 3 — hard (production simulation)
Deploy the Example 2 Postgres StatefulSet with a `Retain` StorageClass. Insert a row. Then:
1. Delete the whole StatefulSet and show `data-postgres-0` (the PVC) is still `Bound` and the row survives when you re-apply.
2. Delete the PVC and observe the PV go to `Released` (not deleted) because of `reclaimPolicy: Retain`.
3. Manually make that `Released` PV reusable by clearing its `claimRef`, then bind a fresh PVC to it and read the old data back. Explain what `claimRef` is and why K8s won't auto-reuse a Released PV.

## Mental model checkpoint

Answer from memory:

1. What is the difference between a PV and a PVC, and which one is namespaced?
2. What does a StorageClass add on top of PV/PVC, and what does "dynamic provisioning" mean?
3. Trace the path of bytes from a container's `/var/lib/postgresql/data` down to the physical disk. Which kernel syscall does the final step?
4. `ReadWriteOnce` restricts write access to one *what* — Pod or node? Why does that matter for scaling a database?
5. Reclaim policy `Delete` vs `Retain`: what happens to the real disk when you `kubectl delete pvc`?
6. When a Postgres Pod reschedules to a new node, list what must happen to the volume before the new Pod can start.
7. Why does a StatefulSet + `volumeClaimTemplate` protect your data better than a Deployment sharing one PVC?

## Quick reference card

| Field / object | What it does | Key detail |
|----------------|--------------|------------|
| `PersistentVolume` | Represents a real piece of storage | Cluster-scoped (not namespaced) |
| `PersistentVolumeClaim` | A request for storage by a Pod | Namespaced; Pods reference it by name |
| `StorageClass` | A template + provisioner for making PVs | Enables dynamic provisioning |
| `provisioner` | CSI driver that creates the disk | e.g. `ebs.csi.aws.com`, `rancher.io/local-path` |
| `accessModes: RWO` | One node may mount read-write | Per **node**, not per Pod; EBS/PD default |
| `accessModes: RWX` | Many nodes mount read-write | Needs NFS/CephFS/EFS, not block storage |
| `reclaimPolicy: Delete` | Destroy disk when PVC deleted | Default for dynamic; dangerous for data |
| `reclaimPolicy: Retain` | Keep disk when PVC deleted | Safe for databases; needs manual cleanup |
| `volumeBindingMode: WaitForFirstConsumer` | Provision after Pod scheduled | Ensures disk lands in the right zone |
| `allowVolumeExpansion: true` | Grow a bound PVC later | Edit PVC `resources.requests.storage` |
| `volumeClaimTemplates` | Per-replica PVC in a StatefulSet | Creates `data-<name>-<ordinal>`, not auto-deleted |
| `persistentVolumeClaim.claimName` | Attach a PVC's disk to a Pod | PVC must be in the same namespace |

## When would I use this at work?

1. **Running the `orders` Postgres inside the cluster:** a StatefulSet with a `volumeClaimTemplate` on a `Retain` StorageClass so the database survives rollouts, node upgrades, and accidental deletes — the standard pattern for self-hosted stateful services.
2. **Redis with AOF/RDB persistence:** a PVC mounted at `/data` so your cache/queue survives Pod restarts instead of cold-starting empty and hammering Postgres on every deploy.
3. **Resizing storage without downtime:** when the `orders` DB fills its 20Gi disk, you bump the PVC's `resources.requests.storage` (with `allowVolumeExpansion: true`) and the CSI driver grows the underlying disk and filesystem online — no data migration.

## Connected topics

- **Study before:** Topic 14 (Volumes in Depth) — the Docker `mount()` foundation this builds on; Topic 32 (YAML and Manifests) — the object structure; Topic 33 (Pods in Depth) — how Pods reference volumes; Topic 36 (ConfigMaps and Secrets) — the Postgres password used here.
- **Study after:** Topic 40 (Kubernetes Networking in Depth) — how the Postgres Pod is reachable; Topic 44 (Resource Requests and Limits) — sizing stateful Pods; Topic 47 (DaemonSets and StatefulSets) — the full StatefulSet story behind the production example.
