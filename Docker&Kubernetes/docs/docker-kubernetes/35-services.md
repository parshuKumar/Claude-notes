# 35 — Services
## Section: Kubernetes in Practice

## ELI5 — The Simple Analogy

Imagine a pizza shop with a single **phone number on the sign**: 555-ORDERS. Behind the scenes there are 3 people who actually answer calls. People quit, new people are hired, they sit at different desks — but customers never learn each person's personal cell number. They always dial the one number on the sign, and *somebody* picks up. The shop keeps a little switchboard that routes each incoming call to whoever is currently on shift and available.

Your Pods (Topic 33) are the people answering phones. They come and go — every time a Deployment (Topic 34) rolls out a new version or replaces a dead Pod, the Pod gets a **brand-new IP address**. You can't hardcode a Pod's IP; it's like memorizing an employee's personal cell that changes weekly.

A **Service** is the phone number on the sign: a single, *stable* address (name + virtual IP) that never changes, with a switchboard that routes each request to one of the healthy Pods currently behind it. Pods churn underneath; the Service stays put.

## The Linux kernel feature underneath

This is the part almost nobody explains, so let's go deep. **A Service's ClusterIP does not belong to any machine.** There is no network interface anywhere with that IP. You cannot ping a route to it in the normal sense. So how does traffic to `10.96.0.42:80` reach a real Pod at `10.244.1.37:3000`?

The answer is **kernel-level packet rewriting** done by `kube-proxy`, using either **iptables** (the `nat` table with DNAT rules) or **IPVS** (the kernel's in-kernel load balancer). Remember from Topic 13 that Docker networking already used iptables and veth pairs — this is the same machinery, scaled up.

Here's the mechanism in **iptables mode** (the common default):

1. `kube-proxy` runs on **every node**. It watches the API server for Services and Endpoints.
2. When a Service exists, kube-proxy writes iptables rules into the kernel's `nat` table on every node. It does **not** sit in the data path — it only *programs* the kernel. The actual packet rewriting is done by the kernel's netfilter, at wire speed.
3. A rule says, roughly: *"any packet whose destination is `10.96.0.42:80` (the ClusterIP) — rewrite the destination to one of these real Pod IPs, chosen randomly, then let normal routing carry it there."* That rewrite is **DNAT** (Destination NAT).

Real rules look like this (simplified from `iptables-save -t nat`):

```
# Traffic to the Service ClusterIP jumps to a per-service chain:
-A KUBE-SERVICES -d 10.96.0.42/32 -p tcp --dport 80 -j KUBE-SVC-ORDERS

# The per-service chain picks ONE endpoint by probability (3 Pods → 1/3 each):
-A KUBE-SVC-ORDERS -m statistic --mode random --probability 0.3333 -j KUBE-SEP-POD1
-A KUBE-SVC-ORDERS -m statistic --mode random --probability 0.5000 -j KUBE-SEP-POD2
-A KUBE-SVC-ORDERS                                                   -j KUBE-SEP-POD3
   │                                            │
   │                                            └── probabilities compound: 1/3, then 1/2 of
   │                                                the rest, then the remainder = even split.
   └── one KUBE-SVC chain per Service.

# Each per-endpoint chain does the actual DNAT to a real Pod IP:
-A KUBE-SEP-POD1 -p tcp -j DNAT --to-destination 10.244.1.37:3000
-A KUBE-SEP-POD2 -p tcp -j DNAT --to-destination 10.244.2.19:3000
-A KUBE-SEP-POD3 -p tcp -j DNAT --to-destination 10.244.3.51:3000
```

So the "load balancer" for a ClusterIP is **just netfilter DNAT rules in every node's kernel.** No proxy process handles your bytes. That's why it's fast and why the ClusterIP is "virtual" — it exists only as a match target in iptables, nowhere as a real interface.

**IPVS mode** does the same job but uses the kernel's IP Virtual Server (a real in-kernel L4 load balancer with hash tables) instead of a long linear chain of iptables rules. For a cluster with thousands of Services, iptables rule evaluation gets slow (it's a list); IPVS uses hash lookups and scales far better. You choose the mode in kube-proxy's config (`mode: ipvs`).

