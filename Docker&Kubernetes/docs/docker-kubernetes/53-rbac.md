# 53 — RBAC (Role-Based Access Control)
## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Think of a big office building with electronic keycards.

Every keycard is blank until someone programs it. The building manager keeps a **rulebook**. A rule says: *"Anyone holding a 'cleaner' card may open the supply closet and the hallways — but NOT the server room or the CEO's office."* That rule is written down once. Then the manager **hands a cleaner-card to Priya**. Now Priya's specific card opens exactly the supply closet and hallways — nothing more.

Notice the three separate things:

- The **rulebook entry** ("cleaner cards may open X, Y") — reusable, describes *permissions*, names no person.
- The **act of handing Priya a cleaner-card** — connects a *person* to a *rulebook entry*.
- **Priya herself** — the identity holding the card.

Kubernetes RBAC is exactly this. The **rulebook entry** is a `Role` (or `ClusterRole`). **Handing someone the card** is a `RoleBinding` (or `ClusterRoleBinding`). The **person/identity** — when it's a program, not a human — is a `ServiceAccount`. And a **card that opens the whole building, every door** is `cluster-admin` — which is exactly the card you must *not* tape to your CI pipeline.

This topic is about writing tight rulebook entries and handing out the smallest possible card, especially the deploy-only card your Topic 52 pipeline should carry.

---

## The Linux kernel feature underneath

RBAC has no kernel primitive — it lives entirely *inside the Kubernetes API server*, above the OS. But there is a concrete mechanism that makes it real, and it's worth seeing physically: **a mounted token file inside every pod, and a request-authorization pipeline in the API server.**

Two physical realities:

1. **Every pod has a bearer token on disk.** When a pod runs, the kubelet mounts its ServiceAccount's JWT token as a plain file:

```
/var/run/secrets/kubernetes.io/serviceaccount/token      ← a signed JWT (the "keycard")
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt      ← CA to verify the API server
/var/run/secrets/kubernetes.io/serviceaccount/namespace   ← "orders"
```

Any process in the pod can `cat` that token and present it to the API server. The token is a **JWT signed by the cluster** — the API server can verify it without a database lookup. This is the same file mechanism as any Kubernetes Secret mounted as a volume (Topic 36).

2. **The API server runs every request through a three-stage gate.** When a request hits `https://<api-server>:6443`, the kernel's TCP/TLS stack terminates the connection, then the API server's Go code runs three stages *in order*:

```
request → [ Authentication ] → [ Authorization (RBAC) ] → [ Admission ] → etcd
          "WHO are you?"       "are you ALLOWED to do    "any extra
          (verify the JWT)      THIS verb on THIS         policy checks?"
                                resource?"  ← RBAC lives here
```

RBAC is *only* the middle stage: **authorization**. Authentication (proving *who* you are) already happened; admission (mutating/validating) happens after. RBAC answers one yes/no question: *"May this identity perform this verb on this resource in this namespace?"*

Hold this: **RBAC is a pure authorization decision made in the API server's memory by matching your identity's granted rules against the (verb, resource, apiGroup, namespace) of your request.** No kernel, no iptables — just rule matching before anything touches etcd.

---

## What is this?

**RBAC (Role-Based Access Control)** is Kubernetes' system for deciding *who can do what* to *which resources*. You write **Roles** that list allowed operations (verbs like `get`, `create`, `delete`) on resource types (like `pods`, `deployments`), and you **bind** those Roles to identities (users, groups, or ServiceAccounts).

A **ServiceAccount** is a non-human identity — an account that *programs* (pods, CI pipelines) authenticate as, instead of a person. RBAC's most common job in production is granting a ServiceAccount exactly the permissions a piece of automation needs, and not one permission more.

---

## Why does it matter for a backend developer?

Your Topic 52 pipeline deploys `orders-api`. Right now it authenticates with a kubeconfig that's bound to `cluster-admin`. Here's what that costs you:

- **One leak = total loss.** That kubeconfig sits in a GitHub secret. If a malicious npm dependency in your Action, or a leaked build log, exposes it, the attacker can now read every Secret in every namespace (including the Postgres and payment-gateway credentials), delete every workload, and create crypto-mining pods. `cluster-admin` means *everything, everywhere.*
- **Blast radius from bugs.** A typo in a script (`kubectl delete ns` with the wrong variable) wipes production, because the token was allowed to. A tight Role that only permits `patch deployments in orders` would have *rejected* the delete.
- **No answer for the auditor.** "Which identities can read Secrets in the `orders` namespace?" You can't answer, because everything is admin.
- **Pods over-permissioned by default.** Your `orders-api` pod uses the namespace's `default` ServiceAccount, whose token is mounted into the container. If your app has an SSRF or RCE bug, the attacker `cat`s that token and calls the API server *as your app*. If that account can do anything, so can the attacker.

Understanding RBAC lets you give each human, pipeline, and pod the **smallest set of permissions that still works** — so a leak, a bug, or a compromised pod causes the least possible damage. This is *least privilege*, and it's the difference between "we lost one deployment" and "we lost the cluster."

---

## The physical reality

RBAC objects are just records in etcd, and identities are tokens on disk. When RBAC is configured for `orders-api`'s deploy pipeline, here's what exists:

**1. In etcd — four kinds of objects:**

```
/registry/serviceaccounts/orders/deployer                 ← the identity (a name + token refs)
/registry/roles/orders/deployer-role                       ← the rulebook entry (namespaced)
/registry/rolebindings/orders/deployer-binding             ← links deployer → deployer-role
/registry/clusterroles/view                                ← a cluster-wide rulebook entry
```

Roles and RoleBindings are **namespaced** (they live inside `orders`). ClusterRoles and ClusterRoleBindings are **cluster-scoped** (no namespace).

**2. The Role object's actual content — a list of rules:**

```yaml
rules:
- apiGroups: ["apps"]          # deployments live in the "apps" API group
  resources: ["deployments"]   # which resource type
  verbs: ["get", "list", "patch"]   # what you may DO to it
```

That is literally three arrays. Authorization is: does *any* rule's arrays all contain the request's (apiGroup, resource, verb)? If yes → allow.

**3. In a running pod — the mounted token:**

```
$ kubectl exec orders-api-abc -n orders -- \
    cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiI...   ← the JWT this pod presents as its identity
```

**4. In etcd — the ServiceAccount's identity claims (inside the JWT), once decoded:**

```
"sub": "system:serviceaccount:orders:deployer"
        │                      │       │
        │                      │       └─ the ServiceAccount name
        │                      └─ its namespace
        └─ the fixed prefix that marks it as a ServiceAccount identity
```

That `system:serviceaccount:orders:deployer` string is the **username** RBAC matches bindings against. RoleBindings reference exactly this identity.

---

## How it works — step by step

Full trace: your CI pipeline (authenticating as the `deployer` ServiceAccount) runs `kubectl patch deployment orders-api -n orders`, and the API server decides whether to allow it.

1. **The client presents a token.** `kubectl` (in GitHub Actions) opens TLS to `:6443` and sends `Authorization: Bearer eyJhbGc...` — the `deployer` ServiceAccount's JWT.

2. **Authentication resolves the identity.** The API server verifies the JWT's signature against the cluster's signing key. Valid → it now knows the requester is `system:serviceaccount:orders:deployer`, with groups `system:serviceaccounts:orders`. (If the token were forged or expired → `401 Unauthorized`, and we stop here.)

3. **The request is described as an attribute tuple.** The API server turns the HTTP request into:

```
user:      system:serviceaccount:orders:deployer
verb:      patch                 (from HTTP PATCH)
apiGroup:  apps
resource:  deployments
name:      orders-api
namespace: orders
```

4. **The RBAC authorizer looks for matching bindings.** It searches:
   - **RoleBindings in namespace `orders`** whose `subjects` include our ServiceAccount. It finds `deployer-binding`, which points to Role `deployer-role`.
   - **ClusterRoleBindings** with our ServiceAccount as a subject (cluster-wide grants). None here.

