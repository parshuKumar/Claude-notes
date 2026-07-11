# 36 — Service-to-Service Communication

> **Phase 6 — Topic 2 of 5**
> Prerequisites: `01-how-the-internet-works.md`, `06-dns-in-depth.md`, `09-tls-ssl-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a giant office building where every team sits in a different room, and the rooms keep moving. The "Payments" team was in room 214 yesterday, but the building manager reshuffled everyone overnight, and now they're in room 809 — and there are three copies of the Payments team spread across three floors.

If you want to send a memo to Payments, you can't write "Room 214" on the envelope anymore. Instead:

1. You ask the **front desk** (internal DNS): "Where does the Payments team sit today?" The front desk always knows, because every team checks in when they move.
2. The front desk gives you a **stable mailbox number** (a Service name) that never changes, even when the team moves rooms.
3. Behind that mailbox, a **mail sorter** (load balancer) quietly spreads your memos across all three copies of the Payments team so no single copy gets buried.
4. Before anyone reads your memo, both you and Payments show each other your **company ID badges** (mutual TLS) — because even inside the building, you don't trust a memo just because it arrived through the internal mail system. Anyone could have snuck in.

Service-to-service communication is exactly this: services find each other by name (not by ever-changing address), get load-balanced automatically, and prove their identity to each other on every single call.

---

## Where This Lives in the Network Stack

Service-to-service communication is not one layer — it's a *pattern* that spans several. But its two headline features land in specific places:

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, gRPC, DNS-based discovery)      │  ← REST/gRPC calls, mesh routing
│  Layer 6 — Presentation  (TLS/mTLS)                             │  ← mTLS: BOTH sides present certs
│  Layer 5 — Session       (connection reuse, keep-alive)         │
│  Layer 4 — Transport     (TCP — Service ClusterIP:port)         │  ← kube-proxy L4 load balancing
│  Layer 3 — Network       (Pod IPs, CNI overlay, NetworkPolicy)  │  ← east-west routing, firewalls
│  Layer 2 — Data Link     (virtual ethernet, veth pairs)         │
│  Layer 1 — Physical      (the actual data-center fabric)        │
└─────────────────────────────────────────────────────────────────┘
```

- **Service discovery** starts at L7 (a DNS query) but resolves to an L4 target (a virtual IP:port).
- **Load balancing** in Kubernetes happens at L3/L4 (kube-proxy/iptables/IPVS/eBPF) OR L7 (a mesh sidecar reading HTTP).
- **mTLS** lives at L6 — the same place as normal TLS (Topic 09), but with a second certificate flowing in the *other* direction.

---

## What Is This?

In a monolith, "calling another module" is a function call — it happens in-process, in nanoseconds, and never fails from a network problem. In a microservices/distributed system, that same call crosses the network: **service A must find service B, connect to it, secure the channel, and survive B being slow, dead, or replaced.**

The hard part is that in modern infrastructure, **service instances are ephemeral**:

- Autoscaling spins Pods up and down based on load — new IPs appear and vanish every minute.
- A crashed container is restarted with a **different IP**.
- A rolling deploy replaces all instances one by one.
- A node dies and its Pods are rescheduled elsewhere.

So the address you called five minutes ago may now belong to a dead process — or worse, a *different* service that got that IP recycled. Service-to-service communication is the collection of mechanisms that solve five problems at once:

| Problem | Mechanism |
|---------|-----------|
| **Discover** — where is B right now? | Internal DNS / service registry |
| **Balance** — spread load across B's instances | kube-proxy / L7 load balancer |
| **Secure** — encrypt + authenticate the call | mTLS |
| **Survive** — B is slow or failing | timeouts, retries, circuit breakers |
| **Observe** — what's happening between services? | mesh telemetry / distributed tracing |

The two dominant answers today are **internal DNS + client resilience code** (the DIY approach) and **a service mesh** (push all of the above out of your app and into infrastructure).

---

## Why Does It Matter for a Backend Developer?

Because in a microservices world, **most of your bugs are now network bugs between your own services.** You cannot ship or operate distributed services without this. Concretely, you need it to:

