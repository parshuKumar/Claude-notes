# 30 — The Reconciliation Loop

## Section: Kubernetes Foundations

---

## ELI5 — The Simple Analogy

Imagine a **thermostat** on your wall. You set it to **21°C**. That number is
your *wish* — the temperature you want the room to be. The thermostat itself
doesn't know or care *why* the room is cold. It just does one dumb, endless
thing, over and over, forever:

```
loop forever:
    wanted   = 21°C          (what you asked for)
    actual   = read the room thermometer
    if actual < wanted:  turn the heater ON
    if actual > wanted:  turn the heater OFF
    wait a moment, then check again
```

Open a window and cold air rushes in — the thermostat notices the room dropped
and fires the heater. Close the window — it notices the room is warm enough and
stops. You never *told* it "a window opened, react!" It simply keeps comparing
**wanted vs actual** and nudges reality toward the wish. It doesn't need to know
the history of *how* the room got cold; it only looks at the room *right now*.

**That is the single most important idea in all of Kubernetes.** Every piece of
Kubernetes behavior — self-healing, scaling, rolling updates, load balancing — is
a fleet of these thermostats. Each one watches one kind of object, compares your
wish (**desired state**) to the world (**actual state**), and takes one small
action to close the gap. Then it does it again. Forever.

Once you truly get the thermostat, you understand Kubernetes.

---

## The Linux kernel feature underneath

There is **no kernel primitive** for reconciliation — it is a pure software
pattern. But it rides on two very concrete mechanisms in the API server + etcd,
and it's worth being precise because the details explain how it's efficient and
correct.

**1. The `watch` — how a controller "sees" changes without hammering the DB.**

A controller does not sit in a tight loop querying "any changes? any changes?".
That would melt the API server. Instead it uses the Kubernetes **watch**
mechanism, which underneath is one long-lived HTTP connection:

```
Controller ──── HTTP GET /apis/apps/v1/deployments?watch=true&resourceVersion=X ──► API server
           ◄─── stream of events: ADDED / MODIFIED / DELETED (chunked, kept open)
```

- The kernel side: the API server holds thousands of these open TCP sockets and
  uses **epoll** to know which have data to push. etcd itself has a native
  `Watch` (gRPC stream) that the API server consumes and fans out to all
  interested controllers.
- `resourceVersion` is a monotonic counter from etcd. It lets a controller say
  "give me everything that changed *after* version X," so no event is missed even
  across reconnects.

**2. `list` + `watch` = the informer, the controller's local cache.**

Every controller first does a **LIST** (get the full current state once) to build
an in-memory cache, then a **WATCH** to keep that cache updated incrementally:

```
┌─────────────────────────────────────────────────────────────────────┐
│  A controller's "informer" (the standard building block)             │
│                                                                       │
│   1. LIST all Deployments  ───► fill local cache (a map in RAM)       │
│   2. WATCH for changes     ───► keep cache in sync via ADDED/MOD/DEL  │
│   3. On any change, enqueue the object's key into a work queue        │
│   4. A worker pops the key and runs reconcile(key)  ← the thermostat  │
│                                                                       │
│  The reconcile function reads DESIRED from cache, reads ACTUAL from   │
│  the cluster, and issues API calls to close the gap. Then returns.    │
└─────────────────────────────────────────────────────────────────────┘
```

So "underneath," reconciliation is: **etcd's monotonic versioning + a gRPC/HTTP
watch stream + an in-memory cache + a work queue + a compare-and-act function.**
No syscalls of its own — just disciplined use of the API server. The kernel work
(clone/cgroups) happens far downstream when a kubelet finally acts on the pod the
loop created.

---

## What is this?

The reconciliation loop is Kubernetes' universal control pattern: a **controller**
continuously compares the **desired state** (what your YAML asked for, stored in
etcd) against the **actual state** (what is really running) and takes actions to
make actual match desired. It is **level-triggered** — it looks at the current
level of the world and corrects it, rather than reacting once to an edge/event.
Every controller in Kubernetes is a variation of this one loop.

---

## Why does it matter for a backend developer?

This one concept dissolves almost every "why did Kubernetes do that?" question:

- **"I deleted a pod but it came back!"** Of course — a controller's desired state
  still says "3 replicas," it saw actual drop to 2, and it created a replacement.
  You didn't change the *wish*, only reality; the loop corrected reality.
