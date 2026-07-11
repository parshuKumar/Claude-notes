# 42 — Network Policies
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Imagine an office building where, by default, **every door is unlocked** and anyone can walk into any room. The accountant's office, the server room, the CEO's office — all wide open. If a stranger sneaks into the lobby, they can wander into *any* room and take *anything*. That's a normal Kubernetes cluster on day one: **every pod can talk to every other pod.** Your `orders-api` can reach your `payments` database, and so can a random `guestbook` app someone deployed for a demo.

Now imagine you hire a security team that installs **keycard locks** on every door and writes a rulebook: "Only people with an *orders-api* badge may enter the *Postgres* room. Everyone else is refused at the door." Suddenly the accountant can't stroll into the server room, and neither can the stranger in the lobby.

A **Network Policy** is that rulebook. Each rule says which pods (badges) are allowed to talk to which other pods, on which ports. And here's the key twist: the moment you put a lock on a door, that door defaults to **locked** — only the explicitly-allowed badges get in. Everything not listed is denied.

But there's a catch the whole topic hinges on: **the keycard locks don't come with the building.** Kubernetes writes the rulebook, but a separate security company (the **CNI plugin**) must actually install and enforce the locks. If you write rules but hired no security company, the rulebook sits in a drawer and every door stays wide open.

---

## The Linux kernel feature underneath

A NetworkPolicy has **no kernel primitive of its own** — like Ingress, it's a pure Kubernetes abstraction. But the *enforcement* is extremely real and lives deep in the Linux kernel's **packet filtering** machinery.

Here is the physical truth. When you apply a NetworkPolicy, the CNI plugin (Calico, Cilium, etc. — Topic 40) translates your YAML into **actual packet-filtering rules on every node**:

1. **iptables rules** (classic CNIs like Calico's iptables mode). Every packet leaving or entering a pod passes through the kernel's **netfilter** hooks. The CNI programs rules like "packets destined for the Postgres pod's IP on port 5432 are ACCEPTed only if the source IP belongs to an orders-api pod; otherwise DROP." You can literally see them:

```
$ iptables -L cali-pi-postgres -n
Chain cali-pi-postgres (1 references)
target     prot  source            destination
ACCEPT     tcp   10.244.2.15/32    anywhere     tcp dpt:5432   ← orders-api pod IP
DROP       all   anywhere          anywhere                    ← everything else
```

2. **eBPF programs** (modern CNIs like Cilium). Instead of iptables, Cilium attaches tiny **eBPF programs** to the network hooks in the kernel. Each pod gets a numeric **security identity**, and an eBPF map decides `allow` or `drop` per packet — faster than walking long iptables chains.

3. **The decision is per-packet, in kernel space.** No userspace process inspects each packet. When a `guestbook` pod sends a TCP SYN to Postgres:5432, the kernel's netfilter/eBPF hook checks the rules and **DROPs the SYN silently**. The guestbook's `connect()` syscall just hangs and eventually times out — because the packet died in the kernel before it ever left the node.

So "a NetworkPolicy blocks traffic" physically means: **the CNI has programmed iptables or eBPF rules in every node's kernel that DROP packets not matching your allow-list, at the netfilter/XDP layer, before they reach the target pod's network namespace.**

The crucial consequence: if your CNI plugin **doesn't implement NetworkPolicy** (e.g. Flannel by default), no iptables/eBPF rules get written, and your policy is **silently ignored** — every packet flows freely. The abstraction exists; the enforcement doesn't.

---

## What is this?

A **NetworkPolicy** is a namespaced Kubernetes object that specifies which network connections are **allowed** to and from a group of pods, selected by labels. It controls Layer 3/4 traffic (IP + port), for both **ingress** (incoming) and **egress** (outgoing).

Network policies are **allow-lists that default to deny**: as soon as *any* policy selects a pod, that pod rejects all traffic **except** what the policies explicitly permit. They are enforced by the CNI plugin, not by Kubernetes core — so they only work on a CNI that supports them.

---

## Why does it matter for a backend developer?

Your `orders` namespace runs `orders-api`, `postgres`, and `redis`. On a fresh cluster, the network is **flat and totally open**: every pod can reach every other pod's IP on every port. That means:

- The `frontend` pod can connect **directly to Postgres:5432**, bypassing your API entirely.
- If an attacker compromises *any* pod — even a throwaway `debug` container — they can immediately reach your database, your Redis, your internal admin service. **One breached pod = the whole cluster breached.** This is called **lateral movement**, and it's how most real Kubernetes breaches escalate.
- There is no network-level enforcement of your architecture. You *intended* "only orders-api talks to Postgres," but nothing enforces it; it's honor-system.

**Without Network Policies** you will:
- Ship a cluster where a single compromised pod owns everything.
- Fail security audits and compliance (PCI-DSS, SOC2 all require network segmentation).
- Have no way to prove "the payments database is only reachable by payments-api."
- Debug mysterious "why can this random pod read my database" incidents.

**With Network Policies** you build **zero-trust networking**: nothing talks to anything unless explicitly allowed. A breached `frontend` pod finds every other door locked. This is the difference between a flat, brittle network and a segmented, defensible one.

---

## The physical reality

When a NetworkPolicy is active in namespace `orders`, here's what exists:

**1. The policy object in etcd:**

```
$ kubectl get networkpolicy -n orders
NAME                    POD-SELECTOR   AGE
postgres-allow-orders   app=postgres   2d
default-deny-all        <none>         2d
```

**2. Real iptables chains on every node** (Calico iptables mode). The CNI reconciled your policy into kernel rules:

```
$ sudo iptables -S | grep cali | grep postgres
-A cali-tw-postgres -m comment --comment "pi:orders-api-only" \
   -s 10.244.2.15/32 -p tcp --dport 5432 -j ACCEPT     ← allow orders-api
-A cali-tw-postgres -m comment --comment "pi:default-drop" -j DROP   ← deny rest
```

**3. Or eBPF maps** (Cilium). Instead of iptables, identity-based maps:

```
$ cilium bpf policy get <postgres-pod-endpoint-id>
DIRECTION   IDENTITY   PORT/PROTO   ACTION
Ingress     12345      5432/TCP     ALLOW      ← identity 12345 = orders-api
Ingress     *          *            DENY
```

**4. Each pod's network namespace** (Topic 01, 40) has these rules applied at its **veth pair** — the virtual cable connecting the pod to the node. Packets are filtered right where the pod's cable meets the node's kernel.

**5. Proof of a dropped connection** — a blocked pod's `connect()` just hangs:

```
$ kubectl exec frontend -n orders -- nc -zv -w3 postgres 5432
nc: connect to postgres port 5432 (tcp) timed out   ← SYN was DROPped in-kernel
```

So a NetworkPolicy on disk/in kernel is: **an object in etcd + iptables chains or eBPF maps on every node + filtering at each pod's veth.** The YAML is desired state; the kernel rules are the enforced reality (Topic 30's reconciliation loop, applied to firewalls).

---

## How it works — step by step

Full trace of locking Postgres down to only `orders-api`, and what happens to a blocked packet:

1. **You `kubectl apply` a NetworkPolicy** selecting `app=postgres`, allowing ingress from `app=orders-api` on 5432. The API server validates and writes it to etcd. Nothing is enforced yet.

2. **The CNI's agent watches NetworkPolicy objects.** Calico's `calico-node` / Cilium's `cilium-agent` runs as a DaemonSet — one pod per node (Topic 47). Each holds a watch (Topic 30) on NetworkPolicy, Pod, and Namespace objects.

3. **The agent resolves label selectors to pod IPs.** It asks: which pods have `app=orders-api`? It gets their current IPs (say `10.244.2.15`). Which pods have `app=postgres`? It finds the target(s). This mapping is **dynamic** — if orders-api scales up, new pod IPs are added automatically.

