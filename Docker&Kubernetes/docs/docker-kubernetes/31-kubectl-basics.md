# 31 — kubectl Basics

## Section: Kubernetes Foundations

---

## ELI5 — The Simple Analogy

Imagine a giant warehouse run entirely by robots. You never touch a box yourself. Instead, there is a **front desk** with one clerk. You walk up, hand the clerk a slip of paper — "I want 3 red boxes on shelf 5" — and walk away. The robots read your slip from the clerk's inbox and make it happen. If a box falls off, the robots put a new one back, because your slip is still pinned to the board.

In Kubernetes:

- The **warehouse** is your cluster.
- The **front desk clerk** is the **API server**.
- **`kubectl`** is the little machine on your desk that writes neat slips and hands them to the clerk over the phone.

You almost never talk to the robots (the nodes) directly. You talk to the clerk. `kubectl` is just a very polite, very precise phone. That is the entire mental model: **kubectl is an HTTP client that turns your commands into REST calls to the API server.** Nothing more magical than that.

---

## The OS / networking mechanism underneath

Kubernetes has no single "kubectl kernel feature" — it is a userland program. But under `kubectl` there are three very concrete mechanisms, and knowing them removes 90% of the confusion.

**1. It is an HTTPS client (TLS + TCP).**
Every `kubectl` command becomes an HTTPS request to the API server, usually on port `6443`. It uses the same TCP/TLS machinery as `curl`. If you remember from Docker networking (Topic 13), a TCP connection is a socket pair `(src_ip:src_port, dst_ip:dst_port)`. `kubectl get pods` opens exactly such a socket to `https://<api-server>:6443`.

**2. Mutual TLS (mTLS) for auth.**
The API server does not trust anyone by default. `kubectl` presents a **client certificate** (or a token), and the server presents its **server certificate**. Both sides verify each other against a **certificate authority (CA)**. This is why your kubeconfig contains three crypto blobs: the CA, your client cert, and your client key. On the wire it is standard X.509 — the same PKI that secures your bank's website, just pointed inward.

**3. `watch` = a long-lived streaming HTTP connection.**
When a controller (or `kubectl get pods --watch`) "watches" for changes, it is not polling in a loop. It opens **one HTTP connection that never closes** and the API server streams newline-delimited JSON events (`ADDED`, `MODIFIED`, `DELETED`) down that pipe as etcd changes. This uses HTTP chunked transfer encoding. Remember this from Topic 30 — the reconciliation loop is built on these watch streams.

So when this doc says "kubectl talks to the API server," it means precisely: **open a TLS socket to :6443, authenticate with a client cert, send an HTTP verb (GET/POST/PATCH/DELETE) to a REST path, read the JSON back.** You can literally replace `kubectl` with `curl` and it still works. We will prove that later.

---

## What is this?

`kubectl` (pronounced "cube-control" or "cube-cuttle" — nobody agrees) is the **official command-line client for Kubernetes**. It reads a config file called a **kubeconfig**, figures out which cluster to talk to, translates your command into a REST call to that cluster's API server, and prints the response. Every graphical dashboard, every CI pipeline, every `helm install` is ultimately doing what `kubectl` does: HTTP calls to the API server.

---

## Why does it matter for a backend developer?

Because `kubectl` is your **only window into the cluster**. When your `orders-api` pod is crash-looping at 2 a.m., you will not SSH into a server. You will run `kubectl logs`, `kubectl describe`, `kubectl exec`. If you do not know these verbs cold, an outage that should take 5 minutes takes 2 hours.

Here is what breaks without this knowledge:

- You `kubectl apply` a change to the **wrong cluster** because you never checked your **context**, and you take down production while trying to fix staging. This is the single most common self-inflicted outage.
- You run `kubectl get pods`, see "No resources found," and panic — when really your pods are in the `orders` **namespace** and you were looking at `default`.
- You cannot read a pod's logs, so you have no idea why the container exited.
- You waste an hour debugging RBAC when a single `-v=8` would have shown you the exact 403 HTTP response.

`kubectl` is not "a tool you look up." It is muscle memory, like `git` or `ls`.

---

## The physical reality

Where does all this live on your machine? Three places.

**1. The kubeconfig file — `~/.kube/config`**

