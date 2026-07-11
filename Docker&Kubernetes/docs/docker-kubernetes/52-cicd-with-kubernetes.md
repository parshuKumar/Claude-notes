# 52 — CI/CD with Kubernetes
## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Imagine a restaurant kitchen. There are two ways new dishes reach the customer.

**Way 1 — the waiter carries the plate out (PUSH).** A cook finishes a dish, walks it out of the kitchen, across the dining room, and puts it on the table. The cook is *inside* the dining room now, touching the tables. If the cook trips, food goes everywhere. The cook needs a key to the dining room door.

**Way 2 — the food goes on a shelf, and a robot in the dining room grabs it (PULL).** The cook finishes a dish and puts it on a pass-through shelf. A trusted robot that *lives inside the dining room* watches the shelf, sees the new plate, and places it on the correct table. The cook never enters the dining room. The cook needs no key.

The "dining room" is your **production Kubernetes cluster**. The "cook" is your **CI pipeline** (GitHub Actions). "Way 1" is a **push** deployment — CI has cluster credentials and runs `kubectl apply` from outside. "Way 2" is **GitOps / pull** — an agent *inside* the cluster (ArgoCD or Flux) watches a Git repo like a shelf, and whenever the repo changes, it pulls the change into the cluster itself.

The whole topic is about how code you commit turns into `orders-api` pods actually running, and which of these two doors you use to make that happen.

---

## The Linux kernel feature underneath

CI/CD has no single kernel primitive — it is glue that stands on top of everything you already learned. But there *is* a concrete OS/networking mechanism that makes the whole thing safe and possible: **TLS-authenticated HTTPS requests to the Kubernetes API server, and containers running your build steps.**

Two physical realities do all the work:

1. **The API server is just an HTTPS endpoint.** Every deploy — whether `kubectl apply`, Helm, or ArgoCD — is ultimately a TCP connection to `https://<api-server>:6443`, carrying a bearer token or client certificate, sending JSON/YAML over the wire. There is no magic "deploy" syscall. When your GitHub Actions runner "deploys," it opens a socket to port 6443 and does an HTTP `PATCH` / `POST`. The kernel feature underneath is the **TCP/TLS stack** plus **x509 certificate verification** (Topic 40 covered how pods reach services; this is the same networking reaching the control plane).

2. **Every CI step is a container.** A GitHub Actions job runs inside a Linux container (or VM) on a runner. When it runs `docker build`, it uses the same **namespaces + cgroups + overlayfs** trinity from Topic 02. Layer caching in CI (Topic 27) is literally reusing `/var/lib/docker/overlay2/` layers between runs. So "CI builds an image" is: a container, starting a build that produces more container layers, pushing them as tarballs+JSON to a registry over HTTPS.

Hold this: **CI/CD is not a Kubernetes feature. It is a sequence of HTTPS calls and container runs, authenticated by a token, that ends with the API server writing your desired state into etcd** (Topic 29). The reconciliation loop (Topic 30) does the rest.

---

## What is this?

**CI/CD** (Continuous Integration / Continuous Delivery) is the automated pipeline that takes a Git commit, builds a container image from it, pushes that image to a registry, and updates your Kubernetes cluster to run the new image — with no human typing `kubectl` by hand.

**GitOps** is a specific *style* of the "CD" half: instead of a pipeline pushing changes *into* the cluster, you commit your desired manifests to Git, and an agent *inside* the cluster continuously pulls Git and reconciles the cluster to match. Git becomes the single source of truth for what should be running.

---

## Why does it matter for a backend developer?

You own `orders-api`. Today your "deploy process" is: SSH somewhere, run `kubectl set image deployment/orders-api orders-api=orders-api:latest`, and pray. Here is what goes wrong without a real pipeline:

- **`latest` bites you.** You pushed `orders-api:latest` twice today. A pod crashed and restarted at 3am, pulled `latest`, and got the *newer, broken* build you never meant to ship. Now prod is running code that was never approved. Nobody can tell which commit is live.
- **"It deployed but nothing changed."** You ran `kubectl apply` but the image tag was identical (`latest` again), so Kubernetes saw no change in the spec and did nothing. Your fix never rolled out. You stared at logs for an hour.
- **No audit trail.** Someone changed the replica count in prod by hand. Three weeks later it's mysteriously back to the old value because someone re-applied an old manifest. There is no record of who changed what, when.
- **Credentials sprayed everywhere.** Your cluster admin `kubeconfig` is pasted into a GitHub secret with full cluster-admin rights. One leaked token = whole cluster owned. (Topic 53 — RBAC — fixes this; this topic sets up *why* you need it.)