4. **The agent programs the kernel on every node.** It writes iptables chains (or eBPF maps) that say: for packets arriving at a Postgres pod's veth on port 5432 — ACCEPT if source ∈ {orders-api IPs}, else DROP. Because a policy now *selects* Postgres, all **other** ingress to Postgres is implicitly denied.

5. **A legitimate request flows.** `orders-api` (10.244.2.15) opens a TCP connection to Postgres:5432. The SYN packet hits Postgres's veth, netfilter checks the chain, source IP matches the allow rule → **ACCEPT**. Connection established, queries flow.

6. **A malicious/accidental request is dropped.** The `frontend` pod (10.244.3.9) tries to connect to Postgres:5432. The SYN hits the same chain, source IP is **not** in the allow-list → **DROP**. The packet is discarded in the kernel; no RST is sent, so `frontend`'s `connect()` **hangs and times out** rather than getting "connection refused."

7. **Pods reschedule, rules follow.** If Postgres crashes and reschedules to another node with a new IP, the CNI agent re-resolves labels and reprograms the kernel rules on the new node. The policy is expressed in **labels**, not IPs, so it survives pod churn automatically.

The key insight: **you never wrote a single firewall rule.** You described intent in labels; the CNI continuously reconciled that intent into live kernel packet filters across every node.

---

## Ingress vs. egress, and the default-deny cornerstone

Two directions, and one make-or-break rule about how selection works:

```
        ┌────────────┐                          ┌────────────┐
        │ orders-api │ ──── egress (out) ───▶    │  postgres  │
        │    pod     │                           │    pod     │
        │            │  ◀─── ingress (in) ────    │            │
        └────────────┘                          └────────────┘
        "what may this pod          "what may reach this pod
         SEND TO?"                   FROM where?"
```

- **Ingress rules** = who is allowed to **connect TO** the selected pods.
- **Egress rules** = where the selected pods are allowed to **connect OUT to**.

**The default-deny cornerstone — memorize this:**

> A pod that is **not selected by any** NetworkPolicy is **wide open** (all traffic allowed).
> The moment a pod **is selected by at least one** policy for a direction, **all traffic in that direction is denied except what the policies allow.**

So policies are purely **additive allow-lists**, and selecting a pod flips its default for that direction from "allow-all" to "deny-all-except-listed." This is why the standard first move in a hardened namespace is a **default-deny-all** policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: orders
spec:
  podSelector: {}              # {} = select EVERY pod in the namespace
  policyTypes:
    - Ingress
    - Egress                   # deny both directions by default
  # no ingress/egress rules listed → nothing is allowed
```

This one object turns the whole `orders` namespace into a locked building. Then you add narrow "allow" policies to open exactly the doors you need. That is **zero-trust**: deny everything, then permit the minimum.

---

## Exact syntax breakdown

A policy allowing `orders-api` → Postgres, with everything annotated:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-allow-orders-api
  namespace: orders
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: orders-api
      ports:
        - protocol: TCP
          port: 5432
```

Annotated:

```
apiVersion: networking.k8s.io/v1
│           │
│           └─ the stable API group for NetworkPolicy since K8s 1.7.
│              Same group as Ingress, but a totally different kind.
│
└─ kind: NetworkPolicy → a firewall rule set, enforced by the CNI

metadata:
  name: postgres-allow-orders-api   ← name of this policy object
  namespace: orders                 ← NetworkPolicy is NAMESPACED. It can only
  │                                    select pods in ITS OWN namespace.
  │
spec:
  podSelector:                      ← WHICH pods this policy applies to (the target)
    matchLabels:
      app: postgres                 ← selects pods labeled app=postgres.
      │                                {} here would mean "every pod in namespace".
      │                                Selecting Postgres flips its INGRESS default
      │                                from allow-all → deny-all-except-below.
      │
  policyTypes:                      ← which DIRECTIONS this policy governs
    - Ingress                       ← we only constrain incoming here.
    │                                  If you list Egress too but give no egress
    │                                  rules, ALL egress is denied.
    │
  ingress:                          ← the allow-list of INCOMING connections
    - from:                         ← WHO may connect (a list of sources; OR-ed)
        - podSelector:              ← a source = pods matching these labels...
            matchLabels:
              app: orders-api       ← ...i.e. only orders-api pods (in THIS namespace)
      ports:                        ← ...AND only on these ports
        - protocol: TCP             ← TCP (or UDP, or SCTP)
          port: 5432                ← Postgres's port. A connection to any OTHER
                                       port on Postgres is still denied.
```