The other kernel-adjacent piece is **DNS**. The name `orders-api.orders.svc.cluster.local` is resolved by **CoreDNS** (a Pod running in the cluster) to the Service's ClusterIP. Every Pod's `/etc/resolv.conf` is written by the kubelet to point at CoreDNS. That's how a name becomes a virtual IP, which iptables then turns into a real Pod IP.

## What is this?

A **Service** is a Kubernetes object that gives a stable name and virtual IP to a *set* of Pods (chosen by label selector), and load-balances traffic across the healthy ones. It decouples "who wants to talk to `orders-api`" from "which exact Pods are running right now," which change constantly.

There are four flavors — **ClusterIP** (internal only), **NodePort** (a port on every node), **LoadBalancer** (a cloud load balancer), and **Headless** (no virtual IP, direct Pod IPs) — plus **ExternalName** (a DNS CNAME alias).

## Why does it matter for a backend developer?

Without Services, your Node.js app cannot reliably talk to anything:

- **Pod IPs are ephemeral.** Your `orders-api` needs to reach Postgres. If you hardcode the Postgres Pod's IP, the next time that Pod restarts (new IP), your API breaks with `ECONNREFUSED`. A Service named `postgres` gives you a name that never changes.
- **Load balancing for free.** Run 3 replicas of `orders-api` and a Service spreads requests across all 3 — no nginx to configure, it's kernel iptables.
- **Service discovery by DNS.** In your code you write `postgres://postgres.orders.svc.cluster.local:5432/db` or just `postgres:5432` (same namespace) — a human-readable name, resolved automatically.
- **Exposing to the outside world.** Users on the internet can't reach a Pod IP (it's private, cluster-internal). A `LoadBalancer` or `NodePort` (or an Ingress, Topic 41) is how external traffic gets in.

Not understanding Services leads to the classic beginner bug: "my API works with `kubectl port-forward` but not from another Pod" — usually a selector or port mismatch in the Service.

## The physical reality

Apply an `orders-api` Service and here's what exists.

**In etcd:**
```
/registry/services/specs/orders/orders-api
    → type=ClusterIP, clusterIP=10.96.0.42, port=80, targetPort=3000, selector{app:orders-api}
/registry/services/endpoints/orders/orders-api          (legacy Endpoints)
/registry/endpointslices/orders/orders-api-abcde        (modern EndpointSlice)
    → the LIST of Pod IPs currently backing this Service:
      10.244.1.37:3000  ready=true
      10.244.2.19:3000  ready=true
      10.244.3.51:3000  ready=true
```

**In every node's kernel (iptables nat table):** the DNAT chains shown above. Run on any node:
```bash
iptables-save -t nat | grep orders-api
```

**In CoreDNS:** an A record mapping `orders-api.orders.svc.cluster.local → 10.96.0.42`.

**In every Pod's `/etc/resolv.conf`** (written by kubelet):
```
nameserver 10.96.0.10                              ← the CoreDNS Service's ClusterIP
search orders.svc.cluster.local svc.cluster.local cluster.local
                                                   ← lets you use short names like "postgres"
options ndots:5
```
That `search` line is why, from a Pod in namespace `orders`, `curl http://orders-api` works: DNS appends `orders.svc.cluster.local` automatically.

## How it works — step by step

Trace one HTTP request from an `orders-api` Pod to a `postgres` Service.

