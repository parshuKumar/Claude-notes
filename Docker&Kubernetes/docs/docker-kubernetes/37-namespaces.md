# 37 — Namespaces
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Imagine one big office building shared by several teams. Every team gets its **own floor**. On the "Orders" floor there's a desk labeled "Postgres". On the "Payments" floor there's *also* a desk labeled "Postgres". Same label, different floor, no confusion — because when you say "go to Postgres" you mean *the Postgres on your floor*. Two desks can share the same name as long as they're on different floors.

A Kubernetes **Namespace** is a floor. It's a way to slice one physical cluster into named sections so teams (or environments — dev, staging, prod) can each have their own `orders-postgres`, their own `orders-api`, their own quotas, without stepping on each other. Names only have to be unique **within** a namespace, just like desk labels only have to be unique within a floor.

But here's the catch the analogy hides, and it's the single most important thing in this doc: the floors have **no walls between them by default**. Anyone on the Payments floor can walk straight over to the Orders floor's Postgres desk and start talking to it. A namespace is a **naming and organization boundary**, not a **security wall**. If you want walls, you have to build them separately (NetworkPolicies, RBAC, quotas). Miss this and you'll wrongly believe your prod namespace is protected from your dev namespace. It is not, until you make it so.

---

## The Linux kernel feature / mechanism underneath

Careful — there are **two totally different things both called "namespace"** and mixing them up is a classic confusion:

- **Linux namespaces** (Topic 02): a kernel feature that isolates what a *process* can see — its PID namespace, network namespace, mount namespace. This is what makes a *container* a container.
- **Kubernetes Namespaces** (this topic): a purely logical partition of the Kubernetes API. It has **nothing to do** with Linux kernel namespaces. A Pod in the `orders` K8s Namespace and a Pod in the `payments` K8s Namespace can run on the *same node* and are isolated by Linux namespaces *per Pod*, not per K8s-Namespace.

So what is a Kubernetes Namespace *physically*? It's a **prefix on etcd keys**. That's it. When you create the `orders` Namespace and put a ConfigMap in it, the API server stores it in etcd at:

```
/registry/configmaps/orders/orders-config
│                     │
│                     └─ the namespace name is a path segment in the etcd key
└─ everything namespaced is filed under its namespace prefix
```

A cluster-scoped object (like a Node or a PersistentVolume) has **no** namespace segment: `/registry/persistentvolumes/pv-1`. That's the whole mechanism. A Namespace is a folder name in etcd's key hierarchy that the API server uses to scope `list`, `get`, and `delete` operations. When you run `kubectl get pods -n orders`, the API server simply lists etcd keys under the `.../pods/orders/` prefix.

The DNS side has a physical mechanism too: **CoreDNS** (the cluster DNS server, a Pod running in `kube-system`) programs a name for every Service that includes its namespace: `orders-postgres.orders.svc.cluster.local`. The namespace is literally a label in the DNS name. We'll use that heavily below.

---

## What is this?

A **Namespace** is a virtual sub-cluster inside one physical Kubernetes cluster — a way to group and scope objects (Pods, Services, Deployments, ConfigMaps, Secrets) under a name, so their names only need to be unique within that group. It provides a scope for names, a target for resource quotas and access policies, and an organizational unit for teams and environments.

It does **not** provide network isolation, node isolation, or security isolation on its own. Those are separate features you layer on top.

---

## Why does it matter for a backend developer?

Without namespaces, everything lands in one bucket called `default`, and three things go wrong fast:

1. **Name collisions.** Your team deploys `postgres`. Another team deploys `postgres`. One overwrites the other. Chaos. With namespaces, `orders/postgres` and `payments/postgres` coexist peacefully.

2. **No blast-radius control.** A runaway `Job` in dev spawns 500 Pods and eats the whole cluster's CPU, starving prod. With a **ResourceQuota** per namespace, dev can be capped so it *cannot* starve prod.

3. **No clean teardown or access control.** You want to give the dev team access to their stuff but not prod. RBAC (Topic 53) is scoped *by namespace* — a `RoleBinding` in `orders-dev` grants access only there. And when a project ends, `kubectl delete namespace orders-experiment` cleanly deletes *everything* inside it in one command.