**Critical selector nuance — the `podSelector` vs `namespaceSelector` trap:**

```
from:
  - podSelector:                    ← matches pods BY LABEL, but ONLY within the
      matchLabels:                     policy's OWN namespace (orders)
        app: orders-api

from:
  - namespaceSelector:              ← matches ALL pods in namespaces whose LABELS match
      matchLabels:
        team: payments                (e.g. every pod in the payments namespace)

from:
  - namespaceSelector:              ← AND-combined: pods that are BOTH in a matching
      matchLabels:                    namespace AND have the matching pod label.
        team: payments                (note: two items under ONE list entry = AND)
    podSelector:
      matchLabels:
        app: gateway
```

The single most common mistake: writing `namespaceSelector` and `podSelector` as **two separate list items** (OR — "any pod in that namespace, OR any pod with that label anywhere") when you meant them as **one item with two keys** (AND — "a pod with that label in that namespace"). The YAML indentation decides your security posture.

---

## Example 1 — basic

Lock down `orders-api` so only the `frontend` may reach it on port 3000. Every line commented.

```yaml
# np-basic.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders-api-allow-frontend    # policy name
  namespace: orders                  # applies within the orders namespace
spec:
  podSelector:                       # TARGET: which pods this protects
    matchLabels:
      app: orders-api                # → the orders-api pods
  policyTypes:
    - Ingress                        # we govern INCOMING traffic only
  ingress:                           # allow-list of who may connect in
    - from:
        - podSelector:               # source = pods labeled...
            matchLabels:
              app: frontend          # ...app=frontend (in this namespace)
      ports:
        - protocol: TCP
          port: 3000                 # only on TCP 3000 (orders-api's port)
```

Apply and prove it:

```bash
kubectl apply -f np-basic.yaml
#      │      │
#      │      └─ read the policy from the file
#      └─ write it to etcd; the CNI reconciles kernel rules

kubectl get networkpolicy -n orders
# NAME                        POD-SELECTOR     AGE
# orders-api-allow-frontend   app=orders-api   5s

# From the ALLOWED pod (frontend) — succeeds:
kubectl exec -n orders deploy/frontend -- nc -zv -w3 orders-api 3000
# → orders-api (10.244.2.20:3000) open        ← ACCEPT

# From a NON-allowed pod (say a debug pod) — times out:
kubectl run debug --rm -it --image=busybox -n orders -- \
  nc -zv -w3 orders-api 3000
# → nc: connect ... timed out                 ← packet DROPped in kernel
```

The timeout (not "connection refused") is your fingerprint that a NetworkPolicy dropped the packet: the kernel silently discarded the SYN, so the sender waits for a reply that never comes.

---

## Example 2 — production scenario

**The situation.** Your `orders` namespace runs `frontend`, `orders-api`, `postgres`, and `redis`. Security review flags: "Anything can reach the database. A compromised frontend could dump every customer's data." You must implement **zero-trust**: nothing talks to anything unless explicitly required by the architecture.

The intended data flow is:

```
  internet ──▶ ingress-nginx ──▶ frontend ──▶ orders-api ──▶ postgres
                                                    │
                                                    └──────▶ redis
```

Nothing else should be possible. Here's the full policy set.

**Step 1 — default-deny the whole namespace:**

```yaml
# 1-default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: orders
spec:
  podSelector: {}                    # every pod in orders
  policyTypes: [Ingress, Egress]     # deny in AND out by default
```