- Understand why hardcoding `http://10.0.4.12:8080` is a time bomb — that Pod will be gone by next week.
- Know why calling `http://payments/charge` (a name) works in Kubernetes and where that name comes from.
- Explain why one slow downstream service can take down your *entire* fleet (cascading failure) — and how timeouts, retries, and circuit breakers stop it.
- Retry safely — knowing that retrying a non-idempotent `POST /charge` can double-charge a customer (ties to Topic 15).
- Answer the security question every audit asks: "Is traffic *inside* your network encrypted and authenticated?" (Zero trust — you can't answer "it's fine, it's private").
- Debug the classic incident: "Service A can *resolve* service B but can't *reach* it" — a NetworkPolicy or security group is silently dropping east-west traffic (ties to Topic 32).
- Reason about whether a service mesh is worth its latency and resource cost, or whether a few lines of retry code are enough.

This is the connective tissue of every system-design interview and every real distributed system you'll ever operate.

---

## The Packet/Protocol Anatomy

Here's what a single service-to-service call physically involves in Kubernetes. Two things are stable (the Service name and its ClusterIP); everything behind them is ephemeral.

```
     STABLE (never changes)                 EPHEMERAL (changes constantly)
┌──────────────────────────────┐     ┌────────────────────────────────────┐
│  DNS name:                   │     │  Pod IPs behind it (autoscaled):   │
│  payments.shop.svc.cluster.  │────►│    10.1.3.7:8080   (Pod A)         │
│  local                       │     │    10.1.5.2:8080   (Pod B)         │
│                              │     │    10.1.9.4:8080   (Pod C)  ← new  │
│  ClusterIP (virtual IP):     │     │    (10.1.2.1  ← died 30s ago)      │
│  10.96.140.22:80             │     └────────────────────────────────────┘
└──────────────────────────────┘
```

The DNS name resolves to the **ClusterIP** — a *virtual* IP that no network card actually owns. When you send a packet to it, the node's kernel (kube-proxy's iptables/IPVS rules) rewrites the destination to a real, live Pod IP via DNAT (Destination NAT):

```
Packet leaving your app:      [src: 10.1.8.5:51234] → [dst: 10.96.140.22:80]   ← ClusterIP
                                                              │
                              kube-proxy DNAT rule fires (random pick of a healthy Pod)
                                                              ▼
Packet actually on the wire:  [src: 10.1.8.5:51234] → [dst: 10.1.5.2:8080]     ← real Pod
```

When mTLS is on (via a mesh), the payload of that TCP segment is a **TLS record**, and the handshake inside carried **two** certificates instead of one:

```
┌────────────────────────────────────────────────────────────────┐
│ IP PACKET  src=10.1.8.5  dst=10.1.5.2                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ TCP SEGMENT  dst port 15006 (sidecar inbound)            │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ TLS 1.3 RECORD (mTLS)                              │  │  │
│  │  │   client cert: spiffe://cluster/ns/shop/sa/web    │  │  │  ← client identity
│  │  │   server cert: spiffe://cluster/ns/shop/sa/pay    │  │  │  ← server identity
│  │  │  ┌──────────────────────────────────────────────┐  │  │  │
│  │  │  │ HTTP/2 (gRPC) or HTTP/1.1 request            │  │  │  │
│  │  │  │ POST /charge   ...                           │  │  │  │
│  │  │  └──────────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

The **SPIFFE ID** (`spiffe://...`) baked into each cert is the workload's cryptographic identity — that's what makes "who is calling me?" answerable at L6 instead of trusting a source IP.

---

## How It Works — Step by Step

Let's trace the full lifecycle of one call: your `web` service calls `payments` in a Kubernetes cluster with a service mesh (Istio) enabled.

### Step 1 — Service Discovery (find B by name)
```
Your app runs:  POST http://payments/charge   (or payments.shop.svc.cluster.local)

1a. App's resolver checks /etc/resolv.conf → nameserver is CoreDNS (10.96.0.10)
1b. Query sent (UDP :53): "A record for payments.shop.svc.cluster.local?"
1c. CoreDNS answers from its view of the Kubernetes API:
        payments.shop.svc.cluster.local  →  10.96.140.22  (the ClusterIP)
1d. Answer cached locally for the record's TTL (default 30s in CoreDNS)

Result: name → stable ClusterIP 10.96.140.22
```
*This is DNS (Topic 06) — internal, cluster-scoped, and pointing at a virtual IP.*