Concretely for `orders-api`: you'll run it in an `orders` namespace in production, `orders-staging` for staging, and `orders-dev` for development — same manifests, different namespace, isolated quotas, separate access. And you must *know* that `orders-dev` can still reach `orders` Postgres over the network unless you add a NetworkPolicy (Topic 42), because that surprise has caused real incidents.

---

## The physical reality

Create three namespaces and here's what exists.

**In etcd**, the Namespace objects themselves plus prefixed keys for everything in them:

```
/registry/namespaces/orders                      ← the Namespace object (a real API object)
/registry/namespaces/orders-staging
/registry/namespaces/orders-dev
/registry/deployments/orders/orders-api          ← Deployment, filed under "orders"
/registry/services/orders/orders-postgres        ← Service, filed under "orders"
/registry/deployments/orders-dev/orders-api      ← a DIFFERENT Deployment, same name, other ns
```

**The default namespaces every cluster ships with:**

```
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   40d   ← where objects go if you don't specify -n
kube-system       Active   40d   ← the control plane's own Pods (CoreDNS, kube-proxy...)
kube-public       Active   40d   ← world-readable, holds cluster info
kube-node-lease   Active   40d   ← node heartbeat Lease objects (node health)
orders            Active   2d
```

**What is NOT namespaced.** Some objects are cluster-scoped and live outside any namespace:

```
$ kubectl api-resources --namespaced=false
NAME                 KIND
namespaces           Namespace          ← a Namespace itself isn't inside a namespace
nodes                Node               ← nodes are physical, shared by all namespaces
persistentvolumes    PersistentVolume   ← PVs are cluster-wide (PVCs are namespaced)
clusterroles         ClusterRole
storageclasses       StorageClass
```

That's the physical truth: namespaces are etcd key prefixes plus a handful of Namespace objects, and roughly two-thirds of resource kinds are namespaced while the rest are cluster-scoped.

---

## How it works — step by step

Full trace of using a namespace, from creation to a cross-namespace call.

1. **You create the namespace.** `kubectl create namespace orders` POSTs a `Namespace` object; the API server writes `/registry/namespaces/orders` to etcd and a controller sets its status to `Active`.

2. **You deploy into it.** Every namespaced object you `apply` with `namespace: orders` (or `-n orders`) gets stored under the `orders` etcd prefix. The API server *rejects* an object whose `metadata.namespace` doesn't match an existing, Active namespace.

3. **Name uniqueness is enforced per namespace.** The API server's uniqueness check is scoped to the prefix. `orders/orders-api` and `orders-dev/orders-api` are two different keys, so both are allowed. Within `orders`, a second `orders-api` Deployment is rejected as already-exists.

4. **CoreDNS registers a namespaced DNS name for each Service.** When you create the `orders-postgres` Service in `orders`, CoreDNS can now answer queries for `orders-postgres.orders.svc.cluster.local`. Pods in the *same* namespace can use the short name `orders-postgres`; Pods in *other* namespaces must use the fully qualified name (FQDN).

5. **The scheduler ignores namespaces entirely for placement.** This is crucial: the scheduler places Pods on nodes based on resources, not namespaces. A Pod from `orders` and a Pod from `payments` can land on the *same* node. Namespaces do **not** partition hardware.

6. **A ResourceQuota (if present) is enforced at admission.** If the `orders` namespace has a ResourceQuota, the API server checks *every* create/update against it. If deploying a Pod would push the namespace's total CPU requests over the quota, the API server **rejects** the create with a `403 exceeded quota` error — before the Pod ever reaches a node.

7. **A LimitRange (if present) mutates Pods at admission.** If the namespace has a LimitRange, the API server fills in default requests/limits on any container that omitted them, and rejects any that exceed the min/max. This happens as a mutating step during admission, before etcd write.

8. **Cross-namespace network calls just... work (by default).** When an `orders-dev` Pod connects to `orders-postgres.orders.svc.cluster.local`, kube-proxy routes it exactly like a same-namespace call (Topic 35). There is **no** network boundary at the namespace edge unless a NetworkPolicy exists. This is step where most people's mental model is wrong.

