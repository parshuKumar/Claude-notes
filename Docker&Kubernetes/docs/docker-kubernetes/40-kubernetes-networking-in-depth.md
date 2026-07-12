# 40 — Kubernetes Networking in Depth
## Section: Kubernetes in Practice

## ELI5 — The Simple Analogy

Imagine a giant city (your cluster). Every house in the city is a **Pod**, and every house has its own unique street address (its IP).

The city has one magical rule: **any house can send a letter directly to any other house's address, and it just arrives** — no post office translation, no fake return addresses, no "which building are you in?" games. A house in the north suburb (node 1) can mail the house in the south suburb (node 2) using the plain address, and the letter travels straight there. This is the **flat network**: one big shared address space where everyone can reach everyone.

But there's a problem. Houses get demolished and rebuilt constantly (Pods die and restart), and each rebuild gets a **new address**. If your pizza place moved every night, nobody could ever order a pizza. So the city sets up **phone numbers that never change** — a **Service**. You call the pizza-place phone number (the Service IP), and the phone company (kube-proxy) instantly connects you to *whichever* pizza house is currently open, at its current address. The phone number is virtual — there's no physical phone booth at that number, it's just a rule in the phone company's switchboard.

And who actually paves the roads between suburbs so letters can cross town? A road-building contractor called the **CNI plugin** (Calico, Flannel, etc.). Kubernetes doesn't build roads itself — it hires a contractor to do it.

## The Linux kernel feature underneath

Kubernetes networking is built entirely from **standard Linux kernel networking** you could set up by hand. There is no K8s-specific kernel module. The pieces:

```
  Pod network namespace           Node (root) network namespace
  ─────────────────────           ─────────────────────────────
   eth0 (10.244.1.5)  ◄──veth pair──►  vethXXwibble
        │                                   │
        │                              attached to →  cni0 bridge  (Flannel)
        │                                          or  routed directly (Calico)
        └── default route via cni0             │
                                               ▼
                                         node routing table + iptables/IPVS
                                               │
                                               ▼
                                      eth0 of the node (192.168.0.11)  → the real LAN
```

The kernel primitives:

- **Network namespaces** (`CLONE_NEWNET`, Topic 02) — each Pod gets its *own* network stack: its own `eth0`, its own routing table, its own `127.0.0.1`. Two Pods can both listen on port 3000 without colliding because each has a private stack.
- **veth pairs** — a virtual Ethernet cable with two ends. One end (`eth0`) lives inside the Pod's namespace; the other end lives in the node's root namespace. Packets in one end come out the other. This is the *exact* mechanism from Docker bridge networking (Topic 13), now driven by the CNI plugin.
- **Bridges** (`cni0`, `docker0`) — a virtual switch in the kernel. All Pod veth ends on a node plug into it, so same-node Pods talk via layer-2 switching.
- **Routing tables** — `ip route`. Cross-node traffic is plain IP routing: "packets for 10.244.2.0/24 go to node-2". The CNI plugin's job is to program these routes on every node.
- **iptables / IPVS / eBPF** — the kernel's packet-mangling engines. `kube-proxy` writes rules here to implement the virtual Service IP by rewriting destination addresses (DNAT).
- **IP forwarding** (`net.ipv4.ip_forward=1`) — the sysctl that lets a node forward packets that aren't addressed to itself, so it can act as a router for its Pods.

Everything in this topic is those six pieces arranged into a system. If you understand veth + bridge + routes + iptables, you understand Kubernetes networking.

## What is this?

Kubernetes networking is the set of rules and machinery that gives **every Pod its own IP**, lets **every Pod reach every other Pod directly** (across nodes, no NAT), and provides **stable virtual IPs (Services)** that load-balance to a changing set of Pods. The Pod-to-Pod plumbing is done by a **CNI plugin** (Calico, Flannel, Cilium…); the Service virtual-IP routing is done by **kube-proxy**.

## Why does it matter for a backend developer?

Because *everything* your `orders-api` does over the network flows through this, and when it breaks, the symptoms are baffling:

- `orders-api` can't reach `postgres` → is it DNS? the Service? the CNI? a NetworkPolicy? You must know the layers to debug it.
- Requests work "sometimes" → you're hitting a Service that load-balances across 3 replicas and one is broken.
- Cross-node traffic fails but same-node works → the CNI's cross-node routing/overlay is misconfigured.
- You `curl` a Service IP and `ping` fails but `curl` works → because the Service IP has **no network interface**; it only exists as an iptables rule, so ICMP/ping has nothing to hit.

Knowing where Pod IPs come from, how packets cross nodes, and how a Service IP is *not a real machine* turns these from mysteries into 5-minute fixes.

## The 4 networking problems Kubernetes solves

Kubernetes networking is defined by **four distinct problems**. Keep them separate in your head — they use different mechanisms.

```
1. Container-to-container (inside ONE Pod)
   → all containers in a Pod share ONE network namespace → talk via localhost.

2. Pod-to-Pod (across the whole cluster)
   → the flat network. Every Pod IP reachable from every other Pod, no NAT.
   → SOLVED BY: the CNI plugin.

3. Pod-to-Service (stable virtual IP → changing Pods)
   → a Service gets a virtual ClusterIP that never changes; load-balances to Pods.
   → SOLVED BY: kube-proxy (iptables/IPVS) + CoreDNS for the name.

4. External-to-Service (outside world → cluster)
   → NodePort / LoadBalancer / Ingress expose Services to outside traffic.
   → SOLVED BY: kube-proxy (NodePort), cloud LB, or Ingress controller (Topic 41).
```

The **Kubernetes network model** demands three guarantees for problem 2:

- Every Pod gets its **own cluster-unique IP**.
- Pods on the **same node** can reach each other **without NAT**.
- Pods on **different nodes** can reach each other **without NAT** — a Pod sees the *real* source IP of the sender, not a translated one.

This "no NAT between Pods, everyone on one flat IP space" is the defining property. It's what makes a Pod feel like a plain VM on a big flat LAN.

## The physical reality

Say `orders-api` (Pod IP `10.244.1.5` on node-1) talks to `postgres` (Pod IP `10.244.2.8` on node-2). Here's what physically exists.

**Inside the `orders-api` Pod's network namespace:**

```
# kubectl exec orders-api -- ip addr
1: lo:   127.0.0.1/8                       ← private loopback, only this Pod
3: eth0: 10.244.1.5/24                     ← the Pod IP; one end of a veth pair
         └── default route: via 10.244.1.1 dev eth0   (the node's cni0 bridge)
```

**On node-1 (root network namespace):**

```
# ip addr
cni0:      10.244.1.1/24                    ← bridge; gateway for all Pods on node-1
veth3f9a2: <up>  master cni0                ← the OTHER end of orders-api's veth, on the bridge
eth0:      192.168.0.11/24                  ← node's real NIC on the LAN

# ip route
10.244.1.0/24 dev cni0                      ← local Pods: switch via the bridge
10.244.2.0/24 via 192.168.0.12 dev eth0     ← node-2's Pods: route to node-2's real IP
                                              (this route is PROGRAMMED BY THE CNI PLUGIN)
```

**The Service IP has no physical existence.** If `postgres` is fronted by a Service at `10.96.0.20`:

```
# ip addr | grep 10.96.0.20        → NOTHING. No interface owns it.
# iptables-save | grep 10.96.0.20  → rules that DNAT it to a real Pod IP
```

The ClusterIP `10.96.0.20` lives **only** as iptables (or IPVS) rules in the kernel's netfilter tables. There is no NIC, no process, no machine at that address. That is why `ping 10.96.0.20` fails while `curl 10.96.0.20:5432` works — `curl` sends a TCP packet that the DNAT rule catches and rewrites; `ping` sends ICMP that no rule matches.

## How it works — step by step

### Part A — How a Pod gets its IP (Pod creation)