5. **It reads the referenced Role's rules.** `deployer-role` contains:

```yaml
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]
```

6. **It checks if any rule covers the request tuple.** Does a rule have `apps` in `apiGroups` AND `deployments` in `resources` AND `patch` in `verbs`? **Yes.** RBAC is *purely additive* — one matching allow rule is enough.

7. **Decision: allow.** The request proceeds to admission control, then writes the patched spec to etcd. The Deployment controller rolls the pods (Topic 34).

8. **What a denied request looks like.** Suppose the same account tries `kubectl delete deployment orders-api`. Step 6 finds no rule with `delete` in `verbs`. There is **no default-allow** — the absence of a matching rule means deny:

```
Error from server (Forbidden): deployments.apps "orders-api" is forbidden:
User "system:serviceaccount:orders:deployer" cannot delete resource
"deployments" in API group "apps" in the namespace "orders"
```

That error message is RBAC telling you the exact tuple that failed: *user, verb, resource, apiGroup, namespace.* Reading it is how you debug every permission problem.

**The core rule to burn in:** RBAC is **default-deny + purely additive**. Nothing is allowed unless some Role, bound to your identity, explicitly allows that (verb, resource, apiGroup). There are no "deny" rules — you shrink permissions by granting *less*, never by writing a deny.

---

## Exact syntax breakdown

**A Role — the namespaced rulebook entry:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1   # the RBAC API group
kind: Role                                 # namespaced (vs ClusterRole)
metadata:
  namespace: orders                        # this Role only exists/works in "orders"
  name: deployer-role
rules:
- apiGroups: ["apps"]                       # "" = core group; "apps" = deployments/replicasets
  resources: ["deployments"]                # resource TYPE (plural, lowercase)
  verbs: ["get", "list", "watch", "patch"]  # allowed operations
- apiGroups: [""]                           # core API group (empty string!)
  resources: ["pods", "pods/log"]           # pods, and the log SUBRESOURCE
  verbs: ["get", "list"]
```

The rule fields, annotated:

```
- apiGroups: ["apps"]  resources: ["deployments"]  verbs: ["get","list","patch"]
  │                     │                            │
  │                     │                            └─ WHAT you may do. Common verbs:
  │                     │                               get, list, watch (read),
  │                     │                               create, update, patch, delete (write),
  │                     │                               deletecollection, "*" (all — avoid)
  │                     └─ WHICH resource TYPE. Plural, lowercase. Subresources use "/":
  │                        pods/log, pods/exec, deployments/scale
  └─ WHICH API group the resource belongs to:
     "" (empty)  = core group: pods, services, secrets, configmaps, nodes
     "apps"      = deployments, replicasets, statefulsets, daemonsets
     "batch"     = jobs, cronjobs
     "networking.k8s.io" = ingresses, networkpolicies
     "rbac.authorization.k8s.io" = roles, rolebindings
```

**A ServiceAccount — the identity a program authenticates as:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: orders
  name: deployer          # identity string becomes: system:serviceaccount:orders:deployer
```

**A RoleBinding — handing the card to the identity:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: orders                     # binding is namespaced → grants apply IN orders
  name: deployer-binding
subjects:                               # WHO gets the permissions
- kind: ServiceAccount
  name: deployer
  namespace: orders
roleRef:                                # WHICH rulebook entry they get
  kind: Role                            # Role (namespaced) or ClusterRole
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

Annotated on the two halves that people mix up:

```
subjects:                 roleRef:
- kind: ServiceAccount      kind: Role
  name: deployer            name: deployer-role
  │                         │
  │                         └─ WHICH permissions (points at a Role/ClusterRole).
  │                            roleRef is IMMUTABLE — to change it, delete & recreate.
  └─ WHO gets them (a ServiceAccount, User, or Group). You can list many subjects.
```

