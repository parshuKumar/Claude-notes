# 54 — Multi-Environment Strategy
## Section: Production Patterns

## ELI5 — The Simple Analogy

Think about a play being staged in a theatre. Before opening night in front of a paying audience, the actors go through stages:

1. **Rehearsal room (dev)** — messy, actors try things, mess up, restart. Nobody outside sees it. Cheap and disposable.
2. **Dress rehearsal on the real stage (staging)** — same stage, same lights, same costumes as the real show, but the audience is just a few critics. If something breaks here, it's embarrassing but not a disaster.
3. **Opening night (production)** — full audience, real tickets, real money. Everything must work. You do **not** experiment here.

The play (your app) is the **same script** at every stage. What changes is the *setting*: the rehearsal room uses a fake prop phone; opening night uses a real one. In software, the "script" is your container image and manifests; the "props" are the config and secrets that differ per environment.

**Multi-environment strategy** is how you run the *same app* through *dev → staging → production* with the *right settings at each stage*, and how you safely promote it from one to the next.

## The Linux kernel feature underneath

There's no single kernel primitive for "environments" — this is an organizational pattern. But the **isolation** that makes multiple environments safe on shared infrastructure comes straight from the kernel features you already know:

- **Namespaces** (Topic 02, Topic 37): a Kubernetes **Namespace** gives each environment its own naming scope, DNS subdomain (`svc.orders-staging.svc.cluster.local`), and RBAC boundary (Topic 53). When you run dev/staging/prod as separate *clusters*, isolation goes even deeper — separate API servers, separate etcd, separate **network namespaces** and nodes entirely, so a runaway dev workload can't touch prod's kernel resources.
- **cgroups** (Topic 44): ResourceQuotas per Namespace ride on cgroup accounting so a noisy dev environment can't starve staging on a shared cluster.

So "environment isolation" is the same kernel isolation (namespaces + cgroups) you learned for containers, applied one level up — either as K8s Namespaces (soft, shared-kernel isolation) or as separate clusters (hard, separate-machine isolation).

## What is this?

A multi-environment strategy defines **how many environments you run** (typically dev, staging, production), **how they're isolated** (separate Namespaces vs separate clusters), **how config/secrets differ** between them (Helm values, Kustomize overlays), and **how a change is promoted** from one to the next (the same image, tested, moving up the chain).

The golden rule: **build the image once, deploy the same image everywhere, and change only configuration between environments.**

## Why does it matter for a backend developer?

Without a clear strategy, teams fall into two traps:

- **The "it works on my machine / dev" trap.** dev uses SQLite, prod uses Postgres; dev has 1 replica, prod has 20; dev has debug logging, prod doesn't. The app behaves differently in each because the *environments*, not just the config, diverged. Bugs appear only in prod.
- **The "cowboy prod" trap.** No staging, so every change is tested live on customers. One bad deploy takes down `orders-api` for real users.

A good strategy gives you **confidence**: a change proven in staging (which mirrors prod) is very likely to work in prod. It gives you **safety**: blast radius is contained; dev can't reach prod's database. And it gives you **reproducibility**: the *same image* that passed staging is the one that ships, so you're not re-rolling the dice.

This is also where several earlier topics come together: image tagging (Topic 24), Secrets management (Topic 50), RBAC (Topic 53), Helm (Topic 49), and GitOps (Topic 52) all exist largely to make multi-environment promotion safe.

## The physical reality

Two dominant models:

**Model A — Namespaces on one cluster (cheaper, softer isolation):**

```
ONE cluster
├── Namespace: orders-dev       (ResourceQuota: small)
├── Namespace: orders-staging   (ResourceQuota: medium)
└── Namespace: orders-prod      (ResourceQuota: large, PodDisruptionBudgets, NetworkPolicies)
     ↑ same API server, same etcd, same nodes — isolation is logical (namespaces + RBAC + quotas)
```

**Model B — Separate clusters (stronger isolation, more cost/ops):**

```
Cluster: dev-cluster        (small, cheap nodes, permissive)
Cluster: staging-cluster    (prod-like sizing)
Cluster: prod-cluster       (HA control plane, multi-AZ, locked-down RBAC, separate cloud account)
     ↑ separate etcd, separate networks — a dev outage physically cannot affect prod
```

On disk, the same app's manifests live once, with per-environment overrides:

```
deploy/
├── base/                       ← shared manifests (Deployment, Service, HPA)
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/     kustomization.yaml   (replicas: 1, debug logs, tiny resources)
    ├── staging/ kustomization.yaml   (replicas: 3, prod-like)
    └── prod/    kustomization.yaml   (replicas: 10, HPA, PDB, real secrets ref)
```

