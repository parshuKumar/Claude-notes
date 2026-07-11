# 32 — YAML and Manifests

## Section: Kubernetes Foundations

---

## ELI5 — The Simple Analogy

Imagine you want a house built. You do **not** stand next to the builder saying "put a brick there… now another brick… now a window." That is exhausting and if you leave, everything stops.

Instead, you hand the builder a **blueprint**: "3 bedrooms, 2 bathrooms, red door." You walk away. The builder reads the blueprint and makes the house match it. If a wall falls down later, the builder looks at the blueprint again and rebuilds that wall — because the blueprint still says the wall should be there.

A **Kubernetes manifest is that blueprint.** It is a YAML file that says "I want 3 copies of `orders-api`, using this image, exposing this port." You hand it to Kubernetes with `kubectl apply`. Kubernetes stores the blueprint and works forever to make reality match it. That is the whole idea of **declarative** infrastructure: you describe the *destination*, not the *turn-by-turn directions*.

---

## The mechanism underneath: etcd, the source of truth

There is no Linux kernel primitive for a "manifest." The mechanism underneath a manifest is a **distributed key-value database called etcd**, and it is worth understanding physically because everything in Kubernetes is really "an entry in etcd that controllers watch."

When you `kubectl apply -f deploy.yaml`:

1. Your YAML becomes an HTTP request to the API server (Topic 31).
2. The API server validates it, then **writes it as a key-value pair into etcd**.
3. The key is a path; the value is the object, serialized.

You can see the raw etcd entry. Kubernetes stores objects under `/registry/`:

```
etcd key:    /registry/deployments/orders/orders-api
              │         │           │      │
              │         │           │      └─ the object's NAME
              │         │           └─ the NAMESPACE
              │         └─ the RESOURCE type (plural)
              └─ every K8s object lives under /registry/

etcd value:  the full object, serialized as protobuf (NOT the YAML you wrote —
             the API server converts your YAML → internal object → protobuf bytes)
```

You can literally pull it out:

```bash
# On a control-plane node (advanced — proves the point):
ETCDCTL_API=3 etcdctl get /registry/deployments/orders/orders-api \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
# → prints the stored object. THIS is the single source of truth for the cluster.
```

etcd matters because of three properties:

- **It is the *only* stateful component.** Everything else (scheduler, controllers, kubelet) is stateless and just reads/writes etcd through the API server. Wipe every node but keep etcd, and you can rebuild the cluster. Wipe etcd and the cluster is gone.
- **It is consistent (Raft consensus).** A write is not acknowledged until a majority of etcd members agree. That is why you run 3 or 5 etcd nodes, never 2 or 4 — you need a majority to survive one failure.
- **It supports *watches*.** Remember from Topic 31 and 30: controllers open a watch on etcd (via the API server) and get streamed every change. So the moment your Deployment lands in etcd, the Deployment controller *wakes up* and starts creating pods. The manifest is inert data; the watch is what makes it *do* something.

So a manifest's real life is: **YAML text on your disk → HTTP → validated → protobuf bytes under `/registry/...` in etcd → a watch fires → a controller reconciles.**

---

## What is this?

A **manifest** is a YAML (or JSON) file that describes a Kubernetes object — a Pod, Deployment, Service, ConfigMap, anything. Every manifest, no matter the type, has the **same four top-level keys**: `apiVersion`, `kind`, `metadata`, and `spec` (plus a read-only `status` the cluster fills in). You write the *desired* state in `spec`; Kubernetes writes the *actual* state in `status`. That four-key skeleton is the grammar of all of Kubernetes.

---

## Why does it matter for a backend developer?

Because **manifests are your infrastructure as code.** Your `orders-api` deployment, its Service, its config, its scaling rules — all of it lives in YAML files checked into git next to your application code. If you can't read and write manifests fluently, you can't ship to Kubernetes at all.

What breaks without this understanding:

- You copy a YAML from a blog, it has `apiVersion: extensions/v1beta1`, and `kubectl apply` fails with a cryptic "no matches for kind" error. You don't know it's a version problem.
- You use **tabs** for indentation (YAML forbids them) and get a parse error you can't decode.
- You mix up `spec` (what you want) and `status` (what is), and try to "set" a field the cluster owns.
- You run `kubectl apply` and `kubectl create` interchangeably, then get "AlreadyExists" errors in CI and don't understand why one is idempotent and the other isn't.
- You get bitten by the infamous **Norway problem** (`NO` becoming `false`) or a version string like `1.10` silently becoming the number `1.1`.

Manifests are the contract between you and the cluster. This topic is about reading that contract precisely.

---

## The physical reality

A manifest exists in **three physical forms** during its life. Knowing all three demystifies everything.

**Form 1 — Text on your disk.** A plain UTF-8 file, `deploy.yaml`, in your git repo.

```
$ ls -l deploy.yaml
-rw-r--r--  1 you  staff  612  Jul 12 10:02 deploy.yaml
$ file deploy.yaml
deploy.yaml: ASCII text
```

It is just text. Indentation is spaces. No magic.

**Form 2 — An in-memory object inside the API server.** When you apply it, the API server parses the YAML into a Go struct, runs it through **conversion** (to a canonical version), **defaulting** (fills in fields you omitted), and **admission** (validation/mutation). At this point it is no longer "your YAML" — it is a fully-populated object with dozens of fields you never wrote.

**Form 3 — Protobuf bytes in etcd.** The canonical object is serialized (protobuf by default, not YAML) and written under `/registry/...`. This is the durable, authoritative copy.

You can watch the round trip: write a tiny YAML, apply it, then ask for it back with `-o yaml`. What comes back is **much bigger** than what you wrote — that's Forms 2 and 3 revealing all the defaulted fields plus the live `status`:

```
what you WROTE (Form 1)          what you GET BACK (Forms 2+3 reflected)
─────────────────────            ──────────────────────────────────────
apiVersion: v1                   apiVersion: v1
kind: Pod                        kind: Pod
metadata:                        metadata:
  name: demo                       name: demo
spec:                              namespace: default        ← defaulted
  containers:                      uid: 8f3a...              ← assigned by API server
  - name: app                      creationTimestamp: ...    ← assigned
    image: nginx                   resourceVersion: "48213"  ← etcd version counter
                                 spec:
                                   containers:
                                   - name: app
                                     image: nginx
                                     imagePullPolicy: Always  ← defaulted
                                     terminationMessagePath:  ← defaulted
                                       /dev/termination-log   ← defaulted
                                 status:                      ← ADDED by the cluster
                                   phase: Running
                                   podIP: 10.244.1.9
```

The gap between the two columns is the entire story of this topic: **you write a small `spec`; the cluster expands it, persists it, and reports back a `status`.**

---

## How it works — step by step

Trace `kubectl apply -f deploy.yaml` from your keystroke to a durable etcd entry.

1. **`kubectl` reads the file** off your disk. Pure local I/O.