This is a YAML file on your laptop. It is the entire address book + keyring for every cluster you can reach.

```
$ ls -l ~/.kube/config
-rw-------  1 you  staff  5342  Jul 12 09:14 /Users/you/.kube/config
        │
        └─ mode 600: read/write for you ONLY. It contains private keys.
           If this leaks, someone can act as you against your clusters.
```

Its structure, physically:

```
~/.kube/config
├── clusters:        # WHERE — the address + CA of each API server
│   └── - name: prod
│         cluster:
│           server: https://34.121.5.9:6443     ← the phone number
│           certificate-authority-data: LS0tLS1C… ← CA to trust the server
│
├── users:           # WHO — your identity / credentials
│   └── - name: alice
│         user:
│           client-certificate-data: LS0tLS1C… ← your ID card (public)
│           client-key-data: LS0tLS1C…         ← your private key (secret!)
│
├── contexts:        # WHICH PAIR — glue: "use THIS user against THIS cluster in THIS namespace"
│   └── - name: prod-orders
│         context:
│           cluster: prod
│           user: alice
│           namespace: orders
│
└── current-context: prod-orders   ← the single active pairing right now
```

Read that structure three times. **A context is just a saved triple: (cluster, user, namespace).** "Switching context" changes nothing on the server — it only changes which triple `kubectl` reads from this local file. That is why context mistakes are so dangerous: there is no server-side guardrail.

**2. The cached discovery + connection info — `~/.kube/cache/`**

```
~/.kube/cache/
├── discovery/      # cached list of every API type the cluster supports
│   └── 34.121.5.9_6443/…/serverresources.json
└── http/           # cached HTTP responses (ETags) to speed up repeat calls
```

`kubectl` caches "what kinds of objects does this cluster know about?" here so it does not re-ask on every command. If a new CRD (custom resource) does not show up, this stale cache is often why.

**3. The binary itself**

```
$ which kubectl
/usr/local/bin/kubectl
$ kubectl version --client
Client Version: v1.30.2
```

One static Go binary. No daemon. `kubectl` is not running in the background — it starts, makes HTTP calls, prints, and exits. Every single command is a fresh process.

---

## How it works — step by step

Let's trace one command end to end: `kubectl get pods`.

```
kubectl get pods
```

1. **`kubectl` process starts.** A fresh OS process. No shared state with your last command except files on disk.

2. **Find the kubeconfig.** It checks, in order:
   - The `--kubeconfig` flag (not set here).
   - The `$KUBECONFIG` env var (may list several files, colon-separated, merged).
   - The default `~/.kube/config`.

3. **Read `current-context`.** Say it is `prod-orders`. `kubectl` looks up that context and finds: cluster = `prod`, user = `alice`, namespace = `orders`.

4. **Resolve the server address.** From cluster `prod`: `server: https://34.121.5.9:6443`, plus the CA cert to trust it.

5. **Resolve credentials.** From user `alice`: client certificate + client key (or a token, or an exec plugin like `aws eks get-token`).

6. **Resolve the resource type.** You typed `pods`. `kubectl` consults its discovery cache (`~/.kube/cache/discovery/…`) to map the friendly name `pods` → the real API group/version/resource: `""/v1/pods` (core group, version v1). This is how `po`, `pods`, `pod` all resolve to the same thing.

7. **Build the URL.** Because you did not pass `-n`, it uses the context's namespace `orders`:
   ```
   GET https://34.121.5.9:6443/api/v1/namespaces/orders/pods?limit=500
   ```

8. **Open the TLS socket.** TCP connect to `34.121.5.9:6443`, TLS handshake. `kubectl` verifies the server cert against the CA (step 4). The server verifies `alice`'s client cert against its CA. **Mutual TLS complete.**

9. **Send the HTTP request.** `GET /api/v1/namespaces/orders/pods`, with header `Accept: application/json`.

10. **API server does its job** (this is the server side, but know it): authenticate the cert → authorize via RBAC ("can alice list pods in orders?") → read the pod list from **etcd** → serialize to JSON → send back `200 OK` with a JSON body.

11. **`kubectl` receives the JSON.** A `PodList` object with an `items` array.

