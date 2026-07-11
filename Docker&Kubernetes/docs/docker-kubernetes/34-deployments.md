# 34 — Deployments
## Section: Kubernetes in Practice

## ELI5 — The Simple Analogy

Imagine you run a coffee shop and you always want **exactly 3 baristas** working the counter. You don't personally hire, watch, and replace each barista. Instead you write a rule on a whiteboard: *"Always keep 3 baristas on shift. They must all wear the v2 uniform."* Then you hire a **floor manager** whose only job is to keep reality matching that whiteboard. If a barista goes home sick, the manager calls in a replacement. If you change the whiteboard to say "v3 uniform," the manager swaps uniforms **one barista at a time** so the counter is never empty.

A **Deployment** is that whiteboard. It says "I want N copies of this Pod, running this image." A controller (the floor manager) constantly makes reality match it — replacing dead Pods, and rolling out new versions gently so your API is never fully down.

You stop babysitting individual Pods. You declare the *desired state* and let Kubernetes do the babysitting.

## The Linux kernel feature underneath

A Deployment, like a Pod (Topic 33), is **not** a kernel object. There is no syscall for it. It is pure software running the **reconciliation loop** (Topic 30) on top of etcd (Topic 29). But it's worth being precise about the mechanism, because "magic" is the enemy of debugging.

There are actually **two** controllers and **three** object types stacked up:

```
   You declare:            DEPLOYMENT  (desired: 3 replicas, image orders-api:1.4.0)
                                │
   Deployment controller manages ↓
                            REPLICASET  (a versioned snapshot: "3 pods of THIS template")
                                │
   ReplicaSet controller manages ↓
                            POD, POD, POD   (the actual running apartments, Topic 33)
```

Each controller is an infinite loop of the same three kernel-boring steps, running inside the `kube-controller-manager` process on the control plane:

```
loop forever:
    desired = read object from API server (backed by etcd)     # a WATCH, not a poll
    actual  = list the objects I own (via ownerReferences + labels)
    if actual != desired:
        make API calls to create/delete objects to close the gap
```

The **Deployment controller** watches Deployments. Its job: make sure the right *ReplicaSet* exists with the right template and replica count. When you change the image, it creates a **new** ReplicaSet and gradually scales the new one up while scaling the old one down.

The **ReplicaSet controller** watches ReplicaSets. Its job: make sure exactly N Pods matching its selector exist. If one dies, it creates another (a new Pod with a new name and new IP).

The glue that lets a controller know "which Pods are mine" is two things stored in etcd:
- **labels + selector** (Topic 38): the ReplicaSet's `selector.matchLabels` must match the Pod template's `labels`.
- **`ownerReferences`**: each Pod's metadata records "I am owned by ReplicaSet X" so garbage collection cleans up correctly.

That's the whole engine. No kernel feature — just watches on etcd and API calls, run in a loop.

## What is this?

A **Deployment** is a Kubernetes object that manages a set of identical Pods for you: it keeps a chosen number of replicas running, replaces ones that die, and rolls out new versions of your app with zero downtime (and lets you roll back). Under the hood it does this by creating and managing **ReplicaSets**, which in turn create the **Pods**.

It is the object you will use 90% of the time to run a stateless service like `orders-api`.

## Why does it matter for a backend developer?

Bare Pods (Topic 33) are mortal and unmanaged. For a real service you need answers to questions a Deployment solves for you:

- **"A node died at 3am — who brings my API back?"** Without a Deployment: nobody, your Pod is gone. With one: the ReplicaSet notices `actual=2, desired=3` and creates a replacement Pod on a healthy node, automatically.
- **"How do I ship v1.4.0 without downtime?"** A Deployment does a **rolling update**: start a new Pod, wait until it's Ready, kill an old one, repeat. Your users never see a gap.
- **"The new version is broken — get me back fast."** `kubectl rollout undo` flips back to the previous ReplicaSet in seconds, because the old ReplicaSet still exists (scaled to 0) in revision history.
- **"Traffic spiked — I need 10 copies."** Change `replicas: 3` to `replicas: 10` (or let an HPU/HPA do it, Topic 45). The controller creates 7 more Pods.

Not understanding Deployments means you'll `kubectl delete pod` by hand and be confused when it *instantly comes back* (the ReplicaSet recreated it). This confuses beginners constantly.

## The physical reality

When you apply a Deployment for `orders-api` with 3 replicas, here is what exists in **etcd** (nothing new on any node beyond the Pods themselves, Topic 33):