**The four-way matrix — the single most important table in RBAC:**

```
                    │ Role (namespaced)      │ ClusterRole (cluster-wide)
────────────────────┼────────────────────────┼───────────────────────────────
RoleBinding         │ grant IN one namespace │ grant a ClusterRole's rules, but
(namespaced)        │ (the classic case)     │ ONLY within the binding's namespace
────────────────────┼────────────────────────┼───────────────────────────────
ClusterRoleBinding  │ ❌ INVALID —           │ grant CLUSTER-WIDE across every
(cluster-wide)      │ can't cluster-bind a   │ namespace + cluster-scoped
                    │ namespaced Role        │ resources (nodes, PVs)
```

Read the tricky cell: a **RoleBinding → ClusterRole** is a real, useful pattern. It reuses a cluster-wide rulebook entry (like the built-in `view`) but *confines* it to one namespace. So you write the ClusterRole once and grant it per-namespace.

---

## Example 1 — basic

Create a read-only identity that can look at pods in `orders` but change nothing. Every line commented.

```yaml
# viewer.yaml — a read-only ServiceAccount for the orders namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: viewer
  namespace: orders
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                          # namespaced rulebook: only affects "orders"
metadata:
  name: pod-reader
  namespace: orders
rules:
- apiGroups: [""]                   # core group (pods live here)
  resources: ["pods", "pods/log"]   # pods and their logs
  verbs: ["get", "list", "watch"]   # READ only — no create/delete/patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                   # hand the pod-reader card to "viewer"
metadata:
  name: viewer-can-read-pods
  namespace: orders
subjects:
- kind: ServiceAccount
  name: viewer
  namespace: orders
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply and test it *as that account* using `--as` impersonation:

```bash
kubectl apply -f viewer.yaml

# Can viewer LIST pods in orders?  Expect: yes
kubectl auth can-i list pods -n orders \
  --as=system:serviceaccount:orders:viewer
# → yes

# Can viewer DELETE pods in orders?  Expect: no (no delete verb granted)
kubectl auth can-i delete pods -n orders \
  --as=system:serviceaccount:orders:viewer
# → no

# Can viewer read pods in a DIFFERENT namespace?  Expect: no (Role is namespaced)
kubectl auth can-i list pods -n default \
  --as=system:serviceaccount:orders:viewer
# → no
```

The three answers prove: read is allowed, write is denied (default-deny), and the grant is confined to `orders` (namespaced Role).

---

## Example 2 — production scenario

**The situation.** Your Topic 52 GitHub Actions pipeline deploys `orders-api`. Security flagged that its kubeconfig is bound to `cluster-admin`. Your job: replace it with a **deploy-only ServiceAccount** that can do *exactly* what the pipeline needs — update the `orders-api` deployment and watch the rollout — and nothing else. No reading Secrets, no touching other namespaces, no deleting anything.

**Step 1 — enumerate the minimum verbs the pipeline actually uses.** The pipeline runs:

```
kubectl set image deployment/orders-api ...   → needs: get, patch on deployments (apps)
kubectl rollout status deployment/orders-api  → needs: get, list, watch on deployments (apps)
                                                        get, list, watch on replicasets (apps)
                                                        get, list on pods (core)
```

That's the whole list. No `create`, no `delete`, no `secrets`.

**Step 2 — the least-privilege manifest:**

```yaml
# deployer-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployer
  namespace: orders
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: orders
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "patch"]   # patch = set image + scale; NO delete/create
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list", "watch"]            # rollout status reads replicasets
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]                      # rollout status inspects pods
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: orders
subjects:
- kind: ServiceAccount
  name: deployer
  namespace: orders
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