12. **Format for humans.** By default `kubectl` prints a table (name, ready, status, restarts, age). It computes "age" locally from each pod's `creationTimestamp`. With `-o yaml` it would skip the table and dump raw YAML instead.

13. **Print to stdout, exit 0.** Process ends. Next command starts fresh from step 1.

The key insight: steps 1–7 are pure local file reading. Only step 8 onward touches the network. That is why `kubectl config` commands (which only edit `~/.kube/config`) are instant and never fail with connection errors.

---

## Exact syntax breakdown

### The anatomy of a kubectl command

```
kubectl get pods orders-api-7d4f -n orders -o wide
  │      │    │        │           │    │    │
  │      │    │        │           │    │    └─ OUTPUT format: table+extra columns
  │      │    │        │           │    └─ value: the namespace name
  │      │    │        │           └─ FLAG: --namespace (short -n)
  │      │    │        └─ NAME: a specific object (optional; omit = list all)
  │      │    └─ RESOURCE TYPE: what kind of object (pods, deploy, svc, node…)
  │      └─ VERB: the action (get, describe, apply, delete, logs, exec…)
  └─ the binary
```

Almost every command follows: `kubectl VERB RESOURCE [NAME] [FLAGS]`.

### `kubectl config` — inspecting and switching context

```
kubectl config get-contexts
        │      │
        │      └─ SUBCOMMAND: list all contexts; * marks the current one
        └─ VERB: "config" edits/reads ~/.kube/config ONLY (never hits the network)
```

```
kubectl config use-context staging-orders
        │      │           │
        │      │           └─ the context name to make current
        │      └─ SUBCOMMAND: set current-context (rewrites one line in ~/.kube/config)
        └─ config
```

```
kubectl config current-context
                └─ prints just the active context name. Put this in your shell prompt!
```

```
kubectl config set-context --current --namespace=orders
        │      │            │          │
        │      │            │          └─ change the default namespace…
        │      │            └─ …of the CURRENT context (don't name it explicitly)
        │      └─ SUBCOMMAND: modify a context's fields
        └─ config    (so you stop typing -n orders on every command)
```

### The core verbs

```
kubectl get pods
        │   │
        │   └─ RESOURCE (plural). "get" = list or show. The workhorse verb.
        └─ VERB
```

```
kubectl get pods -o wide
                 │  │
                 │  └─ "wide": normal table PLUS node name and pod IP
                 └─ -o / --output: choose the output format
```

```
kubectl describe pod orders-api-7d4f
        │        │   │
        │        │   └─ NAME (singular resource is fine here)
        │        └─ RESOURCE
        └─ VERB: human-readable deep dump + EVENTS. Your #1 debugging command.
```

```
kubectl apply -f deploy.yaml
        │     │  │
        │     │  └─ path to a manifest file (or a directory, or - for stdin)
        │     └─ -f / --filename: the file(s) to apply
        └─ VERB: DECLARATIVE. "make the cluster match this file." Idempotent.
```

```
kubectl delete pod orders-api-7d4f
        │      │   │
        │      │   └─ NAME
        │      └─ RESOURCE
        └─ VERB: remove the object. (For pods managed by a Deployment, a new one respawns!)
```

```
kubectl logs orders-api-7d4f -f --tail=100
        │    │                │   │
        │    │                │   └─ show only the last 100 lines to start
        │    │                └─ -f / --follow: stream new logs live (like tail -f)
        │    └─ POD NAME (logs is pod-specific)
        └─ VERB: print a container's stdout/stderr
```

```
kubectl logs orders-api-7d4f -c app --previous
                             │       │
                             │       └─ logs from the PREVIOUS (crashed) container instance
                             └─ -c: which container, if the pod has several
```

```
kubectl exec -it orders-api-7d4f -- sh
        │    ││  │                 │  │
        │    ││  │                 │  └─ the command to run INSIDE the container
        │    ││  │                 └─ "--" ends kubectl's flags; everything after is the command
        │    ││  └─ POD NAME
        │    │└─ -t: allocate a TTY (so the shell renders correctly)
        │    └─ -i: interactive — keep stdin open (so you can type)
        └─ VERB: run a process inside a running container
```

The `--` separator matters a lot. Without it, `kubectl` tries to interpret `sh`'s flags as its own. `--` is the universal Unix "stop parsing flags now" marker.