```
/registry/deployments/orders/orders-api
    → desired: replicas=3, template(image=orders-api:1.4.0), strategy=RollingUpdate
    → status:  observedGeneration, updatedReplicas, availableReplicas, conditions

/registry/replicasets/orders/orders-api-6c8f5b9d4      ← current ReplicaSet (revision 2)
    → replicas=3, selector matchLabels{app:orders-api, pod-template-hash:6c8f5b9d4}
/registry/replicasets/orders/orders-api-5f7a2c1e9      ← OLD ReplicaSet (revision 1), replicas=0
    → kept for rollback! scaled to zero, not deleted

/registry/pods/orders/orders-api-6c8f5b9d4-abcde       ← actual Pods, owned by the current RS
/registry/pods/orders/orders-api-6c8f5b9d4-fghij
/registry/pods/orders/orders-api-6c8f5b9d4-klmno
```

Two things to notice:

1. **`pod-template-hash`.** Kubernetes computes a hash of the Pod template and appends it to the ReplicaSet name and to the Pods' labels. This is how each ReplicaSet only "owns" its own generation of Pods, even though they all share `app: orders-api`.
2. **Old ReplicaSets are kept, scaled to 0.** That is your revision history / undo buffer. How many are kept is controlled by `revisionHistoryLimit` (default 10).

You can see this whole tree:

```bash
kubectl get deploy,rs,pod -n orders -l app=orders-api
```

## How it works — step by step

**Part A — initial rollout (create).**

1. `kubectl apply -f deploy.yaml`. API server validates and writes the Deployment to etcd. `phase`-equivalent status: 0 replicas available yet.
2. The **Deployment controller** (in kube-controller-manager) sees a Deployment with no matching ReplicaSet. It computes the `pod-template-hash`, creates ReplicaSet `orders-api-<hash>` with `replicas: 3`.
3. The **ReplicaSet controller** sees a ReplicaSet wanting 3 Pods but finding 0. It creates 3 Pod objects (Topic 33 lifecycle applies to each: scheduler assigns nodes, kubelets start them).
4. As each Pod passes its readiness probe (Topic 43), it counts toward `availableReplicas`. When `availableReplicas == 3`, the Deployment's condition `Available=True` and `Progressing=True (NewReplicaSetAvailable)`.

**Part B — rolling update (you change the image to 1.5.0).**

5. `kubectl set image deploy/orders-api web=orders-api:1.5.0 -n orders` (or edit the YAML and `apply`). The Deployment's Pod template changed → its `pod-template-hash` changes.
6. Deployment controller notices the current ReplicaSet's template no longer matches desired. It creates a **new** ReplicaSet `orders-api-<newhash>` with `replicas: 0` initially.
7. It now performs the **rolling update dance**, bounded by `maxSurge` and `maxUnavailable`:
   ```
   new RS: 0→1 (start 1 new Pod)   →   wait until it's Ready
   old RS: 3→2 (delete 1 old Pod)
   new RS: 1→2                      →   wait Ready
   old RS: 2→1
   new RS: 2→3                      →   wait Ready
   old RS: 1→0   (old ReplicaSet now empty but KEPT for rollback)
   ```
8. Throughout, at least `desired - maxUnavailable` Pods stay Ready, so the Service (Topic 35) always has healthy endpoints. Zero downtime.
9. The old ReplicaSet stays in etcd with `replicas: 0`. Revision counter increments (revision 2).

**Part C — rollback.**

10. `kubectl rollout undo deploy/orders-api -n orders`. The Deployment controller finds the previous ReplicaSet (still there, scaled to 0), and reverses the dance: scales *it* back up to 3 and the broken one down to 0. Fast, because the old Pods' template already exists.

## Exact syntax breakdown

Full `orders-api` Deployment manifest, every field annotated.

```yaml
apiVersion: apps/v1              # Deployments live in the "apps" API group, version v1
kind: Deployment                 # object type
metadata:
  name: orders-api               # Deployment name (also base for RS/Pod names)
  namespace: orders              # our running namespace
  labels:
    app: orders-api              # labels ON the Deployment object itself
spec:
  replicas: 3                    # DESIRED number of identical Pods
  revisionHistoryLimit: 5        # keep 5 old ReplicaSets for rollback (default 10)
  selector:                      # how this Deployment finds the Pods it owns
    matchLabels:
      app: orders-api            # MUST match template.metadata.labels below, exactly
  strategy:
    type: RollingUpdate          # RollingUpdate (default) or Recreate
    rollingUpdate:
      maxSurge: 1                # how many EXTRA Pods above replicas during rollout
      maxUnavailable: 0          # how many Pods may be missing during rollout (0 = safest)
  template:                      # the Pod TEMPLATE — this is a Pod spec (Topic 33)
    metadata:
      labels:
        app: orders-api          # labels stamped on every Pod; must satisfy the selector
    spec:
      containers:
        - name: web
          image: orders-api:1.4.0
          ports:
            - containerPort: 3000
          readinessProbe:        # gates whether a Pod counts as "available" (Topic 43)
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "500m", memory: "256Mi" }
```