Or, with Helm (Topic 49):

```
charts/orders-api/
├── templates/            ← one set of templated manifests
├── values.yaml           ← defaults
├── values-dev.yaml       ← replicas:1, image tag from CI, debug on
├── values-staging.yaml   ← replicas:3
└── values-prod.yaml      ← replicas:10, resources up, HPA on
```

## How it works — step by step

The lifecycle of a single change through the environments:

1. **Commit & build once.** A merge to `main` triggers CI (Topic 52). CI builds **one image** and tags it by immutable git SHA: `ghcr.io/acme/orders-api:git-9f3a1c2` (Topic 24 — never `latest`).
2. **Deploy to dev automatically.** CI runs `helm upgrade --install orders-api ./charts/orders-api -f values-dev.yaml --set image.tag=git-9f3a1c2 -n orders-dev`. Dev now runs the new image with dev config (1 replica, debug logging, dev database).
3. **Automated tests run against dev/staging.** Integration/e2e tests hit the dev URL. Green? Promote.
4. **Promote the SAME image to staging.** No rebuild. `helm upgrade ... -f values-staging.yaml --set image.tag=git-9f3a1c2 -n orders-staging`. Staging mirrors prod sizing and uses staging secrets. Run smoke tests, load tests, manual QA.
5. **Approve → promote to production.** A human (or a policy) approves. Same image tag: `... -f values-prod.yaml --set image.tag=git-9f3a1c2 -n orders-prod`. Production does a **rolling update** (Topic 46) — zero downtime.
6. **Verify with observability.** Watch error-rate and p99 latency in Grafana (Topic 51). If bad, `helm rollback` / `kubectl rollout undo` (Topic 46).

The invariant across all six steps: **the artifact (image digest) never changes** after step 1. Only the values file changes. If it worked in staging, the *exact same bytes* run in prod — you've removed "the build was different" as a possible cause of prod-only bugs.

## Exact syntax breakdown

**Helm per-environment values** — the difference is *only* config:

```yaml
# values.yaml (defaults — shared)
image:
  repository: ghcr.io/acme/orders-api
  tag: "latest"            # overridden per deploy by --set image.tag=<git-sha>
replicaCount: 2
resources:
  requests: { cpu: 100m, memory: 128Mi }
logLevel: info
```

```yaml
# values-prod.yaml (only the deltas from defaults)
replicaCount: 10                       # prod runs more copies
resources:
  requests: { cpu: 250m, memory: 256Mi }   # prod pods are bigger
  limits:   { cpu: 500m, memory: 512Mi }
logLevel: warn                         # less noise in prod
autoscaling:
  enabled: true                        # HPA only in prod (Topic 45)
  minReplicas: 5
  maxReplicas: 30
```

The deploy command, annotated:

```
helm upgrade --install orders-api ./charts/orders-api -f values-prod.yaml --set image.tag=git-9f3a1c2 -n orders-prod
│    │       │         │          │                    │  │               │    │                        │
│    │       │         │          │                    │  │               │    │                        └── target namespace = environment
│    │       │         │          │                    │  │               │    └── pin the EXACT image proven in staging
│    │       │         │          │                    │  │               └── override one value at deploy time
│    │       │         │          │                    │  └── environment-specific values file
│    │       │         │          │                    └── the chart directory
│    │       │         │          └── release name
│    │       │         └── install if absent, else upgrade (idempotent)
│    │       └── subcommand
│    └── (upgrade)
└── helm
```

**Kustomize overlay** (the non-Helm alternative):

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base                 # inherit the shared manifests
namespace: orders-prod         # stamp every resource into the prod namespace
replicas:
  - name: orders-api
    count: 10                  # patch replica count for prod
images:
  - name: orders-api
    newTag: git-9f3a1c2        # pin the promoted image
patches:
  - path: resources-patch.yaml # bump CPU/mem, add HPA, PDB, NetworkPolicy
```

```
kubectl apply -k overlays/prod    # -k = build the kustomization and apply the result
│             │  │
│             │  └── the overlay directory (base + prod patches)
│             └── kustomize mode
└── kubectl
```

## Example 1 — basic

Three Namespaces on one cluster with quotas — the cheapest way to start.

```bash
kubectl create namespace orders-dev
kubectl create namespace orders-staging
kubectl create namespace orders-prod
```

```yaml
# quota for dev — keep experiments from eating the cluster
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: orders-dev
spec:
  hard:
    requests.cpu: "2"          # dev namespace can request at most 2 cores total
    requests.memory: 4Gi
    pods: "20"