- **"I manually edited a pod the Deployment owns and my change vanished."** The
  controller reconciled the pod back to match the Deployment template. Desired
  wins; your manual drift got overwritten.
- **"My rolling update is slow / paused."** A controller is stepping actual toward
  desired a few pods at a time (respecting `maxSurge`/`maxUnavailable`), checking
  readiness at each step. It's the loop pacing itself.
- **"I applied the same YAML twice and nothing bad happened."** Because
  reconciliation is **idempotent** — if actual already equals desired, the loop
  does nothing. This is why `kubectl apply` is safe to re-run and why GitOps
  (Topic 52) works at all.

Without this model, Kubernetes feels like an unpredictable ghost that undoes your
changes. With it, Kubernetes becomes obvious and even boring: **it is always,
only, trying to make the world match the written wish.** As a backend dev, this
is the mental key that makes everything else in Kubernetes click.

---

## The physical reality

Reconciliation is not a thing you can `cat` on disk — it's running code plus two
data snapshots. But every part of it is observable:

```
DESIRED STATE  (what you want)         ACTUAL STATE  (what is)
──────────────────────────────         ──────────────────────────────
lives in etcd as object.spec           lives partly in etcd as object.status
  /registry/deployments/orders/          (reported by controllers/kubelet)
    orders-api  → spec.replicas: 3     and partly in the real world:
                                         actual containers on worker nodes,
  read it:                               reported UP via the kubelet.
  kubectl get deploy orders-api \        read it:
    -o jsonpath='{.spec.replicas}'       kubectl get deploy orders-api \
                                           -o jsonpath='{.status.replicas}'

THE CONTROLLER  (the loop itself)
──────────────────────────────
  a goroutine inside kube-controller-manager (a pod in kube-system):
    kubectl get pods -n kube-system | grep controller-manager

  its work queue + informer caches live in that process's RAM.
  every action it takes appears as an EVENT you can see:
    kubectl get events -n orders --sort-by=.lastTimestamp
    #  ScalingReplicaSet   Scaled up replica set orders-api-xyz to 3
```

The crucial on-disk truth: **`spec` and `status` are two different sub-objects of
the same record in etcd.** You write `spec`. Controllers write `status`. The loop
exists to drag the world (and `status`) toward `spec`. When you run
`kubectl get`, the columns you see (`READY 3/3`) are literally
`status` compared to `spec`, rendered for you.

---

## How it works — step by step

Let's trace one controller — the **ReplicaSet controller** — reconciling
`orders-api` at 3 replicas, then handling a pod death. This is the loop in full.

```
DESIRED (etcd):  ReplicaSet orders-api  spec.replicas = 3
ACTUAL:          0 pods running (fresh)

────────────────────── RECONCILE PASS #1 ──────────────────────
 1. The controller's informer already LISTed + is WATCHing ReplicaSets
    and Pods, so it has a fresh local cache.
 2. An event fires (RS created). The controller enqueues key
    "orders/orders-api".
 3. A worker pops the key and calls reconcile("orders/orders-api"):
      a. Read DESIRED from cache:  spec.replicas = 3
      b. Compute ACTUAL: count Pods whose ownerRef = this RS AND that
         are not terminating  →  0
      c. diff = desired(3) - actual(0) = +3
      d. Because diff > 0, create 3 Pod objects (via the API server).
         NOTE: it does NOT wait for them to run. It just creates the
         objects and RETURNS. (The scheduler + kubelet take over,
         Topic 29 steps 6–10.)
 4. Creating pods triggers new WATCH events → the key gets re-enqueued.

────────────────────── RECONCILE PASS #2 ──────────────────────
 5. reconcile runs again:
      a. desired = 3
      b. actual  = 3 (the pods now exist)
      c. diff = 0  →  DO NOTHING. Return. (Idempotent — steady state.)

... time passes, the loop keeps running, always finding diff = 0 ...

────────────────────── A POD DIES ──────────────────────
 6. A node crashes; one pod is lost. The API server records the pod as
    gone → a DELETED watch event fires → key re-enqueued.
 7. reconcile runs:
      a. desired = 3
      b. actual  = 2
      c. diff = +1  →  create 1 new Pod object. Return.
 8. Scheduler places it, a kubelet starts it, actual → 3.
 9. reconcile runs once more: diff = 0 → nothing. Back to steady state.
```