1. **The scheduler assigns the Pod to node-1** (Topic 33). The kubelet on node-1 is told to start it.
2. **The kubelet asks the CRI runtime** (containerd) to create the Pod's **pause container** first — a tiny do-nothing container that exists only to *hold the network namespace* open for the whole Pod. All real containers join this namespace.
3. **The kubelet (via the CRI) calls the CNI plugin**, passing the Pod's namespace path (`/proc/<pause-pid>/ns/net`) and reading config from `/etc/cni/net.d/`.
4. **The CNI plugin does the network setup:**
   - Creates a **veth pair**; puts one end in the Pod's netns renamed `eth0`, leaves the other in the root netns (`vethXXXX`).
   - Asks its **IPAM** (IP Address Management) module for a free IP from this node's Pod CIDR (e.g. node-1 owns `10.244.1.0/24`), say `10.244.1.5`. Assigns it to `eth0`.
   - Attaches the root-side veth to the node's bridge (`cni0`) — or, for Calico, sets a direct route to it.
   - Adds a default route inside the Pod (`via 10.244.1.1`).
5. **The CNI plugin returns the assigned IP** to the kubelet, which records it in the Pod's status (visible via `kubectl get pod -o wide`, stored in etcd).
6. **Real containers start** and join the pause container's netns, so they all share `eth0` and `10.244.1.5`. This is why containers in one Pod reach each other on `localhost` (problem 1).

### Part B — Pod-to-Pod across nodes (the flat network)

`orders-api` (10.244.1.5, node-1) opens a TCP connection to `postgres` at 10.244.2.8 (node-2):

7. The packet leaves `eth0` inside the Pod, travels the veth cable into node-1's root namespace, onto the `cni0` bridge.
8. Node-1's **routing table** matches `10.244.2.0/24 via 192.168.0.12 dev eth0`. This route was installed by the CNI plugin so node-1 knows node-2 hosts that Pod range.
9. **Now the two CNI families differ:**
   - **Flannel (VXLAN overlay):** the packet is **encapsulated** — wrapped inside a UDP packet addressed to node-2's real IP (192.168.0.12). It crosses the physical LAN as ordinary UDP. Node-2's `flannel.1` VXLAN device **unwraps** it, revealing the original packet for 10.244.2.8. Works on any network because the LAN only ever sees normal UDP between node IPs.
   - **Calico (BGP, no overlay):** no wrapping. Calico runs a routing protocol (BGP) so every node's kernel routing table knows "10.244.2.8 is reachable via node-2". The raw Pod packet is just **routed** across the LAN like any IP packet. Faster (no encap overhead) but needs the underlying network to permit those routes.
10. On node-2, the packet is delivered to `cni0`/routing, down the veth into the `postgres` Pod's `eth0`, to Postgres listening on 5432.

Crucially, `postgres` sees the source IP as `10.244.1.5` — the **real** `orders-api` Pod IP, **not NAT'd**. That's the flat-network guarantee.

### Part C — Pod-to-Service (kube-proxy)

`orders-api` connects to Postgres via a Service `postgres.orders.svc.cluster.local` → ClusterIP `10.96.0.20:5432`:

11. **DNS resolution:** the Pod's `/etc/resolv.conf` points at CoreDNS's Service IP. CoreDNS returns `10.96.0.20` for the name.
12. `orders-api` opens a connection to `10.96.0.20:5432`. The packet enters the kernel.
13. **kube-proxy has pre-installed rules.** When the Service was created, kube-proxy (watching the API server) wrote netfilter rules. The packet hits an iptables `KUBE-SERVICES` rule matching dest `10.96.0.20:5432`.
14. The rule **DNATs** (rewrites the destination) to one of the Service's healthy backend Pods — picked from the **EndpointSlice** (the live list of Pod IPs behind the Service). Say it rewrites dest to `10.244.2.8:5432`.
15. From here it's **Part B**: a normal Pod-to-Pod packet to 10.244.2.8, routed by the CNI. The reply is un-DNAT'd on the way back so `orders-api` still thinks it's talking to `10.96.0.20`.

kube-proxy never touches the data packets in a proxy process — it only **programs kernel rules** and the kernel does the rewriting at line rate. (Full Service detail is Topic 35; here we focus on the *why-it-has-no-interface* mechanics.)

## Exact syntax breakdown

### Inspecting the CNI config on a node

