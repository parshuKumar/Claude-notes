# 50 — Secrets Management at Scale

## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Imagine your house key. You could write your address on it and leave it under
the doormat — technically "hidden," but anyone who lifts the mat has your key.
That's roughly what a plain Kubernetes Secret is: it *looks* hidden because it's
scrambled, but the scramble is trivial to undo, and it's sitting right there
under the mat (in the cluster's database).

Now imagine a proper solution: a **bank vault** downtown. Your key stays in the
vault. When you need to get into your house, you show ID at the vault, a clerk
hands you the key for exactly as long as you need it, logs that you took it, and
takes it back after. If you're fired, the vault stops giving you the key —
instantly, everywhere. Nobody keeps a copy under any mat.

That vault is **HashiCorp Vault** (or a cloud secrets manager). The clerk who
runs to the vault for you and drops the key on your desk is the **External
Secrets Operator**. And **Sealed Secrets** is a different clever trick: a magic
lockbox that *only your specific house's lock* can open, so it's safe to mail it
through the post (commit it to git) — the postman can carry it but can't open it.

This topic is about *why the doormat isn't good enough* and *what the grown-up
options are.*

---

## The Linux kernel feature underneath

Secrets management is mostly about **cryptography and access control**, not a
single kernel primitive — but there are two very physical, kernel-and-storage
level facts you must understand, because every myth about Secrets comes from not
knowing them.

**Fact 1 — A Kubernetes Secret is stored in etcd, and by default it is NOT
encrypted; it is only base64-encoded.** base64 is **not encryption** — it's a
reversible alphabet swap, like Morse code. Anyone who can read etcd, or has
`get secret` permission, reads your password instantly.

```
   You: kubectl create secret ... DB_PASSWORD=s3cr3t
                     │
                     ▼  base64("s3cr3t") = "czNjcjN0"   ← reversible, NOT secret
   API server ──▶ etcd on the control-plane node's DISK:
     /var/lib/etcd/member/wal/   and   .../snap/db
        └─ the bytes "czNjcjN0" sit here in plaintext-equivalent form
```

You can prove base64 is not protection in one line:
```
$ echo "czNjcjN0" | base64 -d
s3cr3t
```

**Fact 2 — "Encryption at rest" is a kernel/storage feature you turn on
deliberately.** The API server can encrypt Secret values *before* writing them to
etcd, using an `EncryptionConfiguration`. This is real symmetric encryption
(AES-GCM) applied in the API server process:

```
   Secret value ──▶ API server encrypts with AES-GCM (key from EncryptionConfig)
                                     │
                                     ▼
   etcd stores:  k8s:enc:aescbc:v1:key1:<ciphertext-bytes>   ← now unreadable on disk
```

But note the trap: **the encryption key itself has to live somewhere.** If you
use the simple `aescbc` provider, the key sits in a file
(`/etc/kubernetes/enc/enc.yaml`) on the **same control-plane node** as etcd. So
you've locked the safe but taped the key to the safe's door. The grown-up version
points the API server at a **KMS provider** (cloud KMS or Vault) so the master
key never touches the node's disk — the node must call out to KMS to decrypt,
and KMS can revoke/audit that.

So the "mechanism underneath" is: **base64 (encoding, not security) → optional
AES-GCM encryption at rest in the API server → KMS-backed keys so the master key
lives off-node.** Everything else in this topic is about keeping the *plaintext
value* out of etcd and out of git entirely.

---

## What is this?

**Secrets management at scale** is the practice of storing, distributing, and
rotating sensitive values (DB passwords, API keys, TLS private keys) safely
across many services and environments. Native Kubernetes Secrets are only
base64-encoded in etcd, so at scale teams add **encryption at rest**, an external
**secrets store** like HashiCorp Vault, and operators (**External Secrets
Operator**, **Sealed Secrets**, **SOPS**) that inject real values without ever
committing plaintext to git.

---

## Why does it matter for a backend developer?

`orders-api` needs a Postgres password, a Redis password, and a Stripe API key.
If you get secrets wrong, the failure modes are severe and permanent:

- **Leaked passwords are forever.** If you commit `DB_PASSWORD=s3cr3t` to git,
  it's in the history on every clone, fork, and CI cache — deleting the file does
  **not** remove it. A leaked prod DB password means rotating the password *and*
  auditing what was accessed.
- **base64 fools people into a false sense of safety.** Developers see the
  scrambled string in a YAML file and think "it's hidden." It is not. A teammate
  with read access, a backup of etcd, or a misconfigured dashboard exposes it.
- **No rotation = compounding risk.** A password baked into a Secret that never
  changes is a password that's been in Slack messages, laptops, and CI logs for
  two years. Real secret stores rotate automatically and give short-lived
  credentials.
- **Blast radius.** Without per-service, least-privilege access, one compromised
  pod can read every secret in the namespace. Vault + short-lived tokens shrink
  what any single breach can touch.

If you have ever pushed a `.env` file to GitHub by accident (everyone has, once),
you already understand why this topic exists.

---

## The physical reality

Let's look at where a secret actually *is* under each approach, on disk and in
etcd, for `orders-api`'s `DB_PASSWORD`.

### Plain Secret (the default — weak)

```
etcd on control-plane node:
  /registry/secrets/orders/orders-db   → value contains  "czNjcjN0"  (base64)
                                                           └─ decode → s3cr3t

Inside the pod, when mounted as a file:
  /var/run/secrets/... or your mountPath, e.g. /etc/secrets/DB_PASSWORD
    └─ backed by a tmpfs (RAM), NOT written to the node's disk — this part is good
```

One nuance in Kubernetes' favor: when a Secret is **mounted as a volume** into a
pod, kubelet stores it on a **tmpfs** (RAM-backed filesystem), so it never lands
on the node's persistent disk. The weakness is etcd, not the pod mount.

### Encryption at rest enabled

```
/etc/kubernetes/enc/enc.yaml     ← EncryptionConfiguration (the AES key, if aescbc)
etcd:
  /registry/secrets/orders/orders-db → k8s:enc:aescbc:v1:key1:<ciphertext>
                                        └─ unreadable without the key
```

### External Secrets Operator (ESO)

The real value lives **outside the cluster** in Vault/AWS Secrets Manager. ESO
syncs it *into* a normal K8s Secret on demand:

```
   AWS Secrets Manager / Vault           Cluster
   ┌───────────────────────┐   ESO polls  ┌──────────────────────────┐
   │ orders/db_password =  │ ───────────▶ │ Secret orders-db (synced) │──▶ pod
   │   "s3cr3t"  (source    │   & writes   │  refreshed every N min    │
   │    of truth, audited)  │   a Secret   └──────────────────────────┘
   └───────────────────────┘
   git contains only an ExternalSecret CR (a POINTER), never the value
```

### Sealed Secrets

The value is encrypted with the cluster controller's **public key** into a
`SealedSecret` that is **safe to commit to git**. Only the controller's private
key (in the cluster) can decrypt it:

```
git repo (public-key encrypted, safe to commit):
  sealedsecret.yaml → AgBy3i4...long ciphertext...   ← only THIS cluster can open

Cluster:
  sealed-secrets-controller (holds PRIVATE key) ──▶ decrypts ──▶ real Secret ──▶ pod
```

### SOPS

The YAML file is committed to git but the **values are encrypted** (keys stay
readable) using a KMS/age key. You decrypt at deploy time:

```
git repo:
  secrets.enc.yaml:
    DB_PASSWORD: ENC[AES256_GCM,data:9f2a..,iv:..,tag:..]   ← value encrypted
    #  ↑ key readable so diffs are meaningful; value unreadable without KMS/age key
```

---

## How it works — step by step

Trace of the **External Secrets Operator** delivering `DB_PASSWORD` from AWS
Secrets Manager to `orders-api`:

1. **You store the real value once, in the external store.** e.g.
   `aws secretsmanager put-secret-value --secret-id orders/db_password
   --secret-string 's3cr3t'`. This is the single source of truth, with its own
   IAM access control and audit log. Nothing sensitive is in git or the cluster
   yet.

2. **You commit a `SecretStore` (or `ClusterSecretStore`).** It tells ESO *where*
   the store is and *how to authenticate* (e.g. an IAM role via IRSA / a service
   account). This is a pointer + auth config, not a secret.

3. **You commit an `ExternalSecret` custom resource.** It says: "make a K8s
   Secret named `orders-db`; fill key `DB_PASSWORD` from the external key
   `orders/db_password`; refresh every 1h." Again — a pointer, safe in git.

4. **ESO's controller reconciles (Topic 30).** It watches `ExternalSecret`
   objects. On seeing yours, it authenticates to AWS using the `SecretStore`
   credentials and **reads** `orders/db_password`.

5. **ESO writes a normal Kubernetes Secret.** It creates/updates
   `secret/orders-db` in the `orders` namespace with the fetched value
   (base64-encoded in etcd like any Secret — that's fine because the *source of
   truth* and *rotation* live in AWS, and etcd should have encryption-at-rest on).

6. **`orders-api` consumes the Secret normally.** Your Deployment references
   `secretKeyRef: {name: orders-db, key: DB_PASSWORD}` — the app has no idea AWS
   was involved. It just gets an env var.

7. **Rotation happens automatically.** You rotate the value in AWS. On the next
   refresh interval, ESO re-reads it and updates `secret/orders-db`. Combined
   with the `checksum` trick (Topic 49) or a reloader, pods pick up the new
   password on their next roll.

8. **Revocation is central.** Remove ESO's IAM permission and it can no longer
   sync — no scavenger hunt across 40 manifests to find where a secret leaked.

For **Sealed Secrets** the flow differs at steps 1–5: you run
`kubeseal < secret.yaml > sealedsecret.yaml`, which encrypts with the
controller's **public** key; you commit the `SealedSecret`; the in-cluster
controller decrypts it with its **private** key and emits a normal Secret.

---

## Exact syntax breakdown

### A plain Secret — and why `stringData` beats `data`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: orders-db
  namespace: orders
type: Opaque
│     └─ generic key/value secret (other types: kubernetes.io/tls, .../dockerconfigjson)
stringData:
│ └─ stringData: you write PLAINTEXT; the API server base64-encodes it for you
  DB_PASSWORD: s3cr3t
  │            └─ readable here — which is exactly why this file must NOT enter git
data:
│ └─ data: you must supply ALREADY base64-encoded values (common source of bugs)
  REDIS_PASSWORD: cjNkMXNwdw==   ← base64("r3d1spw"); base64 is encoding, not security
```

### Consuming a Secret in the `orders-api` Deployment

```yaml
        env:
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
              │ └─ pull this env var's value from a Secret (not a literal)
                name: orders-db      ← the Secret's name
                key: DB_PASSWORD     ← which key inside that Secret
```

### EncryptionConfiguration — encryption at rest

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    │           └─ encrypt only Secret objects (could add configmaps, etc.)
    providers:
      - aescbc:            ← the FIRST provider is used to ENCRYPT new writes
          keys:
            - name: key1
              secret: <base64-32-byte-key>   ← ⚠ this key lives on the node's disk
      - identity: {}       ← "no encryption" fallback; MUST be last for decrypt
```
The API server is started with `--encryption-provider-config=/etc/kubernetes/enc/enc.yaml`.
Order matters: the first non-identity provider encrypts; all are tried for
decrypt. For production, replace `aescbc` with a `kms:` provider so the master
key lives in cloud KMS, not on the node.

### ExternalSecret (External Secrets Operator)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: orders-db
  namespace: orders
spec:
  refreshInterval: 1h
  │                └─ how often ESO re-reads the source (enables rotation)
  secretStoreRef:
    name: aws-store            ← which SecretStore (where + how to auth)
    kind: SecretStore
  target:
    name: orders-db           ← the K8s Secret ESO will CREATE/UPDATE
  data:
    - secretKey: DB_PASSWORD  ← key in the resulting K8s Secret
      remoteRef:
        key: orders/db_password   ← path/name in AWS Secrets Manager / Vault
```

### SealedSecret (safe to commit)

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: orders-db
  namespace: orders           ← scope: sealed to THIS name+namespace by default
spec:
  encryptedData:
    DB_PASSWORD: AgBy3i4Ok...very-long-ciphertext...   ← only the in-cluster
    │                                                    controller can decrypt
    └─ produced by: kubeseal < secret.yaml > sealedsecret.yaml
```

### SOPS-encrypted file

```
# secrets.enc.yaml  (committed to git)
DB_PASSWORD: ENC[AES256_GCM,data:9f2a7c..,iv:1b..,tag:5d..,type:str]
│            │              │              │      │
│            │              │              │      └─ auth tag (integrity)
│            │              │              └─ nonce/iv
│            │              └─ the encrypted bytes of "s3cr3t"
│            └─ AES-256-GCM ciphertext wrapper
└─ key stays readable → git diffs are meaningful; value is unreadable
# decrypt at deploy time:  sops -d secrets.enc.yaml | kubectl apply -f -
```

---

## Example 1 — basic

Prove that a plain Secret protects nothing, then turn on the first real defense.

```bash
# 1) Create a Secret the normal way (stringData = you write plaintext):
kubectl -n orders create secret generic orders-db \
  --from-literal=DB_PASSWORD=s3cr3t

# 2) "Decrypt" it with zero effort — base64 is not security:
kubectl -n orders get secret orders-db -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# → s3cr3t

# 3) Consume it safely in orders-api (env var from secretKeyRef):
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-api, namespace: orders }
spec:
  replicas: 1
  selector: { matchLabels: { app: orders-api } }
  template:
    metadata: { labels: { app: orders-api } }
    spec:
      containers:
        - name: orders-api
          image: myregistry/orders-api:1.4.2
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: orders-db, key: DB_PASSWORD }  # never hard-code
EOF

# 4) Confirm the app got it WITHOUT the value ever being in the Deployment YAML:
kubectl -n orders exec deploy/orders-api -- printenv DB_PASSWORD   # → s3cr3t
```

The lesson: `secretKeyRef` keeps the value out of your Deployment manifest (good,
commit that), but the Secret object itself is still only base64 in etcd. Next you
add encryption at rest and stop committing the Secret manifest at all.

---

## Example 2 — production scenario

**The situation:** `orders-api` runs in dev, staging, and prod. Early on, someone
committed `orders-db-secret.yaml` (with a base64 password) to the deploy repo "so
CI could apply it." Security review flagged it: the prod DB password is now in
git history on every developer's laptop. You must (a) get plaintext out of git
permanently, (b) rotate the password, and (c) make sure it never happens again.
The team chooses the **External Secrets Operator** backed by AWS Secrets Manager.

**Step 1 — move the real value into AWS (single source of truth):**
```bash
aws secretsmanager create-secret --name orders/prod/db_password \
  --secret-string 'N3w-rotated-p@ss'      # the OLD one is rotated in Postgres too
```

**Step 2 — install ESO and point it at AWS with a `ClusterSecretStore`:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata: { name: aws-store }
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef: { name: eso-sa, namespace: external-secrets }  # IRSA
```

**Step 3 — commit an `ExternalSecret` (a pointer, safe in git):**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: orders-db, namespace: orders }
spec:
  refreshInterval: 15m
  secretStoreRef: { name: aws-store, kind: ClusterSecretStore }
  target: { name: orders-db }        # ESO creates/updates this K8s Secret
  data:
    - secretKey: DB_PASSWORD
      remoteRef: { key: orders/prod/db_password }
```

**Step 4 — `orders-api` consumes `secret/orders-db` exactly as before.** No app
change: it still reads `secretKeyRef: {name: orders-db, key: DB_PASSWORD}`. ESO
created that Secret by syncing from AWS.

**Step 5 — purge the leaked value from git history** (BFG or `git filter-repo`),
force-push, and rotate the exposed password (which you already did in step 1).
The old base64 in history is now useless.

**What you gained:**
- Git contains only *pointers* (`ExternalSecret`, `ClusterSecretStore`) — never a
  real value. A future accidental commit exposes nothing.
- **Rotation is automatic:** update the value in AWS, and within 15 minutes ESO
  refreshes `secret/orders-db`. Add a reloader or the `checksum/secret` annotation
  (Topic 49) so pods roll and pick it up.
- **Revocation is central:** yank ESO's IAM permission and syncing stops
  everywhere at once.
- **Audit:** AWS logs every read of `orders/prod/db_password`.

If the team preferred a **GitOps-only** flow (no external store), the alternative
is **Sealed Secrets**: `kubeseal` encrypts the Secret with the cluster's public
key into a `SealedSecret` you *can* safely commit; the in-cluster controller
decrypts it. Values stay in git but are unreadable to anyone without the
cluster's private key. **SOPS** achieves a similar "encrypted-in-git" result at
the file level, decrypting at apply time with a KMS/age key.

---

## Common mistakes

### Mistake 1 — Thinking base64 protects a Secret

```
$ cat orders-db-secret.yaml
data:
  DB_PASSWORD: czNjcjN0        # "it's encoded, it's fine to commit"
$ echo czNjcjN0 | base64 -d
s3cr3t
```
**Root cause:** base64 is an *encoding*, not *encryption* — fully reversible with
no key. **Fix:** never commit Secret manifests with real values; use ESO/Sealed
Secrets/SOPS, and enable encryption-at-rest so even etcd bytes are ciphertext.

### Mistake 2 — Committing a `.env` or Secret YAML to git

```
$ git log --oneline -- prod.env
a1b2c3d  add prod env for CI
```
**Root cause:** git history is permanent and distributed; `git rm` does not
remove past commits. Once cloned, the secret is out. **Fix:** treat any secret
that touched git as **compromised** — rotate it immediately — then move to a
pointer-based flow (ESO) or encrypted-in-git (Sealed Secrets/SOPS), and add a
pre-commit hook / `gitleaks` scan to block future leaks.

### Mistake 3 — Wrong base64 in the `data:` field

```
$ kubectl apply -f orders-db-secret.yaml
$ kubectl -n orders exec deploy/orders-api -- printenv DB_PASSWORD
czNjcjN0        # ← the app got the base64 string, not the password!
```
**Root cause:** you put an already-plaintext value under `data:` (which expects
base64), so it got base64-encoded *again*. **Fix:** use `stringData:` for
plaintext (the API server encodes once), or run `echo -n 's3cr3t' | base64` for
`data:`. Watch for `echo` adding a trailing newline — always use `echo -n`.

### Mistake 4 — Assuming a mounted Secret updates instantly on rotation

```
# You rotate the DB password and update the Secret, but the app still uses the old one.
```
**Root cause:** env-var Secrets are injected **once at pod start** and never
change; even volume-mounted Secrets update on a delay and the app must re-read
the file. **Fix:** trigger a rolling restart on rotation — the `checksum/secret`
annotation (Topic 49), a tool like Reloader, or `kubectl rollout restart`.

### Mistake 5 — Sealing a SealedSecret for the wrong namespace/name

```
$ kubectl apply -f sealedsecret.yaml
Error: no key could decrypt secret (DB_PASSWORD)
```
**Root cause:** by default `kubeseal` binds the ciphertext to a specific
`name`+`namespace`. Applying it under a different name or namespace (or to a
different cluster whose controller has a different private key) fails to decrypt.
**Fix:** re-seal for the correct name/namespace/cluster, or use a broader
`--scope namespace-wide`/`cluster-wide` when sealing if you intend to reuse it.

---

## Hands-on proof

```bash
# 1) PROVE base64 is not encryption:
kubectl -n orders create secret generic demo --from-literal=pw=hunter2
kubectl -n orders get secret demo -o jsonpath='{.data.pw}' ; echo
kubectl -n orders get secret demo -o jsonpath='{.data.pw}' | base64 -d ; echo  # hunter2

# 2) See a mounted Secret is on tmpfs (RAM), not the node disk:
kubectl -n orders run probe --image=busybox --restart=Never -it --rm \
  --overrides='{"spec":{"volumes":[{"name":"s","secret":{"secretName":"demo"}}],
  "containers":[{"name":"p","image":"busybox","command":["sh","-c","mount | grep secrets; cat /s/pw; sleep 1"],
  "volumeMounts":[{"name":"s","mountPath":"/s"}]}]}}'
# mount line shows: tmpfs on /s type tmpfs   ← RAM-backed, good

# 3) Check whether encryption-at-rest is ON (read the raw etcd bytes):
#    (on a control-plane node with etcdctl access)
ETCDCTL_API=3 etcdctl get /registry/secrets/orders/demo | strings | head
#   plaintext-ish "hunter2/base64" → NOT encrypted
#   "k8s:enc:aescbc:v1:key1" prefix → encryption-at-rest IS on

# 4) Sealed Secrets round-trip (if the controller is installed):
kubectl -n orders create secret generic orders-db \
  --from-literal=DB_PASSWORD=s3cr3t --dry-run=client -o yaml > secret.yaml
kubeseal < secret.yaml > sealedsecret.yaml      # encrypt with cluster public key
grep DB_PASSWORD sealedsecret.yaml              # long ciphertext — safe to commit
kubectl apply -f sealedsecret.yaml              # controller decrypts → real Secret
kubectl -n orders get secret orders-db          # it now exists, unsealed

# 5) SOPS at the file level (needs sops + an age/KMS key):
sops -e secret.yaml > secret.enc.yaml           # values encrypted, keys readable
sops -d secret.enc.yaml | kubectl apply -f -    # decrypt only at apply time
```

---

## Practice exercises

### Exercise 1 — easy
Create a Secret two ways: once with `--from-literal` (stringData path) and once
by hand-writing a manifest with a `data:` field. Deliberately put the plaintext
password under `data:` (the wrong way) and prove with `printenv` inside the pod
that the app receives a double-base64-garbled value. Fix it and confirm.

### Exercise 2 — medium
Enable encryption at rest on a local cluster (kind/minikube): write an
`EncryptionConfiguration` with an `aescbc` key, point the API server at it, and
restart it. Create a Secret, then read the raw etcd bytes with `etcdctl` and
confirm they start with `k8s:enc:aescbc`. Then run
`kubectl get secrets --all-namespaces -o json | kubectl replace -f -` to
re-encrypt existing Secrets, and verify.

### Exercise 3 — hard (production simulation)
Simulate the Example 2 incident. Commit a Secret manifest with a base64 password
to a throwaway git repo, then "discover the leak." Rotate the value into a real
store (use Sealed Secrets if you have no cloud account): seal the new secret,
commit only the `SealedSecret`, and prove the in-cluster controller produces the
working Secret. Finally, purge the original plaintext from git history with
`git filter-repo`, and add a `gitleaks` pre-commit hook that blocks committing
anything matching a Secret pattern. Prove the hook fires.

---

## Mental model checkpoint

Answer from memory:

1. What is the difference between base64 encoding and encryption, and why does it
   matter for Kubernetes Secrets?
2. Where is a Secret physically stored, and is it encrypted there by default?
3. What does "encryption at rest" protect against, and where does its key live
   with the `aescbc` provider vs a `kms` provider?
4. In the External Secrets Operator flow, what lives in git and what lives in the
   external store?
5. Why is a `SealedSecret` safe to commit to git but a plain Secret is not?
6. How does SOPS differ from Sealed Secrets in *what* it encrypts?
7. Why doesn't rotating a value in the external store immediately change the
   password an already-running pod is using?

---

## Quick reference card

| Thing | What it does | Key detail |
|---|---|---|
| Plain Secret | Stores key/value in etcd | base64 only — NOT encrypted; keep out of git |
| `stringData` | Write plaintext, API server encodes | Use this instead of hand-base64ing `data` |
| `secretKeyRef` | Inject a Secret value as env var | Keeps the value out of the Deployment YAML |
| Encryption at rest | API server encrypts Secrets before etcd | `aescbc` key on node; `kms` keeps it off-node |
| HashiCorp Vault | Central secrets store + short-lived creds | Auditing, rotation, revocation, dynamic secrets |
| External Secrets Operator | Syncs external store → K8s Secret | Git holds only a pointer (`ExternalSecret`) |
| Sealed Secrets | Public-key encrypt a Secret for git | Only in-cluster controller can decrypt |
| SOPS | Encrypt YAML *values* in place | Keys stay readable; decrypt at apply time |
| KMS provider | Master key in cloud KMS/Vault | Node never holds the master key |

---

## When would I use this at work?

1. **Getting the prod DB password out of a repo** where someone committed it —
   rotate it, then move to ESO (pointer in git) or Sealed Secrets (encrypted in
   git) so plaintext never returns.
2. **Automatic credential rotation for `orders-api`** — Vault or a cloud secrets
   manager rotates the Postgres password on a schedule and ESO re-syncs it, with
   a reloader restarting pods to pick it up.
3. **A pure-GitOps team (Topic 52) with no external store** — Sealed Secrets or
   SOPS lets you keep everything, including secrets, in git safely, so ArgoCD/Flux
   can apply the whole app declaratively.

---

## Connected topics

- **Study before:** Topic 36 (ConfigMaps and Secrets) — how Secrets are stored in
  etcd and mounted. Topic 29 (Kubernetes Architecture) — the API server and etcd
  you're protecting. Topic 15 (Environment Variables and Secrets) — why secrets
  don't belong baked into images.
- **Study after:** Topic 52 (CI/CD with Kubernetes) — delivering secrets safely
  in a GitOps pipeline. Topic 53 (RBAC) — least-privilege `get secret` so not
  everyone can read them. Topic 54 (Multi-Environment Strategy) — per-environment
  secret stores and promotion.