A real CI/CD pipeline gives you: **every deploy tied to a specific Git commit and a specific immutable image tag, automatically, with a full history you can roll back.** That is the difference between "we deploy" and "we deploy safely."

---

## The physical reality

When CI/CD is set up for `orders-api`, here is what actually exists across three machines:

**1. On the GitHub Actions runner (a throwaway Linux VM/container), during a run:**

```
/home/runner/work/orders-api/orders-api/     ← your checked-out repo
/var/lib/docker/overlay2/<hash>/              ← image layers being built
~/.docker/config.json                         ← registry auth (base64 token) written by login step
~/.kube/config  (or an env var)               ← cluster credentials, if using PUSH deploys
```

**2. In the container registry (e.g. ghcr.io, ECR), after push:**

```
orders-api:  a291f3c   → sha256:9b2f...   (manifest listing layer digests)
             a10b4bf   → sha256:5c7e...
             latest    → points to whichever we last tagged (a moving pointer!)
```

Each tag is just a *human name pointing at a content-addressed digest* `sha256:...`. The digest is the real, immutable identity (Topic 04). `a291f3c` is a git SHA we used as the tag — it never moves. `latest` is a pointer we keep repointing — that's the danger.

**3. In the cluster's etcd, after deploy:**

```
/registry/deployments/orders/orders-api  →  spec.template.spec.containers[0].image:
                                            "ghcr.io/acme/orders-api:a291f3c"
```

That string in etcd is the *desired state*. Change it, and the Deployment controller (Topic 34) creates a new ReplicaSet and rolls pods. **The entire job of CI/CD is to correctly change that one string in etcd** — either by pushing to the API server, or by committing to Git and letting an in-cluster agent push it for you.

**4. If using GitOps, additionally in the cluster:**

```
namespace: argocd
  deployment/argocd-application-controller   ← the in-cluster agent, always running
  Application "orders-api"  → spec.source.repoURL: git@github.com:acme/orders-api-config
                              spec.source.targetRevision: main
                              spec.destination.namespace: orders
```

That `Application` object is the robot's instructions: "watch this Git repo, keep namespace `orders` matching it."

---

## How it works — step by step

Full trace of a commit becoming running pods. We'll do the **push** flow first (GitHub Actions → cluster), then show where **GitOps** diverges.

### Push flow (kubectl/Helm from GitHub Actions)

1. **You `git push` to `main`.** GitHub receives the commit, say SHA `a291f3c`. A webhook fires the Actions workflow.

2. **A runner spins up.** GitHub allocates a fresh Ubuntu VM/container. It has *nothing* of yours yet.

3. **Checkout.** `actions/checkout` clones your repo into `/home/runner/work/...`. Now the Dockerfile and manifests are on disk.

4. **Build the image.** `docker build` reads the Dockerfile, executes each instruction, produces layers in `/var/lib/docker/overlay2/`. The result is tagged `ghcr.io/acme/orders-api:a291f3c` — **the git SHA as the tag**, so this image is forever tied to this exact commit.

5. **Log in to the registry.** The runner writes registry credentials into `~/.docker/config.json` using a token from GitHub secrets.

6. **Push the image.** `docker push` uploads each *new* layer (skips layers the registry already has, by digest) plus a manifest. The registry now serves `orders-api:a291f3c`.

7. **Authenticate to the cluster.** The deploy job loads a `kubeconfig` (from a secret) pointing at `https://<api-server>:6443` with a **ServiceAccount token** (Topic 53). This is a bearer token, not full admin — ideally scoped to only patch deployments in namespace `orders`.

8. **Update the deployment.** The job runs `kubectl set image deployment/orders-api orders-api=ghcr.io/acme/orders-api:a291f3c -n orders` (or `helm upgrade`). This opens TLS to port 6443, sends a `PATCH` with the new image string, and the API server writes it to etcd.