```bash
cat /etc/cni/net.d/10-flannel.conflist
```

```
/etc/cni/net.d/                 ← kubelet reads CNI config from HERE, lowest filename first
                │
                └── 10-flannel.conflist   the "10-" prefix sets ordering
```

```json
{
  "name": "cbr0",
  "cniVersion": "1.0.0",
  "plugins": [
    { "type": "flannel", "delegate": { "hairpinMode": true, "isDefaultGateway": true } },
    { "type": "portmap", "capabilities": { "portMappings": true } }
  ]
}
```

```
"plugins": [ … ]        ← a CHAIN; each plugin runs in order on Pod setup
  "type": "flannel"     ← the main plugin: creates veth, calls IPAM, wires the bridge
  "type": "portmap"     ← a chained plugin: sets up hostPort → Pod DNAT rules
```

### Seeing the pieces with kubectl / ip

```bash
kubectl get pod orders-api -n orders -o wide
```

```
NAME         READY   STATUS    IP            NODE
orders-api   1/1     Running   10.244.1.5    node-1
                                │             │
                                │             └── which node's netns holds this Pod
                                └── the Pod IP the CNI's IPAM assigned (from node-1's CIDR)
```

```bash
kubectl get service postgres -n orders
```

```
NAME       TYPE        CLUSTER-IP     PORT(S)
postgres   ClusterIP   10.96.0.20     5432/TCP
                       │
                       └── VIRTUAL. No interface owns it. Only iptables/IPVS rules.
                           Comes from the service CIDR (--service-cluster-ip-range),
                           a DIFFERENT range from the Pod CIDR.
```

```bash
kubectl get endpointslices -n orders -l kubernetes.io/service-name=postgres
```

```
ADDRESSES        ← the REAL Pod IPs behind the Service. kube-proxy turns these into DNAT targets.
10.244.2.8       ← when this Pod dies/moves, this list updates, and kube-proxy rewrites rules.
```

### Two CIDRs you must not confuse

```
Pod CIDR      (e.g. 10.244.0.0/16)  → real, routable, one /24 slice per node → Pod eth0 IPs
Service CIDR  (e.g. 10.96.0.0/12)   → virtual, never on any interface → ClusterIP addresses
```

```
--cluster-cidr=10.244.0.0/16          ← given to the CNI / controller-manager: Pod IP pool
--service-cluster-ip-range=10.96.0.0/12  ← given to the API server: virtual Service IP pool
```

## CNI plugins compared

| Plugin | How cross-node works | Pros | Cons |
|--------|----------------------|------|------|
| **Flannel** | VXLAN **overlay** (encapsulate Pod packets in UDP between node IPs) | Dead simple, works on any network | Encap overhead; basic; no NetworkPolicy by default |
| **Calico** | **BGP routing**, no overlay (or optional IPIP/VXLAN) | Fast (native routing), rich NetworkPolicy | Needs network to allow Pod routes; more moving parts |
| **Cilium** | **eBPF** (bypasses iptables), overlay or routing | Very fast, deep observability, replaces kube-proxy | Newer, steeper learning curve |

The mental split: **Flannel wraps packets so the physical network doesn't need to know about Pod IPs. Calico teaches the physical network the routes so it can carry Pod packets natively.** Both give you the same flat-network guarantee — the difference is *how the packet crosses the wire between nodes*.

## Example 1 — basic

See the flat network with your own eyes on a 2-node kind cluster.

```bash
# a kind cluster with 2 nodes uses a real CNI (kindnet) — perfect for this
kind create cluster --config - <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
EOF

kubectl create namespace orders

# two simple pods; K8s will likely spread them across the 2 nodes
kubectl -n orders run pod-a --image=nicolaka/netshoot --command -- sleep 3600
kubectl -n orders run pod-b --image=nicolaka/netshoot --command -- sleep 3600
kubectl -n orders wait --for=condition=Ready pod/pod-a pod/pod-b

# note their IPs and which node each is on
kubectl -n orders get pods -o wide
# pod-a  10.244.0.5  control-plane
# pod-b  10.244.1.4  worker          ← different node, different /24

# from pod-a, reach pod-b DIRECTLY by its Pod IP — no NAT, across nodes
kubectl -n orders exec pod-a -- ping -c2 10.244.1.4      # works
kubectl -n orders exec pod-a -- curl -s 10.244.1.4       # direct Pod-to-Pod

# prove NO nat: run a listener in pod-b and check the source IP it sees
kubectl -n orders exec pod-b -- sh -c 'nc -l -p 8080 &' 
kubectl -n orders exec pod-a -- sh -c 'echo hi | nc 10.244.1.4 8080'
# pod-b sees source = 10.244.0.5 (pod-a's REAL IP), not a translated one
```