Char-by-char on the two fields people get wrong.

```
  selector:
    matchLabels:
      app: orders-api
      │    └──────────── this value...
      └───────────────── ...and this key...
  template:
    metadata:
      labels:
        app: orders-api   ← ...MUST appear identically here, or apply is REJECTED.
```
The selector is *immutable* after creation and must be a subset of the template labels. If they don't match, the API server refuses the object with `selector does not match template labels`.

```
  strategy.rollingUpdate:
      maxSurge: 1
      │         └── during a rollout you may temporarily run replicas+1 = 4 Pods.
      └──────────── can be a number (1) or a percent ("25%").
      maxUnavailable: 0
      │               └── at no point may fewer than replicas-0 = 3 Pods be Ready.
      └───────────────── 0 = never dip below full capacity (needs surge room to work).
```

```
  revisionHistoryLimit: 5
   │                    └── keep the 5 most recent OLD ReplicaSets (each scaled to 0).
   └──────────────────────  older ones get garbage-collected; you can't roll back to them.
```

## Example 1 — basic

The smallest useful Deployment, then watch the controller keep it alive.

```yaml
# file: deploy-basic.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: orders
spec:
  replicas: 3                 # want 3 copies
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api       # matches the selector above
    spec:
      containers:
        - name: web
          image: node:20-alpine
          command: ["node","-e",
            "require('http').createServer((_,r)=>r.end('ok')).listen(3000)"]
          ports:
            - containerPort: 3000
```

```bash
kubectl apply -f deploy-basic.yaml
kubectl get deploy,rs,pod -n orders          # 1 deploy, 1 rs, 3 pods

# Now prove self-healing. Delete a Pod by hand:
kubectl delete pod -n orders -l app=orders-api --field-selector status.phase=Running | head -1
kubectl get pod -n orders -w                 # a NEW pod appears within seconds
#   → the ReplicaSet saw actual=2, desired=3, and created a replacement. You can't "win"
#     by deleting Pods; the controller recreates them.
```

## Example 2 — production scenario: zero-downtime deploy then emergency rollback

**The situation.** It's release day. `orders-api:1.5.0` adds a new `/discount` endpoint. You have 3 replicas behind a Service serving live traffic. You need to roll out with **zero downtime**, and if error rates spike, roll back in under a minute. Your strategy uses `maxUnavailable: 0` (never drop capacity) and `maxSurge: 1` (add one at a time), plus a readiness probe so a new Pod only takes traffic once `/healthz` passes.

```bash
# Current state: revision 1, image 1.4.0, 3 pods Ready.
kubectl rollout history deploy/orders-api -n orders
#   REVISION  CHANGE-CAUSE
#   1         initial

# 1. Trigger the rolling update. --record writes CHANGE-CAUSE for the history.
kubectl set image deploy/orders-api web=orders-api:1.5.0 -n orders
kubectl annotate deploy/orders-api -n orders \
  kubernetes.io/change-cause="ship 1.5.0 discount endpoint"

# 2. Watch the dance in real time.
kubectl rollout status deploy/orders-api -n orders
#   Waiting for deployment "orders-api" rollout to finish: 1 out of 3 new updated...
#   Waiting for deployment "orders-api" rollout to finish: 2 out of 3 new updated...
#   deployment "orders-api" successfully rolled out

# 3. See both ReplicaSets — old one now at 0, new one at 3.
kubectl get rs -n orders -l app=orders-api
#   NAME                    DESIRED   CURRENT   READY   AGE
#   orders-api-6c8f5b9d4    3         3         3       40s   ← new (1.5.0)
#   orders-api-5f7a2c1e9    0         0         0       9m    ← old (1.4.0), kept

# ---- Uh oh: monitoring shows 500s on /discount. Roll back. ----

# 4. One command flips back to the previous ReplicaSet.
kubectl rollout undo deploy/orders-api -n orders
kubectl rollout status deploy/orders-api -n orders
#   deployment "orders-api" successfully rolled out   (back on 1.4.0)

# 5. Confirm history: rollback created a NEW revision that re-uses the old template.
kubectl rollout history deploy/orders-api -n orders
#   REVISION  CHANGE-CAUSE
#   2         ship 1.5.0 discount endpoint
#   3         initial            ← rollback re-applied revision 1's template as revision 3
```