### Step 2 — Sidecar Interception (mesh only)
```
Without a mesh: the app connects straight to 10.96.140.22:80.

With Istio: an init container installed iptables rules in the Pod that
REDIRECT all outbound TCP to the local Envoy sidecar on port 15001.

    app  ──(tries to dial 10.96.140.22:80)──► iptables REDIRECT ──► Envoy :15001
```
The app *thinks* it's talking to payments directly. It's actually talking to its own sidecar.

### Step 3 — L4/L7 Load Balancing
```
Without a mesh (kube-proxy does it at L4):
    Packet to ClusterIP 10.96.140.22:80 hits iptables/IPVS rules that
    DNAT it to one healthy Pod IP, picked at random (or round-robin in IPVS).

With a mesh (Envoy does it at L7):
    Envoy knows all payments Pod IPs (pushed by the control plane).
    It picks one using a smarter policy (least-request, round-robin, etc.),
    and can retry on a DIFFERENT Pod if the first fails.
```

### Step 4 — mTLS Handshake (mesh only)
```
The client Envoy opens a TLS 1.3 connection to the server Envoy (port 15006).
Unlike normal TLS, BOTH present certificates (see mTLS section below).
Both certs were auto-issued by the control plane and auto-rotate (~24h).
The app sent plaintext HTTP; the sidecars wrapped it in mTLS transparently.
```

### Step 5 — Resilience Policy Applied
```
Before the request leaves, the sidecar enforces the configured policy:
  - Timeout:        give up after e.g. 2s
  - Retries:        retry up to 3× on 503/connect-failure (idempotent only)
  - Circuit breaker: if payments is failing, trip open and fail fast
  - Deadline:       propagate remaining time budget downstream
```

### Step 6 — Server-Side Delivery
```
6a. Server Pod's iptables REDIRECT inbound traffic to its Envoy (:15006)
6b. Server Envoy terminates mTLS, verifies the CLIENT cert identity
6c. Server Envoy checks authorization policy (is 'web' allowed to call 'payments'?)
6d. Server Envoy forwards plaintext HTTP to the app on localhost:8080
6e. App processes /charge, returns 200
6f. Response flows back through both sidecars (re-encrypted on the wire)
```

### The Complete Picture
```
┌───────────────────────────┐                 ┌───────────────────────────┐
│  Pod: web                 │                 │  Pod: payments (1 of 3)   │
│                           │                 │                           │
│  app ──plaintext──► Envoy │                 │  Envoy ──plaintext──► app │
│         (localhost)   │   │                 │   ▲  (localhost)          │
└───────────────────────┼───┘                 └───┼───────────────────────┘
                        │                         │
                        │   ═══ mTLS (TLS 1.3) ═══│   ← encrypted + mutually
                        │   both certs verified   │     authenticated on the wire
                        └─────────────────────────┘
   CoreDNS resolved the name → ClusterIP.  kube-proxy/Envoy picked a live Pod.
   The apps never touched TLS, retries, or discovery — the mesh did it all.
```

---

## Exact Syntax Breakdown

### Kubernetes internal DNS name — the anatomy
```
payments . shop . svc . cluster.local
│          │      │     │
│          │      │     └── cluster domain (configurable; default cluster.local)
│          │      └── "svc" = this is a Service (vs "pod")
│          └── namespace the Service lives in
└── the Service name
```
Shorthand that also works, thanks to `search` domains in `/etc/resolv.conf`:
```
payments                          ← works ONLY from within the same namespace
payments.shop                     ← works from any namespace
payments.shop.svc                 ← fully qualified enough
payments.shop.svc.cluster.local   ← the canonical FQDN
```

### nslookup inside a cluster
```bash
kubectl run tmp --rm -it --image=nicolaka/netshoot -- \
  nslookup payments.shop.svc.cluster.local
#            │
#            └── resolves against CoreDNS, returns the Service ClusterIP
```

### openssl s_client — proving a server REQUIRES a client cert
```bash
openssl s_client -connect payments:15443 \
  -cert client.crt -key client.key   -CAfile ca.crt
#  │                │                  │
#  │                │                  └── CA to verify the SERVER's cert
#  │                └── the CLIENT cert we present (mTLS requirement)
#  └── open a TLS session and show the handshake
```