9. **The reconciliation loop takes over.** The Deployment controller notices desired image ≠ current image, creates a new ReplicaSet, and does a rolling update (Topic 46): new pods pull `a291f3c`, pass readiness probes (Topic 43), old pods drain. Zero downtime.

10. **The pipeline waits for rollout.** `kubectl rollout status deployment/orders-api -n orders --timeout=120s` blocks until all new pods are Ready, or fails the pipeline if they crashloop. Now CI knows the deploy truly succeeded.

### GitOps flow (pull) — where it diverges

With GitOps, steps 1–6 are identical (build & push image). Then:

7. **CI does NOT touch the cluster.** Instead, CI (or a bot) commits a one-line change to a *separate config repo*: it updates `image: orders-api:a291f3c` in a Helm `values.yaml` or Kustomize file, and pushes to `orders-api-config` repo. **CI has no cluster credentials at all.**

8. **The in-cluster agent notices.** ArgoCD's application-controller polls (or gets a webhook from) the config repo every ~3 minutes. It sees the new commit.

9. **The agent diffs and syncs.** ArgoCD compares "what Git says" vs "what's live in etcd" (via the API server). It sees the image differs, and it — *from inside the cluster* — applies the change to the API server. Same etcd write as step 8 above, but initiated from *inside*, using an in-cluster ServiceAccount.

10. **Reconciliation + self-healing.** Same rolling update. Bonus: if anyone hand-edits the live deployment, ArgoCD sees the drift from Git and *reverts it* automatically (or flags it). Git is now the undeniable source of truth.

**The single deepest difference:** in push, credentials to prod live *outside* the cluster (in GitHub). In pull/GitOps, credentials never leave the cluster — the cluster reaches out to Git, so there's nothing to leak in CI.

---

## Exact syntax breakdown

**Tagging by git SHA, not latest — the core habit:**

```
docker build -t ghcr.io/acme/orders-api:${GITHUB_SHA} -t ghcr.io/acme/orders-api:latest .
│            │  │                        │             │                             │  │
│            │  │                        │             │                             │  └─ build context = current dir
│            │  │                        │             │                             └─ ALSO tag latest (optional convenience;
│            │  │                        │             │                                never DEPLOY from this tag)
│            │  │                        │             └─ second -t: an image can carry many tags at once
│            │  │                        └─ ${GITHUB_SHA}: the commit SHA, e.g. a291f3c...
│            │  │                           THIS is the immutable, deployable tag
│            │  └─ full image ref: registry/namespace/repo:tag
│            └─ -t: tag the resulting image with this name
└─ build the image from the Dockerfile in .
```

**The push-deploy command inside GitHub Actions:**

```
kubectl set image deployment/orders-api orders-api=ghcr.io/acme/orders-api:a291f3c -n orders
│       │   │     │                     │          │                                │  │
│       │   │     │                     │          │                                │  └─ namespace "orders" (Topic 37)
│       │   │     │                     │          │                                └─ -n: scope to that namespace
│       │   │     │                     │          └─ the new image ref (git SHA tag)
│       │   │     │                     └─ container NAME inside the pod = "orders-api"
│       │   │     │                        (must match spec.template.spec.containers[].name)
│       │   │     └─ target object: the Deployment named orders-api
│       │   └─ "image": which field to change
│       └─ set: imperative sub-command that PATCHes one field
└─ kubectl: opens TLS to the API server :6443 with your token
```

**The Helm alternative (safer — full templated release):**

```
helm upgrade --install orders-api ./chart -n orders --set image.tag=a291f3c --wait --atomic
│    │       │          │         │       │  │      │                       │      │
│    │       │          │         │       │  │      │                       │      └─ --atomic: on failure, auto-rollback
│    │       │          │         │       │  │      │                       │         to the previous good release
│    │       │          │         │       │  │      │                       └─ --wait: block until pods are Ready
│    │       │          │         │       │  │      └─ override values.yaml's image.tag with the git SHA
│    │       │          │         │       │  └─ --set: override a single value at deploy time
│    │       │          │         │       └─ -n orders: install into namespace orders
│    │       │          │         └─ path to the chart directory (Topic 49)
│    │       │          └─ release name (Helm tracks this release's history for rollback)
│    │       └─ --install: create it if it doesn't exist yet, else upgrade (idempotent)
│    └─ upgrade: apply a new revision of the release
└─ helm: renders templates locally, then sends the result to the API server
```