1. **Your code connects to `postgres:5432`.** The Node.js `pg` driver calls `getaddrinfo("postgres")`.
2. **DNS resolution.** The libc resolver reads `/etc/resolv.conf`, sends the query to CoreDNS at `10.96.0.10`. Because of `ndots:5` and the `search` list, it tries `postgres.orders.svc.cluster.local` — CoreDNS answers with the Service's ClusterIP `10.96.0.55`.
3. **Your app opens a TCP connection to `10.96.0.55:5432`.** The first SYN packet leaves your Pod's network namespace via its veth into the node's root namespace.
4. **Netfilter intercepts in the `nat` table (PREROUTING/OUTPUT).** The `KUBE-SERVICES` chain matches destination `10.96.0.55:5432`, jumps to `KUBE-SVC-POSTGRES`, which by probability picks one endpoint chain, which **DNATs** the destination to a real Pod IP, say `10.244.4.88:5432`.
5. **The connection is tracked (conntrack).** The kernel records this NAT mapping so *all subsequent packets* of this TCP connection go to the same Pod — the load-balancing decision is per-connection, not per-packet. Return packets are un-NAT'd automatically so your app thinks it talked to `10.96.0.55` the whole time.
6. **Normal pod-to-pod routing** (Topic 40, via the CNI) carries the packet to the Postgres Pod, possibly on another node.
7. **A Pod dies?** The EndpointSlice controller (in kube-controller-manager) notices the Pod is no longer Ready (via its readiness probe, Topic 43), removes its IP from the EndpointSlice. kube-proxy sees the change and **rewrites the iptables rules** to drop that endpoint. New connections never get routed to the dead Pod. This is the whole point: churn underneath, stable address on top.

## Exact syntax breakdown

A full ClusterIP Service for `orders-api`, every field annotated.

```yaml
apiVersion: v1                 # Service is a core object → "v1"
kind: Service
metadata:
  name: orders-api             # THE stable DNS name: orders-api.orders.svc.cluster.local
  namespace: orders
spec:
  type: ClusterIP              # ClusterIP | NodePort | LoadBalancer | ExternalName
  selector:                    # which Pods back this Service (by labels, Topic 38)
    app: orders-api            # any Pod with label app=orders-api and Ready=true
  ports:
    - name: http               # optional name (required if >1 port)
      protocol: TCP            # TCP | UDP | SCTP
      port: 80                 # the port the SERVICE listens on (what clients dial)
      targetPort: 3000         # the port on the POD/container to forward to
```

The single most confusing pair — `port` vs `targetPort`:

```
    - port: 80
      │     └── clients connect to the Service on THIS port: orders-api:80
      └──────── this is the ClusterIP's port.
      targetPort: 3000
      │           └── kube-proxy DNATs to THIS port on the chosen Pod (your app listens here).
      └───────────── must match the container's actual listening port (containerPort).
```
So: `client → orders-api:80  →(DNAT)→  10.244.x.y:3000`. They are usually different on purpose (dial a friendly 80, app runs on 3000).

A **NodePort** adds one field:
```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080          # a port opened on EVERY node's IP (range 30000–32767)
      │                          reach the Service via  <anyNodeIP>:30080
      └───────────────────────  omit it and Kubernetes picks a free one for you.
```

A **LoadBalancer** (only does something on a cloud that provides one):
```yaml
spec:
  type: LoadBalancer           # cloud provisions an external L4 LB with a public IP
  ports:
    - port: 80
      targetPort: 3000
# status.loadBalancer.ingress[0].ip  ← the public IP the cloud assigned, watch with `get svc`
```
A LoadBalancer is a superset: it also allocates a NodePort and a ClusterIP under the hood. The cloud LB forwards to the NodePort on the nodes, which iptables forwards to a Pod.

A **Headless** Service (no virtual IP, no load balancing — you get the raw Pod IPs):
```yaml
spec:
  clusterIP: None              # THIS makes it headless
  │          └── DNS returns ALL Pod IPs (A records), not one virtual IP.
  └───────────── used with StatefulSets (Topic 47) where each Pod needs its own address.
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

## Example 1 — basic

Expose an `orders-api` Deployment internally, then call it from another Pod by name.

```yaml
# file: svc-basic.yaml
apiVersion: v1
kind: Service
metadata:
  name: orders-api            # becomes the DNS name
  namespace: orders
spec:
  type: ClusterIP             # internal-only (default)
  selector:
    app: orders-api           # matches the Deployment's Pod labels (Topic 34)
  ports:
    - port: 80                # dial orders-api:80
      targetPort: 3000        # app listens on 3000