```
kubectl port-forward pod/orders-api-7d4f 8080:3000
        │            │                    │    │
        │            │                    │    └─ CONTAINER port (inside the pod)
        │            │                    └─ LOCAL port (on your laptop)
        │            └─ TYPE/NAME (works on pods, deployments, or services)
        └─ VERB: tunnel localhost:8080 → the pod's port 3000, through the API server
```

`port-forward` is not a normal network route. It tunnels bytes **through the API server over the existing TLS connection**, then out to the pod. That is why it works even when the pod has no public IP — you are riding the same secure pipe as every other kubectl command.

### Output formats

```
-o wide          table + node + pod IP (and a few type-specific extras)
-o yaml          full object as YAML — everything, including status
-o json          full object as JSON — same data, for scripts/jq
-o name          just "pod/orders-api-7d4f" — great for piping
-o jsonpath='…'  extract ONE field with a JSONPath expression
```

```
kubectl get pod orders-api-7d4f -o jsonpath='{.status.podIP}'
        │                          │          │  │      │
        │                          │          │  │      └─ the field to pull out
        │                          │          │  └─ .status: the sub-object (live state)
        │                          │          └─ {…} wraps a JSONPath expression
        │                          └─ jsonpath output mode
        └─ result: prints just "10.244.1.37" with no table, no quotes
```

---

## Example 1 — basic

You just got access to a new cluster. Walk in carefully.

```bash
# 1. Which cluster am I actually pointed at? ALWAYS check first.
kubectl config current-context
# → prod-orders          ← be very awake now: this is production

# 2. List every context I have, and which is active.
kubectl config get-contexts
# CURRENT   NAME            CLUSTER   AUTHINFO   NAMESPACE
# *         prod-orders     prod      alice      orders
#           staging-orders  staging   alice      orders
#           minikube        minikube  minikube   default

# 3. I want to poke around SAFELY, so switch to staging.
kubectl config use-context staging-orders
# Switched to context "staging-orders".

# 4. What's running? (uses the context's namespace: orders)
kubectl get pods
# NAME                          READY   STATUS    RESTARTS   AGE
# orders-api-7d4f9c8b6-2xk4p    1/1     Running   0          3h
# postgres-0                    1/1     Running   0          5d
# redis-6b9c7d5f4-qz8mn         1/1     Running   0          5d

# 5. See more — node placement and pod IPs.
kubectl get pods -o wide
# NAME                          READY   STATUS    ... IP            NODE
# orders-api-7d4f9c8b6-2xk4p    1/1     Running   ... 10.244.1.37   node-2

# 6. Look at a different namespace without switching context.
kubectl get pods -n kube-system
#   (-n overrides the context's namespace, just for THIS command)

# 7. Look at EVERY namespace at once.
kubectl get pods --all-namespaces      # or -A for short
```

Every line above is a separate HTTPS `GET` to the API server — except steps 1, 2, and 3, which only read/write `~/.kube/config` locally and never touch the network.

---

## Example 2 — production scenario

**3:10 a.m. Pager fires.** `orders-api` is returning 500s. Customers can't check out. You have your laptop and `kubectl`. Here is the exact debugging dance.

```bash
# STEP 0 — Confirm you're on the RIGHT cluster before touching anything.
kubectl config current-context
# → prod-orders     ✓ good, this is where the fire is

# STEP 1 — What's the state of the pods?
kubectl get pods -n orders
# NAME                          READY   STATUS             RESTARTS      AGE
# orders-api-7d4f9c8b6-2xk4p    0/1     CrashLoopBackOff   6 (30s ago)   12m
# orders-api-7d4f9c8b6-9dfgh    1/1     Running            0             3h
# postgres-0                    1/1     Running            0             9d
#                               │       │
#                               │       └─ CrashLoopBackOff: container keeps dying & restarting
#                               └─ 0/1: zero of one containers are ready
```

One replica is crash-looping. Investigate it.

```bash
# STEP 2 — WHY? describe shows the Events at the bottom — read those first.
kubectl describe pod orders-api-7d4f9c8b6-2xk4p -n orders
# ...
# Events:
#   Warning  BackOff    2m   kubelet  Back-off restarting failed container
#   Warning  Unhealthy  3m   kubelet  Readiness probe failed: dial tcp 10.244.1.37:3000: connect: connection refused
```