### Istio PeerAuthentication — enforce mTLS STRICT
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: shop
spec:
  mtls:
    mode: STRICT      # ← reject any plaintext; every call MUST be mTLS
#   modes: STRICT | PERMISSIVE (accept both) | DISABLE
```

### Istio VirtualService — timeout + retries (resilience without app code)
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: payments
spec:
  hosts: [payments]
  http:
    - route:
        - destination: { host: payments }
      timeout: 2s              # ← give up after 2 seconds
      retries:
        attempts: 3            # ← up to 3 tries
        perTryTimeout: 700ms   # ← each try capped at 700ms
        retryOn: 5xx,reset,connect-failure   # ← only safe conditions
```

---

## Example 1 — Basic: Resolve and Call a Service by Name

**Goal:** see that a Service name resolves to a stable ClusterIP, and that the ClusterIP load-balances across Pods.

```bash
# 1. See the Service — note its stable ClusterIP
kubectl get svc payments -n shop
# NAME       TYPE        CLUSTER-IP      PORT(S)   AGE
# payments   ClusterIP   10.96.140.22    80/TCP    9d

# 2. See the ephemeral Pods behind it — different IPs, will change on restart
kubectl get endpoints payments -n shop
# NAME       ENDPOINTS
# payments   10.1.3.7:8080,10.1.5.2:8080,10.1.9.4:8080

# 3. From another Pod, resolve the name
kubectl run tmp --rm -it --image=nicolaka/netshoot -n shop -- \
  nslookup payments
```
Output:
```
Server:    10.96.0.10          ← CoreDNS
Address:   10.96.0.10#53

Name:      payments.shop.svc.cluster.local
Address:   10.96.140.22        ← the STABLE ClusterIP (not a Pod IP!)
```

```bash
# 4. Call it by name repeatedly — kube-proxy spreads calls across Pods
kubectl run tmp --rm -it --image=nicolaka/netshoot -n shop -- \
  sh -c 'for i in 1 2 3 4; do curl -s http://payments/whoami; echo; done'
```
Output (each line served by a *different* Pod, chosen by kube-proxy):
```
served-by: payments-6c9  (10.1.3.7)
served-by: payments-5f2  (10.1.5.2)
served-by: payments-9a4  (10.1.9.4)
served-by: payments-6c9  (10.1.3.7)
```
The lesson: you called **one name**, and traffic fanned out across **three ephemeral Pods** — none of whose IPs you ever wrote down.

---

## Example 2 — Production Scenario: "Payments calls succeed, then randomly 500 for 30 seconds after every deploy"

**The situation:** Your `web` service calls `payments`. Everything is fine — until someone deploys `payments`. For ~30 seconds after each rollout, a fraction of `web`'s calls fail with connection errors, then it heals on its own. Latency dashboards show a spike of `ECONNREFUSED` and `connection reset`. No code changed. What's happening?

**Step 1 — Confirm the failures are connection-level, not app-level**
```bash
kubectl logs deploy/web -n shop | grep -i payment | tail
# error: connect ECONNREFUSED 10.1.2.1:8080
# error: socket hang up  (10.1.2.1)
```
That IP — `10.1.2.1` — is the clue. Check whether it's a *current* payments Pod:
```bash
kubectl get endpoints payments -n shop -o wide
# ENDPOINTS: 10.1.3.7:8080,10.1.5.2:8080,10.1.9.4:8080
```
`10.1.2.1` is **not in the list.** Your app is dialing a Pod that was terminated during the deploy. It's a **stale endpoint** problem.

**Step 2 — Find the two root causes**

*Cause A — the app cached DNS/IPs.* Many HTTP clients (Node's default keep-alive agent, Java's `InetAddress` cache, Go with a custom transport) resolve `payments` once and reuse the connection/IP forever. When the rollout kills that Pod, the client keeps dialing the dead IP.

```
Timeline of a rollout:
  t=0    payments Pods:  10.1.2.1  10.1.3.7          web's pool: →10.1.2.1 (cached)
  t=5    rollout starts, 10.1.2.1 gets SIGTERM
  t=6    10.1.2.1 gone; new Pod 10.1.9.4 comes up
  t=6-35 web keeps dialing 10.1.2.1 → ECONNREFUSED until its pool refreshes
```