```

```bash
kubectl apply -f svc-basic.yaml

# The Service got a stable ClusterIP:
kubectl get svc orders-api -n orders
#   NAME         TYPE        CLUSTER-IP     PORT(S)
#   orders-api   ClusterIP   10.96.0.42     80/TCP

# The Endpoints show the actual Pod IPs behind it:
kubectl get endpointslice -n orders -l kubernetes.io/service-name=orders-api
#   ADDRESSES: 10.244.1.37, 10.244.2.19, 10.244.3.51   (your 3 replicas)

# Call it BY NAME from a throwaway Pod in the same namespace:
kubectl run tmp -n orders --rm -it --image=busybox:1.36 -- \
  sh -c 'wget -qO- http://orders-api:80/ ; echo'
#   → "ok"  (DNS resolved orders-api → 10.96.0.42, iptables DNAT'd to a Pod)
```

## Example 2 — production scenario: internal DB, public API

**The situation.** Your `orders` namespace runs three things: `orders-api` (must be reachable from the internet), `postgres` and `redis` (must be reachable ONLY from inside the cluster — never exposed). You wire this with three Services of different types.

```yaml
# file: svc-prod.yaml
---
# 1. Postgres: internal-only, and headless so the API always talks to the SAME instance.
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: orders
spec:
  clusterIP: None             # headless: DNS returns the Pod IP directly (1 stable DB Pod)
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
---
# 2. Redis: plain internal ClusterIP with load balancing across replicas.
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: orders
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
---
# 3. orders-api: exposed to the internet via a cloud LoadBalancer.
apiVersion: v1
kind: Service
metadata:
  name: orders-api
  namespace: orders
spec:
  type: LoadBalancer         # cloud gives it a public IP
  selector:
    app: orders-api
  ports:
    - name: http
      port: 80               # public port 80...
      targetPort: 3000       # ...to app port 3000
```

In your Node.js `orders-api` code, connection strings become clean, stable names:

```js
// Same namespace → short names work thanks to the resolv.conf search list.
const pg    = new Pool({ connectionString: 'postgres://user:pass@postgres:5432/orders' });
const redis = createClient({ url: 'redis://redis:6379' });
// No IPs anywhere. Pods can restart, roll out, reschedule — these names never change.
```

Verify and reach each one:

```bash
kubectl apply -f svc-prod.yaml
kubectl get svc -n orders
#   NAME         TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
#   postgres     ClusterIP      None          <none>          5432/TCP     ← headless
#   redis        ClusterIP      10.96.0.71    <none>          6379/TCP
#   orders-api   LoadBalancer   10.96.0.42    203.0.113.10    80:31544/TCP ← public IP!

# Public traffic (from your laptop) hits the LoadBalancer's EXTERNAL-IP:
curl http://203.0.113.10/health          # → served by one of the orders-api Pods

