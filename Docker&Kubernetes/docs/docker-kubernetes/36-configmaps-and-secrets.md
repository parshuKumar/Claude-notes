# 36 — ConfigMaps and Secrets
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Imagine you build a toy robot and ship the exact same robot to a thousand kids. But every kid's house is different: some have the light switch on the left, some on the right, some speak English, some Spanish. You do **not** build a thousand different robots. You build **one** robot and give each kid a little **settings card** they slot into the back: "light switch = left", "language = Spanish". Same robot, different card, different behavior.

Your `orders-api` image is the robot. The **settings card** is a **ConfigMap** — non-secret settings like the log level, the Postgres hostname, the Redis port. And when the card has *private* stuff on it — the database password, an API key — you don't leave it lying on the kitchen table. You put it in a small **locked drawer** and hand the robot only what it needs, when it needs it. That locked drawer is a **Secret**.

The whole point: the robot (image) stays the same in dev, staging, and prod. Only the card changes. That is the entire idea behind externalizing configuration.

One warning, and it's the most misunderstood thing in all of Kubernetes: the "locked drawer" (Secret) is, by default, **not actually locked**. It's just a drawer with a slightly opaque door. We'll rip that illusion apart later in this doc.

---

## The Linux kernel feature / mechanism underneath

ConfigMaps and Secrets are pure-Kubernetes objects — there's no single kernel syscall called "configmap". But underneath the abstraction there are two very physical mechanisms, and you must understand both:

**1. etcd — the cluster's one source of truth.** Every ConfigMap and Secret you create is stored as a key in **etcd**, the distributed key-value database that backs the Kubernetes API server (Topic 29). When you `kubectl apply` a ConfigMap, the API server writes it to etcd at a key like `/registry/configmaps/orders/orders-config`. Nothing about a ConfigMap "runs" — it is just bytes sitting in etcd until a Pod asks for it.

**2. tmpfs — how Secrets reach a Pod without touching the node's disk.** When a Secret is mounted into a Pod as files, the kubelet does **not** write those files onto the node's real disk (`/var/lib/...`). It mounts a **tmpfs** — a RAM-backed filesystem (the same `tmpfs` you met in Topic 14). The secret bytes live in the node's memory, not on its SSD. When the Pod dies, the tmpfs is torn down and the RAM is reclaimed — the secret never leaves a trace on disk. This is a deliberate security choice by the kubelet.

```
$ kubectl exec orders-api-7d4 -n orders -- mount | grep secret
tmpfs on /var/run/secrets/db type tmpfs (rw,relatime)
│                            │
│                            └─ the mount type is tmpfs = RAM, not disk
└─ the secret files live here, in memory only
```

So the mental model: **config data rests in etcd, and is projected into Pods either as environment variables (copied once at container start) or as files (a live-ish view backed by tmpfs for Secrets).** Everything below is a consequence of those two facts.

---

## What is this?

A **ConfigMap** is a Kubernetes object that holds non-confidential configuration data as key-value pairs, so you can keep configuration *out* of your container image and inject it at runtime. A **Secret** is almost the same object, but meant for sensitive data (passwords, tokens, TLS keys); Kubernetes treats it a little more carefully — it's stored base64-encoded, mounted via tmpfs, and can be encrypted at rest.

Both let one image behave differently in different environments without rebuilding it.

---

## Why does it matter for a backend developer?

Remember the **12-factor app** rule from Topic 15: *store config in the environment, not in code*. In Docker you did this with `-e` and `--env-file`. Kubernetes gives you first-class objects for it.

Without ConfigMaps and Secrets you fall into three classic traps:

1. **You bake config into the image.** You hardcode `DB_HOST=prod-db.internal` in your Dockerfile. Now your dev image accidentally talks to prod. Or you build three separate images (`orders-api-dev`, `orders-api-staging`, `orders-api-prod`) — three times the build time, and the prod image was *never actually tested* because dev ran a different binary.

2. **You commit secrets to git.** You put the Postgres password in a `.env` file and push it. Now it's in your git history forever, visible to every contractor who ever clones the repo. This is the single most common way companies leak credentials.