9. **Deleting the namespace cascades.** `kubectl delete namespace orders` marks it `Terminating`; a controller then deletes every object under the `orders` prefix (Deployments, Pods, Services, ConfigMaps, Secrets, PVCs...) before removing the Namespace object itself.

The key insight: **namespaces scope names, quotas, and access — but not networking, nodes, or the kernel.** Steps 5 and 8 are the ones that surprise people.

---

## Exact syntax breakdown

### Creating a Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: orders
  labels:
    team: commerce
    environment: production
```

```
apiVersion: v1                     ← Namespace is a core (v1) object
kind: Namespace
metadata:
  name: orders                     ← the namespace name; used as the etcd key prefix + DNS segment
  labels:                          ← labels on the namespace itself (Topic 38)
    team: commerce                 │  used by NetworkPolicy namespaceSelectors, tooling, cost reports
    environment: production        │  a common convention to mark env per namespace
└─ a Namespace is cluster-scoped: it has NO namespace of its own
```

### Setting the namespace on an object

```yaml
metadata:
  name: orders-api
  namespace: orders
```

```
metadata:
  name: orders-api                 ← unique only WITHIN the namespace
  namespace: orders                ← which namespace this object lives in
└─ if omitted, it defaults to "default" (or your kubectl context's namespace)
```

### A ResourceQuota (cap total resources in a namespace)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: orders-quota
  namespace: orders
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    persistentvolumeclaims: "5"
```

```
kind: ResourceQuota
spec:
  hard:                            ← the ceilings the whole namespace may not exceed
    requests.cpu: "4"              ← sum of all Pods' CPU requests ≤ 4 cores
    requests.memory: 8Gi          ← sum of all memory requests ≤ 8 GiB
    limits.cpu: "8"               ← sum of all CPU limits ≤ 8 cores
    limits.memory: 16Gi           ← sum of all memory limits ≤ 16 GiB
    pods: "20"                    ← at most 20 Pods total in this namespace
    persistentvolumeclaims: "5"   ← at most 5 PVCs (Topic 39)
└─ IMPORTANT: once a ResourceQuota on cpu/memory exists, every container MUST set
   requests/limits or its Pod is REJECTED — pair this with a LimitRange for defaults
```

### A LimitRange (per-container defaults and bounds)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: orders-limits
  namespace: orders
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: 512Mi
      defaultRequest:
        cpu: "250m"
        memory: 256Mi
      max:
        cpu: "2"
        memory: 2Gi
      min:
        cpu: "50m"
        memory: 64Mi
```

```
kind: LimitRange
spec:
  limits:
    - type: Container              ← these rules apply per CONTAINER (also: Pod, PersistentVolumeClaim)
      default:                     ← LIMITS applied if a container specifies none
        cpu: "500m"                │  500m = half a CPU core
        memory: 512Mi
      defaultRequest:              ← REQUESTS applied if a container specifies none
        cpu: "250m"
        memory: 256Mi
      max:                         ← a single container may not request/limit MORE than this
        cpu: "2"
        memory: 2Gi
      min:                         ← ...nor LESS than this
        cpu: "50m"
        memory: 64Mi
└─ LimitRange fills gaps and enforces bounds; ResourceQuota caps the namespace TOTAL
```

---

## Example 1 — basic

Create the `orders` namespace, deploy `orders-api` and its Postgres into it, and prove name scoping. Every line commented.

```bash
# 1. Create the namespace.
kubectl create namespace orders

# 2. Deploy Postgres and orders-api INTO orders (note -n orders on every command).
kubectl create deployment orders-postgres -n orders --image=postgres:16-alpine
kubectl create deployment orders-api      -n orders --image=orders-api:1.4.0

# 3. Prove they're isolated to orders — the default namespace is empty.
kubectl get pods                 # (no -n) → looks in "default" → No resources found
kubectl get pods -n orders       # → shows orders-postgres-... and orders-api-...

# 4. Prove name reuse: create a DEV namespace with its OWN orders-api of the same name.
kubectl create namespace orders-dev
kubectl create deployment orders-api -n orders-dev --image=orders-api:1.5.0-rc  # SAME name, allowed!

# 5. Stop typing -n every time: pin your context's default namespace.
kubectl config set-context --current --namespace=orders
#                          │                          │
#                          │                          └─ future commands default to "orders"
#                          └─ modify the CURRENT kubeconfig context
kubectl get pods                 # now defaults to orders, no -n needed
```

Two Deployments named `orders-api` now exist, in `orders` and `orders-dev`, running different image tags, completely independent. That's the naming boundary doing its job.

---

## Example 2 — production scenario

**The situation.** Your platform team hosts `orders`, `payments`, and a shared `orders-dev` on one cluster to save money. Last week a broken load-test in `orders-dev` spawned hundreds of Pods, ate every core, and **took down production `orders`** because Pods from all namespaces share the same nodes (trace step 5). You've been asked to (a) cap dev so it can never do that again, and (b) confirm whether dev could also *reach prod's database over the network*.

**Part A — cap dev with a ResourceQuota + LimitRange:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: dev-quota, namespace: orders-dev }
spec:
  hard:
    requests.cpu: "2"          # dev may request at most 2 cores TOTAL across all Pods
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "15"                 # and never more than 15 Pods
---
apiVersion: v1
kind: LimitRange
metadata: { name: dev-limits, namespace: orders-dev }
spec:
  limits:
    - type: Container
      default:        { cpu: "300m", memory: 256Mi }   # so devs who forget limits get sane ones
      defaultRequest: { cpu: "100m", memory: 128Mi }
      max:            { cpu: "1",    memory: 1Gi }      # no single dev container can hog a core+
```

Now when the load-test tries to spin up its 200 Pods:

```
$ kubectl scale deployment loadtest -n orders-dev --replicas=200
Error creating: pods "loadtest-..." is forbidden: exceeded quota: dev-quota,
  requested: pods=1, used: pods=15, limited: pods=15
```

The API server refuses at admission (trace step 6). Prod `orders` is untouched — dev physically cannot claim more than its slice.

**Part B — the network reality check.** You test whether dev can reach the prod database:

```bash
kubectl run probe -n orders-dev --image=postgres:16-alpine -it --rm -- \
  psql -h orders-postgres.orders.svc.cluster.local -U app -c '\l'
#         │                    │
#         │                    └─ ".orders." = the PROD namespace in the FQDN
#         └─ a dev Pod dialing the PROD Postgres Service
```

It **connects.** Dev can talk to prod's database, because namespaces are not a network boundary (trace step 8). To actually wall it off you need a NetworkPolicy in `orders` (Topic 42):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-allow-same-ns, namespace: orders }
spec:
  podSelector: { matchLabels: { app: orders-postgres } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { environment: production }   # only prod-labeled namespaces may connect
```

**The production lesson:** a ResourceQuota gave you *blast-radius* isolation (compute), but you needed a NetworkPolicy for *network* isolation. Namespaces alone gave you neither — they only organized the names. Two different problems, two different tools.

---

## Common mistakes

**Mistake 1 — Believing namespaces isolate the network.**

The #1 misconception. Someone puts prod in `orders` and dev in `orders-dev` and assumes dev can't touch prod. It can — any Pod can reach any Service in any namespace via its FQDN unless a NetworkPolicy blocks it.

```
# from a dev pod, this succeeds by default:
$ curl orders-api.orders.svc.cluster.local:3000/health
{"status":"ok"}
```

Root cause: namespaces scope *names*, not *packets* (trace step 8). **Right fix:** add NetworkPolicies (Topic 42) for network isolation, and RBAC (Topic 53) for API access isolation.

**Mistake 2 — Adding a ResourceQuota without a LimitRange, then everything breaks.**

You add a CPU/memory ResourceQuota to `orders`. Suddenly new Pods fail:

```
Error creating: pods "orders-api-..." is forbidden: failed quota: orders-quota:
  must specify limits.cpu, limits.memory, requests.cpu, requests.memory
```

Root cause: once a quota constrains `requests.cpu`/`limits.cpu`, the API server requires **every** container to declare them; any container that omits them is rejected. **Right fix:** pair the ResourceQuota with a **LimitRange** that supplies `default`/`defaultRequest`, so containers without explicit values get filled in automatically (trace step 7).

**Mistake 3 — Forgetting `-n` and deploying to `default`.**

```bash
kubectl apply -f orders-api.yaml     # oops, no -n orders
kubectl get pods -n orders           # No resources found  ← "where did my app go?!"
kubectl get pods                     # there it is, in default
```

Root cause: with no `namespace` in the manifest and no `-n`, objects go to `default`. **Right fix:** put `namespace: orders` in every manifest's `metadata`, and/or pin your context with `kubectl config set-context --current --namespace=orders`.

**Mistake 4 — Using the short Service name across namespaces.**

Your `orders-dev` app tries to reach prod Postgres with the short name and DNS fails:

```
$ node -e "require('dns').lookup('orders-postgres', console.log)"
Error: getaddrinfo ENOTFOUND orders-postgres
```

Root cause: the short name `orders-postgres` resolves only within the *same* namespace (CoreDNS search domains). Across namespaces you must use the FQDN (trace step 4). **Right fix:** use `orders-postgres.orders.svc.cluster.local` (or at least `orders-postgres.orders`).

**Mistake 5 — A namespace stuck `Terminating` forever.**

```
$ kubectl get ns orders-experiment
NAME                STATUS        AGE
orders-experiment   Terminating   3h      ← won't die
```

Root cause: an object in it has a **finalizer** that never completed (often a custom resource whose controller is gone), so the cascade delete (trace step 9) blocks. **Right fix:** find the stuck resource (`kubectl get all,ingress,... -n orders-experiment`) and remove the finalizer, or as a last resort clear the namespace's `spec.finalizers`. Never just ignore it — the name stays reserved.

---

## Hands-on proof

Run these **right now** on any cluster.

```bash
# 1. See the built-in namespaces.
kubectl get namespaces

# 2. Create two namespaces and prove name reuse across them.
kubectl create namespace demo-a
kubectl create namespace demo-b
kubectl create deployment web --image=nginx -n demo-a
kubectl create deployment web --image=nginx -n demo-b   # SAME name — allowed (different ns)
kubectl get deploy -A | grep web                        # both exist

# 3. PROOF namespaces don't isolate the network: reach a Service across namespaces.
kubectl expose deployment web --port=80 -n demo-a
kubectl run probe -n demo-b --image=busybox -it --rm -- \
  wget -qO- web.demo-a.svc.cluster.local     # succeeds from demo-b into demo-a

# 4. PROOF short names are namespace-local: this FAILS from demo-b.
kubectl run probe2 -n demo-b --image=busybox -it --rm -- \
  nslookup web        # cannot resolve "web" — it's not in demo-b

# 5. PROOF a ResourceQuota blocks at admission.
kubectl create quota tiny --hard=pods=1 -n demo-a
kubectl scale deployment web --replicas=5 -n demo-a
kubectl get events -n demo-a | grep -i quota   # "exceeded quota" — only 1 pod allowed

# 6. See what's namespaced vs cluster-scoped.
kubectl api-resources --namespaced=true  | head
kubectl api-resources --namespaced=false | head   # nodes, PVs, namespaces themselves...

# 7. Clean up — one command deletes EVERYTHING inside.
kubectl delete namespace demo-a demo-b
```

What you verified: name reuse across namespaces (step 2), cross-namespace networking works by default (step 3), short names are namespace-local (step 4), and quotas block at admission (step 5).

---

## Practice exercises

### Exercise 1 — easy
Create a namespace `orders-staging`. Deploy an `nginx` Deployment named `orders-api` into it. Then deploy another `nginx` Deployment *also* named `orders-api` into the `default` namespace. Show both exist with `kubectl get deploy -A`. Explain in one sentence why the duplicate name was allowed.

### Exercise 2 — medium
On the `orders-staging` namespace, apply a ResourceQuota of `pods: "3"` and a LimitRange giving containers a default request of `100m` CPU / `128Mi`. Deploy a container that specifies **no** resources and confirm it starts (the LimitRange filled them in). Then scale to 5 replicas and show that only 3 Pods run, with an "exceeded quota" event for the rest.

### Exercise 3 — hard (production simulation)
Create `orders` (prod) and `orders-dev`. Run Postgres in `orders`. From a Pod in `orders-dev`, prove you can `nc`/`wget` the Postgres Service in `orders` using its FQDN (no isolation). Then write and apply a NetworkPolicy in `orders` that only allows ingress to Postgres from Pods in namespaces labeled `environment=production`. Re-run the dev probe and show it now **times out**. Write down the two separate mechanisms you used: one for names, one for network.

---

## Mental model checkpoint

Answer from memory:

1. Physically, what *is* a Kubernetes Namespace in etcd?
2. Name three things a namespace does NOT isolate.
3. Can two Pods from different namespaces run on the same node? Why or why not?
4. What DNS name does a Pod in `orders-dev` need to reach the `orders-postgres` Service in `orders`?
5. What happens to new Pods if you add a CPU/memory ResourceQuota but no LimitRange?
6. What's the difference between a ResourceQuota and a LimitRange?
7. Which four namespaces ship with every cluster, and what is `kube-system` for?

---

## Quick reference card

| Command / field | What it does | Key detail |
|---|---|---|
| `kubectl create namespace NAME` | Create a namespace | Cluster-scoped; becomes an etcd key prefix |
| `kubectl get pods -n NAME` | List objects in a namespace | No `-n` → uses `default` |
| `kubectl get pods -A` | List across ALL namespaces | `--all-namespaces` shorthand |
| `kubectl config set-context --current --namespace=NAME` | Set default namespace | Stop typing `-n` every command |
| `kubectl delete namespace NAME` | Delete a namespace | Cascades to ALL objects inside |
| `ResourceQuota` | Cap a namespace's TOTAL resources | Enforced at admission; blocks over-quota creates |
| `LimitRange` | Per-container defaults & min/max | Fills in missing requests/limits |
| `<svc>.<ns>.svc.cluster.local` | Cross-namespace Service DNS | Short name only works same-namespace |
| `kubectl api-resources --namespaced=false` | List cluster-scoped kinds | Nodes, PVs, namespaces, ClusterRoles |
| NetworkPolicy (Topic 42) | Actual network isolation | Namespaces do NOT provide this |

**Naming conventions & when to use multiple namespaces:**
- Common patterns: `team-app` (`commerce-orders`), or `app-env` (`orders`, `orders-staging`, `orders-dev`), or per-team (`team-payments`).
- **Use multiple namespaces when:** separating teams, separating environments on one cluster, applying different quotas/RBAC, or wanting clean one-command teardown.
- **Don't over-split:** a namespace per microservice is usually overkill — group by *team* or *environment*, not per-service, or you drown in RBAC and NetworkPolicy rules.
- **For hard multi-tenant isolation** (untrusted tenants), namespaces are *not enough* — consider separate clusters (Topic 54).

---

## When would I use this at work?

1. **Running dev, staging, and prod on one cluster.** You deploy the same `orders-api` manifests to `orders-dev`, `orders-staging`, and `orders`, each with its own ConfigMap/Secret (Topic 36), its own quota, and RBAC that lets the dev team touch only `orders-dev`.

2. **Capping a noisy team so they can't starve prod.** After a load-test outage, you put a ResourceQuota + LimitRange on the dev namespace so no experiment can ever consume more than its allotted CPU/memory/Pod count — protecting production that shares the same nodes.

3. **Clean project teardown.** A team spins up `orders-experiment` for a two-week spike. When it's done, `kubectl delete namespace orders-experiment` removes every Pod, Service, ConfigMap, Secret, and PVC in one command — no orphaned resources left billing you.

---

## Connected topics

- **Study before:**
  - **Topic 02 — Linux Foundations of Containers:** so you don't confuse *Linux* namespaces (kernel) with *Kubernetes* Namespaces (etcd prefixes).
  - **Topic 35 — Services:** the Service DNS names that gain a namespace segment here.
  - **Topic 36 — ConfigMaps and Secrets:** the per-namespace config that makes one image behave per-environment.
- **Study after:**
  - **Topic 42 — Network Policies:** the actual network wall namespaces don't give you.
  - **Topic 44 — Resource Requests and Limits:** the per-container values that ResourceQuota/LimitRange govern.
  - **Topic 53 — RBAC:** namespace-scoped access control, the security boundary namespaces enable.
  - **Topic 54 — Multi-Environment Strategy:** namespaces-vs-clusters for dev/staging/prod at scale.