Every line proves a rule of the model: unique Pod IPs, cross-node reachability, no NAT.

## Example 2 — production scenario

**The situation:** Your `orders-api` Deployment (3 replicas) talks to Postgres via the Service `postgres.orders.svc.cluster.local`. It works. Then you deploy a new feature and suddenly **1 in 3 requests to Postgres hang**, but `kubectl logs` shows no crashes. Same-node requests seem fine; the failures feel random.

**The investigation, using this topic's mental model:**

```bash
# 1. Is DNS the problem? (Pod-to-Service starts with a name → IP)
kubectl -n orders exec deploy/orders-api -- nslookup postgres
#  → resolves to 10.96.0.20  �so DNS is fine

# 2. What are the REAL Pod IPs behind the Service? (the EndpointSlice)
kubectl -n orders get endpointslices -l kubernetes.io/service-name=postgres -o wide
#  ADDRESSES        READY
#  10.244.2.8       true
#  10.244.3.9       false   ← a backend Pod is listed but NOT ready/healthy
```

There's the smoking gun. The Service load-balances (kube-proxy DNAT, step 14) across the EndpointSlice. One backend Pod (`10.244.3.9`) is in the slice but broken. Roughly 1/N connections get DNAT'd to the bad Pod and hang.

**Why was a broken Pod in the endpoints list?** Because it had **no readiness probe** (Topic 43). Without a readiness probe, Kubernetes assumes a running Pod is ready and adds its IP to the EndpointSlice the instant the container starts — even while Postgres is still doing crash recovery and not yet accepting connections.

**The fix:** add a readiness probe so kube-proxy only routes to Pods that actually accept traffic.

```yaml
# in the postgres pod template
readinessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]   # only "ready" when PG accepts connections
  periodSeconds: 5
```

Now a not-yet-ready or unhealthy Postgres Pod is **removed from the EndpointSlice**; kube-proxy's rules drop it as a DNAT target, and no connection is routed there. The "random 1-in-3 hang" disappears.

**The lesson tying the whole topic together:** the Service IP is virtual and load-balances via kube-proxy across the *live* EndpointSlice. The EndpointSlice is only trustworthy if your Pods have honest readiness probes. Networking, Services, and health checks are one system.

## Common mistakes

**Mistake 1 — `ping <ClusterIP>` fails and you conclude the Service is down.**

```
kubectl exec pod-a -- ping 10.96.0.20
#  100% packet loss
kubectl exec pod-a -- curl 10.96.0.20:5432
#  connects fine
```

Root cause: a ClusterIP has **no network interface** — it exists only as iptables/IPVS DNAT rules that match **TCP/UDP** to the Service port. `ping` sends ICMP, which no rule catches, so it's dropped. This is normal, not a fault.
Right: test Services with the actual protocol/port (`curl`, `nc`, `psql`), never `ping`.

**Mistake 2 — All Pods stuck `ContainerCreating`, `NetworkPluginNotReady`.**

```
kubectl describe pod orders-api
  Warning  FailedCreatePodSandBox  ... failed to setup network for sandbox:
           plugin type="..." failed: ... /etc/cni/net.d has no valid config
```

Root cause: no CNI plugin is installed (or its config in `/etc/cni/net.d/` is missing/broken). Kubernetes deliberately does **not** ship a CNI — you must install one (Calico/Flannel/Cilium). Until then the kubelet can't create Pod network namespaces, so no Pod can start.
Right: install a CNI (`kubectl apply -f <calico|flannel>.yaml`). On kind/minikube one is preinstalled.