**Waiting for the rollout so CI fails loudly on a bad deploy:**

```
kubectl rollout status deployment/orders-api -n orders --timeout=120s
│       │       │      │                      │          │
│       │       │      │                      │          └─ give up (and exit non-zero) after 120s → fails the CI job
│       │       │      │                      └─ namespace
│       │       │      └─ which deployment to watch
│       │       └─ "status": stream progress until complete or timeout
│       └─ rollout: the sub-command family for deploy progress/history/undo
└─ kubectl: same TLS call, but this one polls until pods are Ready
```

**An ArgoCD Application (the GitOps "robot's instructions"):**

```yaml
apiVersion: argoproj.io/v1alpha1     # ArgoCD's own CRD, not core k8s
kind: Application                    # "watch this repo, keep this namespace matching it"
metadata:
  name: orders-api
  namespace: argocd                  # the Application object lives in the argocd namespace
spec:
  project: default
  source:
    repoURL: https://github.com/acme/orders-api-config   # the CONFIG repo (not app source)
    targetRevision: main                                 # which branch/tag/commit to track
    path: overlays/prod                                  # subfolder of manifests to apply
  destination:
    server: https://kubernetes.default.svc               # THIS cluster (in-cluster address)
    namespace: orders                                    # deploy into namespace orders
  syncPolicy:
    automated:
      prune: true        # delete resources removed from Git (Git is the truth)
      selfHeal: true     # if the live cluster drifts from Git, revert it automatically
```

Annotated on the fields that trip people up:

```
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      │        │
      │        └─ selfHeal: a human runs `kubectl scale ... --replicas=1`?
      │           ArgoCD sees Git still says 3, and scales it back to 3. Drift is undone.
      └─ prune: you delete a Service from Git?
         ArgoCD deletes it from the cluster too. Without prune, orphans linger forever.
```

---

## Example 1 — basic

A minimal, complete GitHub Actions workflow that builds `orders-api`, pushes it tagged by commit SHA, and deploys via `kubectl`. Every line commented.

```yaml
# .github/workflows/deploy.yml
name: build-and-deploy
on:
  push:
    branches: [main]          # run only when main advances (i.e. a merged PR)

permissions:
  contents: read              # workflow may read the repo
  packages: write             # workflow may push images to ghcr.io (GitHub's registry)

jobs:
  ship:
    runs-on: ubuntu-latest    # a fresh throwaway Linux runner
    steps:
      - name: Checkout code
        uses: actions/checkout@v4       # clones repo onto the runner

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}          # the user who triggered the run
          password: ${{ secrets.GITHUB_TOKEN }}  # auto-provided, scoped token

      - name: Build and push image tagged by commit SHA
        run: |
          IMAGE=ghcr.io/${{ github.repository }}/orders-api
          # Build once, tag with the immutable git SHA. NEVER deploy :latest.
          docker build -t "$IMAGE:${GITHUB_SHA}" .
          docker push "$IMAGE:${GITHUB_SHA}"
          echo "IMAGE_REF=$IMAGE:${GITHUB_SHA}" >> "$GITHUB_ENV"   # pass to next step

      - name: Configure cluster access
        run: |
          # KUBE_CONFIG is a base64'd kubeconfig stored as a secret.
          # It uses a deploy-only ServiceAccount token (Topic 53), NOT cluster-admin.
          mkdir -p ~/.kube
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > ~/.kube/config

      - name: Deploy the new image
        run: |
          kubectl set image deployment/orders-api \
            orders-api="$IMAGE_REF" -n orders
          # Block until the new pods are actually Ready, or fail the job.
          kubectl rollout status deployment/orders-api -n orders --timeout=120s
```

What this guarantees: the image that runs in prod is `orders-api:<the exact commit you merged>`. If it crashloops, `rollout status` times out, the job goes red, and you know immediately.

---

## Example 2 — production scenario

**The situation.** Your team scaled from 2 to 15 engineers. The push pipeline from Example 1 now has three problems: (1) the `KUBE_CONFIG` secret has near-admin rights and lives in GitHub, which security flagged; (2) people occasionally hotfix prod by hand with `kubectl edit`, and nobody knows the cluster's true state; (3) you want staging and prod to deploy the *same tested image*, not two separately-built images.