The app isn't listening on 3000. Read its logs.

```bash
# STEP 3 — Logs of the CURRENT container. Might be empty if it died instantly.
kubectl logs orders-api-7d4f9c8b6-2xk4p -n orders

# STEP 4 — The container already restarted, so read the PREVIOUS instance's logs.
kubectl logs orders-api-7d4f9c8b6-2xk4p -n orders --previous
# → Error: getaddrinfo ENOTFOUND postgres-primary
#          at GetAddrInfoReqWrap.onlookup (node:dns:...)
#   Cannot connect to Postgres. Exiting.
```

Root cause found: the app is trying to reach a host `postgres-primary` that doesn't resolve. Someone changed a config. Confirm what the pod thinks its DB host is.

```bash
# STEP 5 — Peek at the live env inside the HEALTHY replica (exec into it).
kubectl exec -it orders-api-7d4f9c8b6-9dfgh -n orders -- env | grep DB_HOST
# → DB_HOST=postgres          ← healthy pod has the RIGHT value

kubectl get pod orders-api-7d4f9c8b6-2xk4p -n orders \
  -o jsonpath='{.spec.containers[0].env[?(@.name=="DB_HOST")].value}'
# → postgres-primary          ← crashing pod has the WRONG value

# STEP 6 — Confirm: is there even a "postgres" service to reach?
kubectl get svc -n orders
# NAME       TYPE        CLUSTER-IP     PORT(S)    AGE
# postgres   ClusterIP   10.96.14.2     5432/TCP   9d
#   → yes, "postgres" exists; "postgres-primary" does not. Bad deploy confirmed.
```

You've localized the incident in six commands: a bad rollout set `DB_HOST=postgres-primary`. The fix is to roll back the Deployment (Topic 46), but the **diagnosis** was pure `kubectl`: `get` → `describe` → `logs --previous` → `exec`/`jsonpath`. Burn this sequence into memory.

---

## Common mistakes

**Mistake 1 — Running the command against the wrong cluster.**

```bash
$ kubectl apply -f new-config.yaml
deployment.apps/orders-api configured
# ...30 seconds later...
# "Why is STAGING behaving weird?" — because you were on prod-orders.
```

**Error you'll actually see:** none. That's the horror. There is no error — it succeeds against the wrong cluster.
**Root cause:** `current-context` is a single line in a local file with zero server-side guardrails. Nothing stops you.
**Wrong:** blindly running `kubectl apply` after a long break.
**Right:** make the context visible. Add it to your shell prompt, or alias:
```bash
alias kctx='kubectl config current-context'
# and put current-context in your PS1, or use the `kube-ps1` tool.
```

**Mistake 2 — "No resources found" because of the namespace.**

```bash
$ kubectl get pods
No resources found in default namespace.
                              │
                              └─ the clue is RIGHT THERE: it looked in "default"
```

**Root cause:** your pods are in `orders`, but your context's namespace is `default` (or you never set one).
**Wrong:** assuming the cluster is empty.
**Right:** `kubectl get pods -n orders`, or better, set the default once: `kubectl config set-context --current --namespace=orders`. Use `-A` to search all namespaces when unsure.

**Mistake 3 — `logs` shows nothing on a crash-looping pod.**

```bash
$ kubectl logs orders-api-7d4f9c8b6-2xk4p
$      ← empty. The pod already restarted; you're reading the NEW, silent instance.
```

**Root cause:** `kubectl logs` reads the **current** container. When it crashes, kubelet starts a fresh one; the crash output belonged to the old one.
**Wrong:** concluding "there are no logs."
**Right:** `kubectl logs <pod> --previous` (or `-p`) to read the instance that actually died.

**Mistake 4 — Forgetting `--` before the exec command.**

```bash
$ kubectl exec -it orders-api-7d4f9c8b6-2xk4p sh
error: unable to upgrade connection: container not found ("sh")
       │
       └─ kubectl tried to treat "sh" oddly without the separator (behavior varies by version)
```

**Root cause:** `kubectl` cannot tell where its own flags end and your command begins.
**Wrong:** `kubectl exec -it POD sh`
**Right:** `kubectl exec -it POD -- sh` (the `--` says "the command starts here").