**Mistake 3 — Same-node traffic works, cross-node traffic silently fails.**

```
# pod on node-1 → pod on node-1: OK
# pod on node-1 → pod on node-2: times out
```

Root cause: the CNI's cross-node path is broken — for an overlay (Flannel), the VXLAN UDP port (8472) is blocked by a firewall/security group; for Calico BGP, the node routes aren't propagating. Same-node works because it's just the local bridge; cross-node needs the overlay/routing to function.
Right: open the CNI's transport (e.g. UDP 8472 for Flannel VXLAN, TCP 179 for Calico BGP) between nodes; verify with `kubectl get nodes -o wide` and node routes.

**Mistake 4 — Overlapping Pod and Service CIDRs (or Pod CIDR clashing with your LAN).**

```
# Pods randomly can't reach a database that lives on the corporate LAN at 10.244.x.x
```

Root cause: the Pod CIDR (`10.244.0.0/16`) overlaps with a real network your Pods need to reach, or the Service CIDR overlaps the Pod CIDR. The kernel routing gets ambiguous and packets go to the wrong place.
Right: choose Pod and Service CIDRs that don't overlap each other **or** any external network the cluster talks to; set them at cluster-init time (they're painful to change later).

**Mistake 5 — Expecting a Pod IP to be stable and hardcoding it.**

```
DATABASE_URL=postgres://10.244.2.8:5432/orders   # WRONG
```

Root cause: Pod IPs are ephemeral — reassigned on every restart/reschedule from the node's CIDR. Hardcoding one means the connection breaks the first time Postgres moves.
Right: connect via the Service DNS name (`postgres.orders.svc.cluster.local`) — that's exactly the stable-virtual-IP problem Services solve.

## Hands-on proof

```bash
# 1. Every Pod has its own IP from the Pod CIDR
kubectl get pods -A -o wide | awk '{print $7, $1"/"$2}' | head

# 2. A ClusterIP owns NO interface — prove it
SVC=$(kubectl -n orders get svc postgres -o jsonpath='{.spec.clusterIP}')
kubectl -n orders exec deploy/orders-api -- ip addr | grep "$SVC" || echo "no interface has $SVC — it's virtual"

# 3. See kube-proxy's DNAT rules for that Service (on a node)
docker exec kind-worker sh -c "iptables-save -t nat | grep $SVC"
#   → KUBE-SERVICES ... -d 10.96.0.20/32 ... -j KUBE-SVC-xxxx
#     KUBE-SVC-xxxx ... -j KUBE-SEP-yyyy      (one SEP per backend Pod)
#     KUBE-SEP-yyyy ... DNAT --to-destination 10.244.2.8:5432   ← the real Pod

# 4. The EndpointSlice = the live backend list kube-proxy uses
kubectl -n orders get endpointslices -l kubernetes.io/service-name=postgres -o yaml | grep -A2 addresses

# 5. Look at the veth pair on the node (Pod eth0's partner)
docker exec kind-worker sh -c "ip link | grep -A1 veth | head"

# 6. See the cross-node route the CNI programmed
docker exec kind-worker sh -c "ip route | grep 10.244"
#   → 10.244.0.0/24 via <control-plane-ip>   ← how this node reaches other nodes' Pods
```

You verified: Pod IPs exist on interfaces, the Service IP does *not*, kube-proxy's DNAT rules map it to real Pods, and the CNI's routes carry cross-node traffic.

## Practice exercises

### Exercise 1 — easy
On a 2-node kind cluster, create two Pods and use `kubectl get pods -o wide` to confirm they have different IPs. From one Pod, `curl`/`ping` the other's Pod IP directly. State which of the "4 problems" you just demonstrated and why no NAT was involved.

### Exercise 2 — medium
Create a `postgres` Deployment (2 replicas) and a ClusterIP Service in `orders`. From an `orders-api` Pod: (a) `nslookup postgres` to get the ClusterIP, (b) confirm `ping <ClusterIP>` fails but `nc -zv <ClusterIP> 5432` succeeds, (c) explain in two sentences why, referencing "no interface" and "iptables DNAT".