**The fix: GitOps with ArgoCD + a separate config repo + promotion by SHA.**

You split into two repos:

```
orders-api          ← application source code + Dockerfile (CI builds & pushes image here)
orders-api-config   ← Kubernetes manifests, one folder per environment (ArgoCD watches this)
    ├── base/                    # shared manifests
    ├── overlays/staging/        # staging patches (replicas, resources, image tag)
    └── overlays/prod/           # prod patches
```

**Step 1 — CI builds and pushes, then bumps *staging* config (no cluster access):**

```yaml
      - name: Build, push, and open a staging bump PR
        run: |
          IMAGE=ghcr.io/acme/orders-api:${GITHUB_SHA}
          docker build -t "$IMAGE" .
          docker push "$IMAGE"
          # Update staging's image tag in the CONFIG repo and push.
          git clone https://x-access-token:${{ secrets.CONFIG_TOKEN }}@github.com/acme/orders-api-config
          cd orders-api-config
          # Rewrite the tag in staging's kustomization to the new SHA.
          sed -i "s|newTag:.*|newTag: ${GITHUB_SHA}|" overlays/staging/kustomization.yaml
          git commit -am "staging: orders-api ${GITHUB_SHA}"
          git push
```

**Step 2 — ArgoCD (in-cluster) syncs staging automatically.** Within ~3 minutes, staging runs the new SHA. No CI credentials touched the cluster.

**Step 3 — promote to prod by copying the *same* SHA.** After staging soaks and QA passes, a human (or a "promote" workflow) opens a PR that copies the identical tag from staging into `overlays/prod/kustomization.yaml`:

```
overlays/staging/kustomization.yaml:  newTag: a291f3c   ← already soaked in staging
overlays/prod/kustomization.yaml:     newTag: a291f3c   ← promote the EXACT same digest
```

The prod PR is reviewed and merged. ArgoCD syncs prod to the *same bytes* that were tested in staging — not a rebuild that might differ.

**Step 4 — self-heal kills manual drift.** An on-call engineer runs `kubectl scale deployment/orders-api --replicas=10 -n orders` during an incident. ArgoCD's `selfHeal: true` sees Git still says `replicas: 3` and reverts it. Lesson learned: the fix must go in Git, not the cluster. Now the cluster's state is *always* exactly what's in the config repo — auditable via `git log`.

**The payoff:**
- No prod credentials in CI (they leaked from a different team last quarter; yours can't).
- Every prod state is a Git commit — `git blame overlays/prod/kustomization.yaml` tells you who shipped what, when.
- Staging and prod run byte-identical images (promotion copies the SHA, no rebuild).
- Rollback is `git revert` — ArgoCD syncs the cluster back automatically.

---

## Common mistakes

**Mistake 1 — Deploying `:latest`.**

```
$ kubectl set image deployment/orders-api orders-api=orders-api:latest -n orders
deployment.apps/orders-api image updated
$ kubectl rollout status deployment/orders-api -n orders
Waiting for deployment "orders-api" rollout to finish...
# ...hangs, or "successfully rolled out" but NOTHING actually changed
```

Root cause: the image *string* in the spec (`orders-api:latest`) is byte-identical to what's already in etcd, so the Deployment controller (Topic 30's reconciliation) sees **no diff** and does nothing — even though the `latest` tag now points to different bytes in the registry. Kubernetes compares the *spec string*, not the registry digest. Worse, when a pod restarts later with `imagePullPolicy: Always`, it silently pulls whatever `latest` points to *now* — a build you never approved. Right way: **tag by immutable git SHA** so every deploy changes the spec string and pins exact bytes.

**Mistake 2 — `imagePullPolicy` mismatch with mutable tags.**

You use `:latest` *and* set `imagePullPolicy: IfNotPresent`. Now a node that already cached an old `latest` **never re-pulls**, so different nodes run different code under the same tag:

```
$ kubectl get pods -n orders -o wide
orders-api-abc  Running  node-1   # running OLD latest (cached)
orders-api-def  Running  node-2   # running NEW latest (freshly pulled)
```

Root cause: mutable tag + `IfNotPresent` = the tag is a lie, resolved differently per node. Right way: immutable SHA tags make caching *safe* (the tag always means the same bytes), and you can freely use `IfNotPresent` for speed.