Why this is zero-downtime: with `maxUnavailable: 0`, Kubernetes first *adds* a new Ready Pod (surge to 4) before it removes an old one, so the Service (Topic 35) always has at least 3 healthy endpoints. The readiness probe ensures a new Pod is only counted (and only receives traffic) after `/healthz` succeeds — a Pod that fails to start never displaces a working one, and the rollout simply *stalls* instead of taking your API down.

If you'd used `strategy.type: Recreate` instead, Kubernetes would kill all 3 old Pods first, then start 3 new ones — a visible outage window. Use `Recreate` only when two versions truly cannot run at once (e.g. an incompatible schema lock).

## Common mistakes

**Mistake 1 — selector doesn't match template labels.**
```yaml
selector:  { matchLabels: { app: orders } }
template:  { metadata: { labels: { app: orders-api } } }   # mismatch!
```
```
The Deployment "orders-api" is invalid: spec.template.metadata.labels:
Invalid value: map[string]string{"app":"orders-api"}: `selector` does not match template `labels`
```
**Root cause:** the controller finds owned Pods by selector; if the template's Pods wouldn't match the selector, the Deployment could never "see" its own Pods. The API server rejects it upfront.
**Right:** make `selector.matchLabels` a subset of `template.metadata.labels`.

**Mistake 2 — deleting a Pod expecting it to stay gone.**
```bash
kubectl delete pod orders-api-6c8f5b9d4-abcde -n orders
kubectl get pods -n orders   # it's back with a new name!
```
**Root cause:** the ReplicaSet's reconciliation loop sees `actual < desired` and immediately creates a replacement. This is a feature (self-healing), not a bug.
**Right:** to actually reduce Pods, `kubectl scale deploy/orders-api --replicas=0` or delete the Deployment.

**Mistake 3 — `maxUnavailable: 1` with only 1 replica = downtime.**
```yaml
spec: { replicas: 1, strategy: { rollingUpdate: { maxUnavailable: 1, maxSurge: 0 } } }
```
During a rollout, the single Pod is removed before the new one is Ready → your API is down.
**Root cause:** with 1 replica and `maxUnavailable: 1`, Kubernetes is *allowed* to have 0 Pods available.
**Right:** run ≥2 replicas, or set `maxUnavailable: 0` and `maxSurge: 1` so a new Pod is added before the old is removed.

**Mistake 4 — rollout stuck forever because readiness never passes.**
```
kubectl rollout status ... hangs:
Waiting for deployment "orders-api" rollout to finish: 1 out of 3 new updated...
```
```bash
kubectl get pods -n orders   # new pod shows 0/1 READY, Running
```
**Root cause:** the new image's readiness probe (Topic 43) never succeeds (wrong path, wrong port, app not listening). With `maxUnavailable: 0` the rollout correctly refuses to proceed — it won't kill healthy old Pods to make room for broken new ones. Good! It protects you.
**Right:** fix the probe/app, or `kubectl rollout undo`. The stall is the safety net working.

**Mistake 5 — expecting `kubectl apply` of the same image to redeploy.**
```bash
kubectl apply -f deploy.yaml   # same file, nothing happens
```
**Root cause:** Deployments are declarative. If the Pod template is byte-for-byte identical, there's nothing to reconcile — no new ReplicaSet, no restart.
**Right:** to force a restart (e.g. to pick up a changed ConfigMap), use `kubectl rollout restart deploy/orders-api -n orders`, which bumps a template annotation and triggers a fresh rolling update.

## Hands-on proof

```bash
kubectl create namespace orders 2>/dev/null

# 1. Create a Deployment and watch the three-layer tree appear.
kubectl create deployment orders-api --image=nginx:1.25 --replicas=3 -n orders
kubectl get deploy,rs,pod -n orders
#   deployment.apps/orders-api        3/3
#   replicaset.apps/orders-api-xxxx   3
#   pod/orders-api-xxxx-....          Running   (x3)

# 2. Prove ownerReferences link Pod → ReplicaSet → Deployment.
POD=$(kubectl get pod -n orders -l app=orders-api -o name | head -1)
kubectl get $POD -n orders -o jsonpath='{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}{"\n"}'
#   → ReplicaSet/orders-api-xxxx

# 3. Do a rolling update and see a SECOND ReplicaSet born.
kubectl set image deploy/orders-api orders-api=nginx:1.27 -n orders
kubectl rollout status deploy/orders-api -n orders
kubectl get rs -n orders            # two ReplicaSets: new=3, old=0

# 4. Roll back and confirm you're on the old image again.
kubectl rollout undo deploy/orders-api -n orders
kubectl get rs -n orders            # counts flip: old=3, new=0
kubectl rollout history deploy/orders-api -n orders

# 5. Prove self-healing.
kubectl delete $POD -n orders
kubectl get pod -n orders           # a fresh Pod with a new name is already Running
```