### Exercise 3 — hard (production simulation)
Reproduce Example 2's bug on purpose. Deploy Postgres with **no readiness probe** and make one replica fail to accept connections (e.g. wrong `PGDATA` so it crash-loops but stays "Running" briefly, or a sidecar that holds the port). From `orders-api`, loop `nc -zv postgres 5432` 20 times and observe intermittent failures. Then: inspect the EndpointSlice to find the bad IP, add a `pg_isready` readiness probe, and show the bad IP leaves the EndpointSlice and the failures stop. Explain the exact chain: readiness → EndpointSlice → kube-proxy DNAT targets.

## Mental model checkpoint

Answer from memory:

1. Name the 4 networking problems Kubernetes solves and which component solves each.
2. What are the three guarantees of the Kubernetes flat network model? What does "no NAT between Pods" mean concretely?
3. Trace how a Pod gets its IP: which container holds the netns, who calls the CNI, and where does the IP come from?
4. Flannel vs Calico: how does each move a packet from a Pod on node-1 to a Pod on node-2?
5. Why does `ping <ClusterIP>` fail while `curl <ClusterIP>:port` works? Where does the ClusterIP actually "live"?
6. What is an EndpointSlice, and how does kube-proxy use it? What happens to it when a Pod becomes not-ready?
7. Why must you never hardcode a Pod IP in a connection string?

## Quick reference card

| Concept | What it is | Key detail |
|---------|-----------|------------|
| Pod network namespace | Each Pod's private network stack | Held open by the pause container |
| Pod IP | Unique cluster-wide address per Pod | From node's slice of the Pod CIDR; ephemeral |
| veth pair | Virtual cable Pod `eth0` ↔ node root ns | One end in Pod, one on the node bridge |
| CNI plugin | Wires Pod networking (veth, IPAM, routes) | K8s ships none; you install Calico/Flannel/Cilium |
| Flannel | Overlay CNI | VXLAN: encapsulates Pod packets in UDP (port 8472) |
| Calico | Routed CNI | BGP: no encap, native IP routing + NetworkPolicy |
| Flat network | Every Pod reaches every Pod, no NAT | Real source Pod IP is preserved |
| ClusterIP (Service) | Stable virtual IP for a set of Pods | No interface; only iptables/IPVS rules |
| kube-proxy | Programs Service routing in the kernel | DNATs ClusterIP → a Pod from the EndpointSlice |
| EndpointSlice | Live list of ready backend Pod IPs | Updated as Pods become ready/unready |
| Pod CIDR vs Service CIDR | Pod IP pool vs virtual IP pool | Must not overlap each other or the LAN |

## When would I use this at work?

1. **Debugging "orders-api can't reach postgres":** walk the layers — DNS (CoreDNS) → Service ClusterIP → EndpointSlice → kube-proxy DNAT → CNI cross-node route — to pinpoint whether it's a name, a Service, a dead backend, or the network fabric in minutes instead of guessing.
2. **Choosing/operating a CNI:** picking Flannel for a simple on-prem cluster vs Calico when you need NetworkPolicy (Topic 42) and native routing performance, and knowing which firewall ports (VXLAN 8472, BGP 179) to open between nodes.
3. **Explaining a flaky Service:** recognizing that intermittent failures across replicas point at the EndpointSlice/readiness (kube-proxy is fanning out to a bad Pod), not at the app code — and fixing it with a proper readiness probe.

## Connected topics

- **Study before:** Topic 02 (Linux Foundations) — network namespaces and veth; Topic 13 (Networking in Docker) — the veth+bridge+iptables mechanics this generalizes; Topic 29 (Kubernetes Architecture) — kube-proxy and the kubelet; Topic 35 (Services) — the Service object in full.
- **Study after:** Topic 41 (Ingress) — exposing Services to the outside world; Topic 42 (Network Policies) — restricting this flat network into a zero-trust one; Topic 43 (Health Checks) — readiness probes that keep the EndpointSlice honest; Topic 51 (Observability) — tracing traffic across the mesh.