Now internalize the two properties that make this robust:

**Level-triggered, not edge-triggered.** The controller does **not** say "a
DELETE event happened, so create exactly one pod." It says "let me look at the
world *as it is now* — I see 2, I want 3, so I create the difference." Why does
this matter? Suppose the controller was asleep/restarted and *missed* three
separate delete events. An edge-triggered system would be hopelessly out of sync
(it reacted to 0 of 3 edges). A level-triggered system just wakes up, counts
what's actually there (say 0), compares to desired (3), and creates 3. **It
self-corrects from any state, even after missing events or crashing.** This is
why Kubernetes is so resilient.

```
 EDGE-TRIGGERED (fragile)          LEVEL-TRIGGERED (robust — Kubernetes)
 ─────────────────────────         ─────────────────────────────────────
 "react once per event"            "look at reality now, fix the gap"
 miss an event → permanently       miss any number of events → next pass
 wrong                             still converges to desired
 needs a perfect event stream      tolerates crashes, restarts, missed
                                   events, duplicate events
```

**Idempotent + convergent.** Run the same reconcile 1000 times when the world
already matches: nothing happens 999 extra times. Each pass only ever moves
toward desired and stops when it arrives. This is exactly the thermostat.

---

## Exact syntax breakdown

The reconciliation model shows up in the object schema itself. The `spec`/`status`
split *is* the loop, encoded in every object.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: orders
spec:                      # ◄── DESIRED STATE. You write this. The loop's goal.
  replicas: 3              #     "I want 3." A controller will make it true.
status:                    # ◄── ACTUAL STATE. You NEVER write this. Controllers do.
  replicas: 3              #     how many pods actually exist
  readyReplicas: 3         #     how many pass readiness (Topic 43)
  availableReplicas: 3     #     ready long enough to count as available
  observedGeneration: 7    #     which spec version the controller has acted on
```

Field-by-field, with the loop meaning:

```
spec:
│
└── The WISH. Everything under here is what YOU want. The reconciliation loop's
    entire job is to make the world (and status) match this.

  replicas: 3
  │         │
  │         └── the target the loop maintains. Change it → loop acts. Delete a
  │             pod → loop restores it. This integer is "desired state" made real.
  └── under spec = desired.

status:
│
└── The REALITY, as last reported by the responsible controller. Written by
    Kubernetes, read-only to you. If you kubectl edit it, the next reconcile
    overwrites it with the truth.

  observedGeneration: 7
  │                   │
  │                   └── the metadata.generation of the spec the controller has
  │                       already processed. If spec.generation > observedGeneration,
  │                       the controller has NOT yet caught up to your latest change.
  └── This is how you tell "has the loop seen my update yet?" — a real debugging tool.
```

The command that lets you *watch the loop work*:

```
kubectl get deployment orders-api -w -o wide
│       │   │          │          │  │
│       │   │          │          │  └── extra columns incl. UP-TO-DATE, AVAILABLE
│       │   │          │          └── "-w" = WATCH: stream every change live, so you
│       │   │          │              literally see status crawl toward spec
│       │   │          └── the object to watch
│       │   └── kind
│       └── read from the API server
└── each printed line is a reconcile result: READY 1/3 → 2/3 → 3/3
```

And the command that shows the loop's *actions* as a narrative:

```
kubectl describe deployment orders-api
│                                       ...
│   Events:
│     ScalingReplicaSet  Scaled up replica set orders-api-abc to 3
│     └── every line here is one reconcile decision, in plain English.
└── "describe" surfaces the controller's Events — the loop's diary.
```

---

## Example 1 — basic

Watch the loop restore desired state with your own eyes. Every line commented.

```bash
# Declare desired state: 3 replicas of orders-api
kubectl create namespace orders
kubectl create deployment orders-api -n orders --image=nginx --replicas=3

# See desired vs actual side by side
kubectl get deployment orders-api -n orders
#   NAME         READY   UP-TO-DATE   AVAILABLE
#   orders-api   3/3     3            3          <- status(3) matches spec(3)

# List the 3 pods the ReplicaSet controller created
kubectl get pods -n orders
#   orders-api-6d4f...-aaa   Running
#   orders-api-6d4f...-bbb   Running
#   orders-api-6d4f...-ccc   Running

