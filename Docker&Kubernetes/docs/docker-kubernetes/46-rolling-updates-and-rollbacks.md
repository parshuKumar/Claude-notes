# 46 — Rolling Updates and Rollbacks
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Imagine a restaurant with 4 waiters on the floor, all wearing the old blue uniform. The manager wants everyone in the new red uniform — but the restaurant **cannot close**. Customers are eating right now.

So the manager does it **one waiter at a time**. Pull waiter #1 off the floor, hand them a red uniform, but *only let them back on the floor once they're fully dressed and ready to serve*. Meanwhile the other 3 blue waiters keep taking orders. Once red waiter #1 is confirmed serving customers, pull waiter #2. Repeat until all 4 are red.

At no single moment did the restaurant have fewer than 3 working waiters. Customers never noticed. That is a **rolling update**: replace old pods with new pods *a few at a time*, never dropping below a safe number, and only counting a new pod as "working" once it says *"I'm ready."*

And a **rollback**? Halfway through, the manager notices the red uniforms have no pockets and waiters can't carry pens. Disaster. So the manager says "everyone back to blue" — and reverses the exact same swap, one at a time, back to the old uniform. That's `kubectl rollout undo`.

---

## The underlying mechanism (etcd + controllers, no direct kernel primitive)

A rolling update is **not** a kernel feature. It is a pure Kubernetes control-plane behavior, built entirely on the reconciliation loop from **Topic 30**. Let's name the real machinery underneath, because "Kubernetes does a rolling update" is exactly the kind of hand-wave this guide forbids.

The physical truth: a **Deployment** object in etcd owns one or more **ReplicaSet** objects, and each ReplicaSet owns some **Pods**. When you change the Deployment's pod template (e.g. a new image tag), here is what actually happens in etcd and in the controllers:

1. The **Deployment controller** (part of `kube-controller-manager`, Topic 29) is watching the Deployment key in etcd. It notices the pod template changed.
2. It computes a hash of the new pod template — the `pod-template-hash` — and looks for a ReplicaSet whose hash matches. None exists, so it **creates a brand-new ReplicaSet** with `replicas: 0`.
3. It then plays a slow tug-of-war: **scale the new ReplicaSet up by a bit, scale the old ReplicaSet down by a bit**, over and over, respecting two dials (`maxSurge`, `maxUnavailable`), until the new ReplicaSet has all the replicas and the old one has zero.
4. The old ReplicaSet is **not deleted** — it is kept at `replicas: 0` as a saved "revision" so you can roll back to it instantly.

So the deep reality: a rolling update is **two ReplicaSets existing at the same time**, one shrinking and one growing, coordinated by a controller that re-reads desired vs actual state every fraction of a second. There is no magic, just etcd records and a loop. We rely heavily on the readiness probe from **Topic 43** to decide when a new pod "counts."

---

## What is this?

A **rolling update** is how a Deployment replaces old pods with new ones gradually, so your `orders-api` never goes fully offline during a deploy. Kubernetes creates a new ReplicaSet, shifts pods over a few at a time, and waits for each new pod to pass its **readiness probe** before sending it traffic.

A **rollback** (`kubectl rollout undo`) reverses this by re-activating a previous ReplicaSet that Kubernetes kept around as a saved revision — restoring the old version without you having to remember the old image tag.

---

## Why does it matter for a backend developer?

You push `orders-api:1.5.0` on a Friday. It has a bug that crashes on startup. What happens next depends entirely on whether you understand rolling updates:

- **If you understand it:** you configured a readiness probe, so the broken v1.5.0 pods never pass readiness, never receive traffic, and the rollout **automatically stalls** with your old v1.4.0 pods still serving every customer. You type `kubectl rollout undo` and you're safe. Zero customer impact.
- **If you don't:** you had no readiness probe, so Kubernetes assumed the crashing pods were "ready" the instant the container started, sent live customer traffic to them, they 500'd, and you took a production outage — while also having killed your good old pods because nothing stopped the rollout.