*Cause B — no graceful shutdown + no retries.* The old Pod stopped accepting connections *before* it was removed from the Service endpoints, AND `web` had no retry, so every in-flight call to it hard-failed instead of being retried on a healthy Pod.

**Step 3 — Prove the connection is going to a dead Pod**
```bash
# From inside web's Pod, watch which IP it dials
kubectl exec -it deploy/web -n shop -- \
  sh -c 'ss -tnp | grep :8080'
# ESTAB  0  0  10.1.8.5:51234  10.1.2.1:8080   ← still bound to the DEAD Pod IP
```

**Step 4 — The fixes (in order of impact)**

1. **Stop caching the IP / use short-lived connections or re-resolve.** Configure your HTTP client to honor DNS TTL and cap connection lifetime. In Node, don't pin a single agent forever; in Java set `networkaddress.cache.ttl` low.

2. **Add graceful shutdown to `payments`** — on SIGTERM, stop accepting new connections but finish in-flight ones, and add a `preStop` sleep so it leaves the endpoints list *before* it dies:
```yaml
lifecycle:
  preStop:
    exec: { command: ["sh","-c","sleep 5"] }   # keep serving while k8s de-registers us
terminationGracePeriodSeconds: 30
```

3. **Add retries + timeouts on the caller** so a single dead endpoint is retried on a live Pod instead of surfacing as a 500. If you run a mesh, this is one YAML block (the VirtualService above) — zero app code. Without a mesh, add it in the client (idempotent calls only!).

**Step 5 — Bonus: the mTLS variant of this same incident.** Same symptom, different cause. You enabled `PeerAuthentication: STRICT` in the `shop` namespace. Suddenly `web`'s calls to a *different* service, `legacy-billing`, fail with `connection reset`. Reason: `legacy-billing` runs *outside* the mesh (no sidecar), so it can't complete an mTLS handshake — STRICT mode rejected its plaintext.

```bash
istioctl proxy-config secret deploy/web -n shop   # confirm web HAS a workload cert
kubectl get pod -l app=legacy-billing -n shop -o jsonpath='{..containers[*].name}'
# app     ← only ONE container: no "istio-proxy" sidecar => not in the mesh
```
Fix: either add `legacy-billing` to the mesh, or scope mTLS to `PERMISSIVE` for that workload while you migrate it.

**The lesson:** "The network is private, so it's fine" is false twice over. Ephemeral IPs mean *discovery must be live*, and internal traffic still needs *timeouts, retries, and identity* — the same defenses you'd use against the public internet.

---

## Common Mistakes

### Mistake 1: Hardcoding a Pod/instance IP instead of the Service name
```javascript
// WRONG — this IP belongs to a Pod that will be gone after the next deploy
const res = await fetch("http://10.1.2.1:8080/charge");
```
```javascript
// RIGHT — the Service name is stable; discovery + LB happen for you
const res = await fetch("http://payments.shop.svc.cluster.local/charge");
```
**Why it bites:** Pod IPs are recycled. Tomorrow `10.1.2.1` might be a *different service* entirely — now you're POSTing charges to the wrong app.

---

### Mistake 2: No timeout — one slow service freezes your whole fleet
```
web → payments → fraud-check (suddenly takes 30s)

With no timeout:
  Every web request waiting on payments holds a thread/connection for 30s.
  web's connection pool fills up → web stops answering ITS callers →
  the frontend stalls → the WHOLE system is down because ONE service is slow.
  This is a CASCADING FAILURE.
```
**Fix:** Every network call gets a timeout (e.g. 1–2s) shorter than your caller's timeout. Fail fast, shed load, and use a circuit breaker so you stop hammering a service that's already down.

---

### Mistake 3: Retrying non-idempotent calls
```
web → payments  POST /charge  ($50)
  network blips AFTER payments charged but BEFORE the 200 came back
  web retries → payments charges AGAIN → customer charged $100
```
**Fix:** Only auto-retry **idempotent** operations (GET, PUT, DELETE — Topic 15). For non-idempotent ones, use an **idempotency key** so the server dedupes retries:
```
POST /charge
Idempotency-Key: 5f3c-...   ← server ignores a second request with the same key
```

---