2. **`kubectl` parses YAML → JSON.** The Kubernetes API speaks JSON on the wire; YAML is a convenience for humans. `kubectl` converts your YAML into an equivalent JSON body. (This is why every field in YAML has a JSON equivalent — they're the same data model.)

3. **`kubectl` decides the HTTP verb.** For `apply`, it uses a `PATCH` with a special `apply` content-type (server-side apply) or diffs against a stored annotation (client-side apply). For `create` it would be a `POST`; for `delete`, a `DELETE`. It builds the URL from `apiVersion` + `kind`:
   ```
   apiVersion: apps/v1  +  kind: Deployment
        │                       │
        └──────────┬────────────┘
                   ▼
   PATCH /apis/apps/v1/namespaces/orders/deployments/orders-api
   ```

4. **API server authenticates & authorizes** (mTLS + RBAC — Topic 31 step 10).

5. **Decode & convert.** The API server decodes the JSON into an internal Go object and converts it to its canonical internal version. (If you sent an old `apiVersion`, this is where it gets upgraded — or rejected if that version no longer exists.)

6. **Schema validation.** Does `spec.replicas` expect an integer? Is `spec.selector` present and valid? Unknown fields are (by default) **rejected** with `strict` decoding via `apply`, or dropped/warned otherwise. A typo like `replcas: 3` gets caught here.

7. **Defaulting.** The server fills in every omitted-but-required field with defaults: `imagePullPolicy`, `restartPolicy: Always`, `strategy: RollingUpdate`, and so on. This is why the object you get back is bigger than what you sent.

8. **Admission control.** A chain of plugins runs:
   - **Mutating** admission may *change* the object (inject a sidecar, add default labels).
   - **Validating** admission may *reject* it (e.g., a policy: "no containers may run as root").
   ResourceQuota checks also happen here (Topic 37/44).

9. **Persist to etcd.** The final canonical object is serialized to protobuf and written to `/registry/deployments/orders/orders-api`. etcd's Raft ensures a majority of members commit it. etcd bumps a global `resourceVersion` counter.

10. **The watch fires.** Any controller watching Deployments (here, the Deployment controller) receives a `MODIFIED`/`ADDED` event over its open watch stream. It wakes up.

11. **Reconciliation begins** (Topic 30). The Deployment controller sees "desired 3 replicas" and creates a ReplicaSet; the ReplicaSet controller creates 3 Pod objects (more etcd writes); the scheduler assigns each Pod to a node; each node's kubelet pulls the image and starts the container. Every one of these is another loop of "write desired state to etcd → a watcher reconciles."

12. **`status` gets written back.** As pods come up, controllers and kubelet **PATCH the `status`** subresource of your objects — `availableReplicas`, `podIP`, `phase: Running`. You never write `status`; the cluster owns it. That's why `kubectl get -o yaml` shows a rich `status` you never typed.

The manifest itself did nothing after step 9. Everything from step 10 on is the *watch + reconcile* engine reacting to the new desired state. **The manifest is the goal; the controllers are the workers.**

---

## Exact syntax breakdown

### The universal four-key skeleton

Every manifest — Pod, Deployment, Service, ConfigMap, Secret, Ingress — has this shape:

```
apiVersion: apps/v1     ← 1. WHICH API (group + version) validates this object
kind: Deployment        ← 2. WHAT kind of object this is
metadata:               ← 3. WHO it is: name, namespace, labels, annotations
  name: orders-api
spec:                   ← 4. WHAT YOU WANT: the desired state (you write this)
  ...
status:                 ← 5. WHAT IS: the actual state (the CLUSTER writes this; you don't)
  ...
```

Memorize the four you write: **apiVersion, kind, metadata, spec.** The fifth, `status`, is read-only to you.

### `apiVersion` — which API endpoint

```
apiVersion: apps/v1
            │    │
            │    └─ VERSION: v1 (stable), v1beta1 (beta), v1alpha1 (experimental)
            └─ GROUP: "apps" (Deployments, StatefulSets). Empty for core objects.
```

```
apiVersion: v1        ← core group has NO group name — just the version.
            │          Used by Pod, Service, ConfigMap, Secret, Namespace.
            └─ core/legacy group (historically served at /api/v1, not /apis/...)
```

Common pairings you must recognize:

```
Pod, Service, ConfigMap, Secret, Namespace  →  apiVersion: v1
Deployment, ReplicaSet, StatefulSet, DaemonSet →  apiVersion: apps/v1
Job, CronJob                                 →  apiVersion: batch/v1
Ingress                                      →  apiVersion: networking.k8s.io/v1
HorizontalPodAutoscaler                      →  apiVersion: autoscaling/v2
```

Getting this pair wrong is the #1 beginner error — see Common Mistakes.

### `kind` — the object type

```
kind: Deployment
      │
      └─ PascalCase, SINGULAR. "Deployment" not "deployments".
         (In kubectl commands you use lowercase/plural: `kubectl get deployments`.
          In the manifest you use PascalCase singular. Two different conventions.)
```

### `metadata` — identity

```
metadata:
  name: orders-api                 ← REQUIRED. Unique within (namespace + kind).
  namespace: orders                ← optional; defaults to "default" or your context's ns.
  labels:                          ← key/value tags. USED BY selectors to find this object.
    app: orders-api                   (Topic 38 — labels are how objects find each other.)
    tier: backend
  annotations:                     ← key/value metadata NOT used for selection.
    kubernetes.io/change-cause: "deploy v1.4.2"   (notes, tooling data, checksums)
    │                          │
    │                          └─ arbitrary value
    └─ often namespaced with a domain to avoid collisions
```

`labels` vs `annotations` is a crucial distinction: **labels are for *selecting/grouping* (queryable), annotations are for *describing* (not queryable).** A selector can find "all pods with `app=orders-api`"; it cannot find "all pods with a certain annotation."

### `spec` — desired state (a Deployment example)

```
spec:
  replicas: 3                      ← DESIRED count. The reconciliation loop's target (Topic 30).
  selector:                        ← how THIS Deployment finds ITS pods…
    matchLabels:
      app: orders-api              ← …it manages every pod with this label.
  template:                        ← the POD BLUEPRINT stamped out `replicas` times
    metadata:
      labels:
        app: orders-api            ← MUST match selector.matchLabels above, or apply fails.
    spec:                          ← this inner spec is a POD spec
      containers:
      - name: app                  ← "- " starts a LIST ITEM (a container)
        image: myrepo/orders-api:1.4.2
        ports:
        - containerPort: 3000      ← the port the Node app listens on
        env:
        - name: DB_HOST
          value: postgres          ← reach the Postgres Service by name (Topic 35)
```

The `selector.matchLabels` and `template.metadata.labels` **must agree** — this is a self-referential link that trips up everyone once. The Deployment uses the selector to know which pods are "its own."

### YAML syntax essentials (annotated)

```
key: value            ← a mapping (dictionary) entry. Colon-SPACE is required.
  │  │
  │  └─ scalar value (string, number, bool)
  └─ key

parent:               ← nesting is done with SPACES only. NEVER tabs.
  child: value        ← 2 spaces = one level deeper (2 is convention; be consistent)

list:                 ← a sequence
- item1               ← "- " (dash space) marks each element
- item2

inline_list: [a, b, c]        ← flow style; same as a dash list
inline_map: {name: app, port: 3000}   ← flow style mapping

multiline: |          ← "|" keeps newlines (literal block) — great for scripts/configs
  line one
  line two

folded: >             ← ">" folds newlines into spaces (one long line)
  this becomes
  one line

quoted: "1.10"        ← QUOTE version strings & risky values so YAML doesn't retype them
comment: value  # everything after # is ignored
---                   ← three dashes separate MULTIPLE documents in one file
```

### Multiple objects in one file

```
apiVersion: v1
kind: Service
metadata:
  name: orders-api
spec:
  ...
---                   ← document separator: everything below is a SECOND object
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  ...
```

`kubectl apply -f` this file and it applies **both** objects in order. This is the standard way to keep an app's Service + Deployment together.

### Imperative vs declarative verbs

```
DECLARATIVE (preferred, git-friendly):
  kubectl apply -f deploy.yaml     ← "make the cluster match this file." Idempotent.
                                      Run it 100 times → same result. Diffs & patches.

IMPERATIVE (quick, one-off, no file):
  kubectl create deployment orders-api --image=orders-api:1.4.2 --replicas=3
                │                                                 │
                │                                                 └─ flags describe the object
                └─ "create" = POST. Fails with AlreadyExists if it's already there.

  kubectl run tmp --image=busybox -it -- sh    ← create a throwaway pod
  kubectl scale deployment orders-api --replicas=5   ← imperative edit
  kubectl delete -f deploy.yaml                ← delete what the file describes
```

Rule of thumb: **`apply` with files for anything real (it's your infrastructure-as-code).** Use imperative commands only for throwaway debugging or to *generate* YAML:

```
kubectl create deployment orders-api --image=nginx --dry-run=client -o yaml > deploy.yaml
                                       │                │
                                       │                └─ print the object instead of creating it
                                       └─ --dry-run=client: build it locally, DON'T send to server
# → a great way to scaffold a manifest you then edit and check into git.
```

---

## Example 1 — basic

The smallest complete, real manifest: a single Pod. Every line explained.

```yaml
# pod.yaml — one Pod running the orders-api container
apiVersion: v1            # Pod is a core object → group is empty, just "v1"
kind: Pod                 # PascalCase, singular
metadata:
  name: orders-api        # unique name within the namespace
  namespace: orders       # place it in the "orders" namespace
  labels:
    app: orders-api       # a tag other objects (Services) can select on
spec:                     # ← desired state starts here
  containers:             # a list (a Pod can have several containers)
  - name: app             # "- " = first (only) list item; container name
    image: node:20-alpine # the image to run (Topic 04)
    command: ["node", "server.js"]   # override the container's default command
    ports:
    - containerPort: 3000 # informational: the port the app listens on
    env:
    - name: NODE_ENV      # an environment variable name…
      value: "production" # …and its value (quoted to be safe)
```

Apply and inspect:

```bash
kubectl apply -f pod.yaml
# pod/orders-api created

kubectl get pod orders-api -n orders -o yaml | head -40
# → you'll see YOUR fields PLUS defaulted fields (imagePullPolicy, restartPolicy…)
#   PLUS a whole status: block you never wrote. That's the cluster talking back.
```

Note: a bare Pod is fine for learning, but in production you almost never create Pods directly — a **Deployment** (Topic 34) creates and heals them for you. This example is to see the skeleton clearly.

---

## Example 2 — production scenario

**The setup:** your team is deploying `orders-api` v1.4.2. It needs 3 replicas, a Service so other pods can reach it, and it talks to Postgres and Redis. You keep everything in one git-tracked file, `orders-api.yaml`, and ship with `kubectl apply`.

```yaml
# orders-api.yaml — Service + Deployment for the orders API
apiVersion: v1
kind: Service                     # a stable network name/IP in front of the pods
metadata:
  name: orders-api                # other pods reach it at http://orders-api.orders:3000
  namespace: orders
  labels:
    app: orders-api
spec:
  selector:
    app: orders-api               # this Service routes to any pod labeled app=orders-api
  ports:
  - port: 3000                    # the port the Service listens on
    targetPort: 3000              # the port on the pod to forward to
---                               # ↓ second document: the Deployment ↓
apiVersion: apps/v1               # Deployment lives in the "apps" group
kind: Deployment
metadata:
  name: orders-api
  namespace: orders
  labels:
    app: orders-api
  annotations:
    kubernetes.io/change-cause: "deploy orders-api v1.4.2"   # shows in rollout history
spec:
  replicas: 3                     # desired: 3 identical pods
  selector:
    matchLabels:
      app: orders-api             # this Deployment owns pods with this label…
  template:                       # …and stamps out pods from this blueprint
    metadata:
      labels:
        app: orders-api           # MUST equal selector.matchLabels (and the Service selector)
    spec:
      containers:
      - name: app
        image: myrepo/orders-api:1.4.2   # PINNED version — never rely on :latest in prod
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          value: postgres         # the Postgres Service name in this namespace
        - name: REDIS_HOST
          value: redis            # the Redis Service name
        - name: NODE_ENV
          value: "production"
```

Ship and verify:

```bash
# 1. Preview WHAT WILL CHANGE without changing anything (safe on prod).
kubectl diff -f orders-api.yaml
#    → shows a red/green diff of the live objects vs your file.

# 2. Apply both objects.
kubectl apply -f orders-api.yaml
# service/orders-api created
# deployment.apps/orders-api created

# 3. Watch the desired→actual convergence (the reconciliation loop, Topic 30).
kubectl get pods -n orders -w
#    → 0/1 Pending → ContainerCreating → 1/1 Running, three times.

# 4. Later, ship v1.4.3: edit the image tag in the file, then apply again.
#    apply is DECLARATIVE — it diffs, sees only the image changed, and does a rolling update.
kubectl apply -f orders-api.yaml
# deployment.apps/orders-api configured    ← "configured", not "created": it patched.
```

The magic of step 4: you did not tell Kubernetes *how* to update. You changed the desired state in the file and re-applied. The Deployment controller computed the diff and rolled out the change with zero downtime. **That is declarative infrastructure — you edit the blueprint, the cluster does the work.**

---

## Common mistakes

**Mistake 1 — Wrong `apiVersion` for the `kind`.**

```bash
$ kubectl apply -f deploy.yaml
error: unable to recognize "deploy.yaml": no matches for kind "Deployment" in version "extensions/v1beta1"
                                                                                    │
                                                                                    └─ that API version was removed in k8s 1.16
```

**Root cause:** you copied an old manifest. `Deployment` moved from `extensions/v1beta1` to `apps/v1` years ago, and the old versions were deleted from the API server. The server literally has no endpoint for that group/version anymore.
**Wrong:** `apiVersion: extensions/v1beta1` with `kind: Deployment`.
**Right:** `apiVersion: apps/v1`. Check the correct pairing with `kubectl explain deployment` or `kubectl api-resources`.

**Mistake 2 — Tabs for indentation.**

```bash
$ kubectl apply -f pod.yaml
error: error parsing pod.yaml: error converting YAML to JSON: yaml: line 6: found character that cannot start any token
                                                                              │
                                                                              └─ that "character" is a TAB
```

**Root cause:** YAML **forbids tabs** for indentation — only spaces. Your editor inserted a tab.
**Wrong:** a `\t` anywhere in the indentation.
**Right:** use spaces only. Configure your editor: for YAML, expand tabs to 2 spaces. Run `cat -A pod.yaml` and look for `^I` (that's a tab) to hunt them down.

**Mistake 3 — The "Norway problem" / unquoted scalars.**

```yaml
countries:
  norway: NO          # ← you meant the string "NO"…
db_version: 1.10      # ← you meant the string "1.10"…
enabled: yes          # ← you meant the string "yes"…
```
```
# What YAML actually parses:
#   norway     → false        (NO is a boolean!)
#   db_version → 1.1          (1.10 is a number; trailing zero dropped!)
#   enabled    → true         (yes is a boolean!)
```

**Root cause:** YAML 1.1 auto-types bare scalars. `NO`, `yes`, `on`, `off`, `true` become booleans; `1.10` becomes a float. Your string got silently converted.
**Wrong:** `image: myrepo/app:1.10` unquoted (becomes `1.1`).
**Right:** **quote anything that could be misread:** `db_version: "1.10"`, `enabled: "yes"`, version tags, and country/language codes. When in doubt, quote it.

**Mistake 4 — Selector doesn't match template labels.**

```bash
$ kubectl apply -f deploy.yaml
The Deployment "orders-api" is invalid: spec.template.metadata.labels: Invalid value:
map[string]string{"app":"orders"}: `selector` does not match template `labels`
```

**Root cause:** `spec.selector.matchLabels` says `app: orders-api` but `spec.template.metadata.labels` says `app: orders`. A Deployment must be able to select the very pods it creates; mismatched labels make that impossible, so the API server rejects it in validation (step 6).
**Wrong:** selector and template labels differ.
**Right:** make them identical. The Service selector should also match, so traffic reaches the pods.

**Mistake 5 — Using `create` when you should `apply` (and vice-versa).**

```bash
$ kubectl create -f deploy.yaml
Error from server (AlreadyExists): error when creating "deploy.yaml":
deployments.apps "orders-api" already exists
```

**Root cause:** `create` is imperative (`POST`) — it only makes *new* objects and fails if one exists. `apply` is declarative (`PATCH`) — it creates *or* updates, so it's idempotent.
**Wrong:** `kubectl create -f` in a CI pipeline that re-runs.
**Right:** `kubectl apply -f` for anything you'll deploy more than once. Reserve `create` for one-shot resources or when you explicitly want it to fail on a duplicate.

---

## Hands-on proof

Prove that (a) the object grows after applying, (b) it lands in etcd, and (c) you own `spec` but the cluster owns `status`.

```bash
# 1. Write the smallest possible manifest.
cat > /tmp/demo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
  - name: app
    image: nginx
EOF

# 2. Count the lines you wrote.
wc -l /tmp/demo.yaml
# → 8 lines

# 3. Apply it.
kubectl apply -f /tmp/demo.yaml
# pod/demo created

# 4. Ask for it back and count — it's WAY bigger (defaulting + status).
kubectl get pod demo -o yaml | wc -l
# → ~80+ lines. The cluster added defaults and a whole status: block.

# 5. See the fields YOU never wrote.
kubectl get pod demo -o yaml | grep -E 'imagePullPolicy|restartPolicy|uid|resourceVersion|phase|podIP'
# imagePullPolicy: Always      ← defaulted
# restartPolicy: Always        ← defaulted
# uid: 8f3a...                 ← assigned by the API server
# resourceVersion: "48213"     ← etcd's version counter
# phase: Running               ← STATUS: written by the cluster, not you
# podIP: 10.244.1.9            ← STATUS: assigned at runtime

# 6. Prove apply is idempotent — run it again.
kubectl apply -f /tmp/demo.yaml
# pod/demo unchanged           ← no diff → no change. Safe to re-run forever.

# 7. Prove create is NOT idempotent.
kubectl create -f /tmp/demo.yaml
# Error from server (AlreadyExists): pods "demo" already exists

# 8. Ask the cluster what a field means (built-in schema docs).
kubectl explain pod.spec.containers.image
# → "Container image name. More info: ... "  (great for discovering valid fields)

# 9. Cleanup.
kubectl delete -f /tmp/demo.yaml
# pod "demo" deleted
```

Steps 4–5 are the whole lesson made visible: **you write a small `spec`, the cluster expands it with defaults and reports a live `status`.**

---

## Practice exercises

### Exercise 1 — easy
1. Write a manifest for a single Pod named `web` running `nginx`, with a label `app: web`.
2. `kubectl apply -f` it, then `kubectl get pod web -o yaml`.
3. Find three fields in the output that you did **not** write. State whether each is a *default* or part of *status*.
4. Delete it with `kubectl delete -f`.

### Exercise 2 — medium
1. Generate a Deployment manifest *without writing YAML by hand*:
   `kubectl create deployment orders-api --image=nginx --replicas=2 --dry-run=client -o yaml > orders.yaml`.
2. Open `orders.yaml`. Identify the four top-level keys and the `selector` / `template.labels` pair.
3. Add a second container port and an env var `NODE_ENV=production` to the container. Quote the value.
4. `kubectl apply -f orders.yaml`, confirm 2 pods run, then change `replicas` to 4 in the file and `apply` again. Watch the reconciliation add 2 pods.
5. Add a Service to the **same file** below a `---` separator (selector `app: orders-api`, port 80). Re-apply — confirm both objects update.

### Exercise 3 — hard (production simulation)
1. Deliberately introduce the **Norway problem**: set an env var `value: yes` (unquoted). Apply, then `kubectl get pod <name> -o yaml` and grep for that env var. What value did it actually become? Explain why in terms of YAML typing.
2. Deliberately break the label link: make `selector.matchLabels.app: orders-api` but `template.metadata.labels.app: orders`. Apply and capture the **exact** error message. Explain which validation step (from "How it works") rejected it.
3. Take a working manifest and change `apiVersion: apps/v1` to `apiVersion: apps/v1beta1`. Apply and capture the error. Explain what the API server's *conversion* step did (or couldn't do).
4. Prove the etcd round-trip conceptually: apply a Deployment, then run `kubectl get deployment orders-api -o jsonpath='{.metadata.resourceVersion}'`. Apply an unchanged file (`apply` again) — does the resourceVersion change? Now edit `replicas` and apply — does it change now? Explain what `resourceVersion` tracks.

---

## Mental model checkpoint

Answer from memory:

1. What are the four top-level keys in **every** manifest that you write?
2. Which of those keys is read-only to you, and who writes it?
3. What is the difference between `spec` and `status` in one sentence?
4. `Deployment` uses which `apiVersion`? `Pod` uses which? Why is one different?
5. Where does a manifest physically end up after `kubectl apply`, and in what format (not YAML)?
6. Why is `kubectl apply` idempotent but `kubectl create` is not?
7. What's the difference between a **label** and an **annotation**?
8. Why must you quote `"1.10"` and `"yes"` in YAML?

---

## Quick reference card

| Field / Concept | What it does | Key detail |
|---|---|---|
| `apiVersion` | Which API group+version validates the object | `apps/v1` for Deployment; bare `v1` for core (Pod/Service) |
| `kind` | The object type | PascalCase singular (`Deployment`), unlike kubectl's plural |
| `metadata.name` | Unique identity | Unique per (namespace + kind) |
| `metadata.labels` | Queryable tags | Used by **selectors** to link objects (Topic 38) |
| `metadata.annotations` | Non-queryable metadata | Notes/tooling data; selectors can't match on these |
| `spec` | Desired state | **You** write this — the reconciliation target |
| `status` | Actual state | **Cluster** writes this — read-only to you |
| `selector` + `template.labels` | Links a controller to its pods | Must match, or the API server rejects the object |
| `---` | Document separator | Multiple objects in one file, applied in order |
| `\|` / `>` | Multiline block / folded string | `\|` keeps newlines; `>` folds them to spaces |
| `kubectl apply -f` | Declarative sync | Idempotent (PATCH); safe to re-run; git-friendly |
| `kubectl create -f` | Imperative create | POST; fails with AlreadyExists on duplicates |
| `--dry-run=client -o yaml` | Scaffold a manifest | Build locally, don't send; pipe to a file |
| `kubectl explain TYPE.field` | Built-in schema docs | Discover valid fields without leaving the terminal |
| `kubectl diff -f` | Preview changes | Shows live-vs-file diff before you apply |

---

## When would I use this at work?

1. **Every deploy is a manifest.** Your `orders-api` Deployment, Service, ConfigMap, and HPA all live as YAML in git. Shipping a change = edit the YAML, open a PR, and let CI run `kubectl apply -f`. You'll read and write these files daily.

2. **Debugging "why won't it apply?"** A teammate's PR fails CI with "no matches for kind" or a selector mismatch. Because you understand the four-key skeleton and the validation steps, you spot the wrong `apiVersion` or mismatched labels in seconds instead of guessing.

3. **Scaffolding fast, then hardening.** You prototype with `kubectl create deployment … --dry-run=client -o yaml > deploy.yaml`, then edit that generated file — pin the image tag, add env vars, add probes — and commit it. You get the speed of imperative commands with the durability of declarative infrastructure-as-code.

---

## Connected topics

- **Study before:** Topic 29 (Kubernetes Architecture — what the API server and etcd are), Topic 30 (The Reconciliation Loop — how desired state in a manifest becomes actual state), Topic 31 (kubectl Basics — the client that ships your manifests to the API server).
- **Study after:** Topic 33 (Pods in Depth), Topic 34 (Deployments — the manifest you'll write most), Topic 36 (ConfigMaps and Secrets — more manifest kinds), Topic 38 (Labels and Selectors — the mechanic behind `selector`/`matchLabels`), Topic 49 (Helm — templating manifests so you don't hand-edit YAML per environment).