**Step 2 — but every pod still needs DNS** (or nothing can resolve `postgres` to an IP). This is the #1 gotcha of default-deny egress — allow DNS to kube-system:

```yaml
# 2-allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: orders
spec:
  podSelector: {}                    # all pods may...
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system   # ...reach CoreDNS
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**Step 3 — Postgres accepts ingress ONLY from orders-api:**

```yaml
# 3-postgres-lockdown.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-allow-orders-api
  namespace: orders
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: orders-api        # ONLY orders-api → Postgres:5432
      ports:
        - protocol: TCP
          port: 5432
```

**Step 4 — orders-api may egress to Postgres and Redis, and accept from frontend:**

```yaml
# 4-orders-api-rules.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders-api-rules
  namespace: orders
spec:
  podSelector:
    matchLabels:
      app: orders-api
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend          # frontend → orders-api:3000
      ports:
        - protocol: TCP
          port: 3000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres          # orders-api → postgres:5432
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis             # orders-api → redis:6379
      ports:
        - protocol: TCP
          port: 6379
```

**The result — a proven, segmented network:**

- `frontend` compromised? It can reach `orders-api:3000` and DNS. It **cannot** touch Postgres or Redis — the SYNs are dropped in-kernel. The blast radius is one hop, not the whole database.
- A rogue `debug` pod someone `kubectl run`s? It's selected by `default-deny-all`, allowed nothing but DNS. It can reach **nothing**.
- Postgres is reachable **only** by `orders-api`, on **only** 5432. This is now auditable: you can point at the YAML and prove the segmentation to a compliance reviewer.

**The payoff:** you turned a flat "anyone reaches anything" network into a **zero-trust** one where each pod can talk to exactly the pods its job requires — and a single breach can no longer walk the whole cluster.

---

## Common mistakes

**Mistake 1 — Policy has zero effect because the CNI doesn't support it.**

```
$ kubectl apply -f postgres-lockdown.yaml
networkpolicy.networking.k8s.io/postgres-allow-orders-api created
$ kubectl exec frontend -n orders -- nc -zv -w3 postgres 5432
postgres (10.244.1.8:5432) open      ← STILL OPEN despite the policy!
```

Root cause: your CNI (e.g. plain **Flannel**) does not implement NetworkPolicy, so **no iptables/eBPF rules are ever written**. The object sits in etcd doing nothing. Right way: run a policy-capable CNI — **Calico, Cilium, Weave, Antrea**. Check with `kubectl get pods -n kube-system | grep -E 'calico|cilium'`. This is the single most surprising Network Policy failure.

**Mistake 2 — default-deny egress breaks DNS, and everything "randomly" fails.**

```
$ kubectl exec orders-api -n orders -- node -e "require('pg').connect('postgres')"
Error: getaddrinfo EAI_AGAIN postgres      ← can't even RESOLVE the name
```

Root cause: your default-deny-egress policy blocked **UDP/TCP 53 to CoreDNS**, so pods can't resolve `postgres` to an IP — the connection fails before it starts. Right way: always pair default-deny-egress with an explicit **allow-DNS-to-kube-system** policy (Example 2, step 2). Everyone forgets this once.

**Mistake 3 — `namespaceSelector` OR `podSelector` when you meant AND.**

```yaml
from:
  - namespaceSelector:
      matchLabels: {team: payments}
  - podSelector:                       # SEPARATE list item = OR, not AND!
      matchLabels: {app: gateway}