**Mistake 5 — Deleting a pod expecting it to stay gone.**

```bash
$ kubectl delete pod orders-api-7d4f9c8b6-2xk4p
pod "orders-api-7d4f9c8b6-2xk4p" deleted
$ kubectl get pods
# orders-api-7d4f9c8b6-hj2wp   1/1   Running   0   4s      ← a NEW one appeared!
```

**Error you'll see:** none — and that's confusing.
**Root cause:** the pod is owned by a ReplicaSet (from a Deployment). The reconciliation loop (Topic 30) notices "desired 3, actual 2" and immediately creates a replacement. You can't kill managed pods by deleting them.
**Wrong:** `kubectl delete pod …` to "remove" an app.
**Right:** scale or delete the **Deployment**: `kubectl delete deployment orders-api`, or `kubectl scale deployment orders-api --replicas=0`.

---

## Hands-on proof

Prove that **kubectl is just an HTTP client**. You'll need a running cluster (`minikube start`, `kind create cluster`, or Docker Desktop's Kubernetes).

```bash
# 1. Confirm you can reach a cluster.
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   2m    v1.30.2

# 2. THE BIG REVEAL: run any command with -v=8 to see the raw HTTP.
kubectl get pods -v=8 2>&1 | grep -E 'GET|Response Status|Request Headers' | head
# You'll see literal lines like:
#   GET https://127.0.0.1:63421/api/v1/namespaces/default/pods?limit=500
#   Request Headers: Accept: application/json
#   Response Status: 200 OK in 14 milliseconds
#   → That's it. That's the whole "magic." A GET request.

# 3. Now do the SAME request yourself, bypassing kubectl entirely.
#    `kubectl proxy` opens a local, already-authenticated tunnel to the API server:
kubectl proxy --port=8001 &
curl -s http://127.0.0.1:8001/api/v1/namespaces/default/pods | head -c 300
# → {"kind":"PodList","apiVersion":"v1","metadata":{...
#    You just talked to the Kubernetes API with plain curl. kubectl = curl + kubeconfig.
kill %1   # stop the proxy

# 4. Inspect your OWN kubeconfig (secrets are redacted with this flag).
kubectl config view
# See clusters:, users:, contexts:, current-context: — the exact structure from earlier.

# 5. See the discovery cache that maps "pods" → "/api/v1/pods".
ls ~/.kube/cache/discovery/
# One directory per API server you've talked to.

# 6. Prove -o jsonpath extracts a single field.
kubectl get nodes -o jsonpath='{.items[0].status.nodeInfo.kubeletVersion}'
# → v1.30.2      (no table, just the value — perfect for scripts)
```

After step 2 you will never again think of `kubectl` as mysterious. It is `curl` with a config file and pretty tables.

---

## Practice exercises