## Practice exercises

### Exercise 1 — easy
Create an `orders-api` Deployment with 2 replicas of `node:20-alpine` running a tiny HTTP server. Scale it to 5 with `kubectl scale`, watch 3 new Pods appear, then scale back to 2 and watch 3 terminate. Confirm with `kubectl get rs` that it's still one ReplicaSet the whole time (scaling doesn't make a new RS — only template changes do).

### Exercise 2 — medium
Roll out an image change from `1.4.0` to `1.5.0` with `maxSurge: 1, maxUnavailable: 0`. In a second terminal run a tight loop of `kubectl get pods -n orders` (or `-w`) and record the exact sequence of Pod creations/deletions. Then `kubectl rollout undo` and observe how many revisions `kubectl rollout history` now shows and why the rollback added a *new* revision number.

### Exercise 3 — hard (production simulation)
Deploy `orders-api` with a `readinessProbe` on `/healthz`. Push a "broken" image whose `/healthz` returns 500. Run `kubectl rollout status` and observe it stall. Explain, using `maxUnavailable: 0`, why your live traffic stayed healthy the entire time. Now set a `progressDeadlineSeconds: 60` and observe the Deployment condition flip to `Progressing=False (ProgressDeadlineExceeded)` after a minute. Recover with `kubectl rollout undo` and verify all Pods are back to Ready.

## Mental model checkpoint

1. What three object types stack up when you create a Deployment, and which controller manages each link?
2. What is `pod-template-hash` and why does it exist?
3. Where does the old version "go" after a rolling update, and how does rollback use it?
4. Why does deleting a Pod managed by a Deployment bring it right back?
5. What do `maxSurge` and `maxUnavailable` control, and which combo gives strict zero-downtime?
6. Why does an identical `kubectl apply` do nothing, and how do you force a restart?
7. Why does a failing readiness probe *stall* a rollout instead of causing an outage (with `maxUnavailable: 0`)?

## Quick reference card

| Command / field | What it does | Key detail |
|---|---|---|
| `kind: Deployment` (`apps/v1`) | Manages replicas + rollouts of a Pod template | Creates ReplicaSets, which create Pods |
| `spec.replicas` | Desired number of Pods | Change it → controller scales; same RS stays |
| `spec.selector.matchLabels` | Which Pods this Deployment owns | Must be a subset of template labels; immutable |
| `spec.strategy.type` | `RollingUpdate` (default) or `Recreate` | Recreate = kill all then create all (downtime) |
| `maxSurge` / `maxUnavailable` | Rollout pacing | `surge:1,unavail:0` = strict zero-downtime |
| `revisionHistoryLimit` | Old ReplicaSets kept for rollback | Default 10; each is a scaled-to-0 RS |
| `kubectl rollout status` | Watch a rollout finish | Stalls (safely) if new Pods never become Ready |
| `kubectl rollout undo` | Roll back to previous revision | Fast — old ReplicaSet already exists |
| `kubectl rollout restart` | Force a fresh rolling update | Use to pick up ConfigMap/Secret changes |
| `kubectl scale --replicas=N` | Change replica count | Does NOT create a new ReplicaSet |

## When would I use this at work?

1. **Running any stateless service.** `orders-api`, an auth service, a worker with an HTTP health port — Deployments are the default way to run them with self-healing and N replicas.
2. **Shipping a new version safely.** `kubectl set image` + rolling update + readiness probe gives zero-downtime releases; `kubectl rollout undo` is your instant "oh no" button when a release misbehaves.
3. **Reacting to load or incidents.** Scale replicas up during a traffic spike (or let the HPA in Topic 45 do it), or `rollout restart` to cycle Pods that have gone stale or to reload injected config.

## Connected topics

- **Study before:** Topic 30 (reconciliation loop — the engine), Topic 32 (manifests), Topic 33 (Pods — what a Deployment ultimately creates), Topic 38 (labels & selectors — how ownership works).
- **Study after:** Topic 35 (Services — put a stable address in front of these ever-changing Pods), Topic 43 (probes — the readiness gate rollouts depend on), Topic 45 (HPA — automatic replica scaling), Topic 46 (rolling updates & rollbacks in even more depth).