Without this knowledge you will:
- Cause outages on every deploy because there's nothing gating traffic to unready pods.
- Not understand why `kubectl rollout status` "hangs" (it's correctly waiting for pods that will never become ready).
- Panic during an incident because you don't know `kubectl rollout undo` exists or how revision history works.
- Set `maxUnavailable: 100%` "to make deploys faster" and cause a total blackout every release.

This is the single most-used operation in your week-to-week life as a backend dev on Kubernetes. You deploy constantly. This is *how* deploys happen.

---

## The physical reality

When you run a rolling update on `orders-api` in namespace `orders`, here is what **actually exists** in the cluster at the halfway point.

**1. Two ReplicaSets, both alive, in etcd.** Query them:

```
$ kubectl get rs -n orders
NAME                    DESIRED   CURRENT   READY   AGE
orders-api-6c9f8b7d4    2         2         2       20m   ← OLD (v1.4.0), shrinking
orders-api-7f4d9c2a1    2         2         2       15s   ← NEW (v1.5.0), growing
```

Both ReplicaSets are real objects with real etcd keys:

```
/registry/replicasets/orders/orders-api-6c9f8b7d4   ← old revision, kept for rollback
/registry/replicasets/orders/orders-api-7f4d9c2a1   ← new revision
```

**2. A `pod-template-hash` label on every pod.** That gibberish suffix (`6c9f8b7d4`) is not random — it's a hash of the pod template. Look:

```
$ kubectl get pods -n orders --show-labels
NAME                          READY   STATUS    LABELS
orders-api-6c9f8b7d4-abcde    1/1     Running   app=orders-api,pod-template-hash=6c9f8b7d4
orders-api-7f4d9c2a1-fghij    1/1     Running   app=orders-api,pod-template-hash=7f4d9c2a1
```

This label is how each ReplicaSet knows which pods are "its own" (Topic 38 — Labels and Selectors).

**3. Revision annotations on each ReplicaSet.** Kubernetes stamps a revision number so it can order them for rollback:

```
$ kubectl get rs orders-api-6c9f8b7d4 -n orders -o jsonpath='{.metadata.annotations}'
{"deployment.kubernetes.io/revision":"7", ...}
```

**4. The Deployment's status, tracking the swap live.** In etcd the Deployment object carries a `status` block the controller updates continuously:

```
$ kubectl get deploy orders-api -n orders -o jsonpath='{.status}'
{"replicas":4,"updatedReplicas":2,"readyReplicas":4,"availableReplicas":4, ...}
#              │                   │
#              │                   └─ 4 pods total are Ready (2 old + 2 new)
#              └─ 2 pods run the NEW template so far
```

So the "physical reality" of a rolling update is: **two ReplicaSet records, a bunch of pods each tagged with which template they came from, and a Deployment status field being rewritten many times a second** as the controller drives the numbers.

---

## How it works — step by step

Full trace of `kubectl set image deployment/orders-api api=orders-api:1.5.0 -n orders` with `replicas: 4`, `maxSurge: 1`, `maxUnavailable: 1`, and a working readiness probe.

1. **kubectl PATCHes the Deployment.** The `set image` command sends an HTTP PATCH to the API server, changing `spec.template.spec.containers[0].image` from `orders-api:1.4.0` to `orders-api:1.5.0`. The API server writes the new Deployment object to etcd. That's the *only* thing you directly caused.

2. **The Deployment controller wakes up.** It has a watch on the Deployment. etcd notifies it: "this object changed." The controller reads the new desired state.

3. **It hashes the new pod template.** It computes `pod-template-hash=7f4d9c2a1` and searches existing ReplicaSets for that hash. Not found → this is a genuinely new revision.

4. **It creates a new ReplicaSet at `replicas: 0`.** `orders-api-7f4d9c2a1` is born in etcd, desired 0 pods, stamped `deployment.kubernetes.io/revision: 8`.

5. **It computes the surge/unavailable budget.** With 4 replicas, `maxSurge: 1` → it may run up to **5** pods total. `maxUnavailable: 1` → at least **3** pods must always be *available*. So the safe operating window is 3–5 pods.

6. **First move: surge up the new RS.** It scales `orders-api-7f4d9c2a1` from 0 → 1. Now: 4 old + 1 new = 5 pods (at the surge ceiling). The new pod's container starts.

7. **The readiness gate holds traffic back.** The new pod is `Running` but `0/1 READY`. The kubelet runs its readiness probe (Topic 43). Until it passes, the Service's `EndpointSlice` (Topic 35) does **not** include this pod, so **no customer traffic reaches it.** This is the entire mechanism of zero-downtime.

8. **New pod becomes Ready.** Readiness probe returns 200. The endpoints controller adds the pod's IP to the Service's EndpointSlice. kube-proxy updates iptables/IPVS. *Now* traffic flows to the new pod. availableReplicas is 5.

9. **Now it's safe to remove an old one.** Because 5 available ≥ 4, the controller scales the **old** RS `orders-api-6c9f8b7d4` from 4 → 3. One old pod gets a SIGTERM, drains, and terminates. Available drops to 4 (still ≥ 3, safe).

10. **Repeat the dance.** Surge new to 2, wait for Ready, scale old to 2. Surge new to 3, wait, scale old to 1. Surge new to 4, wait, scale old to 0.

11. **Convergence.** New RS = 4/4 ready, old RS = 0/0. The Deployment's `status.updatedReplicas == status.replicas == 4`. The controller marks the Deployment `Progressing → Complete`. `kubectl rollout status` prints `successfully rolled out` and exits 0.

12. **The old RS is kept.** `orders-api-6c9f8b7d4` stays in etcd at `replicas: 0`. It costs nothing (no pods) but preserves the full old template for instant rollback. How many old RSes are kept is governed by `revisionHistoryLimit` (default 10).

**The critical branch:** if at step 8 the new pod *never* becomes Ready (bad image, crash loop, failing probe), step 9 **never fires**. The controller never scales the old RS down past the `maxUnavailable` floor. Your old pods keep serving. The rollout sits stuck — which is a *feature*, not a bug. It's Kubernetes refusing to break production.

---

## Exact syntax breakdown

The Deployment strategy block that controls all of this:

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

```
spec:
  replicas: 4
  │        │
  │        └─ desired steady-state pod count. The surge/unavailable math is relative to THIS.
  │
  strategy:
    type: RollingUpdate          ← the default. The alternative is "Recreate" (kill all, then start all — causes downtime)
    rollingUpdate:
      maxSurge: 1
      │        │
      │        └─ how many pods ABOVE `replicas` may exist during the update.
      │           1 → up to 5 pods total. Can be a % (e.g. 25% of 4 = 1, rounded UP).
      │           Higher = faster rollout, more temporary resource use.
      │
      maxUnavailable: 1
               │
               └─ how many pods BELOW `replicas` may be unavailable during the update.
                  1 → at least 3 must always be available. Can be a % (rounded DOWN).
                  Set to 0 for the safest (never drop capacity) but you MUST have maxSurge ≥ 1 then.
```

The `Recreate` alternative (know it exists, rarely want it):

```
strategy:
  type: Recreate     ← terminates ALL old pods, THEN creates new ones. Guarantees downtime.
                       Use only when two versions truly cannot run at once (e.g. incompatible DB schema lock).
```

The commands you'll actually type:

```
kubectl set image deployment/orders-api api=orders-api:1.5.0 -n orders
│       │         │                     │   │                 │
│       │         │                     │   │                 └─ namespace (Topic 37)
│       │         │                     │   └─ new image:tag to roll out
│       │         │                     └─ container NAME inside the pod (must match spec.template...containers[].name)
│       │         └─ resource: the Deployment named orders-api
│       └─ set image: shortcut that PATCHes just the image field (triggers a rolling update)
└─ kubectl

kubectl rollout status deployment/orders-api -n orders
│       │       │      │
│       │       │      └─ which Deployment to watch
│       │       └─ status: block and stream progress until rollout completes or fails; exit code reflects success
│       └─ rollout: the subcommand family for managing Deployment revisions
└─ kubectl

kubectl rollout history deployment/orders-api -n orders
│               │
│               └─ history: list every saved revision (number + change-cause) so you know what to roll back to

kubectl rollout undo deployment/orders-api -n orders --to-revision=7
│               │                                     │
│               │                                     └─ roll back to a SPECIFIC revision.
│               │                                        Omit it to go to the immediately previous revision.
│               └─ undo: re-activate an old ReplicaSet (a rolling update in reverse)

kubectl rollout pause deployment/orders-api -n orders     ← freeze mid-rollout (for canary-style checking)
kubectl rollout resume deployment/orders-api -n orders    ← continue a paused rollout
kubectl rollout restart deployment/orders-api -n orders   ← roll all pods (same image) — e.g. to pick up a new Secret
```

---

## Example 1 — basic

Deploy `orders-api`, then roll it forward and back. Every line commented.

```yaml
# orders-api-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: orders
spec:
  replicas: 4                    # steady state: 4 pods serving traffic
  revisionHistoryLimit: 5        # keep the last 5 old ReplicaSets for rollback (default is 10)
  selector:
    matchLabels:
      app: orders-api            # this Deployment owns pods with this label (Topic 38)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                # allow 1 extra pod during the swap (up to 5)
      maxUnavailable: 0          # never drop below 4 available — safest for customer traffic
  template:
    metadata:
      labels:
        app: orders-api          # pods get this label; must match selector above
    spec:
      containers:
        - name: api              # container name — this is what `set image` targets
          image: orders-api:1.4.0
          ports:
            - containerPort: 3000
          readinessProbe:        # THE gate. No traffic until this passes. (Topic 43)
            httpGet:
              path: /healthz     # your Express route that returns 200 when ready to serve
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 5
```

```bash
# Apply it. --record stamps the command into the revision's change-cause (deprecated but still handy;
# alternatively annotate manually as shown below).
kubectl apply -f orders-api-deploy.yaml -n orders

# Watch the initial rollout complete.
kubectl rollout status deployment/orders-api -n orders
# waiting for deployment "orders-api" rollout to finish: 0 of 4 updated replicas are available...
# deployment "orders-api" successfully rolled out

# Now roll forward to v1.5.0 and record WHY (the change-cause).
kubectl set image deployment/orders-api api=orders-api:1.5.0 -n orders
kubectl annotate deployment/orders-api -n orders \
  kubernetes.io/change-cause="upgrade to 1.5.0 - new discount engine"

# Watch the rolling update swap ReplicaSets one pod at a time.
kubectl rollout status deployment/orders-api -n orders

# See the revision history.
kubectl rollout history deployment/orders-api -n orders
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         upgrade to 1.5.0 - new discount engine

# Oops — v1.5.0 is bad. Roll back to the previous revision (v1.4.0).
kubectl rollout undo deployment/orders-api -n orders

# Confirm we're back on 1.4.0.
kubectl get deployment orders-api -n orders -o jsonpath='{.spec.template.spec.containers[0].image}'
# orders-api:1.4.0
```

Notice the rollback was **one command** and you never had to remember the old image tag — Kubernetes kept the old ReplicaSet and just re-activated it.

---

## Example 2 — production scenario

**The situation.** It's 2pm Tuesday. You deploy `orders-api:1.5.0`, which accidentally ships a bad `DATABASE_URL` default and **crashes on startup** with `ECONNREFUSED`. You have 6 replicas, `maxSurge: 2`, `maxUnavailable: 0`, and — critically — a readiness probe hitting `/healthz`, which only returns 200 *after* the DB connection pool is established.

Here's the play-by-play of what Kubernetes does **for** you:

```bash
kubectl set image deployment/orders-api api=orders-api:1.5.0 -n orders
kubectl rollout status deployment/orders-api -n orders
# Waiting for deployment "orders-api" rollout to finish: 2 out of 6 new replicas have been updated...
# ...and it just sits here. Forever. Good.
```

In another terminal:

```bash
kubectl get pods -n orders
# NAME                          READY   STATUS             RESTARTS   AGE
# orders-api-6c9f8b7d4-aaaa     1/1     Running            0          40m   ← OLD, healthy, serving
# orders-api-6c9f8b7d4-bbbb     1/1     Running            0          40m   ← OLD, healthy, serving
# orders-api-6c9f8b7d4-cccc     1/1     Running            0          40m   ← OLD (6 old still up!)
# orders-api-6c9f8b7d4-dddd     1/1     Running            0          40m
# orders-api-6c9f8b7d4-eeee     1/1     Running            0          40m
# orders-api-6c9f8b7d4-ffff     1/1     Running            0          40m
# orders-api-7f4d9c2a1-gggg     0/1     CrashLoopBackOff   4          2m    ← NEW, broken, NO traffic
# orders-api-7f4d9c2a1-hhhh     0/1     CrashLoopBackOff   4          2m    ← NEW, broken, NO traffic
```

**Read what happened.** Because `maxUnavailable: 0`, the controller was **forbidden** from removing a single old pod until a new pod became available. The new pods never became Ready (readiness probe fails — DB refuses). So the old ReplicaSet stayed at 6/6, the new one got stuck at 2 broken pods (the `maxSurge: 2` ceiling), and **every customer request kept hitting the healthy v1.4.0 pods.** Your dashboards show zero error rate. Nobody paged.

The fix is one line, and it's instant because the old ReplicaSet is still fully staffed:

```bash
kubectl rollout undo deployment/orders-api -n orders
# deployment.apps/orders-api rolled back
# The 2 broken new pods are deleted, new RS scales to 0, you're 100% on 1.4.0 again.
```

**The lesson.** The two things that saved you were (1) a **real readiness probe** that only passes when the app can actually serve, and (2) `maxUnavailable: 0` so a failed rollout can never subtract capacity. Together they turn a would-be outage into a non-event. This is *the* pattern to internalize.

---

## Common mistakes

**Mistake 1 — No readiness probe, so broken pods get traffic.**

You deploy a version that starts its process fine but isn't actually ready (DB pool still connecting). With no readiness probe, Kubernetes treats "container started" as "ready" and immediately routes customers to it.

```
# Users see:
HTTP 502 Bad Gateway
# or
{"error":"ECONNREFUSED 10.0.4.12:5432"}
```

Root cause: without a readiness probe, the pod's IP is added to the Service EndpointSlice the moment the container is `Running`, regardless of whether your app can serve. The rolling update also happily scales down old pods because new ones look "available." Wrong: no probe. Right: a readiness probe that returns 200 **only** when the app can truly handle a request (DB connected, migrations checked, caches warm).

**Mistake 2 — `maxUnavailable: 100%` or high, killing all capacity.**

Someone sets it to speed up deploys:

```yaml
rollingUpdate:
  maxUnavailable: 100%   # "make it fast"
```

Result during every deploy:

```
$ kubectl get deploy orders-api -n orders
NAME         READY   UP-TO-DATE   AVAILABLE
orders-api   0/6     6            0          ← ZERO available pods = full outage window
```

Root cause: `maxUnavailable: 100%` lets the controller remove *all* old pods before new ones are Ready, so there's a window with zero serving pods. Wrong: 100%. Right: `maxUnavailable: 0` (or a small number/percent) with `maxSurge ≥ 1` so you always keep capacity.

**Mistake 3 — Both `maxSurge: 0` and `maxUnavailable: 0`.**

```
$ kubectl apply -f deploy.yaml
The Deployment "orders-api" is invalid: spec.strategy.rollingUpdate.maxUnavailable:
Invalid value: intstr.IntOrString{...}: may not be 0 when maxSurge is 0
```

Root cause: if you can't add a pod (surge 0) *and* can't remove a pod (unavailable 0), the controller has no legal move to make progress — a deadlock. Kubernetes rejects it at admission. Right: at least one of the two must be non-zero.

**Mistake 4 — Expecting `rollout undo` to restore data or Secrets.**

`kubectl rollout undo` only restores the **pod template** (image, env, resources) from the old ReplicaSet. It does **not** revert database migrations, ConfigMaps, or Secrets you changed separately.

```
# You rolled back the image, but the DB schema is still the migrated v1.5.0 schema → app errors.
ERROR: column "legacy_status" does not exist
```

Root cause: revision history is *only* the Deployment's pod template, nothing else in the cluster. Right: make schema migrations **backward-compatible** (expand/contract pattern) so an image rollback still works against the migrated DB.

**Mistake 5 — `kubectl set image` typo in the container name.**

```
$ kubectl set image deployment/orders-api orders-api=orders-api:1.5.0 -n orders
error: unable to find container named "orders-api"
```

Root cause: `set image` targets the **container name** (`api` in our manifest), not the Deployment name or image name. They're often different. Right: check `kubectl get deploy orders-api -n orders -o jsonpath='{.spec.template.spec.containers[*].name}'` first.

---

## Hands-on proof

Run these right now (a `kind`/`minikube`/any cluster works). We'll use a public image so you don't need to build anything, and watch two ReplicaSets exist at once.

```bash
kubectl create namespace orders 2>/dev/null

# 1. Create a Deployment with 4 replicas of nginx as a stand-in for orders-api.
kubectl create deployment orders-api --image=nginx:1.25 --replicas=4 -n orders

# 2. Set an explicit rolling strategy so the surge/unavailable is visible.
kubectl patch deployment orders-api -n orders -p \
  '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0},"type":"RollingUpdate"}}}'

# 3. Watch the rollout. In one terminal, keep this running:
kubectl get rs -n orders -w &

# 4. Trigger a rolling update by changing the image.
kubectl set image deployment/orders-api nginx=nginx:1.26 -n orders
kubectl annotate deployment/orders-api -n orders kubernetes.io/change-cause="bump to 1.26"

# 5. Watch: you'll SEE two ReplicaSets — old shrinking, new growing.
kubectl rollout status deployment/orders-api -n orders

# 6. Prove the old ReplicaSet is kept at 0 replicas (for rollback).
kubectl get rs -n orders
#   NAME                    DESIRED   CURRENT   READY
#   orders-api-<oldhash>    0         0         0        ← kept, ready to roll back to
#   orders-api-<newhash>    4         4         4

# 7. Look at the revision history.
kubectl rollout history deployment/orders-api -n orders

# 8. Roll back and confirm the image reverts to 1.25 in ONE command.
kubectl rollout undo deployment/orders-api -n orders
kubectl get deploy orders-api -n orders -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
#   nginx:1.25

# 9. Clean up.
kill %1 2>/dev/null
kubectl delete namespace orders
```

What you verified: a rolling update creates a **second ReplicaSet** (step 5), keeps the **old one at 0** for rollback (step 6), records **numbered revisions** (step 7), and `undo` **re-activates the previous revision** without you supplying the old tag (step 8).

---

## Practice exercises

### Exercise 1 — easy
Create `orders-api` from `nginx:1.25` with 3 replicas in namespace `orders`. Run `kubectl rollout history` — how many revisions exist? Now `kubectl set image` to `nginx:1.26` and check history again. Explain in one sentence why revision 2 appeared and what object was created behind it.

### Exercise 2 — medium
Set `maxSurge: 0` and `maxUnavailable: 1` on your Deployment, then roll to a new image while running `kubectl get pods -n orders -w`. Count: at any moment, how many pods are `Running`? Now flip to `maxSurge: 1, maxUnavailable: 0` and roll again. Describe the difference in pod count during the two rollouts and which one is safer for a customer-facing API and why.

### Exercise 3 — hard (production simulation)
Simulate a failed deploy. Roll `orders-api` to an image that does not exist (`nginx:9.9.9-does-not-exist`). Run `kubectl rollout status` in one terminal (it will hang) and `kubectl get pods -n orders` in another (you'll see `ImagePullBackOff` on new pods while old pods stay `Running`). Prove that (a) no old pod was removed because `maxUnavailable: 0`, and (b) `kubectl rollout undo` fully restores service. Then write down the two configuration choices that made this a non-outage.

---

## Mental model checkpoint

Answer from memory:

1. During a rolling update, how many ReplicaSets exist and what is each one's role?
2. What single mechanism prevents customer traffic from reaching a new-but-not-yet-ready pod?
3. What do `maxSurge` and `maxUnavailable` each control, relative to `replicas`?
4. Why does a rollout with `maxUnavailable: 0` **stall** (rather than break prod) when the new image is broken?
5. Where does the old version "live" after a successful rollout, and what makes rollback instant?
6. What does `kubectl rollout undo` restore — and what does it NOT restore?
7. Why is `maxSurge: 0` + `maxUnavailable: 0` rejected by the API server?

---

## Quick reference card

| Command / field | What it does | Key detail |
|---|---|---|
| `strategy.type: RollingUpdate` | Replace pods gradually (default) | Alternative `Recreate` causes downtime |
| `maxSurge` | Extra pods allowed above `replicas` | Number or %; rounded **up** |
| `maxUnavailable` | Pods allowed unavailable below `replicas` | Number or %; rounded **down** |
| `readinessProbe` | Gates traffic to a pod | The engine of zero-downtime; no probe = broken deploys |
| `revisionHistoryLimit` | How many old ReplicaSets to keep | Default 10; each old RS = 0 pods, cheap |
| `kubectl set image deploy/X c=img:tag` | Trigger a rolling update | Targets **container** name `c`, not deploy name |
| `kubectl rollout status deploy/X` | Stream progress; exit code = success | "Hangs" = correctly waiting for readiness |
| `kubectl rollout history deploy/X` | List revisions + change-cause | Annotate `kubernetes.io/change-cause` to fill it |
| `kubectl rollout undo deploy/X` | Roll back to previous revision | Re-activates old ReplicaSet; template only |
| `kubectl rollout undo --to-revision=N` | Roll back to a specific revision | Get N from `rollout history` |
| `kubectl rollout restart deploy/X` | Roll all pods, same image | Picks up new Secret/ConfigMap values |
| `kubectl rollout pause/resume deploy/X` | Freeze/continue mid-rollout | Enables manual canary checks |

---

## When would I use this at work?

1. **Every single deploy.** Shipping a new `orders-api` version is a rolling update whether you think about it or not. Understanding it means you set `maxUnavailable`/`maxSurge` and readiness probes correctly so releases are boring and safe.

2. **Firefighting a bad release.** A version is 500ing in prod. You `kubectl rollout undo` to restore the last-known-good version in seconds — no rebuild, no remembering the old tag, no redeploy pipeline — because the old ReplicaSet is still there.

3. **Rolling a config/Secret change.** You rotated the DB password in a Secret (Topic 36) but running pods still hold the old env value. `kubectl rollout restart deployment/orders-api` cycles all pods through a fresh rolling update so they pick up the new Secret, with zero downtime.

---

## Connected topics

- **Study before:**
  - **Topic 34 — Deployments**: the object that owns ReplicaSets; this topic is the deep dive on *how it updates* them.
  - **Topic 30 — The Reconciliation Loop**: the controller mechanics that drive the surge/shrink dance.
  - **Topic 43 — Health Checks in Kubernetes**: the readiness probe is the gate that makes zero-downtime possible.
  - **Topic 35 — Services**: EndpointSlices and kube-proxy are what actually route traffic to Ready pods.
- **Study after:**
  - **Topic 45 — Horizontal Pod Autoscaler**: how `replicas` changes automatically, and how that interacts with rollouts.
  - **Topic 49 — Helm Basics**: templating deploys and using `helm rollback`, which wraps this mechanism.
  - **Topic 52 — CI/CD with Kubernetes**: automating `set image` + `rollout status` in a pipeline, and GitOps rollbacks.
