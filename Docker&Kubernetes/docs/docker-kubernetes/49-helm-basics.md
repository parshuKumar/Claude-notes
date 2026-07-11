# 49 — Helm Basics

## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Imagine you love a particular sandwich. Every time you want it, you go to the
kitchen and repeat the same 12 steps: get bread, spread butter, add cheese, add
tomato, toast for 4 minutes, cut diagonally… If you make it for 3 friends, you
do all 12 steps three times, and if you forget the tomato once, that sandwich
comes out wrong.

Now imagine a **recipe card**. The card has the fixed steps written down, and at
the top it has **blanks** you fill in: "cheese type: ____", "toast time: ____".
To make a sandwich you just hand the card to the kitchen with the blanks filled
in ("cheddar", "4 min") and out comes exactly the sandwich you wanted. Want a
slightly different one for a friend who hates tomato? Same card, different
blanks.

**Helm is that recipe-card system for Kubernetes.** The recipe card is called a
**chart**. The blanks are `values.yaml`. Handing the filled-in card to the
kitchen is `helm install`. And Helm remembers every sandwich it ever made for
you (a **release** with **revisions**), so if the new recipe is bad you can say
"give me back the sandwich from yesterday" — that's `helm rollback`.

---

## The Linux kernel feature underneath

Helm has **no kernel primitive of its own**. This is important to be honest
about: Helm is a **client-side templating and release-tracking tool**. It does
not run in the kernel, it does not talk to the scheduler, and it does not create
namespaces, cgroups, or network namespaces itself. Everything that actually
*runs* on a node — the pods, the cgroups (Topic 44), the veth pairs (Topic 40) —
is created by the same Kubernetes machinery you already learned, from ordinary
manifests.

So the "mechanism underneath" for Helm is really **two things**:

**1. The Kubernetes API server + etcd (Topic 29).** Helm's whole job ends at
"produce plain YAML manifests and POST them to the API server." Once the objects
are in etcd, normal controllers (Topic 30 reconciliation loop) take over. Helm
is just a very smart `kubectl apply` wrapper.

```
   helm install          renders templates          kubectl-style apply
  ┌────────────┐  Go     ┌──────────────────┐  YAML  ┌──────────────────┐
  │ chart +    │ ─────▶  │ fully-rendered   │ ─────▶ │ API server → etcd │
  │ values.yaml│ templ.  │ Deployment/Svc/… │        │ (Topic 29 / 30)  │
  └────────────┘         └──────────────────┘        └──────────────────┘
        ↑ recipe + blanks      ↑ plain manifests            ↑ real objects
```

**2. Where Helm stores its own memory: Kubernetes Secrets.** Helm 3 needs to
remember what it installed and every past revision. It stores that as a **Secret
of type `helm.sh/release.v1`** in the same namespace as the release. Inside that
Secret is a **gzip-compressed, base64-encoded** blob of the rendered manifests
and metadata. So Helm's "database" is literally rows in etcd, encoded like this:

```
$ kubectl -n orders get secret -l owner=helm
NAME                              TYPE                 DATA
sh.helm.release.v1.orders.v1      helm.sh/release.v1   1
sh.helm.release.v1.orders.v2      helm.sh/release.v1   1   ← each revision = one Secret
```

That is the entire "physical reality" of a Helm release: **one Secret per
revision**, sitting in etcd like everything else. There is no Helm daemon
running on your cluster (that was Helm 2's "Tiller", removed in Helm 3 for
security). Helm runs entirely as a CLI on your laptop or in CI.

---

## What is this?

**Helm is the package manager for Kubernetes** — think `apt`/`npm`, but for
clusters. A **chart** is a versioned bundle of templated Kubernetes manifests
plus a `values.yaml` file of defaults. You `helm install` a chart to create a
**release** (a named, running instance), `helm upgrade` to change it, and
`helm rollback` to jump back to any previous **revision**. Helm renders Go
templates into plain YAML and applies it to the cluster.

---

## Why does it matter for a backend developer?

Without Helm, deploying `orders-api` means hand-maintaining a pile of nearly
identical YAML files, and the pain shows up fast:

- **Copy-paste sprawl.** `orders-api` needs a Deployment, Service, ConfigMap,
  Secret, HPA, Ingress — that's 6 files. Now do dev, staging, prod: 18 files,
  95% identical, differing only in replica count, image tag, and hostnames.
  Someone will edit the staging file and forget prod. Helm makes it **one chart,
  three `values` files**.
- **Unsafe upgrades.** With raw `kubectl apply` you have no built-in "undo." If
  a new image tag crashes every pod at 2 a.m., you're grepping git history for
  the old tag. Helm gives you `helm rollback orders 4` — one command, back to
  revision 4, in seconds.
- **No atomic view of "the app."** Your app is really 6+ objects. `kubectl` sees
  them as unrelated. Helm treats them as **one release** you can install,
  upgrade, inspect, and delete as a unit (`helm uninstall orders` removes all 6).
- **Reusing other people's infra.** Need Redis or Postgres for `orders-api`? A
  public chart (e.g. Bitnami) installs a production-grade StatefulSet, Service,
  and Secret with one command instead of you authoring 200 lines of YAML.

If you have ever kept `deployment-prod.yaml` and `deployment-staging.yaml` in
sync by hand and gotten it wrong, Helm is the fix.

---

## The physical reality

Two separate "physical" locations matter: **what's on your disk** (the chart)
and **what's in the cluster** (the release Secrets).

### On disk — the chart directory

A chart is just a directory with a required shape:

```
orders-api/                     ← chart root (the recipe box)
├── Chart.yaml                  ← chart metadata: name, version, appVersion
├── values.yaml                 ← default "blanks" (image tag, replicas, ports…)
├── templates/                  ← the recipe cards (templated manifests)
│   ├── deployment.yaml         ← Deployment with {{ }} placeholders
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── _helpers.tpl            ← reusable named template snippets (no output)
│   └── NOTES.txt               ← message printed after install
├── charts/                     ← subcharts (dependencies, e.g. redis) live here
└── .helmignore                 ← like .dockerignore: files to exclude when packaging
```

When you run `helm package orders-api/`, Helm tars+gzips this into
`orders-api-0.1.0.tgz` — that `.tgz` is the shippable artifact you push to a
**chart repository** (a plain HTTP server hosting an `index.yaml`, or an OCI
registry like Docker Hub / ECR).

### In the cluster — the release Secrets

After `helm install orders ./orders-api -n orders`, the cluster holds:

```
$ kubectl -n orders get all,secret -l app.kubernetes.io/instance=orders
deployment.apps/orders-api      ← the rendered Deployment
service/orders-api              ← the rendered Service
configmap/orders-api-config     ← the rendered ConfigMap
secret/sh.helm.release.v1.orders.v1   ← Helm's OWN record of revision 1

# Decode Helm's memory of what it deployed:
$ kubectl -n orders get secret sh.helm.release.v1.orders.v1 \
    -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip | head -c 200
{"name":"orders","info":{"status":"deployed","first_deployed":...},"manifest":"---\n# Source: orders-api/templates/deployment.yaml\n..."}
```

That double-decode (`base64 -d | base64 -d | gunzip`) is real: Helm base64s the
gzipped JSON, and Kubernetes base64s Secret data on top. Inside you find the
**fully rendered manifest** and the status. This is how `helm rollback` works —
it reads an old revision's Secret and re-applies the manifest stored inside.

---

## How it works — step by step

Full trace of `helm install orders ./orders-api -n orders -f values-prod.yaml`:

1. **Helm loads the chart from disk.** It reads `Chart.yaml` (to learn the chart
   name/version), `values.yaml` (defaults), and every file under `templates/`.
   It also pulls in any subcharts under `charts/`.

2. **Helm merges values.** It layers value sources in a strict order, each
   overriding the one before: chart's `values.yaml` → any parent-chart values →
   `-f values-prod.yaml` files (left to right) → `--set key=val` flags. The
   result is one final "values" object.

3. **Helm builds the render context.** It assembles the objects your templates
   can reference: `.Values` (the merged values), `.Release` (name=orders,
   namespace=orders, revision=1, `.IsInstall`=true), `.Chart` (from Chart.yaml),
   `.Capabilities` (the cluster's API versions), and `.Files` (non-template
   files in the chart).