```

Root cause: two `-` list items under `from` are **OR-ed**. This accidentally allows **every pod in the payments namespace** AND **every pod named gateway in the local namespace** — far more than intended. Right way: to mean "gateway pods *in* payments," put both selectors under **one** list item (no second `-`).

**Mistake 4 — Forgetting policies are namespaced; cross-namespace traffic slips or breaks.**

```
# orders-api in `orders` can't reach a shared `auth` service in `platform` namespace.
```

Root cause: a `podSelector` only matches pods in the **policy's own namespace**. To allow traffic from/to another namespace you MUST use `namespaceSelector`. Conversely, a policy in `orders` cannot protect pods in `platform` at all. Right way: use `namespaceSelector` (with the built-in `kubernetes.io/metadata.name` label) for cross-namespace rules, and remember each namespace needs its own default-deny.

**Mistake 5 — Expecting "connection refused" instead of a timeout, and misdiagnosing.**

```
$ kubectl exec frontend -n orders -- curl -m5 postgres:5432
curl: (28) Connection timed out after 5001 ms      ← NOT "connection refused"
```

Root cause: a NetworkPolicy **DROPs** packets (silent discard), it doesn't **REJECT** them (which would send a TCP RST = "connection refused"). So blocked traffic **hangs and times out**. Devs waste hours suspecting a slow database or DNS. Right way: a **timeout** to a service you know is up is the signature of a NetworkPolicy drop — check `kubectl get networkpolicy -n <ns>` first.

---

## Hands-on proof

Run on a cluster with a policy-capable CNI (Calico/Cilium). `minikube start --cni=calico` works.

```bash
# 0. CONFIRM your CNI enforces policy (else nothing below works).
kubectl get pods -n kube-system | grep -E 'calico|cilium'

# 1. Set up the orders namespace with three labeled pods.
kubectl create namespace orders
kubectl run postgres   --image=postgres:16-alpine -n orders \
  --labels=app=postgres --env=POSTGRES_PASSWORD=pw
kubectl run orders-api --image=busybox -n orders --labels=app=orders-api -- sleep 3600
kubectl run frontend   --image=busybox -n orders --labels=app=frontend   -- sleep 3600

# 2. BEFORE any policy: prove the network is WIDE OPEN.
kubectl exec frontend -n orders -- nc -zv -w3 postgres 5432
#   → postgres open   ← frontend can reach the DB (bad!)

# 3. Apply a lockdown: only orders-api may reach Postgres.
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: postgres-allow-orders-api, namespace: orders}
spec:
  podSelector: {matchLabels: {app: postgres}}
  policyTypes: [Ingress]
  ingress:
    - from: [{podSelector: {matchLabels: {app: orders-api}}}]
      ports: [{protocol: TCP, port: 5432}]
EOF

# 4. PROOF it works: orders-api ALLOWED, frontend DENIED.
kubectl exec orders-api -n orders -- nc -zv -w3 postgres 5432
#   → postgres open              ← ACCEPT (allowed source)
kubectl exec frontend   -n orders -- nc -zv -w3 postgres 5432
#   → nc: timed out              ← DROP (packet died in kernel)

# 5. PROOF in the kernel (Calico) — see the real iptables rule on the node:
#    (run on the node, or via a privileged pod)
#    sudo iptables -S | grep -i postgres