**Mistake 3 — Not waiting for rollout, so CI goes green on a broken deploy.**

```yaml
- run: kubectl set image deployment/orders-api orders-api=$IMAGE_REF -n orders
# job ends here, exits 0, CI is GREEN
# ...meanwhile the new pods are in CrashLoopBackOff
```

Root cause: `set image` returns as soon as the *spec* is updated, not when pods are *healthy*. Your pipeline reports success while prod is down. Right way: always follow with `kubectl rollout status ... --timeout=Ns` (or `helm upgrade --wait --atomic`), which exits non-zero if pods don't become Ready — turning CI red and auto-rolling-back.

**Mistake 4 — Storing cluster-admin kubeconfig in CI secrets.**

Root cause: `secrets.KUBE_CONFIG` with cluster-admin means a compromised Action, a malicious dependency, or a leaked log = full cluster takeover. Wrong:

```
# a kubeconfig whose ServiceAccount is bound to cluster-admin
```

Right way: create a **deploy-only ServiceAccount** with a Role that can only `get/patch` deployments in namespace `orders` (Topic 53), or eliminate CI credentials entirely by going GitOps (pull), where the cluster reaches out to Git and nothing sensitive lives in CI.

**Mistake 5 — Building a *new* image when promoting staging → prod.**

```
# staging deploy: docker build ... (produces image X)
# prod deploy:    docker build ... (produces image Y — DIFFERENT bytes!)
```

Root cause: rebuilding from source at promotion time can pick up new base-image layers, dependency versions, or a dirty cache, so prod runs code that was *never* tested in staging. Right way: **build once, promote the digest.** Copy the exact `sha`/digest from staging to prod. The bytes that soaked in staging are the bytes that reach prod.

---

## Hands-on proof