```

Deploy the same chart into each, changing only the values file and namespace:

```bash
helm upgrade --install orders-api ./charts/orders-api -f values-dev.yaml     -n orders-dev
helm upgrade --install orders-api ./charts/orders-api -f values-staging.yaml -n orders-staging
helm upgrade --install orders-api ./charts/orders-api -f values-prod.yaml    -n orders-prod
```

## Example 2 — production scenario

**Context:** Your team ships `orders-api` several times a week. You use **GitOps** (Topic 52) with Argo CD and separate clusters for staging and prod. Here's the promotion flow your team actually operates:

1. A dev merges a PR. CI builds `orders-api:git-9f3a1c2`, pushes to GHCR, and **opens a PR** that bumps the image tag in `env/staging/values.yaml` in the *config repo*.
2. That PR auto-merges; **Argo CD** (watching the config repo) syncs staging to the new tag. Staging is prod-like, so this is a realistic test.
3. E2e + load tests run against staging. Green.
4. A release manager opens a PR bumping the tag in `env/prod/values.yaml` — **the exact same `git-9f3a1c2`**. This PR requires **two approvals** (enforced by branch protection) — the human gate before prod.
5. On merge, Argo CD syncs prod with a rolling update. PodDisruptionBudgets (Topic 46) and readiness probes (Topic 43) guarantee zero downtime.
6. Grafana SLO dashboards (Topic 51) are watched for 15 minutes. Argo CD's UI shows prod is `Synced / Healthy`.

**Why this is safe:**
- **Same digest** flowed dev → staging → prod; no "works in staging, breaks in prod because of a rebuild".
- **Config, not code, differs per env** — replicas, resources, secret references (Topic 50) — all in Git, all reviewed.
- **Prod is gated** by human approval + RBAC (Topic 53): CI's service account can deploy to staging automatically but **cannot** write to prod without the approved PR.
- **Drift is impossible** — Argo CD continuously reconciles each env's live state to Git (Topic 30). If someone `kubectl edit`s prod by hand, Argo reverts it and flags the drift.

**A common failure this prevents:** someone hotfixes prod directly with `kubectl edit`, it works, but the change isn't in Git — the next deploy silently reverts it and the bug returns. GitOps + multi-env config-in-Git makes that class of "config drift" bug structurally impossible.

## Common mistakes

**Mistake 1 — Rebuilding the image per environment.**

```
# CI builds a fresh image for prod from the same commit
docker build -t orders-api:prod .
```

**Root cause:** two builds of "the same" code can differ (dependency drift, base image moved, build-time timestamps). You tested staging's bytes but shipped *different* bytes to prod. → **Fix:** build **once**, tag by immutable git SHA/digest, and promote that *exact* artifact through every environment.

**Mistake 2 — `latest` tag across environments.**

```yaml
image: ghcr.io/acme/orders-api:latest
```

**Root cause:** `latest` is mutable — dev, staging, and prod may all resolve to *different* images at different times, and you can't tell which commit is running or roll back deterministically (Topic 24). → **Fix:** pin immutable tags (`git-<sha>` or the `@sha256:` digest). Never deploy `latest`.

**Mistake 3 — Secrets copied by hand between environments.**

Pasting prod DB credentials into a dev ConfigMap "to test quickly." Now prod secrets leak into a low-trust environment and often into Git. → **Fix:** per-environment secret *references* only (Topic 50) — External Secrets Operator / Vault / Sealed Secrets, with each env pulling from its own secret store. Dev never holds prod credentials.

**Mistake 4 — Staging that doesn't resemble prod.**

Staging runs 1 replica, no HPA, an in-memory DB; prod runs 20 replicas, HPA, Postgres. A concurrency bug or connection-pool bug appears **only** in prod because staging never exercised that shape. → **Fix:** make staging *prod-like* in topology (same DB engine, similar replica count, same probes/quotas) even if smaller. The whole point of staging is to be a faithful dress rehearsal.

**Mistake 5 — No environment isolation (shared database).**

dev and prod point at the *same* Postgres to "save money." A dev test truncates a table and destroys prod data. → **Fix:** every environment gets its own datastore and network boundary (separate Namespace + NetworkPolicy, or separate cluster/account). Isolation is the point of having environments at all.

## Hands-on proof

```bash
# 1. Create three namespaces = three environments on one cluster
for e in dev staging prod; do kubectl create namespace orders-$e; done

# 2. Deploy the SAME chart with different values into each
helm upgrade --install orders-api ./charts/orders-api -f values-dev.yaml     -n orders-dev
helm upgrade --install orders-api ./charts/orders-api -f values-prod.yaml    -n orders-prod

# 3. PROVE only config differs — compare rendered manifests, not rebuilt images
helm template orders-api ./charts/orders-api -f values-dev.yaml  | grep -E 'replicas|image:'
helm template orders-api ./charts/orders-api -f values-prod.yaml | grep -E 'replicas|image:'
#   → same image line, different replicas/resources