**Step 3 — mint a token for CI (Kubernetes 1.24+, tokens aren't auto-created as Secrets anymore):**

```bash
kubectl apply -f deployer-rbac.yaml
# Short-lived token (rotate it; or use a longer duration for CI):
TOKEN=$(kubectl create token deployer -n orders --duration=8760h)
```

**Step 4 — build a scoped kubeconfig with that token** (no admin cert anywhere):

```bash
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
kubectl config set-cluster prod --server="$APISERVER" --insecure-skip-tls-verify=false
kubectl config set-credentials deployer --token="$TOKEN"
kubectl config set-context deploy --cluster=prod --user=deployer --namespace=orders
# base64 this file into the GitHub secret KUBE_CONFIG (replacing the admin one).
```

**Step 5 — prove the blast radius is tiny.** Impersonate the account and confirm it can deploy but nothing else:

```bash
SA=system:serviceaccount:orders:deployer
kubectl auth can-i patch deployments -n orders --as=$SA          # → yes (needed)
kubectl auth can-i delete deployments -n orders --as=$SA         # → no  (blocked)
kubectl auth can-i get secrets -n orders --as=$SA                # → no  (blocked!)
kubectl auth can-i patch deployments -n payments --as=$SA        # → no  (other ns blocked)
kubectl auth can-i '*' '*' --all-namespaces --as=$SA             # → no  (not admin)
```

**The payoff:** the worst an attacker can do with a leaked CI token is patch the `orders-api` deployment in one namespace — they cannot read your Postgres password (Secret), cannot delete workloads, cannot touch `payments`. You turned "leaked token = lost cluster" into "leaked token = someone can restart orders-api." That is least privilege in action, and it's exactly the Role your Topic 52 pipeline should authenticate with.

---

## Common mistakes

**Mistake 1 — Using a ClusterRoleBinding when you meant a namespaced grant.**

```yaml
kind: ClusterRoleBinding   # grants the ClusterRole in EVERY namespace!
roleRef:
  kind: ClusterRole
  name: admin              # built-in admin ClusterRole
subjects:
- kind: ServiceAccount
  name: deployer
  namespace: orders
```

Root cause: a `ClusterRoleBinding` applies the role **cluster-wide**, so your "deployer" now has admin in `payments`, `kube-system`, everywhere — the opposite of least privilege. The `namespace: orders` on the subject does *not* confine it; only the *binding kind* controls scope. Right way: use a namespaced `RoleBinding` (which confines even a ClusterRole to its own namespace), or a `Role` + `RoleBinding` pair.

**Mistake 2 — Forgetting the core apiGroup is the empty string.**

```yaml
- apiGroups: ["core"]        # WRONG — there is no group literally named "core"
  resources: ["pods"]
  verbs: ["get"]
```

Then:

```
Error from server (Forbidden): pods is forbidden:
User "..." cannot get resource "pods" in API group "" in the namespace "orders"
```

Root cause: pods/services/secrets/configmaps are in the **core group**, whose name is the *empty string* `""`, not `"core"`. Your rule targeted a group that doesn't exist, so it matched nothing. Right way: `apiGroups: [""]` for core resources; `apiGroups: ["apps"]` for deployments; etc.

**Mistake 3 — Wrong resource name (singular, or the kind instead of the resource).**

```yaml
- apiGroups: ["apps"]
  resources: ["Deployment"]   # WRONG — capitalized, singular (that's the KIND)
  verbs: ["patch"]
```

Root cause: RBAC matches the **resource name** as it appears in the REST API — **plural, lowercase** (`deployments`), not the manifest `kind` (`Deployment`). Your rule silently matches nothing and the request is denied. Right way: `resources: ["deployments"]`. Find exact names with `kubectl api-resources`.

**Mistake 4 — Granting `verbs: ["*"]` or `resources: ["*"]` "to be safe."**

```yaml
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]        # this IS cluster-admin-level power on those objects
```

Root cause: wildcards defeat the entire point of RBAC. A pipeline that only needs to patch one deployment now can delete Secrets and create pods. When it leaks, you lose everything. Right way: list the *exact* verbs and resources the automation uses (see Example 2). If you don't know them, run the task and read the `Forbidden` errors — each one tells you the precise missing (verb, resource).

**Mistake 5 — Editing `roleRef` and being surprised it's rejected.**

```
The RoleBinding "deployer-binding" is invalid:
roleRef: Invalid value: ... : cannot change roleRef
```

Root cause: `roleRef` is **immutable** by design — you can't repoint a binding at a different Role after creation (it would silently change who has what). Right way: `kubectl delete rolebinding deployer-binding` and recreate it pointing at the new Role. (You *can* freely add/remove `subjects`.)

---

## Hands-on proof

Run against any cluster to watch default-deny, additive grants, and namespace confinement happen for real.

```bash
# 0. Setup.
kubectl create namespace orders
kubectl create serviceaccount deployer -n orders

# 1. Before any binding: default-deny. Prove the account can do NOTHING.
kubectl auth can-i patch deployments -n orders \
  --as=system:serviceaccount:orders:deployer          # → no

# 2. Create a Role that allows only get/patch on deployments.
kubectl create role deployer-role -n orders \
  --verb=get --verb=list --verb=watch --verb=patch \
  --resource=deployments

# 3. Bind it to the ServiceAccount.
kubectl create rolebinding deployer-binding -n orders \
  --role=deployer-role \
  --serviceaccount=orders:deployer

# 4. Now patch is allowed (additive: one matching rule flips no → yes).
kubectl auth can-i patch deployments -n orders \
  --as=system:serviceaccount:orders:deployer          # → yes

# 5. But delete is still denied (no delete verb granted).
kubectl auth can-i delete deployments -n orders \
  --as=system:serviceaccount:orders:deployer          # → no

# 6. And it's confined to "orders" — another namespace stays denied.
kubectl auth can-i patch deployments -n default \
  --as=system:serviceaccount:orders:deployer          # → no

# 7. See the exact rules that were granted.
kubectl describe role deployer-role -n orders

# 8. Decode the request pipeline: what username does the SA present?
kubectl create token deployer -n orders | cut -d. -f2 | base64 -d 2>/dev/null; echo
#   → look for "sub":"system:serviceaccount:orders:deployer"

# 9. Clean up.
kubectl delete namespace orders
```

What you verified: (1) nothing is allowed by default; (4) a single Role+Binding adds exactly one capability; (5) unlisted verbs stay denied; (6) a namespaced Role never leaks to other namespaces; (8) the SA's identity string is what bindings match.

---

## Practice exercises

### Exercise 1 — easy
Create namespace `orders` and ServiceAccount `viewer`. Using `kubectl create role` and `kubectl create rolebinding`, grant `viewer` read-only access (`get`, `list`, `watch`) to `pods` only. Then run three `kubectl auth can-i ... --as=system:serviceaccount:orders:viewer` checks proving: (a) it can list pods, (b) it cannot delete pods, (c) it cannot list secrets. Explain why (b) and (c) are denied even though you never wrote a "deny" rule.

### Exercise 2 — medium
Build the deploy-only `deployer` ServiceAccount from Example 2 with a `Role` + `RoleBinding` giving `get/list/watch/patch` on `deployments` and `get/list` on `pods`. Then deliberately try to make the pipeline do something it shouldn't: run `kubectl auth can-i create secrets -n orders --as=...` and `kubectl auth can-i patch deployments -n kube-system --as=...`. Both should be `no`. Now change your `RoleBinding` into a `ClusterRoleBinding` pointing at a ClusterRole with the same rules, re-check the `kube-system` question, and explain what changed and why that's dangerous.

### Exercise 3 — hard (production simulation)
You have three consumers of the cluster that each need a different, minimal identity: (1) the **CI pipeline** — patch `orders-api` deployment in `orders` only; (2) a **read-only dashboard** that must list pods/services in *all* namespaces but read nothing sensitive; (3) the **`orders-api` pod itself**, which must read one specific ConfigMap named `orders-config` and nothing else. For each, decide `Role` vs `ClusterRole` and `RoleBinding` vs `ClusterRoleBinding`, write the manifests, and for consumer (3) use the `resourceNames` field to restrict access to *just* the `orders-config` ConfigMap. Then prove with `kubectl auth can-i` (including `--as` for the pod's ServiceAccount) that each identity can do its job and nothing more. Explain why the dashboard needs a ClusterRole but the CI pipeline must not.

---

## Mental model checkpoint

Answer from memory:

1. What are the three separate objects in RBAC, and which one names the *permissions*, which names the *identity*, and which *links* them?
2. RBAC is "default-____ and purely ____." Fill both blanks. Are there "deny" rules?
3. What's the difference in *scope* between a `Role` and a `ClusterRole`? Between a `RoleBinding` and a `ClusterRoleBinding`?
4. Which of the four Role/ClusterRole × Binding combinations is invalid, and which one lets you reuse a cluster-wide rulebook entry inside a single namespace?
5. What is the `apiGroups` value for core resources like pods and secrets, and why do people get it wrong?
6. What identity string does a ServiceAccount named `deployer` in namespace `orders` present, and where does a pod find its token on disk?
7. Which three attributes of a request does the `Forbidden` error report, and how do you use it to fix a missing permission?

---

## Quick reference card

| Command / concept | What it does | Key detail |
|---|---|---|
| `Role` | Namespaced list of allowed (apiGroup, resource, verb) | Only works in its own namespace |
| `ClusterRole` | Cluster-wide rulebook entry | Needed for cluster-scoped resources (nodes, PVs) and all-namespace grants |
| `RoleBinding` | Grants a Role/ClusterRole in ONE namespace | Confines even a ClusterRole to its namespace |
| `ClusterRoleBinding` | Grants a ClusterRole across ALL namespaces | Where over-privilege accidents happen |
| `ServiceAccount` | Non-human identity for pods/CI | Token mounted at `/var/run/secrets/.../token` |
| `apiGroups: [""]` | The core group | pods, services, secrets, configmaps |
| `verbs` | get/list/watch/create/update/patch/delete | RBAC is additive; no deny verbs |
| `resourceNames` | Restrict a rule to named objects | e.g. only ConfigMap `orders-config` |
| `kubectl auth can-i VERB RES --as=SA` | Test a permission without doing it | The fastest RBAC debugging tool |
| `kubectl create token SA -n ns` | Mint a bearer token (1.24+) | Tokens no longer auto-created as Secrets |

---

## When would I use this at work?

1. **Locking down a CI/CD pipeline.** When security asks "why does GitHub have cluster-admin?", you create the Example 2 deploy-only ServiceAccount — patch on one deployment in one namespace — and swap the kubeconfig. A leaked CI token can now only restart `orders-api`, not read your database Secret.

2. **Giving a teammate safe access.** A new backend dev needs to view logs and restart pods in `orders` but must not touch `payments` or read Secrets. You bind them (via a Role + RoleBinding) to exactly `pods`, `pods/log`, and `deployments` (get/list/patch) in `orders` only.

3. **Hardening the app's own pod.** You realize `orders-api` runs as the `default` ServiceAccount with a mounted token. You create a dedicated ServiceAccount with access to only the one ConfigMap it reads (via `resourceNames`), so an RCE in the app can't pivot to the whole cluster through that token.

---

## Connected topics

- **Study before:**
  - **Topic 29 — Kubernetes Architecture**: the API server, where the authentication → authorization → admission pipeline runs.
  - **Topic 36 — ConfigMaps and Secrets**: what the tokens RBAC protects actually are, and how they mount into pods.
  - **Topic 37 — Namespaces**: the isolation boundary that Roles and RoleBindings are scoped to.
  - **Topic 52 — CI/CD with Kubernetes**: the pipeline whose credentials this topic locks down.
- **Study after:**
  - **Topic 50 — Secrets Management at Scale**: RBAC controls *who can read* Secrets; that topic controls *how they're stored and rotated*.
  - **Topic 54 — Multi-Environment Strategy**: per-environment ServiceAccounts and least-privilege promotion across dev/staging/prod.