### Mistake 4: Trusting the private network — no encryption or authN inside
```
Belief:  "It's a private VPC / cluster network, so we don't need TLS internally."
Reality: One compromised Pod, a misconfigured NetworkPolicy, or a rogue insider
         can now sniff or spoof ANY internal call. A source IP is not an identity —
         it can be spoofed or recycled. This is exactly what ZERO TRUST rejects.
```
**Fix:** mTLS everywhere (every call encrypted + both sides authenticated by cert, not by IP), plus authorization policies. A mesh gives you this without touching app code (ties to Topics 09 and 31).

---

### Mistake 5: Assuming the mesh is free
```
Every call now goes:  app → sidecar → network → sidecar → app
Costs:  +0.5–2ms per hop latency (two extra proxies)
        +50–150MB RAM and CPU PER POD for the sidecar
        + operational complexity (control plane, cert rotation, upgrades)
```
**Fix:** Adopt a mesh when you have enough services that uniform mTLS/retries/observability outweigh the overhead. For 3 services, a retry library is cheaper. (Sidecar-less/ambient meshes and eBPF are emerging to cut this cost.)

---

### Mistake 6: "It resolves, so it must be reachable" — forgetting NetworkPolicies/security groups
```
$ nslookup payments        → 10.96.140.22   ✓ DNS works
$ curl http://payments/    → hangs, times out ✗

DNS resolution and L3/L4 reachability are SEPARATE.
A NetworkPolicy (or AWS security group) can allow the DNS query but
DROP the actual east-west traffic to port 8080.
```
**Fix:** Check the NetworkPolicy / security group governing east-west traffic, not just DNS (ties to Topic 32).

---

### Mistake 7: Stale DNS from an over-long TTL / client-side caching
```
payments record TTL = 300s, but Pods churn every 30s.
Your client cached the answer → keeps sending to instances that are gone.
```
**Fix:** Keep internal DNS TTLs short (CoreDNS defaults to 30s), and make sure your HTTP client actually respects TTL rather than caching the first resolution forever.

---

## Hands-On Proof

```bash
# 1. Show the STABLE Service vs the EPHEMERAL Pods behind it
kubectl get svc payments -n shop
kubectl get endpoints payments -n shop     # these IPs change on every deploy

# 2. Resolve a service name from inside the cluster
kubectl run tmp --rm -it --image=nicolaka/netshoot -n shop -- \
  nslookup payments.shop.svc.cluster.local

# 3. Watch CoreDNS actually answer the query
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20

# 4. See kube-proxy's load-balancing rules for the ClusterIP (iptables mode)
#    Run on a node (or a privileged Pod):
iptables -t nat -L KUBE-SERVICES -n | grep 10.96.140.22

# 5. Prove a server REQUIRES a client cert (mTLS) — expect a handshake failure
#    without a cert, success WITH one:
openssl s_client -connect payments:15443 -CAfile ca.crt </dev/null
#   → "peer did not return a certificate" / handshake fails
openssl s_client -connect payments:15443 -cert client.crt -key client.key -CAfile ca.crt </dev/null
#   → handshake completes, prints BOTH the server cert AND "Acceptable client CA names"

# 6. If running Istio: confirm each app Pod has a sidecar and a workload cert
kubectl get pod -n shop -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[*].name}{"\n"}{end}'
#   look for "istio-proxy" alongside your app container
istioctl proxy-config secret deploy/web -n shop      # shows the auto-issued SPIFFE cert

# 7. Prove east-west traffic can be blocked even when DNS works
kubectl get networkpolicy -n shop
```

---

## Practice Exercises

### Exercise 1 — Easy: Stable name vs ephemeral IPs
```bash
# In any Kubernetes cluster (minikube/kind/kind works):
kubectl create deployment hello --image=nginxdemos/hello --replicas=3
kubectl expose deployment hello --port=80

# Now answer from the output of these:
kubectl get svc hello              # 1. What is the ClusterIP? Does it look like a Pod IP?
kubectl get endpoints hello        # 2. How many IPs are behind the Service?
kubectl delete pod -l app=hello --field-selector=... # (delete one Pod)
kubectl get endpoints hello        # 3. Did the endpoint list change? Did the ClusterIP change?

# Questions:
#  - Why does the ClusterIP stay constant while the endpoints change?
#  - If you had hardcoded one of the endpoint IPs, what would happen now?
```