# 4. PROVE namespace isolation (DNS scoping, Topic 37)
kubectl run -it --rm probe --image=busybox -n orders-dev -- \
  nslookup orders-api.orders-prod.svc.cluster.local   # cross-namespace name is fully-qualified

# 5. PROVE quota isolation
kubectl describe resourcequota -n orders-dev

# 6. See per-env RBAC (Topic 53): CI can deploy to staging but not prod
kubectl auth can-i create deployments -n orders-prod --as=system:serviceaccount:ci:deployer
```

## Practice exercises

### Exercise 1 — easy
Create `orders-dev`, `orders-staging`, `orders-prod` Namespaces. Deploy the same simple app into all three using a single manifest but overriding replica count per namespace (via `kubectl scale` or three values files). Confirm with `kubectl get deploy -A` that the image is identical everywhere but the replica counts differ.

### Exercise 2 — medium
Build a Helm chart with `values.yaml`, `values-dev.yaml`, and `values-prod.yaml`. Make prod enable an HPA (Topic 45) and larger resources, dev use 1 replica and debug logging. Use `helm template` to *diff* the rendered output of dev vs prod and confirm the **image tag line is identical** while the env-specific fields differ. Add a `ResourceQuota` to dev.

### Exercise 3 — hard (production simulation)
Design (and, if you have Argo CD/Flux, implement) a GitOps promotion pipeline: one config repo with `env/dev`, `env/staging`, `env/prod` folders. Write a flow where CI bumps the image tag in `env/staging` automatically, tests run, and a *manually approved* PR bumps the **same** tag in `env/prod`. Then simulate a "cowboy" `kubectl edit` on the prod Deployment and show that the GitOps controller reverts it (proving drift protection). Document what RBAC (Topic 53) prevents CI from touching prod directly.

## Mental model checkpoint

1. What is the one invariant that must hold as a change moves dev → staging → prod?
2. Namespaces vs separate clusters — what does each isolate, and what are the trade-offs?
3. Why is `latest` a dangerous tag in a multi-environment world?
4. What *should* differ between environments, and what should *never* differ?
5. Why must staging resemble production, and what bugs does a non-prod-like staging hide?
6. How do GitOps + config-in-Git make "config drift" structurally impossible?
7. Which earlier topics (image tagging, secrets, RBAC, Helm, rollouts) each play a role in safe promotion, and why?

## Quick reference card

| Concept | What it does | Key detail |
|---|---|---|
| dev / staging / prod | The three standard environments | Same image, different config |
| Namespace-per-env | Cheap, soft isolation on one cluster | Logical isolation via RBAC + quotas |
| Cluster-per-env | Strong isolation | Separate etcd/network/account; more cost |
| Build once, promote | Same image digest everywhere | Removes "rebuild differed" as a bug cause |
| Immutable tags (`git-<sha>`) | Deterministic, rollback-able | Never `latest` (Topic 24) |
| Helm values-per-env | Config deltas per environment | `-f values-prod.yaml --set image.tag=...` |
| Kustomize overlays | Base + per-env patches | `kubectl apply -k overlays/prod` |
| ResourceQuota | Cap an env's resource use | Prevents noisy-neighbor across namespaces |
| GitOps (Argo/Flux) | Reconcile live state to Git | Auto-reverts manual drift (Topic 52) |
| Human approval gate | Manual step before prod | Enforced by branch protection + RBAC (Topic 53) |

## When would I use this at work?

1. **Standing up a new service.** You define the dev/staging/prod story for `orders-api` on day one — Namespaces or clusters, one chart with three values files, immutable image tags — so the team never falls into "cowboy prod".
2. **Making releases safe and boring.** A change flows through automated dev/staging tests and a gated prod promotion of the *same image*, so shipping multiple times a day is low-drama and every deploy is rollback-able.
3. **Passing an audit / incident review.** When asked "how do you know prod runs what you tested?" you point at immutable digests promoted through environments and Git history — every prod change is reviewed, attributable, and reversible.

## Connected topics

- **Study before:** Topic 37 (Namespaces — the isolation unit), Topic 49 (Helm — values-per-env), Topic 50 (Secrets — per-env secret refs), Topic 52 (CI/CD & GitOps — the promotion engine), Topic 24 (image tagging — immutable artifacts), Topic 53 (RBAC — gating prod).
- **Study after:** This is the capstone — revisit Topic 46 (rolling updates in prod), Topic 51 (observability per environment), and Topic 45 (autoscaling enabled only where it belongs) with the whole picture in mind.