4. **Helm renders every template with Go's `text/template` engine.** Each `{{ }}`
   is evaluated. `{{ .Values.replicaCount }}` becomes `3`. Loops, conditionals,
   and functions (`| quote`, `| default`, `| nindent`) run. Named templates in
   `_helpers.tpl` are expanded. Files starting with `_` produce **no** manifest;
   `NOTES.txt` is rendered but only printed, not applied.

5. **Helm validates the output.** It parses the rendered text as YAML, checks
   each doc has `apiVersion`/`kind`, and (unless `--disable-openapi-validation`)
   checks it against the cluster's schema. Bad templates fail **here**, before
   anything touches the cluster.

6. **Helm orders the resources.** It sorts objects into a sane install order
   (Namespace → NetworkPolicy → ... → ConfigMap/Secret → ... → Deployment →
   Service → ...) so dependencies exist first. This ordering is built into Helm.

7. **Helm sends the manifests to the API server.** Effectively a create/apply
   for each object. Normal Kubernetes reconciliation (Topic 30) then schedules
   pods, pulls images, and starts containers — Helm's job is essentially done.

8. **Helm records the release.** It writes `sh.helm.release.v1.orders.v1` (a
   Secret) containing the rendered manifest, the merged values, and
   `status: deployed`. This is revision 1.