### Exercise 2 — Medium: Prove kube-proxy load-balances
```bash
# Using the "hello" service above (nginxdemos/hello shows the serving Pod's hostname):
kubectl run curler --rm -it --image=nicolaka/netshoot -- \
  sh -c 'for i in $(seq 1 12); do curl -s http://hello/ | grep -i "Server name"; done'

# Questions:
#  1. How many distinct Pod names appear across the 12 requests?
#  2. Is the distribution roughly even? (iptables mode = random; IPVS = round-robin)
#  3. What did you connect to — a Pod IP or the ClusterIP? Where did the fan-out happen?
```

### Exercise 3 — Hard (Production Simulation): mTLS + resilience
```bash
# Install Istio (istioctl install --set profile=demo -y) and label a namespace:
kubectl create ns shop
kubectl label ns shop istio-injection=enabled

# Deploy two services (web -> payments), then:

# A) Turn on STRICT mTLS for the namespace:
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default, namespace: shop }
spec: { mtls: { mode: STRICT } }
EOF

# B) Deploy a THIRD service WITHOUT the sidecar (istio-injection disabled on its Pod)
#    and have web call it.

# Questions:
#  1. Capture traffic between the two MESHED services (istioctl or tcpdump on port 15006).
#     Is the payload readable plaintext or encrypted? Why?
#  2. web's call to the NON-meshed service now fails. What's the exact error, and why
#     does STRICT mode reject it?
#  3. Add a VirtualService with timeout:1s + 3 retries on payments. Kill 1 of its 3 Pods
#     mid-request-loop. Do callers see errors, or do retries mask the dead Pod?
#  4. Switch mTLS mode to PERMISSIVE. Does the non-meshed call work now? What did you
#     trade away by doing so?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **In Kubernetes, a Service name resolves to a ClusterIP, but no network card owns that IP. So how does a packet sent to it actually reach a running Pod?** (What component, what mechanism?)

2. **What is the difference between client-side and server-side service discovery? Which one is Kubernetes' Service + kube-proxy model, and which one is a client querying Consul/Eureka directly?**

3. **Explain the exact difference between normal TLS and mTLS in one sentence. In mTLS, who presents a certificate?**

4. **Your service A can `nslookup` service B successfully but every `curl` to B times out. DNS is clearly working. Name two things (at different layers) that could be silently dropping the traffic.**

5. **Why is retrying a `POST /charge` dangerous, but retrying a `GET /balance` safe? What header makes the POST safe to retry?**

6. **Describe the sidecar pattern in one sentence. What moves OUT of your application code and INTO the mesh, and what are the two costs you pay for that?**

7. **What does "zero trust" mean for internal service-to-service traffic, and why is "it's inside our private VPC" not a sufficient security argument?**

---

## Quick Reference Card

| Concept | What it is | Key detail |
|---------|-----------|-----------|
| Service discovery | Finding B's current location by name | Client-side (query registry) vs server-side (LB in front) |
| Internal DNS | Cluster-scoped name → ClusterIP | `svc.ns.svc.cluster.local`, served by CoreDNS, TTL ~30s |
| ClusterIP | Stable virtual IP for a Service | No NIC owns it; kube-proxy DNATs it to a Pod IP |
| Pod IP | An instance's real, ephemeral IP | Changes on restart/reschedule — never hardcode it |
| Headless Service | `clusterIP: None` | DNS returns Pod IPs directly (for stateful/gRPC client-LB) |
| kube-proxy | L4 load balancer | iptables (random) / IPVS (round-robin) / eBPF (Cilium) |
| East-west | Service ↔ service traffic | The subject of this topic |
| North-south | Client ↔ cluster (ingress) traffic | Topic 34 (API gateways) |
| Service mesh | Sidecar proxies + control plane | Envoy (data plane) + Istio/Linkerd (control plane) |
| Data plane | The Envoy sidecars carrying traffic | One per Pod, intercepts all in/out |
| Control plane | Configures + issues certs to sidecars | Istiod; pushes routes, rotates mTLS certs |
| mTLS | TLS where BOTH sides present certs | Encrypt + mutually authenticate; identity = SPIFFE ID |
| Circuit breaker | Fail fast when a dependency is down | Stops cascading failures |
| Idempotency key | Makes a retry-unsafe call safe to retry | Server dedupes repeats of the same key |

**Sync vs async — when to choose which:**
```
Synchronous (REST/gRPC over HTTP — Topics 15/37):
  Use when: caller needs the answer NOW (read a value, confirm a write).
  Cost:     temporal coupling — B down means A's request fails.