# Postgres/redis have NO external IP — unreachable from outside, only cluster-internal.
```

Why headless for Postgres: a single primary database Pod (Topic 47, StatefulSet) doesn't want a virtual IP load-balancing across "replicas" — you want DNS to hand back the *one* Pod's real IP so the connection (and connection pooling) goes straight to it. `clusterIP: None` does exactly that.

## Common mistakes

**Mistake 1 — Service selector doesn't match Pod labels → no endpoints.**
```yaml
Service:    selector: { app: orders-api }
Deployment: template.labels: { app: orders }   # mismatch
```
Symptom: the Service exists but connections hang or refuse.
```bash
kubectl get endpointslice -n orders -l kubernetes.io/service-name=orders-api
#   ADDRESSES: <none>          ← EMPTY. No Pods matched → no DNAT targets → nothing to route to.
```
**Root cause:** a Service finds Pods purely by label selector. No match → empty EndpointSlice → kube-proxy has nothing to DNAT to.
**Right:** make the Service `selector` equal the Pod template labels. `kubectl describe svc orders-api` shows the `Endpoints:` line — if it's empty, it's always a selector or readiness problem.

**Mistake 2 — Confusing `port` and `targetPort`.**
```yaml
ports: [{ port: 3000, targetPort: 80 }]   # app actually listens on 3000, not 80!
```
Symptom: connection refused *after* DNS resolves fine.
```
curl: (7) Failed to connect ... Connection refused
```
**Root cause:** kube-proxy DNATs to `targetPort` (80) on the Pod, but your app listens on 3000. The packet arrives at a closed port.
**Right:** `targetPort` must equal the container's real listening port; `port` is whatever clients dial.

**Mistake 3 — Endpoints empty because Pods aren't Ready.**
```bash
kubectl get pods -n orders     # pods show 0/1 READY
kubectl get endpointslice ...  # ADDRESSES: <none>
```
**Root cause:** only Pods passing their **readiness probe** (Topic 43) are added to the EndpointSlice. If readiness never passes, the Pods run but the Service refuses to route to them (correctly — they're not ready to serve).
**Right:** fix the readiness probe / app startup. This is a feature: it prevents traffic hitting a not-yet-ready Pod.

**Mistake 4 — Expecting `type: LoadBalancer` to work on bare-metal/minikube.**
```bash
kubectl get svc orders-api -n orders
#   EXTERNAL-IP: <pending>        ← stuck forever
```
**Root cause:** `LoadBalancer` needs a cloud controller (AWS/GCP/Azure) to provision a real LB. On minikube/bare-metal there's nobody to fulfill it, so `EXTERNAL-IP` stays `<pending>`.
**Right:** use `minikube tunnel`, or install MetalLB on bare-metal, or use `NodePort`/Ingress instead.

**Mistake 5 — Cross-namespace call using the short name.**
```js
// orders-api Pod (namespace "orders") trying to reach a Service in namespace "payments":
fetch('http://billing:8080')     // fails — resolves to billing.orders, which doesn't exist
```
```
getaddrinfo ENOTFOUND billing
```
**Root cause:** the `resolv.conf` search list only appends the *caller's* namespace first. `billing` becomes `billing.orders.svc.cluster.local` — wrong namespace.
**Right:** use the fully-qualified name across namespaces: `http://billing.payments.svc.cluster.local:8080` (or at least `billing.payments`).

## Hands-on proof

```bash
kubectl create namespace orders 2>/dev/null
kubectl create deployment orders-api --image=hashicorp/http-echo \
  -- /http-echo -text=ok -listen=:3000  -n orders
kubectl scale deploy/orders-api --replicas=3 -n orders

# 1. Create a ClusterIP Service and see its virtual IP.
kubectl expose deployment orders-api --port=80 --target-port=3000 -n orders
kubectl get svc orders-api -n orders          # note the CLUSTER-IP (e.g. 10.96.0.42)

# 2. See the real Pod IPs behind it.
kubectl get endpointslice -n orders -l kubernetes.io/service-name=orders-api -o wide
#   → three addresses, one per replica.

# 3. Resolve the DNS name from inside the cluster.
kubectl run dns -n orders --rm -it --image=busybox:1.36 -- \
  nslookup orders-api.orders.svc.cluster.local
#   → Address: 10.96.0.42   (the ClusterIP, from CoreDNS)

# 4. See the actual kernel DNAT rules (on a node).
minikube ssh -- 'sudo iptables-save -t nat | grep -i orders-api'
#   → KUBE-SVC-... chains and KUBE-SEP-... DNAT --to-destination 10.244.x.y:3000

# 5. Watch endpoints update live as Pods churn.
kubectl get endpointslice -n orders -w &
kubectl delete pod -n orders -l app=orders-api --wait=false | head -1
#   → the deleted Pod's IP disappears from the slice; the replacement's IP appears.
```

## Practice exercises

### Exercise 1 — easy
Create an `orders-api` Deployment (3 replicas) and a ClusterIP Service on `port: 80 → targetPort: 3000`. From a `busybox` Pod in the same namespace, `wget` the Service by its short name `orders-api` ten times and confirm it answers. Then run `kubectl describe svc orders-api` and read the `Endpoints:` line — count the IPs.