3. **You can't rotate anything.** The DB password leaked. To change it you must rebuild and redeploy every image. With a Secret, you update one object and restart the Pods.

Concretely for `orders-api`: the log level, the Postgres host, the Redis host, feature flags → **ConfigMap**. The Postgres password, the Redis password, the JWT signing key, the Stripe API key → **Secret**. Same image, injected differently per namespace (`orders` in prod, `orders-dev` in dev).

---

## The physical reality

Let's make it concrete. You create a ConfigMap and a Secret in the `orders` namespace. Here's what physically exists.

**In etcd**, two keys:

```
/registry/configmaps/orders/orders-config     ← your ConfigMap, values in PLAIN TEXT
/registry/secrets/orders/orders-secrets        ← your Secret, values BASE64-encoded
```

Read the raw etcd bytes for the Secret and you get a shock:

```
$ ETCDCTL_API=3 etcdctl get /registry/secrets/orders/orders-secrets
...
DB_PASSWORD
c3VwZXItc2VjcmV0LXBhc3N3b3Jk          ← this is base64, NOT encryption
```

Run that base64 through a decoder and it says `super-secret-password` in plain text. **Base64 is not encryption. It is not a password. It is a reversible encoding anyone can undo in one second.** Hold that thought — it's the biggest gotcha in this whole topic.

**Inside a running Pod that mounts them as files:**

```
/etc/orders-config/LOG_LEVEL          ← file, contents: "info"          (from ConfigMap)
/etc/orders-config/DB_HOST            ← file, contents: "orders-postgres" (from ConfigMap)
/var/run/secrets/db/DB_PASSWORD       ← file on TMPFS, contents: "super-secret-password"
```

Each *key* becomes a *filename*, and each *value* becomes that file's *contents*. The ConfigMap files sit on a normal projected volume; the Secret files sit on a **tmpfs** (RAM) as we saw above.

**Inside a Pod that uses them as env vars** — nothing on disk at all, just entries in the process environment:

```
$ kubectl exec orders-api-7d4 -n orders -- env | grep -E 'LOG_LEVEL|DB_PASSWORD'
LOG_LEVEL=info                         ← copied from ConfigMap at container start
DB_PASSWORD=super-secret-password      ← copied from Secret at container start (now plain text in env!)
```

That's the entire physical footprint: two etcd keys, and either environment entries (copied once) or files (a live view, tmpfs for Secrets).

---

## How it works — step by step

Full trace of injecting config into `orders-api`, from `kubectl apply` to your Node process reading it.

1. **You author a ConfigMap/Secret manifest and `kubectl apply` it.** kubectl POSTs it to the API server.