Asynchronous (message queue/event — Topic 38):
  Use when: fire-and-forget, decoupling, buffering spikes, fan-out.
  Cost:     eventual consistency; harder to reason about ordering/failure.
```

**Resilience defaults (put on EVERY network call):**
```
timeout:        1–2s (shorter than your caller's timeout)
retries:        2–3, exponential backoff + jitter, IDEMPOTENT calls only
retryOn:        5xx, connection-reset, connect-failure (NOT on 4xx)
circuit breaker: trip open after N consecutive failures
deadline:       propagate remaining time budget downstream
```

**Key commands:**
```bash
kubectl get svc,endpoints NAME -n NS          # stable IP vs live Pod IPs
nslookup NAME.NS.svc.cluster.local            # internal DNS resolution
kubectl get networkpolicy -n NS               # what gates east-west traffic
istioctl proxy-config secret POD -n NS        # workload mTLS cert (SPIFFE)
openssl s_client -connect H:P -cert c -key k  # prove client-cert requirement
```

---

## When Would I Use This at Work?

### Scenario 1: "Intermittent 500s right after every deploy"
You now recognize the fingerprint: a caller dialing a stale Pod IP because it cached the resolution or lacked graceful shutdown/retries. You'd check `kubectl get endpoints`, confirm the failing IP isn't in the list, then fix it with a `preStop` hook + retries — not with more app-level error handling.

### Scenario 2: "Security audit asks: is internal traffic encrypted and authenticated?"
"It's a private VPC" is not an acceptable answer anymore. You'd reach for mTLS — ideally via a mesh (`PeerAuthentication: STRICT`) so every service call is encrypted and both sides prove identity by certificate, with no app changes and automatic cert rotation. You'd also add authorization policies so identity is *enforced*, not just verified.

### Scenario 3: "One flaky downstream is taking down the whole product"
You'd diagnose it as a cascading failure from missing timeouts/circuit breakers, not a capacity problem. Add tight timeouts, a circuit breaker, and backpressure so the flaky service fails fast and isolated instead of holding every upstream thread hostage.

### Scenario 4: "Should we adopt a service mesh?"
You can now weigh it honestly: uniform mTLS + retries + observability across many services vs. the per-Pod latency/memory cost and operational complexity. For a handful of services, a retry library and manual TLS may be cheaper; past a few dozen, the mesh's uniform policy usually wins.

### Scenario 5: "Service A can't reach service B in the new namespace"
You'd separate the layers: does DNS resolve (`nslookup`)? If yes but traffic hangs, it's a NetworkPolicy/security group blocking east-west traffic, or STRICT mTLS rejecting a non-meshed peer — not a discovery problem.

---

## Connected Topics

**Study before this:**
- `06-dns-in-depth.md` — internal DNS is the discovery backbone; CoreDNS is just DNS scoped to a cluster
- `09-tls-ssl-in-depth.md` — mTLS is TLS with a second certificate; you need the handshake cold first
- `01-how-the-internet-works.md` — the packet/connection lifecycle every internal call still follows

**Study alongside / next:**
- `15-rest-over-http.md` — method semantics and idempotency, which decide what's safe to retry
- `37-grpc-and-protocol-buffers.md` — the other synchronous style; gRPC over HTTP/2 is mesh-friendly and common east-west
- `38-message-queues-at-network-level.md` — the asynchronous alternative to synchronous service calls
- `32-firewalls-and-security-groups.md` — NetworkPolicies/security groups gate east-west traffic even when DNS works
- `31-vpns-and-private-networks.md` — why "private network" is necessary but not sufficient (zero trust)
- `34-api-gateways.md` — the north-south counterpart to this topic's east-west focus
- `35-network-in-system-design.md` — the latency/bandwidth math behind choosing sync vs async

**This topic is the glue of distributed systems.** Discovery, load balancing, resilience, and mTLS are the four things that turn a pile of independent services into a system that stays up.

---

*Doc saved: `/docs/networking/36-service-to-service-communication.md`*