# 6. Clean up.
kubectl delete namespace orders
```

What you verified: the network is open by default (step 2), one small YAML **flipped Postgres to deny-by-default** (step 3), the allowed pod still connects while the denied pod **times out** (step 4), and the rule physically exists as an in-kernel packet filter (step 5).

---

## Practice exercises

### Exercise 1 — easy
In a namespace with a policy-capable CNI, deploy `redis` (label `app=redis`) and two busybox pods labeled `app=orders-api` and `app=frontend`. Write a NetworkPolicy so **only** `orders-api` can reach `redis:6379`. Prove `orders-api` connects and `frontend` times out with `nc -zv -w3 redis 6379`. Which single field decided who was allowed?

### Exercise 2 — medium
Add a `default-deny-all` policy (`podSelector: {}`, both directions) to the namespace, then try to have `orders-api` connect to `redis` by **name** (`nc -zv redis 6379`). It will fail at DNS resolution. Diagnose why, then write the minimal extra policy to fix it *without* opening any other traffic. Explain in one sentence what default-deny egress broke.

### Exercise 3 — hard (production simulation)
Implement the full Example 2 zero-trust set (default-deny + allow-DNS + Postgres-from-orders-api + orders-api ingress/egress + frontend→orders-api) in a real namespace with `frontend`, `orders-api`, `postgres`, `redis`. Then simulate a breach: `kubectl exec` into `frontend` and attempt to reach `postgres:5432` and `redis:6379` directly — both must time out — while `frontend`→`orders-api:3000` succeeds and `orders-api`→both datastores succeeds. Finally, `kubectl run` a rogue `attacker` pod with **no matching labels** and show it can reach **nothing** but DNS. Write down: for the attacker pod, which policy denied it, and why it could still resolve DNS.

---

## Mental model checkpoint

Answer from memory:

1. What is a pod's default network behavior before any policy selects it, and what changes the instant one policy does?
2. Why does a NetworkPolicy do **nothing** on a cluster running plain Flannel?
3. What is the difference between `podSelector` and `namespaceSelector` in a `from` block, and how does list indentation flip AND vs OR?
4. Why does blocked traffic **time out** instead of returning "connection refused"?
5. What must you *always* allow alongside a default-deny-egress policy, and why?
6. At the kernel level, what does the CNI actually write to enforce your policy?
7. Are NetworkPolicies namespaced or cluster-wide, and what does that imply for cross-namespace traffic?

---

## Quick reference card

| Field / concept | What it does | Key detail |
|---|---|---|
| `kind: NetworkPolicy` | L3/L4 allow-list firewall for pods | Enforced by the CNI, not K8s core |
| `podSelector: {}` | Select every pod in the namespace | Basis of default-deny |
| `podSelector: {matchLabels}` | Select the target/source pods by label | Only within the policy's namespace |
| `policyTypes` | Which directions govern (Ingress/Egress) | Listing a type with no rules = deny all of it |
| `ingress.from` | Who may connect TO the selected pods | List items are OR-ed |
| `egress.to` | Where selected pods may connect OUT | Forgetting DNS breaks name resolution |
| `namespaceSelector` | Match pods by their namespace's labels | Required for cross-namespace rules |
| default-deny policy | Empty selector + no rules | Turns namespace zero-trust |
| Behavior when blocked | Packet DROPped in kernel | Times out, not "connection refused" |
| CNI requirement | Calico / Cilium / Weave / Antrea | Flannel (default) ignores policies |

---

## When would I use this at work?

1. **Passing a security audit.** Compliance (PCI-DSS, SOC2) requires that your card-processing database is network-isolated. You write a policy locking Postgres to only `payments-api` and hand the auditor the YAML as proof of segmentation.

2. **Containing a breach.** After a dependency CVE lets an attacker run code in your `frontend` pod, your default-deny + narrow-allow policies mean they can reach only `orders-api:3000` — not the database, not Redis, not other namespaces. The incident is a contained hop, not a full data exfiltration.

3. **Enforcing architecture, not just documenting it.** Your design says "only orders-api talks to the DB," but nothing stopped a teammate's new `reports` pod from querying Postgres directly. A NetworkPolicy makes the architecture **enforced at the kernel**, so wrong connections fail loudly in dev instead of shipping to prod.

---

## Connected topics

- **Study before:**
  - **Topic 40 — Kubernetes Networking in Depth**: how pods get IPs and how the CNI wires the network — the layer that enforces policies.
  - **Topic 38 — Labels and Selectors**: policies target pods entirely by labels; you must be fluent in `matchLabels`.
  - **Topic 37 — Namespaces**: policies are namespaced, and `namespaceSelector` governs cross-namespace traffic.
- **Study after:**
  - **Topic 53 — RBAC**: NetworkPolicy locks the *network*; RBAC locks the *API*. Together they form defense in depth.
  - **Topic 50 — Secrets Management at Scale**: network segmentation complements secret isolation for a hardened cluster.
  - **Topic 51 — Observability in Kubernetes**: how to see dropped-connection metrics (e.g. Cilium/Hubble flow logs) to debug policies.