# --- TRIGGER THE LOOP: break actual, watch it heal ---
# In one terminal, watch live:
kubectl get pods -n orders -w    # keep this running

# In another terminal, delete a pod (make actual = 2, desired still = 3):
kubectl delete pod -n orders orders-api-6d4f...-aaa

# In the watch terminal you'll see, within a second or two:
#   orders-api-...-aaa   Terminating
#   orders-api-...-ddd   Pending      <- the loop created a REPLACEMENT
#   orders-api-...-ddd   Running      <- actual back to 3

# Read the loop's diary — the Events it emitted:
kubectl describe deployment orders-api -n orders | sed -n '/Events/,$p'
#   Normal  ScalingReplicaSet  Scaled up replica set orders-api-... to 3

# Prove idempotency: apply the SAME spec again, nothing changes
kubectl scale deployment orders-api -n orders --replicas=3   # already 3
kubectl get pods -n orders     # still the same 3 pods; loop found diff=0
```

You never told Kubernetes "recreate pod aaa." You only ever stated a wish
(`replicas: 3`). The loop noticed reality fell short and fixed it. That is the
whole of Kubernetes in one demo.

---

## Example 2 — production scenario

**The situation.** A well-meaning teammate is debugging `orders-api` in prod. A
pod is misbehaving, so they run `kubectl edit pod orders-api-xyz` and hand-edit
the container image to an older tag to "test a theory." Ten seconds later their
change is gone and the pod is back on the new image. They ping you: *"Kubernetes
keeps overwriting my fix, is the cluster haunted?"* You explain the loop.

**What actually happened, step by step:**

1. The `orders-api` **Deployment** owns a **ReplicaSet**, which owns the pods.
   The ReplicaSet's desired state (its pod **template**) says
   `image: orders-api:1.4.0`. That template is the *wish* for every pod it owns.

2. Your teammate edited the **Pod** object directly, setting
   `image: orders-api:1.3.0`. But editing a pod's image isn't even applied to a
   running container by that path, and more importantly it created **drift**:
   actual pod spec no longer matched the ReplicaSet's template.

3. The **ReplicaSet controller**, on its next reconcile, compared its owned pods
   to its template. It doesn't patch a pod's image in place — its reconciliation
   for a pod that no longer matches (or that it needs to replace) is to ensure the
   right number of *correct* pods exist. Combined with the Deployment controller
   enforcing the template, the drifted pod is reconciled away and a
   template-correct pod (image `1.4.0`) stands in its place. **Desired won.**

4. The lesson your teammate missed: **you cannot pin a change by editing an object
   a controller owns.** The controller's desired state always wins on the next
   pass. To actually change the image, you must change the *desired state at the
   top* — edit the **Deployment's** template:

```bash
# WRONG (gets reconciled away):
kubectl edit pod orders-api-xyz -n orders          # drift; loop reverts it

# RIGHT (changes desired state; the loop rolls it out everywhere):
kubectl set image deployment/orders-api \
  orders-api=orders-api:1.3.0 -n orders            # updates the template
#   → Deployment controller reconciles: new ReplicaSet, rolling update to 1.3.0
```

**Why this design is a feature, not a bug.** Because desired state always wins,
your cluster is **self-correcting against configuration drift**. Nobody can
quietly hand-mutate production into a snowflake state; whatever isn't written in
the owning object's desired state gets erased on the next reconcile. This is the
foundation of **GitOps** (Topic 52): put desired state in Git, let controllers
relentlessly reconcile the cluster to match it. Human drift is automatically
undone. Your teammate experienced the single most important safety property of
Kubernetes and mistook it for a haunting.

---

## Common mistakes

**Mistake 1 — Editing an object a controller owns and expecting it to stick.**

```
$ kubectl edit pod orders-api-xyz     # change something
   # ...seconds later, kubectl get pod shows your change is gone
```
**Root cause:** the pod's desired state is owned by a ReplicaSet/Deployment; the
loop reconciles the pod back to the owner's template. Right: **edit the top-level
owner** (the Deployment), which changes desired state and rolls out properly.

**Mistake 2 — Thinking reconciliation is event-driven ("it reacts to my delete").**

```
Belief:  "It saw my delete event, so it created one pod to undo it."
Reality: it doesn't count your events; it counts the WORLD. desired(3) vs
         actual(2) → create 1. It would still converge if it missed the event.