### Exercise 1 — easy
1. Run `kubectl config current-context` and note the answer.
2. Run `kubectl config get-contexts` and identify which line has the `*`.
3. Create the `orders` namespace: `kubectl create namespace orders`.
4. Make `orders` your default namespace: `kubectl config set-context --current --namespace=orders`.
5. Run `kubectl config current-context` and `kubectl get pods` — confirm it now looks in `orders` (it'll say "No resources found in orders namespace").

### Exercise 2 — medium
1. Deploy something to look at: `kubectl create deployment orders-api --image=nginx --replicas=2 -n orders`.
2. List the pods with `-o wide` and record each pod's IP and node.
3. Use `-o jsonpath` to print **only** the name of the first pod:
   `kubectl get pods -n orders -o jsonpath='{.items[0].metadata.name}'`.
4. `kubectl describe` one of the pods and find the `Events` section — what events fired during startup?
5. `kubectl exec -it <pod> -n orders -- sh`, then inside run `hostname` and `ls /`. Exit.
6. Run `kubectl port-forward deploy/orders-api 8080:80 -n orders`, then in another terminal `curl localhost:8080` — you should get nginx's welcome page through the tunnel.

### Exercise 3 — hard (production simulation)
1. Break a pod on purpose: create a deployment whose container exits immediately:
   `kubectl create deployment breaker --image=busybox -n orders -- /bin/false`.
2. Watch it enter `CrashLoopBackOff`: `kubectl get pods -n orders -w` (Ctrl-C to stop the watch).
3. Try `kubectl logs <breaker-pod> -n orders`. Is there output? Now try `--previous`. Explain the difference in one sentence.
4. Use `kubectl describe pod <breaker-pod> -n orders` and quote the exact Event line that shows the back-off.
5. Prove the reconciliation loop: `kubectl delete pod <breaker-pod> -n orders`, then immediately `kubectl get pods -n orders`. A new pod appears. Explain WHO recreated it and why (reference Topic 30).
6. Finally, actually stop the crashing: `kubectl delete deployment breaker -n orders`. Confirm the pods are gone for good.

---

## Mental model checkpoint

Answer these from memory:

1. In one sentence, what does `kubectl` fundamentally *do* under the hood?
2. What three things does a **context** bind together?
3. Where does switching context take effect — on the server, or on your laptop? Why does that make it dangerous?
4. Your pod crashed and restarted. Which flag do you add to `kubectl logs` to see why it died?
5. What does the `--` do in `kubectl exec -it pod -- sh`?
6. You `kubectl delete pod` a pod and a new one appears seconds later. Who created it and why?
7. What's the difference between `-o wide`, `-o yaml`, and `-o jsonpath`?

---

## Quick reference card

| Command | What it does | Key detail |
|---|---|---|
| `kubectl config current-context` | Print active context | Check this BEFORE any risky command |
| `kubectl config get-contexts` | List all contexts | `*` marks the current one |
| `kubectl config use-context NAME` | Switch active context | Only edits `~/.kube/config`; no network |
| `kubectl config set-context --current --namespace=X` | Set default namespace | Stops you retyping `-n X` |
| `kubectl get RESOURCE` | List/show objects | The workhorse verb |
| `kubectl get RESOURCE -o wide` | Table + node + IP | Extra columns, still a table |
| `kubectl describe TYPE NAME` | Deep human dump + Events | Read the **Events** section first when debugging |
| `kubectl apply -f FILE` | Declarative sync to file | Idempotent; safe to re-run |
| `kubectl delete TYPE NAME` | Remove an object | Managed pods respawn — delete the controller instead |
| `kubectl logs POD` | Container stdout/stderr | Add `-f` to stream, `-c` to pick a container |
| `kubectl logs POD --previous` | Logs of the crashed instance | Essential for CrashLoopBackOff |
| `kubectl exec -it POD -- sh` | Shell inside a container | `--` separates kubectl flags from the command |
| `kubectl port-forward TYPE/NAME L:R` | Tunnel localhost:L → pod:R | Rides the API server's TLS connection |
| `kubectl get X -o jsonpath='{…}'` | Extract one field | Great for scripts; no table, no quotes |
| `-n NAME` / `-A` | Target one / all namespaces | `-n` overrides context namespace for one command |
| `-v=8` | Show raw HTTP requests | Proves kubectl is just an HTTP client |

---

## When would I use this at work?

1. **Incident response.** `orders-api` is throwing 500s. You run `get pods` → `describe pod` → `logs --previous` → `exec` to diagnose in minutes, exactly like Example 2. This is 80% of what you'll do with kubectl.

2. **Safe deploys.** Before shipping, you `kubectl config current-context` to be 100% sure you're targeting staging, `kubectl apply -f` your manifest, then `kubectl get pods -w` to watch the rollout. Context discipline prevents the classic "oops, that was prod" outage.

3. **Ad-hoc verification and scripting.** A teammate asks "what version of Node is the API pod running?" You `kubectl exec -it orders-api-… -- node --version`. Or you script a health check in CI with `kubectl get deploy orders-api -o jsonpath='{.status.readyReplicas}'` and fail the pipeline if it's not the expected count.

---

## Connected topics

- **Study before:** Topic 29 (Kubernetes Architecture — you must know what the API server and etcd are before you can understand what kubectl talks to), Topic 30 (The Reconciliation Loop — explains why deleted managed pods come back).
- **Study after:** Topic 32 (YAML and Manifests — the files you feed to `kubectl apply`), Topic 33 (Pods in Depth — the objects you'll be `get`/`describe`/`logs`-ing constantly), Topic 37 (Namespaces — the isolation boundary that `-n` selects).