### Exercise 2 — medium
Change the Service selector to a label that no Pod has (e.g. `app: nope`). Observe the EndpointSlice go empty and requests start hanging. Fix the selector and watch endpoints repopulate. Explain in one sentence, using "kube-proxy has no DNAT target," why the requests hung.

### Exercise 3 — hard (production simulation)
Stand up all three Services from Example 2 (headless `postgres`, ClusterIP `redis`, `NodePort` `orders-api` since you're on minikube). Prove: (a) `orders-api` is reachable from your laptop via `<minikube-ip>:<nodePort>`; (b) `postgres`'s DNS returns a Pod IP directly (not a virtual IP) because it's headless; (c) `redis` is reachable from another Pod but has no external route. Then inspect the iptables `nat` table and identify the `KUBE-SVC` chain and probability rules for `redis`, and explain how three replicas would each get ~1/3 of connections.

## Mental model checkpoint

1. Why can't you hardcode a Pod's IP, and what does a Service give you instead?
2. Where does a ClusterIP "live" — which interface has it? (Trick question.)
3. What does kube-proxy actually do, and who rewrites the packets?
4. What's the difference between `port` and `targetPort`?
5. What determines which Pod IPs appear in a Service's EndpointSlice?
6. Give the full DNS name for a Service `redis` in namespace `orders`.
7. When would you use a headless Service instead of a normal ClusterIP?
8. Why does `type: LoadBalancer` stay `<pending>` on minikube?

## Quick reference card

| Type / field | What it does | Key detail |
|---|---|---|
| `ClusterIP` (default) | Internal virtual IP + load balancing | Only reachable inside the cluster |
| `NodePort` | Opens a port (30000–32767) on every node | `<nodeIP>:<nodePort>` → Service; superset of ClusterIP |
| `LoadBalancer` | Cloud provisions an external L4 LB | Superset of NodePort; `<pending>` without a cloud controller |
| `clusterIP: None` (headless) | No virtual IP; DNS returns Pod IPs | For StatefulSets / when you need the real Pod addresses |
| `ExternalName` | DNS CNAME to an external host | No proxying, no selector; just a DNS alias |
| `port` vs `targetPort` | Service port vs Pod/container port | Client dials `port`; DNAT goes to `targetPort` |
| `selector` | Which Pods back the Service | Empty endpoints ⇒ selector or readiness problem |
| EndpointSlice | The live list of ready backing Pod IPs | Only Ready Pods listed; kube-proxy turns it into DNAT rules |
| DNS name | `svc.ns.svc.cluster.local` | Short name works within same namespace via `search` list |
| kube-proxy modes | `iptables` (default) or `ipvs` | IPVS scales better for thousands of Services |

## When would I use this at work?

1. **Every internal dependency.** Give Postgres, Redis, and internal microservices stable DNS names (`postgres:5432`, `redis:6379`) so `orders-api` code never touches an IP and survives Pod churn.
2. **Exposing an API.** Front `orders-api` with a `LoadBalancer` (cloud) or, more commonly, a ClusterIP Service plus an Ingress (Topic 41) for host/path routing and TLS.
3. **Debugging "it can't connect."** Check `kubectl describe svc` for empty `Endpoints:` (selector/readiness), verify `targetPort` matches the app's port, and confirm you're using the right namespace in the DNS name.

## Connected topics

- **Study before:** Topic 13 (Docker networking — veth + iptables, the same primitives), Topic 33 (Pods & their ephemeral IPs), Topic 34 (Deployments — the churning Pods a Service fronts), Topic 38 (labels & selectors — how a Service picks Pods).
- **Study after:** Topic 40 (Kubernetes networking in depth — CNI, pod-to-pod routing, kube-proxy modes), Topic 41 (Ingress — L7 routing/TLS in front of Services), Topic 42 (Network Policies — restricting who can reach a Service), Topic 43 (readiness probes — what gates Service endpoints), Topic 47 (StatefulSets — the main user of headless Services).