```
**Root cause:** an edge-triggered mental model. Right: level-triggered — the loop
looks at current reality each pass and fixes the gap, which is why it survives
missed events and controller restarts.

**Mistake 3 — Writing to `status` or reading `spec` to learn "what is."**

```
$ kubectl edit deployment orders-api   # you change status.replicas by hand
   # next reconcile overwrites it; also you were reading the wrong field
```
**Root cause:** confusing `spec` (desired, you write) with `status` (actual,
controllers write). Right: **write `spec`, read `status`.** To know "what's really
running," read `status.readyReplicas`, not `spec.replicas`.

**Mistake 4 — Expecting instant reconciliation / not accounting for `observedGeneration`.**

```
$ kubectl apply -f new-deploy.yaml
$ kubectl get pods    # old pods still there — "did it not work?!"
```
**Root cause:** the loop is asynchronous and paced (rolling updates step pod by
pod; controllers process a work queue). Right: check
`status.observedGeneration` vs `metadata.generation` and watch with `-w`; the
loop is converging, just not instantaneously.

**Mistake 5 — Assuming reconciliation can fix things it doesn't control.**

```
   Pod: ImagePullBackOff — loop keeps trying, never succeeds
```
The loop faithfully keeps *creating/retrying* the pod, but it cannot invent a
valid image. desired says "run image X"; if X doesn't exist, actual can never
reach desired, so the loop retries forever. **Root cause:** believing "the loop
heals everything." Right: the loop closes gaps *it has the power to close*; a bad
image, a full node, or a wrong config is a wish that reality cannot satisfy — you
must fix the wish or the environment.

---

## Hands-on proof

```bash
# 1. Desired state in, watch actual converge
kubectl create namespace demo
kubectl create deployment web -n demo --image=nginx --replicas=3
kubectl get deployment web -n demo -o wide -w   # watch READY climb 0/3→3/3, Ctrl-C

# 2. See spec (desired) vs status (actual) as two distinct fields in etcd
kubectl get deployment web -n demo -o jsonpath='DESIRED={.spec.replicas}{"\n"}'
kubectl get deployment web -n demo -o jsonpath='ACTUAL_READY={.status.readyReplicas}{"\n"}'

# 3. LEVEL-TRIGGERED PROOF: delete TWO pods at once; loop restores exactly two
kubectl get pods -n demo
kubectl delete pod -n demo <podA> <podB>     # actual drops to 1
kubectl get pods -n demo -w                  # loop creates 2 replacements → 3
#   it didn't create "one per delete event" — it looked at the world (1) vs
#   desired (3) and made up the difference (2). Ctrl-C when back to 3.

# 4. IDEMPOTENCY PROOF: apply the same desired state repeatedly, no churn
for i in 1 2 3; do kubectl scale deployment web -n demo --replicas=3; done
kubectl get pods -n demo     # still the same 3 pods; diff was 0 each time