2. **The API server validates and writes to etcd.** For a ConfigMap, values are stored as-is (plain text). For a Secret, the `data:` values must already be base64; the API server stores those base64 bytes. If [encryption at rest](#) is configured, the API server encrypts the bytes with a provider (e.g. AES-CBC or a KMS) *before* handing them to etcd. Without that config, etcd stores the base64 in the clear.

3. **You reference the ConfigMap/Secret in a Pod spec** (via `envFrom`, `valueFrom`, or a `volume`). This reference is what wires them together.

4. **The scheduler places the Pod on a node** (Topic 29). The Pod spec — including the config references — is sent to that node's **kubelet**.

5. **The kubelet fetches the referenced objects from the API server.** Before starting the container, the kubelet pulls the current ConfigMap and Secret values. If a referenced object doesn't exist, the Pod won't start (you'll see `CreateContainerConfigError` — a common mistake below).

6. **The kubelet materializes the config, two possible ways:**
   - **As env vars:** the kubelet builds the list of environment variables and passes them to the container runtime (containerd → runc, Topic 03). These values are **copied once** into the process environment at `execve()` time. They are frozen for the life of the container.
   - **As a volume:** the kubelet creates a directory, writes each key as a file. For a ConfigMap it's a normal projected volume; for a Secret it mounts a **tmpfs** and writes the decoded (plain-text) values into it. This volume is bind-mounted into the container's filesystem at your `mountPath`.

7. **The container starts.** Your Node process runs `process.env.DB_HOST` (env var path) or `fs.readFileSync('/etc/orders-config/DB_HOST')` (file path) and gets the value.

8. **On update (files only):** if you later change the ConfigMap, the kubelet has a sync loop (default ~1 minute) that notices the change and **atomically updates the mounted files** via a symlink swap (so your app never sees a half-written file). **Env vars do NOT update** — the value was copied at step 6 and the process must restart to see a new one. This asymmetry is the source of endless "I changed the ConfigMap but nothing happened" confusion.

The key insight: **env vars are a one-time copy; mounted files are a live-ish projection.** That single difference drives most design decisions in this topic.

---

## Exact syntax breakdown

### A ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
  namespace: orders
data:
  LOG_LEVEL: "info"
  DB_HOST: "orders-postgres"
  REDIS_HOST: "orders-redis"
  app.properties: |
    retries=3
    timeout=30
```

```
apiVersion: v1                     ← ConfigMap is a core (v1) object, no group prefix
kind: ConfigMap                    ← the object type
metadata:
  name: orders-config              ← the name Pods will reference
  namespace: orders                ← lives in the "orders" namespace (Topic 37)
data:                              ← the key-value payload (all values are strings)
│ LOG_LEVEL: "info"               ← simple key → becomes env var or a file named LOG_LEVEL
│ DB_HOST: "orders-postgres"      ← the K8s Service DNS name of Postgres (Topic 35)
│ app.properties: |              ← a multi-line value using YAML block scalar "|"
│   retries=3                     │  becomes a file "app.properties" with these two lines
│   timeout=30                    │  (great for config files your app reads whole)
└─ every value here is stored in etcd as PLAIN TEXT — never put secrets in a ConfigMap
```

### A Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: orders-secrets
  namespace: orders
type: Opaque
data:
  DB_PASSWORD: c3VwZXItc2VjcmV0LXBhc3N3b3Jk
stringData:
  REDIS_PASSWORD: "r3dis-pass"
```

```
kind: Secret
type: Opaque                       ← "Opaque" = arbitrary user-defined key/values.
│                                     Other types exist: kubernetes.io/tls (TLS certs),
│                                     kubernetes.io/dockerconfigjson (registry creds),
│                                     kubernetes.io/basic-auth, kubernetes.io/service-account-token
data:                              ← values here MUST be base64-encoded already
│ DB_PASSWORD: c3VwZXItc2VjcmV0...  ← base64 of "super-secret-password".
│                                     base64 is ENCODING, not encryption — trivially reversible
stringData:                        ← convenience: write PLAIN text, API server base64s it for you
│ REDIS_PASSWORD: "r3dis-pass"     ← never appears in the stored object; merged into data as base64
└─ stringData is write-only sugar; when you read the Secret back you only see base64 in "data"
```

### Consuming via `envFrom` (all keys → env vars)

```yaml
    envFrom:
      - configMapRef:
          name: orders-config
      - secretRef:
          name: orders-secrets
```

```
envFrom:                           ← import EVERY key from the source as an env var
  - configMapRef:                  ← pull all keys from a ConfigMap
      name: orders-config          │  LOG_LEVEL, DB_HOST, REDIS_HOST all become env vars
  - secretRef:                     ← pull all keys from a Secret
      name: orders-secrets         │  DB_PASSWORD becomes an env var (copied once, plain text)
└─ warning: envFrom keys must be valid env var names; "app.properties" would be skipped with a warning
```

### Consuming via `valueFrom` (pick ONE key → ONE env var)

```yaml
    env:
      - name: DATABASE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: orders-secrets
            key: DB_PASSWORD
```

```
env:
  - name: DATABASE_PASSWORD        ← the env var name YOUR app reads (you choose it)
    valueFrom:                     ← its value comes from elsewhere, not a literal
      secretKeyRef:                ← specifically, one key of a Secret
        name: orders-secrets       ← which Secret
        key: DB_PASSWORD           ← which key inside it
└─ use configMapKeyRef instead of secretKeyRef to pull a single ConfigMap key
```

### Consuming as a mounted volume (keys → files)

```yaml
    volumeMounts:
      - name: config-vol
        mountPath: /etc/orders-config
        readOnly: true
  volumes:
    - name: config-vol
      configMap:
        name: orders-config
```

```
volumeMounts:                      ← where to attach the volume inside the container
  - name: config-vol               ← must match a volume name below
    mountPath: /etc/orders-config  ← the directory; each key becomes a file here
    readOnly: true                 ← config should never be written by the app
volumes:
  - name: config-vol
    configMap:                     ← this volume is backed by a ConfigMap
      name: orders-config          │  files: /etc/orders-config/LOG_LEVEL, /DB_HOST, /app.properties
└─ swap "configMap:" for "secret:" to mount a Secret (backed by tmpfs) instead
```

---

## Example 1 — basic

Externalize `orders-api`'s log level and DB host, and inject its DB password from a Secret. Every line commented.

```yaml
# configmap.yaml — non-secret settings
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config              # the name we'll reference from the Pod
  namespace: orders                # our app's namespace (Topic 37)
data:
  LOG_LEVEL: "info"                # your Node app reads process.env.LOG_LEVEL
  DB_HOST: "orders-postgres"       # the Postgres Service name (Topic 35 gives DNS)
  DB_PORT: "5432"                  # note: MUST be quoted — YAML would read 5432 as a number
---
# secret.yaml — the sensitive value
apiVersion: v1
kind: Secret
metadata:
  name: orders-secrets
  namespace: orders
type: Opaque
stringData:                        # stringData = we write plain text, K8s base64s it
  DB_PASSWORD: "super-secret-password"
---
# deployment.yaml — wire them into orders-api
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: orders
spec:
  replicas: 2
  selector:
    matchLabels: { app: orders-api }
  template:
    metadata:
      labels: { app: orders-api }
    spec:
      containers:
        - name: orders-api
          image: orders-api:1.4.0
          ports:
            - containerPort: 3000
          envFrom:                 # bulk-import ALL ConfigMap keys as env vars
            - configMapRef:
                name: orders-config
          env:                     # cherry-pick ONE secret key into a named env var
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: orders-secrets
                  key: DB_PASSWORD
```

Apply and verify:

```bash
kubectl apply -f configmap.yaml -f secret.yaml -f deployment.yaml
# Prove the values landed in the container's environment:
kubectl exec deploy/orders-api -n orders -- env | grep -E 'LOG_LEVEL|DB_HOST|DB_PASSWORD'
# LOG_LEVEL=info
# DB_HOST=orders-postgres
# DB_PASSWORD=super-secret-password    ← plain text in the process env, copied at start
```

Your Node code doesn't change at all — it just reads `process.env.DB_HOST` and `process.env.DB_PASSWORD` as always. The image is identical in every environment; only the ConfigMap/Secret differ.

---

## Example 2 — production scenario

**The situation.** Your team runs `orders-api` in prod and needs the TLS certificate for Postgres and the Stripe API key available as **files** (the Postgres client library wants a `sslcert` file path, and you want the secret to update without a redeploy when it's rotated). You also want to prove the "env vars don't refresh, files do" rule so on-call stops filing false bugs.

**The manifests:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: orders-tls
  namespace: orders
type: Opaque
stringData:
  postgres-ca.pem: |
    -----BEGIN CERTIFICATE-----
    MIIB...fakecert...==
    -----END CERTIFICATE-----
  STRIPE_KEY: "sk_live_fakeexample"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: orders
spec:
  replicas: 3
  selector: { matchLabels: { app: orders-api } }
  template:
    metadata: { labels: { app: orders-api } }
    spec:
      containers:
        - name: orders-api
          image: orders-api:1.4.0
          volumeMounts:
            - name: tls                       # mount the Secret as files
              mountPath: /var/run/secrets/pg  # app reads the cert from this path
              readOnly: true
      volumes:
        - name: tls
          secret:
            secretName: orders-tls
            defaultMode: 0400                 # files readable ONLY by owner (tight perms)
```

Inside the Pod, the app reads `/var/run/secrets/pg/postgres-ca.pem` for the TLS cert. That path is on **tmpfs** (RAM) — the cert never hits the node's disk.

**Proving the update rule to your on-call team.** You update the Stripe key:

```bash
kubectl create secret generic orders-tls -n orders \
  --from-literal=STRIPE_KEY=sk_live_rotated ... --dry-run=client -o yaml | kubectl apply -f -
```

- If `STRIPE_KEY` was injected as a **file** (mounted): within ~60s the kubelet's sync loop rewrites `/var/run/secrets/pg/STRIPE_KEY` atomically. If your app re-reads the file per request (or watches it), it picks up the new key **with no restart**. Zero downtime.
- If `STRIPE_KEY` was injected as an **env var** (`secretKeyRef`): the running process still has the OLD value. Nothing changes until the Pod restarts. On-call swears "I rotated it and it didn't work" — because env vars are a frozen one-time copy.

**The production lesson:** for anything you want to rotate live (certs, keys read per-request), mount it as a **file** and have the app re-read it. For values that are fine to freeze until the next deploy (DB host, log level), env vars are simpler. And to force a refresh of env-var config on purpose, add a checksum annotation so changing the ConfigMap changes the Pod template and triggers a rolling restart:

```yaml
  template:
    metadata:
      annotations:
        checksum/config: "{{ sha256 of the ConfigMap }}"   # Helm pattern (Topic 49)
```

Changing the ConfigMap changes this checksum → changes the Pod template → Kubernetes does a rolling update → new Pods get the new env vars. This is the standard trick for "make env-var config reload safely."

---

## Common mistakes

**Mistake 1 — Thinking a Secret is encrypted. It is base64 by default.**

```bash
$ kubectl get secret orders-secrets -n orders -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
super-secret-password        # decoded in one command by anyone with get-secret rights
```

Root cause: base64 is a reversible **encoding** for safely storing binary bytes in etcd, not a security control. Anyone who can `get` the Secret, or read etcd, or read a backup of etcd, sees the plain value. **Right fix:** enable **encryption at rest** on the API server so etcd stores ciphertext, and lock down who can read Secrets with RBAC (Topic 53):

```yaml
# EncryptionConfiguration (given to the API server via --encryption-provider-config)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - aescbc:                 # or "kms:" for a cloud KMS — the recommended option
          keys:
            - name: key1
              secret: <32-byte base64 key>
      - identity: {}            # fallback so existing plaintext can still be read
```

For real production, use a KMS provider or an external system (Topic 50 — Vault, External Secrets, sealed secrets).

**Mistake 2 — Referencing a ConfigMap/Secret that doesn't exist yet.**

```
$ kubectl get pod orders-api-7d4 -n orders
NAME             READY   STATUS                       RESTARTS
orders-api-7d4   0/1     CreateContainerConfigError   0
$ kubectl describe pod orders-api-7d4 -n orders
  Warning  Failed  ...  Error: configmap "orders-config" not found
```

Root cause: at step 5 of the trace, the kubelet couldn't fetch the referenced object, so it refused to create the container. **Right fix:** apply the ConfigMap/Secret *before* (or together with) the Deployment, and make sure the name and namespace match exactly. Mark a reference optional with `optional: true` only if the app truly has a default.

**Mistake 3 — Unquoted numeric/boolean values in a ConfigMap.**

```yaml
data:
  DB_PORT: 5432        # WRONG: YAML parses this as an integer
```

```
error: cannot convert int64 to string
```

Root cause: ConfigMap `data` values must be **strings**. YAML reads `5432` as a number and `true` as a boolean. **Right fix:** quote everything: `DB_PORT: "5432"`, `FEATURE_X: "true"`. (In `binaryData` you'd use base64 instead.)

**Mistake 4 — Expecting env-var config to hot-reload.**

You edit the ConfigMap, wait, and your app still logs at the old level. You conclude Kubernetes is broken. Root cause: you injected it via `envFrom`/`valueFrom`, and env vars are copied **once** at container start (step 6). The mounted-file path *does* refresh; env vars never do. **Right fix:** either mount config as a file and re-read it, or roll the Deployment (`kubectl rollout restart deploy/orders-api -n orders`) to recreate Pods with the new values.

**Mistake 5 — A `subPath` mount that silently never updates.**

```yaml
    volumeMounts:
      - name: config-vol
        mountPath: /app/config.json
        subPath: config.json      # mounts a single file, not a directory
```

Everything works at first, but when you update the ConfigMap the file **never changes**, even after 10 minutes. Root cause: the kubelet's atomic symlink-swap update works on the *directory*; a `subPath` mount bypasses the symlink and pins the file to the version present at container start. **Right fix:** mount the whole directory (no `subPath`) if you need live updates, or accept that `subPath` requires a Pod restart to change.

---

## Hands-on proof

Run these **right now** against any cluster (minikube/kind/k3s) to watch every claim in this doc come true.

```bash
# 0. Namespace.
kubectl create namespace orders 2>/dev/null

# 1. Create a ConfigMap and a Secret imperatively.
kubectl create configmap demo-config -n orders \
  --from-literal=LOG_LEVEL=info --from-literal=DB_HOST=orders-postgres
kubectl create secret generic demo-secret -n orders \
  --from-literal=DB_PASSWORD=super-secret-password

# 2. PROOF base64, not encryption: decode the stored secret in one line.
kubectl get secret demo-secret -n orders -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
# → super-secret-password

# 3. Run a Pod that consumes BOTH as env vars and as files.
kubectl run demo -n orders --image=busybox --restart=Never -- sleep 3600
kubectl set env pod/demo -n orders --from=configmap/demo-config 2>/dev/null || true
# (for a full env+volume demo, apply the Example 1 Deployment instead)

# 4. PROOF the Secret file lives on tmpfs (RAM), not disk — from Example 2 Pod:
#    kubectl exec <pod> -n orders -- mount | grep secret
#    → tmpfs on /var/run/secrets/... type tmpfs

# 5. PROOF env vars are frozen: change the ConfigMap, env stays old.
kubectl create configmap demo-config -n orders \
  --from-literal=LOG_LEVEL=debug --from-literal=DB_HOST=orders-postgres \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl exec demo -n orders -- env | grep LOG_LEVEL   # still shows old value in a running pod

# 6. Inspect where it lives in the API (and thus etcd):
kubectl get secret demo-secret -n orders -o yaml   # note: data is base64, type: Opaque

# 7. Clean up.
kubectl delete pod demo -n orders
kubectl delete configmap demo-config -n orders
kubectl delete secret demo-secret -n orders
```

What you verified: a Secret decodes with a single `base64 -d` (step 2), secret files sit on tmpfs (step 4), and env vars do not refresh while a Pod runs (step 5).

---

## Practice exercises

### Exercise 1 — easy
Create a ConfigMap named `orders-config` in the `orders` namespace with `LOG_LEVEL=warn` and `MAX_CONNECTIONS=20`. Run a `busybox` Pod that imports it with `envFrom` and prints its environment. Confirm both keys appear. Then explain in one sentence why `MAX_CONNECTIONS` had to be a quoted string in the YAML.

### Exercise 2 — medium
Create a Secret `orders-secrets` with `DB_PASSWORD=hunter2`. Mount it as a **volume** at `/var/run/secrets/db` in a Pod. `exec` in and (a) `cat /var/run/secrets/db/DB_PASSWORD` to see the plain value, and (b) run `mount | grep db` to prove it's tmpfs. Then `kubectl get secret orders-secrets -o yaml` and confirm the stored value is base64, not `hunter2`.

### Exercise 3 — hard (production simulation)
Deploy `orders-api` (use any image, e.g. `nginx`) with a ConfigMap key `FEATURE_FLAG` injected **two ways at once**: as an env var (`valueFrom`) and as a mounted file. Change `FEATURE_FLAG` in the ConfigMap. Wait 90 seconds. Show that the **mounted file** updated but the **env var** did not. Then trigger `kubectl rollout restart deploy/orders-api -n orders` and show the env var finally updated. Write down which mechanism the kubelet used for the file update (hint: atomic symlink swap).

---

## Mental model checkpoint

Answer from memory:

1. Where is a ConfigMap physically stored, and in what form is a Secret stored there by default?
2. Is base64 encryption? What one command reveals a Secret's real value?
3. What must you configure so etcd stores Secrets as ciphertext instead of base64?
4. When a Secret is mounted as files, what filesystem type backs those files, and why does that matter for disk security?
5. Which injection method (env var vs mounted file) updates live when you change the source, and which is frozen until Pod restart?
6. What does a `subPath` mount do to live updates?
7. What error do you get if a Pod references a ConfigMap that doesn't exist, and at which step of the trace does it occur?

---

## Quick reference card

| Command / field | What it does | Key detail |
|---|---|---|
| `kubectl create configmap NAME --from-literal=k=v` | Make a ConfigMap | Values stored plain text in etcd |
| `kubectl create secret generic NAME --from-literal=k=v` | Make a Secret | Values stored base64 (NOT encrypted) |
| `envFrom: [configMapRef/secretRef]` | Import all keys as env vars | Copied once at container start; frozen |
| `valueFrom.secretKeyRef` / `configMapKeyRef` | One key → one named env var | Cherry-pick a single value |
| `volumes: {configMap:/secret:}` + `volumeMounts` | Mount keys as files | Secret files on tmpfs (RAM); files update ~60s |
| `stringData:` | Write plain text in a Secret | K8s base64-encodes it; write-only sugar |
| `type: Opaque` | Arbitrary user key/values | Other types: tls, dockerconfigjson, basic-auth |
| `defaultMode: 0400` | File permissions for mounted secret | Restrict to owner-read only |
| `immutable: true` | Freeze a ConfigMap/Secret | Faster kubelet, prevents edits; must delete to change |
| Encryption at rest | API server encrypts before etcd | Use a KMS provider in production |
| `kubectl rollout restart deploy/X` | Recreate Pods | Only reliable way to refresh env-var config |

**Immutable config note:** setting `immutable: true` on a ConfigMap/Secret tells the kubelet it will never change, so the kubelet stops watching it — reducing API-server load at scale and preventing accidental edits. The tradeoff: to change it you must delete and recreate it (then roll the Deployment). Great for versioned config like `orders-config-v3`.

---

## When would I use this at work?

1. **Same image across dev/staging/prod.** You build `orders-api:1.4.0` once and deploy it to the `orders-dev`, `orders-staging`, and `orders` namespaces, each with its own ConfigMap/Secret pointing at that environment's Postgres. The exact binary you tested in staging is the one that runs in prod.

2. **Rotating a leaked credential.** Security says the Postgres password leaked. You update the `orders-secrets` Secret and `kubectl rollout restart deploy/orders-api` — no image rebuild, no code change, done in a minute across all replicas.

3. **Delivering TLS certs and files to your app.** Your Postgres client needs a CA cert on disk. You mount a Secret as a file at `/var/run/secrets/pg/ca.pem` (on tmpfs, never touching the node's disk), and when the cert is rotated the kubelet updates the file within ~60s with no redeploy.

---

## Connected topics

- **Study before:**
  - **Topic 15 — Environment Variables and Secrets:** the Docker-level 12-factor config story this builds on.
  - **Topic 32 — YAML and Manifests:** the `apiVersion/kind/metadata/spec` structure used here.
  - **Topic 29 — Kubernetes Architecture:** the API server + etcd + kubelet roles that make injection work.
- **Study after:**
  - **Topic 37 — Namespaces:** where per-environment ConfigMaps/Secrets live and how quotas apply.
  - **Topic 50 — Secrets Management at Scale:** Vault, External Secrets Operator, and sealed secrets — the real answer to "base64 isn't secure."
  - **Topic 53 — RBAC:** restricting *who* can read Secrets, the other half of securing them.
  - **Topic 49 — Helm Basics:** the `checksum/config` annotation trick for auto-restart on config change.