Run these against a local cluster (`kind`, `minikube`, or Docker Desktop's Kubernetes) to see the SHA-vs-latest difference and the reconciliation trigger.

```bash
# 0. Setup: namespace + a deployment on a pinned tag.
kubectl create namespace orders
kubectl create deployment orders-api --image=nginx:1.25.3 -n orders
kubectl rollout status deployment/orders-api -n orders

# 1. Prove: changing to the SAME image string does NOTHING (no new ReplicaSet).
kubectl get rs -n orders                       # note the single ReplicaSet + its age
kubectl set image deployment/orders-api nginx=nginx:1.25.3 -n orders
kubectl get rs -n orders                       # SAME rs, unchanged — spec didn't change

# 2. Prove: changing to a NEW immutable tag triggers a rollout (new ReplicaSet).
kubectl set image deployment/orders-api nginx=nginx:1.25.4 -n orders
kubectl get rs -n orders                       # a SECOND ReplicaSet appears
kubectl rollout status deployment/orders-api -n orders

# 3. See the exact image string stored in etcd (the "desired state").
kubectl get deployment orders-api -n orders \
  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
# → nginx:1.25.4   (this string IS what the controller reconciles toward)

# 4. See the rollout history — every deploy is a revision you can roll back to.
kubectl rollout history deployment/orders-api -n orders

# 5. Prove rollback works (like `git revert` for deploys).
kubectl rollout undo deployment/orders-api -n orders
kubectl get deployment orders-api -n orders \
  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo   # back to 1.25.3

# 6. Clean up.
kubectl delete namespace orders
```

What you verified: (1) identical spec strings cause no deploy — the `latest` trap; (2) a new tag *does* trigger a new ReplicaSet — why SHA tags work; (3) the image string in etcd is the single source of desired state; (4)+(5) every deploy is a recorded, reversible revision.

---

## Practice exercises

### Exercise 1 — easy
Create namespace `orders` and a deployment `orders-api` from `nginx:1.25.3`. Run `kubectl set image` twice with the *same* tag and confirm with `kubectl get rs -n orders` that no new ReplicaSet is created. Then set a new tag and confirm a second ReplicaSet appears. Write one sentence explaining why the first two commands did nothing.

### Exercise 2 — medium
Write a GitHub Actions workflow file (it doesn't need to actually run) that: builds an image tagged `${GITHUB_SHA}`, pushes it, and deploys with `kubectl set image` followed by `kubectl rollout status --timeout=90s`. Then deliberately break it: point the deploy at an image tag that doesn't exist, and explain (a) what state the pods enter, and (b) how `rollout status` makes the CI job fail instead of falsely reporting success.

### Exercise 3 — hard (production simulation)
Model the push-vs-pull tradeoff on paper for `orders-api`. Draw two diagrams: (A) push — GitHub Actions holds a kubeconfig and calls the API server; (B) pull — ArgoCD runs in the cluster and watches a config repo. For each, mark *where the prod credentials physically live*. Then answer: if an attacker compromises your GitHub Actions runner, what can they reach in scenario A vs scenario B? Finally, write the exact two-file change (staging `kustomization.yaml` and prod `kustomization.yaml`) that promotes commit `a291f3c` from staging to prod by digest, and explain why you copy the tag instead of rebuilding.

---

## Mental model checkpoint

Answer from memory:

1. In one sentence each, what's the difference between **push** and **pull (GitOps)** deployment, and where do prod credentials live in each?
2. Why does deploying `orders-api:latest` twice often result in "nothing happened," in terms of what Kubernetes actually compares?
3. What is the immutable, deployable thing you should tag images with, and why not `latest`?
4. What does `kubectl rollout status --timeout` protect your pipeline from?
5. In GitOps, what do `prune: true` and `selfHeal: true` each do when the cluster drifts from Git?
6. When promoting staging → prod, why copy the image SHA instead of rebuilding from source?
7. What single string in etcd does every CD strategy ultimately change, and which controller reacts to it?

---

## Quick reference card

| Command / concept | What it does | Key detail |
|---|---|---|
| `docker build -t repo:${GITHUB_SHA}` | Tag image by immutable commit SHA | Never deploy `:latest` — the tag must pin exact bytes |
| `kubectl set image deploy/x c=repo:sha -n ns` | PATCH one container's image in etcd | Changes desired state; controller does the rollout |
| `kubectl rollout status --timeout=Ns` | Block until pods Ready, else fail | Stops CI going green on a crashlooping deploy |
| `kubectl rollout undo deploy/x` | Roll back to previous revision | Every deploy is a recorded revision |
| `helm upgrade --install --wait --atomic` | Deploy a chart, auto-rollback on failure | Safer than raw `set image` for templated apps |
| GitOps (ArgoCD/Flux) | In-cluster agent pulls Git → cluster | Credentials never leave the cluster |
| `Application` (ArgoCD CRD) | Declares "watch repo, sync into namespace" | `prune` + `selfHeal` enforce Git as truth |
| Push vs Pull | CI has cluster creds vs cluster has Git creds | Pull removes prod creds from CI entirely |

---

## When would I use this at work?

1. **Setting up a new service's first pipeline.** When `orders-api` needs to go from "I deploy by hand" to "merging to main ships it," you build the Example 1 workflow: build → push by SHA → `set image` → `rollout status`. Ten minutes of YAML replaces a fragile manual ritual.

2. **Passing a security review.** When security says "why does GitHub have a cluster-admin kubeconfig?" you either scope it down to a deploy-only ServiceAccount (Topic 53) or migrate to GitOps so CI holds *no* cluster credentials at all — and you can explain the exact threat model difference.

3. **Debugging "the fix didn't deploy."** When a teammate swears they shipped a fix but prod is unchanged, you immediately check: did they deploy `:latest` (so the spec string never changed)? Did the pipeline skip `rollout status` and go green on a crashloop? You know the two classic failure modes cold.

---

## Connected topics

- **Study before:**
  - **Topic 27 — Docker in CI/CD**: building and pushing images in GitHub Actions with layer caching — the "CI" half this topic builds on.
  - **Topic 30 — The Reconciliation Loop**: why changing one image string in etcd is enough to roll pods.
  - **Topic 34 — Deployments** and **Topic 46 — Rolling Updates and Rollbacks**: the mechanics `set image` triggers.
  - **Topic 49 — Helm Basics**: the templated-deploy alternative to raw `kubectl`.
- **Study after:**
  - **Topic 53 — RBAC**: create the least-privilege, deploy-only ServiceAccount your pipeline should authenticate as.
  - **Topic 54 — Multi-Environment Strategy**: the dev/staging/prod promotion workflow this topic's Example 2 introduced, in full.