# 5. DRIFT-CORRECTION PROOF: try to change a pod the RS owns; watch it revert
POD=$(kubectl get pod -n demo -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl label pod -n demo $POD app=hacked --overwrite   # remove it from the RS's
                                                        # selector match
kubectl get pods -n demo -w   # RS sees only 2 matching its selector → creates a
                              # 3rd; your relabeled pod is now orphaned/extra
#   desired (3 pods matching app=web) always wins.

# 6. Read the loop's diary
kubectl describe deployment web -n demo | sed -n '/Events/,$p'

# 7. Clean up
kubectl delete namespace demo
```

Step 3 is the proof that Kubernetes is level-triggered: two deletes, exactly two
replacements, driven by counting reality — not by reacting to each event.

---

## Practice exercises

### Exercise 1 — easy
Create a Deployment with `--replicas=4`. Print `spec.replicas` and
`status.readyReplicas` separately with `-o jsonpath`. Delete one pod and, without
looking at events, predict how many pods will exist 10 seconds later and why.
Confirm with `kubectl get pods`. **Write one sentence** explaining which field is
"desired" and which is "actual."

### Exercise 2 — medium
Demonstrate the difference between edge- and level-triggered by deleting **three**
pods of a 5-replica Deployment in a single `kubectl delete pod A B C` command.
Immediately run `kubectl get pods -w`. **Record** how many new pods appear and
explain, in terms of "desired vs actual counting," why Kubernetes created exactly
the right number even though it received the deletes essentially at once.

### Exercise 3 — hard (production simulation)
Reproduce the Example 2 "haunting." Create a Deployment `orders-api` (image
`nginx:1.25`). Now try to change one running pod to `nginx:1.24` via
`kubectl edit pod`. Observe the result. Then do it the correct way with
`kubectl set image deployment/orders-api`. **Write a short incident note** that
(a) explains why the pod-level edit didn't persist, referencing ownerReferences
and the reconcile pass; (b) explains why the Deployment-level change did persist
and rolled out; and (c) connects this drift-correction property to why GitOps
(Topic 52) is safe. Bonus: use `kubectl get deploy -o jsonpath` to show
`observedGeneration` advancing after your correct change.

---

## Mental model checkpoint

1. State the reconciliation loop in one sentence, using the words *desired*,
   *actual*, and *gap*.
2. What is the difference between **level-triggered** and **edge-triggered**, and
   why does level-triggered make Kubernetes resilient to missed events and
   controller crashes?
3. Which sub-object do *you* write, and which does a controller write:
   `spec` or `status`?
4. Why does deleting a pod owned by a Deployment bring it back, even though you
   changed nothing in any YAML?
5. Why is a reconcile function required to be **idempotent**, and how does that
   make `kubectl apply` safe to re-run?
6. Your hand-edit to a controller-owned object disappeared. Why — and where
   *should* you have made the change?
7. Name one thing the loop *cannot* fix no matter how many times it runs, and
   explain why (desired vs achievable).

---

## Quick reference card

| Concept | What it means | Key detail |
|---|---|---|
| Reconciliation loop | Compare desired vs actual, act to close the gap, repeat | The core pattern behind ALL K8s behavior |
| Desired state | What you want | `spec`; written by you; stored in etcd |
| Actual state | What is really running | `status` + real containers; written by controllers |
| Controller | Code running one reconcile loop | Lives in kube-controller-manager (and others) |
| Level-triggered | Act on current state, not on events | Converges even after missed events/crashes |
| Idempotent | Re-running changes nothing if already matched | Why `kubectl apply` is safe to repeat |
| Watch (list+watch) | How controllers see changes efficiently | Long-lived HTTP stream + `resourceVersion` |
| Drift correction | Manual changes get reverted to desired | Foundation of GitOps (Topic 52) |
| `observedGeneration` | Which spec version the loop has processed | `< metadata.generation` = not caught up yet |
| `kubectl get ... -w` | Watch actual crawl toward desired live | See the loop working in real time |

---

## When would I use this at work?

**1. Debugging "why did Kubernetes do that?"** Almost every surprising behavior —
a pod reappearing, a manual edit vanishing, a slow rollout — is the loop
enforcing desired state. Framing incidents as "what is desired vs actual, and
which controller is closing the gap?" turns confusion into a two-minute diagnosis.

**2. Adopting GitOps.** When your team puts manifests in Git and uses ArgoCD/Flux
(Topic 52), you are literally making Git the desired state and letting the
reconciliation loop enforce it continuously. Understanding level-triggering is
what lets you trust that a drifted cluster will self-heal back to what's in Git.

**3. Writing your own automation / operators.** The day you need to manage a
custom resource (say, a "TenantDatabase") you'll write a controller — and it will
be exactly this loop: watch desired, read actual, reconcile the gap, idempotently.
Every operator you ever install or build is a thermostat.

---

## Connected topics

**Study before this:**
- **Topic 28** — What Kubernetes is; the self-healing/declarative promise this
  loop actually delivers.
- **Topic 29** — Architecture. The controllers here live in the
  controller-manager; the watch/API-server plumbing is the loop's substrate.

**Study after this:**
- **Topic 32** — YAML and manifests: the `spec`/`status` structure this topic
  leans on, in full.
- **Topic 34** — Deployments: the most-used controller, with ReplicaSets, rolling
  updates, and revision history — reconciliation made concrete.
- **Topic 43 / 45 / 46** — Probes, the HorizontalPodAutoscaler, and rolling
  updates: each is another reconciliation loop with a different desired-state
  input.
- **Topic 52** — GitOps: making Git the desired state and letting reconciliation
  enforce it forever.
```