9. **Helm prints `NOTES.txt`.** The rendered notes (e.g. "Your API is at
   http://orders.example.com") appear in your terminal.

On a later `helm upgrade`, steps 1–7 repeat with `.Release.Revision=2` and
`.IsUpgrade=true`, a **new** Secret `...orders.v2` is written, and the old one is
kept so rollback still works. On `helm rollback orders 1`, Helm reads the v1
Secret, re-applies its stored manifest, and writes a **new** revision (v3) that
is a copy of v1 — revisions only ever move forward in number.

---

## Exact syntax breakdown

### `Chart.yaml` — the identity card

```yaml
apiVersion: v2
│           └─ v2 = Helm 3 chart format (v1 was Helm 2). Always v2 today.
name: orders-api
│     └─ the chart's name. Used in default resource names and the .tgz filename.
description: The orders-api service (Express + Postgres + Redis)
type: application
│     └─ "application" (installable) vs "library" (shared helpers, not installable)
version: 0.1.0
│        └─ the CHART version (semver). Bump when you change templates.
appVersion: "1.4.2"
│           └─ the version of the APP inside (your orders-api image tag). Quoted
│              so "1.10" isn't read as a float. Purely informational by default.
dependencies:
  - name: redis
    │     └─ a subchart to pull in (e.g. Bitnami redis) under charts/
    version: "19.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled   ← only install if .Values.redis.enabled is true
```

### `values.yaml` — the default blanks

```yaml
replicaCount: 2
│            └─ referenced in templates as {{ .Values.replicaCount }}
image:
  repository: myregistry/orders-api
  │           └─ {{ .Values.image.repository }} — NESTED access with dots
  tag: ""          # empty → template falls back to .Chart.AppVersion
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
  targetPort: 3000   ← the container port orders-api listens on
resources:
  requests: { cpu: 100m, memory: 128Mi }   ← Topic 44
  limits:   { cpu: 500m, memory: 256Mi }
```

### The template action — Go templating inside a manifest

```
replicas: {{ .Values.replicaCount }}
│         │  │└──────┬─────────────┘ │
│         │  │       │               └─ }} closes the action
│         │  │       └─ the PATH: .Values (merged values) → key replicaCount
│         │  └─ leading dot = "root context"; .Values is always available
│         └─ {{ opens a template ACTION (code that produces text)
└─ literal YAML key; everything left of {{ is emitted as-is
```

### A pipeline with functions

```
name: {{ .Values.name | default "orders-api" | quote }}
│     │              │ │                     │ │
│     │              │ │                     │ └─ quote: wrap result in "..."
│     │              │ │                     └─ pipe: send left value into next fn
│     │              │ └─ default "orders-api": use this if the value is empty
│     │              └─ pipe: feed .Values.name into the default function
│     └─ the value we start with
└─ result for empty name → name: "orders-api"
```

### `nindent` — the indentation workhorse

```
      labels:
        {{- include "orders-api.labels" . | nindent 8 }}
        │  │       │                    │ │       │  └─ indent by 8 spaces
        │  │       │                    │ │       └─ nindent = newline + indent
        │  │       │                    │ └─ pass current context (.) to the template
        │  │       │                    └─ include: render a NAMED template
        │  │       └─ the named template defined in _helpers.tpl
        │  └─ include a reusable label block
        └─ {{-  the dash trims the whitespace/newline BEFORE the action
```

`{{-` trims preceding whitespace, `-}}` trims following whitespace. Getting
these wrong is the #1 cause of "YAML indentation" errors in Helm.

### The install command

```
helm install orders ./orders-api -n orders -f values-prod.yaml --create-namespace
│    │       │      │             │  │      │  │                └─ make ns if missing
│    │       │      │             │  │      │  └─ values file (overrides defaults)
│    │       │      │             │  │      └─ -f: supply a values file
│    │       │      │             │  └─ namespace to install into
│    │       │      │             └─ -n: target namespace
│    │       │      └─ chart location: a local dir (could be repo/chart or a .tgz)
│    │       └─ RELEASE NAME — your name for this running instance
│    └─ the subcommand: create a new release
└─ the Helm CLI
```

### Upgrade and rollback

```
helm upgrade orders ./orders-api -n orders --set image.tag=1.5.0 --atomic
│    │       │                            │  │                    └─ roll back
│    │       │                            │  │                       automatically
│    │       │                            │  │                       if upgrade fails
│    │       │                            │  └─ --set: override one value inline
│    │       │                            └─ (image.tag beats values.yaml + -f files)
│    │       └─ SAME release name → this is a change to the existing app
│    └─ change an existing release (creates revision N+1)
└─ helm

helm rollback orders 4 -n orders
│    │        │      │
│    │        │      └─ REVISION number to return to (from `helm history`)
│    │        └─ the release to roll back
│    └─ revert to a previous revision (creates a NEW revision that copies it)
└─ helm
```

---

## Example 1 — basic

A minimal `orders-api` chart. Four files, every line explained.

**`orders-api/Chart.yaml`**
```yaml
apiVersion: v2                 # Helm 3 chart format
name: orders-api               # chart name
description: Orders API service
type: application              # installable chart
version: 0.1.0                 # chart version (bump on template changes)
appVersion: "1.0.0"            # the orders-api app version (image tag default)
```

**`orders-api/values.yaml`**
```yaml
replicaCount: 2                # how many orders-api pods
image:
  repository: myregistry/orders-api
  tag: ""                      # empty ⇒ template uses appVersion (1.0.0)
  pullPolicy: IfNotPresent
service:
  port: 80                     # the Service port
  targetPort: 3000             # the container port orders-api listens on
```

**`orders-api/templates/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-api          # → orders-api  (.Release.Name = "orders")
  namespace: {{ .Release.Namespace }}    # the ns you installed into (orders)
spec:
  replicas: {{ .Values.replicaCount }}   # → 2  (from values.yaml)
  selector:
    matchLabels:
      app: {{ .Release.Name }}-api       # ties Deployment to its pods (Topic 38)
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-api
    spec:
      containers:
        - name: orders-api
          # use image.tag, but if empty fall back to Chart.AppVersion:
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}   # → 3000
```

**`orders-api/templates/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-api          # orders-api
spec:
  selector:
    app: {{ .Release.Name }}-api         # sends traffic to those pods
  ports:
    - port: {{ .Values.service.port }}         # 80  (Service port)
      targetPort: {{ .Values.service.targetPort }}  # 3000 (container port)
```

Install and inspect it:

```bash
# See EXACTLY what will be applied — render locally, apply nothing:
helm template orders ./orders-api -n orders

# Install for real:
helm install orders ./orders-api -n orders --create-namespace

# Confirm the release exists:
helm list -n orders
# NAME     NAMESPACE  REVISION  STATUS     CHART             APP VERSION
# orders   orders     1         deployed   orders-api-0.1.0  1.0.0
```

---

## Example 2 — production scenario

**The situation:** Your team runs `orders-api` in **three environments** — dev,
staging, prod — that differ only in replica count, image tag, resource limits,
and whether debug logging is on. You currently keep three near-identical folders
of YAML. Last week someone bumped the image in staging, forgot prod, and prod
sat on a two-week-old build during a customer demo. You are switching to **one
chart + one values file per environment**.

**One chart, three values files.** The chart's `templates/` stay the same; only
the blanks change.

**`values-dev.yaml`**
```yaml
replicaCount: 1
image: { tag: "dev-latest" }
env:   { LOG_LEVEL: "debug" }
resources:
  requests: { cpu: 50m, memory: 64Mi }
```

**`values-prod.yaml`**
```yaml
replicaCount: 6
image: { tag: "1.4.2" }             # pinned, immutable tag — never "latest" in prod
env:   { LOG_LEVEL: "info" }
resources:
  requests: { cpu: 250m, memory: 256Mi }
  limits:   { cpu: "1",  memory: 512Mi }
```

**A template that adapts** — `templates/deployment.yaml` (excerpt):
```yaml
spec:
  replicas: {{ .Values.replicaCount }}          # 1 in dev, 6 in prod
  template:
    spec:
      containers:
        - name: orders-api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: LOG_LEVEL
              value: {{ .Values.env.LOG_LEVEL | quote }}   # "debug" vs "info"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}   # whole block from values
          # Force a rolling restart when config changes, by hashing the ConfigMap:
          # (see checksum annotation trick below)
```

**Deploy each environment** — the release name and namespace change, the chart
does not:
```bash
helm upgrade --install orders-dev  ./orders-api -n orders-dev  \
  -f values-dev.yaml  --create-namespace
helm upgrade --install orders-prod ./orders-api -n orders      \
  -f values-prod.yaml --create-namespace --atomic --timeout 3m
```

`upgrade --install` is the idempotent workhorse: install if it does not exist,
upgrade if it does. Perfect for CI (Topic 52) — the same command works on the
first deploy and the hundredth.

**The rollback that saves the demo.** You ship `1.4.3` to prod and pods
CrashLoop because of a bad migration. Because you used `--atomic`, Helm **already
rolled the failed upgrade back automatically**. If you needed to do it by hand:

```bash
helm history orders-prod -n orders
# REVISION  UPDATED    STATUS      CHART             APP VERSION  DESCRIPTION
# 1         Mon 10:00  superseded  orders-api-0.1.0  1.4.2        Install complete
# 2         Tue 09:00  superseded  orders-api-0.1.0  1.4.2        Upgrade complete
# 3         Wed 14:00  failed      orders-api-0.2.0  1.4.3        Upgrade failed
# 4         Wed 14:01  deployed    orders-api-0.1.0  1.4.2        Rollback to 2

helm rollback orders-prod 2 -n orders   # explicit: go back to the known-good rev 2
```

**Bonus — auto-restart pods on ConfigMap change.** A classic gotcha: editing a
ConfigMap does **not** restart pods that mount it as env vars. Add a checksum
annotation so any config change forces a new pod template hash (Topic 46):
```yaml
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
Now `helm upgrade` after a config edit triggers a rolling update automatically.

**Result:** environments can no longer drift, because they share one chart. The
only difference is the values file, which lives in git and is reviewed in a PR.

---

## Common mistakes

### Mistake 1 — Broken indentation from missing `nindent`

```yaml
      labels:
        {{ toYaml .Values.labels | indent 8 }}
```
```
$ helm install orders ./orders-api -n orders
Error: YAML parse error on orders-api/templates/deployment.yaml:
  error converting YAML to JSON: yaml: line 14: did not find expected key
```
**Root cause:** `indent` adds spaces to **every** line including the first, but
the first line is already positioned right after `{{`, so it ends up
double-indented and the block misaligns. **Fix:** use `nindent` (newline +
indent) and put the action on its own line with `{{-` to trim the leading
newline:
```yaml
      labels:
        {{- toYaml .Values.labels | nindent 8 }}
```

### Mistake 2 — `appVersion` unquoted, read as a number

```yaml
appVersion: 1.10
```
Your image tag renders as `1.1` because YAML parses `1.10` as the float `1.10 =
1.1`. **Fix:** always quote it: `appVersion: "1.10"`. Same for any value that
looks numeric but is a string (`"true"`, `"08"`, phone numbers, git SHAs).

### Mistake 3 — Editing live objects with `kubectl` instead of Helm

```
$ kubectl -n orders scale deployment orders-api --replicas=10   # manual hotfix
$ helm upgrade orders ./orders-api -n orders                    # next deploy
# ... pods drop back to replicaCount: 2
```
**Root cause:** Helm re-applies the **rendered chart**, which still says
`replicas: 2`. Your manual change is silently overwritten. **Fix:** treat the
chart/values as the single source of truth — change `replicaCount` in the values
file and `helm upgrade`, never `kubectl edit` a Helm-managed object.

### Mistake 4 — `helm install` fails: release already exists

```
$ helm install orders ./orders-api -n orders
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
```
**Root cause:** `install` is for **new** releases only; a release named `orders`
already exists in that namespace. **Fix:** use `helm upgrade orders ...` to
change it, or `helm upgrade --install orders ...` to do "install-or-upgrade" in
one idempotent command (what CI should always use).

### Mistake 5 — A failed upgrade leaves the release "stuck"

```
$ helm upgrade orders ./orders-api -n orders
Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
```
**Root cause:** a previous `helm` process was killed mid-upgrade, leaving the
release in a `pending-upgrade` state. **Fix:** `helm rollback orders -n orders`
to return to the last good revision, or in newer Helm use
`helm upgrade ... --force`, and going forward add `--atomic --timeout 5m` so a
failed upgrade cleans itself up instead of getting stuck.

---

## Hands-on proof

Run these right now (needs a cluster — `kind`/`minikube` is fine — and Helm 3).

```bash
# 1) Scaffold a real chart (Helm writes a working starter chart):
helm create orders-api
ls orders-api/                 # Chart.yaml  values.yaml  templates/  charts/  .helmignore

# 2) See the recipe rendered WITHOUT touching the cluster:
helm template demo ./orders-api | head -40      # plain YAML, {{ }} all resolved

# 3) Install it and prove a release + Secret were created:
kubectl create namespace orders
helm install orders ./orders-api -n orders --set replicaCount=2
helm list -n orders                              # STATUS = deployed, REVISION = 1
kubectl -n orders get secret -l owner=helm       # sh.helm.release.v1.orders.v1

# 4) Read Helm's OWN memory of the release (the double-decode is real):
kubectl -n orders get secret sh.helm.release.v1.orders.v1 \
  -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip | head -c 300; echo

# 5) Upgrade → watch the revision number tick to 2, and a v2 Secret appear:
helm upgrade orders ./orders-api -n orders --set replicaCount=3
helm history orders -n orders                    # two rows now
kubectl -n orders get secret -l owner=helm       # ...orders.v1 AND ...orders.v2

# 6) Roll back to revision 1 → creates revision 3 (a copy of 1):
helm rollback orders 1 -n orders
helm history orders -n orders                    # rev 3, DESCRIPTION "Rollback to 1"
kubectl -n orders get deploy -o jsonpath='{.items[0].spec.replicas}'; echo   # → 2

# 7) See values precedence: --set beats -f beats values.yaml:
helm get values orders -n orders                 # what values are ACTUALLY in effect

# 8) Clean up — one command removes ALL the release's objects:
helm uninstall orders -n orders
```

---

## Practice exercises

### Exercise 1 — easy
Run `helm create orders-api`, then open `templates/deployment.yaml`. Change
`replicaCount` in `values.yaml` to 3 and run `helm template demo ./orders-api |
grep replicas`. Confirm the rendered output shows `replicas: 3` **without ever
installing**. Then override it on the command line with `--set replicaCount=5`
and prove `--set` wins.

### Exercise 2 — medium
Take the four-file `orders-api` chart from Example 1. Add a `configmap.yaml`
template that renders `LOG_LEVEL` and `DATABASE_URL` from `.Values.env`. Wire the
Deployment to load them via `envFrom.configMapRef`. Then add the
`checksum/config` annotation so that editing a value and running `helm upgrade`
triggers a rolling restart. Prove it by changing `LOG_LEVEL`, upgrading, and
watching `kubectl -n orders rollout status`.

### Exercise 3 — hard (production simulation)
Build one chart that deploys `orders-api` to **dev** and **prod** with
`values-dev.yaml` (1 replica, `dev-latest`, debug logs) and `values-prod.yaml`
(6 replicas, pinned `1.4.2`, `--atomic`). Deploy prod, then simulate a bad
release: `helm upgrade orders-prod ./orders-api -n orders --set image.tag=does-not-exist
--atomic --timeout 60s`. Prove that `--atomic` automatically rolled the failed
upgrade back (check `helm history` for a `Rollback` row and confirm pods still
run the old `1.4.2` image). Finally, do a manual `helm rollback` to an even
earlier revision and confirm the running image changed.

---

## Mental model checkpoint

Answer from memory:

1. Where does Helm store its record of each release, and in what encoding?
2. Is there a Helm server component running in your cluster in Helm 3? What was
   removed from Helm 2 and why?
3. In what order do value sources override each other (values.yaml, -f files,
   --set)?
4. What is the difference between the chart `version` and `appVersion` in
   `Chart.yaml`?
5. What does `helm rollback orders 4` actually do to the revision numbers?
6. Why does `helm upgrade` overwrite a `kubectl scale` you did by hand?
7. What is the difference between `helm install`, `helm upgrade`, and
   `helm upgrade --install`?

---

## Quick reference card

| Command / field | What it does | Key detail |
|---|---|---|
| `helm create NAME` | Scaffold a working starter chart | Great for learning template structure |
| `helm template REL ./chart` | Render manifests locally | Applies nothing — debug-safe |
| `helm install REL ./chart -n NS` | Create a new release (revision 1) | Fails if release name already exists |
| `helm upgrade REL ./chart -n NS` | Change a release (revision N+1) | Keeps old revision Secrets for rollback |
| `helm upgrade --install REL ...` | Install-or-upgrade | Idempotent — use this in CI |
| `helm rollback REL REV -n NS` | Return to a prior revision | Creates a NEW revision copying the old |
| `helm history REL -n NS` | List all revisions + status | Read revision numbers here |
| `helm list -n NS` | List releases in a namespace | Shows status, chart, app version |
| `helm get values REL -n NS` | Show values in effect | Debug value precedence |
| `helm uninstall REL -n NS` | Delete the whole release | Removes all its objects at once |
| `-f file.yaml` / `--set k=v` | Override values | `--set` beats `-f` beats `values.yaml` |
| `--atomic` | Auto-rollback on failed upgrade | Pair with `--timeout` |
| `Chart.yaml: version` | Chart version | Bump on template changes |
| `Chart.yaml: appVersion` | App/image version | Quote it to avoid float bugs |

---

## When would I use this at work?

1. **Managing dev/staging/prod for `orders-api`** with one chart and three
   values files, so environments can't silently drift and every change is a
   reviewed PR to a values file.
2. **Fast, safe rollbacks during incidents** — `helm rollback orders 4` (or
   `--atomic` doing it automatically) beats grepping git for the last-good image
   tag while orders pile up.
3. **Pulling in dependencies** like Redis or Postgres for `orders-api` as
   subcharts (Bitnami), getting a production-grade StatefulSet + Service + Secret
   without hand-writing hundreds of lines of YAML.

---

## Connected topics

- **Study before:** Topic 32 (YAML and Manifests) — Helm just produces these.
  Topic 34 (Deployments) and Topic 35 (Services) — the objects your templates
  render. Topic 36 (ConfigMaps and Secrets) — Helm stores its release state as a
  Secret. Topic 30 (The Reconciliation Loop) — what runs your objects after Helm
  applies them.
- **Study after:** Topic 50 (Secrets Management at Scale) — how to keep real
  secret *values* out of your chart and git. Topic 52 (CI/CD with Kubernetes) —
  `helm upgrade --install` in a pipeline and GitOps. Topic 54
  (Multi-Environment Strategy) — values-per-environment promotion workflows.
