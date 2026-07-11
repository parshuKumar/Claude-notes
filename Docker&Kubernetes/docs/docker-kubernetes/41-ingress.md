# 41 — Ingress
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Imagine a giant office building with hundreds of small companies inside. Every visitor arrives at the **front door**, where a single **receptionist** sits with a big directory board. You walk up and say "I'm here for Orders Inc." The receptionist reads the board, sees "Orders Inc. → Floor 3, Room 12," and points you there. You never wander the hallways yourself; the receptionist routes you.

Now imagine the *wasteful* alternative: give every company its **own separate front door on the street**, its own security guard, its own street address. One hundred companies, one hundred doors, one hundred guards. Expensive and messy.

In Kubernetes, giving every app its own street door is a **LoadBalancer Service** — each one asks the cloud for its own public IP and its own cloud load balancer (which costs real money every month). The single smart receptionist with the directory board is an **Ingress**: one front door, one public IP, and a set of rules that say "this hostname or this URL path goes to that app inside."

The receptionist reading the directory board is the **Ingress controller**. The directory board itself — the printed list of "who goes where" — is the **Ingress resource** (a YAML object). The board is just paper; it does nothing until a receptionist reads it and acts on it. That split — dumb rules vs. the program that enforces them — is the single most important idea in this whole topic.

---

## The Linux kernel feature underneath

An Ingress has **no direct Linux kernel primitive** of its own — it is a pure Kubernetes abstraction. But underneath the abstraction sits a very real, very old piece of software doing very real kernel work: a **reverse proxy** (usually nginx) running inside a normal pod.

So the "kernel-level truth" of Ingress is this chain:

1. **A real listening socket.** The Ingress controller pod runs nginx, which calls `bind()` and `listen()` on TCP ports **80** and **443** inside its own network namespace (remember network namespaces from **Topic 01** and **Topic 40**). That socket is a real file descriptor in a real process on a real node.

2. **Layer 7 (HTTP) parsing.** When bytes arrive, nginx does something a plain Service **cannot** do: it reads the raw HTTP request, parses the `Host:` header and the request path (`GET /api/orders HTTP/1.1`), and *decides* where to send it based on the text. A Service (Topic 35) works at **Layer 4** — it only sees IP + port and blindly forwards; it has no idea what a URL is. Ingress works at **Layer 7** — it reads the actual HTTP words. That difference is the entire reason Ingress exists.

3. **TLS termination via OpenSSL.** For HTTPS, nginx uses the kernel-adjacent OpenSSL library to do the TLS handshake: it holds the private key, decrypts the `https://` traffic, and hands **plaintext** HTTP to your app. The CPU-heavy asymmetric crypto happens once, at the edge, in the controller pod.

4. **A second proxy hop.** After nginx decides the target, it opens a *new* TCP connection to your app's pod IP (or Service ClusterIP) and copies bytes across. Under the hood that is `iptables`/IPVS routing (Topic 35/40) plus normal socket forwarding.

So when someone says "Ingress routes traffic," what physically happens is: **a pod running nginx accepts a socket, parses HTTP text, terminates TLS, and proxies the request to another pod.** The Ingress YAML you write is merely the *configuration file* that this nginx reloads.

---

## What is this?

An **Ingress** is a Kubernetes object that defines HTTP/HTTPS routing rules — "requests for host `api.shop.com` path `/orders` go to Service `orders-api`" — for traffic entering the cluster from outside. It operates at Layer 7, so it can route on hostname, URL path, and headers, and it can terminate TLS.

An Ingress is **just rules**. It does nothing by itself. You must also run an **Ingress controller** (a pod, commonly ingress-nginx) that watches Ingress objects and reconfigures a real reverse proxy to obey them.

---

## Why does it matter for a backend developer?

You have three apps: `orders-api`, a `payments-api`, and a `frontend`. You want the outside world to reach them at:

- `https://shop.com/` → frontend
- `https://shop.com/api/orders` → orders-api
- `https://shop.com/api/payments` → payments-api

**Without Ingress**, your only "expose to the internet" tool is a **LoadBalancer Service** (Topic 35). That means:

- Three LoadBalancer Services → three cloud load balancers → **three monthly bills** and **three separate public IPs**. Your users would have to hit `shop.com:30001`, `shop.com:30002`, `shop.com:30003`. Ugly, and you can't do path routing at all.
- **No TLS.** A LoadBalancer Service is Layer 4 — it can't read URLs, can't hold a certificate, can't do `https`. You'd have to bake TLS into every single app yourself.
- **No shared hostname.** You can't send `/api/orders` to one app and `/api/payments` to another, because L4 doesn't know what a path is.

Without understanding Ingress you will:
- Burn money on a LoadBalancer per service (each is ~$15–25/month on AWS/GCP).
- Reimplement TLS certificates inside every Node.js app instead of terminating once at the edge.
- Not understand why your `503 Service Temporarily Unavailable` is coming from nginx and not your app.
- Confuse the **Ingress resource** (rules) with the **Ingress controller** (the program) — the #1 reason people's Ingress "does nothing."

With one Ingress + one controller you get **one public IP, one certificate, and clean host/path routing** for your entire cluster.

---

## The physical reality

When Ingress is active in your `orders` namespace, here is what **actually exists**:

**1. The Ingress controller pods** (installed once, cluster-wide, usually in namespace `ingress-nginx`):

```
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS
ingress-nginx-controller-7d9f8c6b5-abcde    1/1     Running   0
```

**2. A real nginx config file inside that pod.** The controller writes your rules into a genuine `nginx.conf`:

```
$ kubectl exec -n ingress-nginx ingress-nginx-controller-7d9f8c6b5-abcde -- cat /etc/nginx/nginx.conf | grep -A6 orders
  location /api/orders/ {
      set $service_name  "orders-api";
      set $service_port  "80";
      proxy_pass http://upstream_balancer;   ← proxies to orders-api pods
  }
```

That file is the *physical proof* that your YAML became a running proxy config.

**3. A LoadBalancer Service in front of the controller** — the ONE cloud load balancer for the whole cluster:

```
$ kubectl get svc -n ingress-nginx
NAME                       TYPE           EXTERNAL-IP      PORT(S)
ingress-nginx-controller   LoadBalancer   203.0.113.50     80:31234/TCP,443:31567/TCP
```

`203.0.113.50` is the single public IP the internet hits.

**4. Your Ingress object stored in etcd** (Topic 29) in the `orders` namespace:

```
$ kubectl get ingress -n orders
NAME             CLASS   HOSTS          ADDRESS         PORTS     AGE
orders-ingress   nginx   shop.com       203.0.113.50    80, 443   3d
```

**5. A TLS Secret holding the certificate + private key** (Topic 36), also in etcd:

```
$ kubectl get secret shop-com-tls -n orders
NAME           TYPE                DATA   AGE
shop-com-tls   kubernetes.io/tls   2      3d      ← tls.crt + tls.key
```

So Ingress on disk/in memory is: **an nginx pod with a generated nginx.conf + one shared cloud LB + an Ingress object in etcd + a TLS secret.** No magic — a proxy reading a config.

---

## How it works — step by step

Full trace of a request `https://shop.com/api/orders/42` reaching `orders-api`:

1. **You `kubectl apply` the Ingress object.** The API server validates it and writes it to etcd (Topic 29). At this instant nothing is routing yet — it's just data.

2. **The Ingress controller's watch fires.** The ingress-nginx controller holds a **watch** on Ingress objects (the reconciliation loop, Topic 30). The API server streams it the event "new Ingress `orders-ingress` in namespace orders."

3. **The controller regenerates nginx.conf.** It reads the rules, looks up the backing Services and their Endpoints (the actual pod IPs of `orders-api`), and writes a fresh `/etc/nginx/nginx.conf` with `server {}` and `location {}` blocks.

4. **nginx reloads.** The controller signals nginx to reload config gracefully (`nginx -s reload`) — old connections drain, new config takes effect, zero dropped requests.

5. **DNS points `shop.com` at the LB.** You (once) set an `A` record: `shop.com → 203.0.113.50` (the LoadBalancer Service's external IP).

6. **A user's browser resolves and connects.** Browser looks up `shop.com` → `203.0.113.50`, opens a TCP connection to port **443**. The cloud LB forwards it to the ingress-nginx controller pod.

7. **TLS handshake + termination.** nginx presents the certificate from Secret `shop-com-tls`, completes the TLS handshake using the private key, and **decrypts** the request. It now holds plaintext: `GET /api/orders/42 HTTP/1.1\r\nHost: shop.com`.

8. **Rule matching (Layer 7).** nginx reads `Host: shop.com` and path `/api/orders/42`. It matches your rule `host: shop.com, path: /api/orders`. The winning backend is Service `orders-api:80`.

9. **Proxy to a pod.** nginx opens a **new** TCP connection to a healthy `orders-api` pod IP (it load-balances across the Service's Endpoints itself, or via the ClusterIP). It copies the plaintext HTTP request across.

10. **Your Node.js app responds.** `orders-api` (which never saw TLS — it just got plain HTTP) returns the JSON. nginx copies the response back over the still-encrypted browser connection, re-encrypting it on the way out.

The key insight: **your app never deals with certificates, hostnames, or paths from the internet.** nginx did all of it at the edge, and handed your app a boring plaintext HTTP request as if it came from next door.

---

## Ingress vs. LoadBalancer Service — the core comparison

```
                        LoadBalancer Service              Ingress
                        ────────────────────              ───────────────────────
OSI Layer               L4 (TCP/UDP)                      L7 (HTTP/HTTPS)
Sees                    IP + port only                    Hostname, path, headers, cookies
Routing granularity     one Service per LB                many Services, one LB
Cost                    one cloud LB per Service ($$$)    one cloud LB total ($)
TLS                     no (app must do it)               yes, terminated at edge
Host-based routing      no                                yes (shop.com vs admin.shop.com)
Path-based routing      no                                yes (/api/orders vs /api/payments)
Needs a controller?     no (cloud provides the LB)        YES (you must install one)
```

The punchline: a **LoadBalancer Service is the raw front door**; **Ingress is a smart HTTP router that usually sits behind one shared LoadBalancer.** In fact, the ingress-nginx controller is *itself* exposed by a single LoadBalancer Service — Ingress doesn't replace LoadBalancer, it multiplexes many apps behind one.

---

## Exact syntax breakdown

A full Ingress manifest for `orders-api`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders-ingress
  namespace: orders
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.com
      secretName: shop-com-tls
  rules:
    - host: shop.com
      http:
        paths:
          - path: /api/orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: orders-api
                port:
                  number: 80
```

Annotated:

```
apiVersion: networking.k8s.io/v1
│           │
│           └─ the stable API group for Ingress since K8s 1.19. Older docs use
│              extensions/v1beta1 — that is REMOVED, do not use it.
│
└─ kind: Ingress → tells the API server this is routing rules, not a workload

metadata:
  name: orders-ingress            ← the object's name inside the namespace
  namespace: orders               ← Ingress is namespaced; it routes to Services in THIS namespace
  annotations:                    ← controller-specific knobs (NOT standard fields)
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    │                                            │
    │                                            └─ rewrite the URL before proxying:
    │                                               strip "/api/orders" so the app sees "/42",
    │                                               using capture group $2 from the path regex
    │
    └─ every nginx-specific behavior is an annotation with this prefix.
       A DIFFERENT controller (Traefik, HAProxy) uses DIFFERENT annotations.

    cert-manager.io/cluster-issuer: letsencrypt-prod
    │                               │
    │                               └─ which ClusterIssuer cert-manager should use
    │                                  to auto-fetch a Let's Encrypt certificate
    │
    └─ read by cert-manager (a separate operator), NOT by nginx

spec:
  ingressClassName: nginx         ← WHICH controller should own this Ingress.
  │                                  If you have both nginx and Traefik installed,
  │                                  this decides who reads it. Omitting it = nobody
  │                                  may act on your Ingress (silent no-op bug).
  │
  tls:                            ← enable HTTPS + TLS termination
    - hosts:
        - shop.com                ← the cert must be valid for this hostname
      secretName: shop-com-tls    ← the Secret (type kubernetes.io/tls) holding
      │                              tls.crt + tls.key. nginx loads the key to
      │                              decrypt. cert-manager can CREATE this for you.
      │
  rules:                          ← the routing table (a list)
    - host: shop.com              ← match requests whose HTTP "Host:" header = shop.com
      http:
        paths:
          - path: /api/orders(/|$)(.*)
            │    │
            │    └─ the URL path to match. The regex here works WITH rewrite-target
            │       to capture the tail. Simpler form: just "/api/orders".
            │
            pathType: Prefix      ← how to match the path. THREE choices:
            │                        • Prefix  — match this path and everything under it
            │                        • Exact   — match ONLY this exact string
            │                        • ImplementationSpecific — controller decides (regex)
            │
            backend:
              service:
                name: orders-api  ← the Service (Topic 35) to forward to
                port:
                  number: 80      ← the Service's port (its clusterIP port)
```

`pathType` values you must know:

```
pathType: Prefix    /api/orders   matches /api/orders, /api/orders/42, /api/orders/x/y
pathType: Exact     /api/orders   matches ONLY /api/orders (not /api/orders/42)
pathType: Implementation…         controller-defined; nginx treats path as a regex
```

---

## Example 1 — basic

The smallest possible Ingress: send everything for host `shop.com` to `orders-api`. Every line commented.

```yaml
# ingress-basic.yaml
apiVersion: networking.k8s.io/v1   # the v1 (stable) Ingress API
kind: Ingress                      # this object is a set of routing rules
metadata:
  name: orders-basic               # name of THIS ingress object
  namespace: orders                # lives in the orders namespace
spec:
  ingressClassName: nginx          # the ingress-nginx controller should handle this
  rules:                           # the routing table
    - host: shop.com               # only match requests with Host: shop.com
      http:
        paths:
          - path: /                # match every path (root and below)
            pathType: Prefix       # "/" as Prefix = catch-all
            backend:
              service:
                name: orders-api   # forward to the orders-api Service
                port:
                  number: 80       # on port 80
```

Apply and check:

```bash
kubectl apply -f ingress-basic.yaml
#      │      │
#      │      └─ -f: read the object from this file
#      └─ create/update the Ingress in etcd

kubectl get ingress -n orders
# NAME           CLASS   HOSTS      ADDRESS        PORTS   AGE
# orders-basic   nginx   shop.com   203.0.113.50   80      10s
#                                   │
#                                   └─ once ADDRESS is filled in, the controller
#                                      has accepted it and nginx is routing

# Test it WITHOUT real DNS by faking the Host header:
curl -H "Host: shop.com" http://203.0.113.50/api/orders
#    │  │                 │
#    │  │                 └─ hit the ingress-nginx LB's public IP
#    │  └─ pretend we are shop.com (so the host rule matches)
#    └─ -H: set a raw HTTP header
# → {"orders":[...]}   ← the request reached orders-api through nginx
```

`curl -H "Host: shop.com"` is the trick every backend dev should know: it lets you test host-based Ingress rules **before** you own the DNS.

---

## Example 2 — production scenario

**The situation.** Your team runs the whole shop behind Kubernetes. You need:

- `https://shop.com/` → `frontend` (the React app)
- `https://shop.com/api/orders` → `orders-api`
- `https://shop.com/api/payments` → `payments-api`
- `https://admin.shop.com/` → `admin-dashboard` (a *different* hostname, internal team only)

And all of it must be HTTPS with an auto-renewing certificate, because your `payments-api` handles card data and plaintext is not an option.

This is **host-based AND path-based routing + TLS** in one Ingress:

```yaml
# ingress-prod.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  namespace: orders
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # auto-provision the cert
    nginx.ingress.kubernetes.io/ssl-redirect: "true"   # force http → https
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.com
        - admin.shop.com
      secretName: shop-tls        # cert-manager will create+fill this Secret
  rules:
    # ---- Public site: one hostname, THREE paths (path-based routing) ----
    - host: shop.com
      http:
        paths:
          - path: /api/orders     # most specific paths first
            pathType: Prefix
            backend:
              service:
                name: orders-api
                port:
                  number: 80
          - path: /api/payments
            pathType: Prefix
            backend:
              service:
                name: payments-api
                port:
                  number: 80
          - path: /               # catch-all LAST → the frontend
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    # ---- Admin site: a DIFFERENT hostname (host-based routing) ----
    - host: admin.shop.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-dashboard
                port:
                  number: 80
```

**What happens now, request by request:**

- `GET https://shop.com/api/orders/42` → nginx matches host `shop.com`, longest path `/api/orders` → `orders-api`.
- `GET https://shop.com/api/payments/charge` → same host, path `/api/payments` → `payments-api`.
- `GET https://shop.com/products` → same host, no `/api/*` match, falls to `/` catch-all → `frontend`.
- `GET https://admin.shop.com/` → different host entirely → `admin-dashboard`.
- `GET http://shop.com/...` (plain HTTP) → `ssl-redirect` annotation → nginx returns `308` redirecting to `https://`.

**Why order matters.** nginx picks the **longest matching prefix**, so `/api/orders` wins over `/` for a `/api/orders/...` request even though both technically match. But listing the catch-all `/` last is good hygiene and avoids surprises with some controllers.

**The payoff:** four apps, two hostnames, full HTTPS, **one public IP, one certificate, one monthly LB bill.** Adding a fifth app tomorrow is three lines of YAML, not a new cloud load balancer.

---

## TLS termination with cert-manager

Manually getting, installing, and *renewing* certificates is painful — Let's Encrypt certs expire every 90 days. **cert-manager** is an operator (Topic 30's reconciliation loop applied to certificates) that automates the whole thing.

**How the pieces fit:**

```
   ┌─────────────────┐   watches    ┌──────────────────┐
   │  Ingress object │◀─────────────│   cert-manager   │
   │ (has annotation │              │    (operator)    │
   │  cert-manager   │              └────────┬─────────┘
   │  .io/issuer)    │                       │ 1. sees annotation
   └─────────────────┘                       │ 2. creates a Certificate object
                                             │ 3. proves you own shop.com
                                             │    (HTTP-01 challenge via nginx)
                                             │ 4. gets cert from Let's Encrypt
                                             ▼
                                   ┌──────────────────────┐
                                   │  Secret: shop-tls     │  ← tls.crt + tls.key
                                   │  (type kubernetes.io/ │     nginx loads this
                                   │        tls)           │     to terminate TLS
                                   └──────────────────────┘
```

A **ClusterIssuer** tells cert-manager *where* to get certificates:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer           # cluster-wide (not namespaced) certificate source
metadata:
  name: letsencrypt-prod
spec:
  acme:                        # the ACME protocol (Let's Encrypt speaks it)
    server: https://acme-v02.api.letsencrypt.org/directory   # LE production
    email: ops@shop.com        # LE emails you before expiry as backup
    privateKeySecretRef:
      name: letsencrypt-prod-account-key   # LE account key, auto-created
    solvers:
      - http01:                # prove domain ownership via an HTTP challenge
          ingress:
            class: nginx       # cert-manager routes the challenge through nginx
```

Then you only add **one annotation** to your Ingress:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [shop.com]
      secretName: shop-tls     # cert-manager CREATES and FILLS this Secret for you
```

cert-manager sees the annotation, does the ACME dance, writes `shop-tls`, and **auto-renews it every ~60 days forever.** You never touch a certificate again. This is TLS termination the way real teams run it.

---

## Common mistakes

**Mistake 1 — Creating an Ingress with no controller installed.**

```
$ kubectl apply -f ingress-prod.yaml
ingress.networking.k8s.io/shop-ingress created

$ kubectl get ingress -n orders
NAME           CLASS   HOSTS      ADDRESS   PORTS   AGE
shop-ingress   nginx   shop.com             80,443  5m
                                  │
                                  └─ ADDRESS is EMPTY forever, curl times out
```

Root cause: an Ingress object is **just rules in etcd**. With no controller pod watching, nobody ever configures a proxy — the receptionist's desk is empty. `kubectl get pods -n ingress-nginx` returns nothing. Right way: install a controller first (`helm install ingress-nginx ...` — Topic 49), *then* create Ingress objects.

**Mistake 2 — Forgetting `ingressClassName`.**

```
# Ingress applied fine, but no controller acts on it.
$ kubectl describe ingress shop-ingress -n orders
Events:  <none>          ← no controller ever claimed it
```

Root cause: with multiple controllers (or strict controllers), an Ingress with no `ingressClassName` is **owned by nobody**. Right way: always set `spec.ingressClassName: nginx` (or set a default IngressClass cluster-wide).

**Mistake 3 — TLS Secret in the wrong namespace.**

```
$ kubectl describe ingress shop-ingress -n orders
Warning  Secret orders/shop-tls does not exist
```

Root cause: an Ingress can **only** reference a TLS Secret in its **own namespace**. If your cert Secret is in `default` but your Ingress is in `orders`, nginx can't find the key and silently serves a fake self-signed cert (browser shows `NET::ERR_CERT_AUTHORITY_INVALID`). Right way: the TLS Secret must live in the same namespace as the Ingress.

**Mistake 4 — Path routing sends the wrong URL to the app.**

Your `orders-api` serves routes at `/42`, but you route `/api/orders` to it. Requests arrive as `/api/orders/42` and your app returns `404` because it has no `/api/orders/42` route.

```
$ curl -H "Host: shop.com" http://203.0.113.50/api/orders/42
Cannot GET /api/orders/42        ← Express 404: your app never had this path
```

Root cause: Ingress forwards the **full original path** unless you rewrite it. Right way: use `nginx.ingress.kubernetes.io/rewrite-target` to strip the prefix, OR make your app mount its router at `/api/orders`.

**Mistake 5 — Expecting Ingress to route non-HTTP traffic (Postgres, gRPC-raw, TCP).**

```
# You try to expose Postgres (port 5432) via Ingress → it just doesn't work.
```

Root cause: Ingress is **HTTP/HTTPS only** by spec — it parses HTTP, so raw TCP like Postgres has nothing to parse. Right way: expose Postgres with a `LoadBalancer` or `NodePort` Service (Topic 35), or use ingress-nginx's separate TCP-services ConfigMap. Ingress is for web traffic, not databases.

---

## Hands-on proof

Run these on a cluster with ingress-nginx installed (e.g. `minikube addons enable ingress`, or Kind + helm).

```bash
# 1. Confirm the controller (the receptionist) actually exists.
kubectl get pods -n ingress-nginx
kubectl get svc  -n ingress-nginx        # note the EXTERNAL-IP / NodePort

# 2. Deploy a tiny orders-api to route to.
kubectl create namespace orders 2>/dev/null
kubectl create deployment orders-api --image=hashicorp/http-echo \
  -- -text="hello from orders-api" -listen=:5678 -n orders
kubectl expose deployment orders-api --port=80 --target-port=5678 -n orders

# 3. Apply the basic Ingress from Example 1.
kubectl apply -f ingress-basic.yaml

# 4. Watch the controller ACCEPT it (ADDRESS gets filled in).
kubectl get ingress -n orders -w        # Ctrl-C once ADDRESS appears

# 5. PROOF the YAML became real nginx config:
POD=$(kubectl get pod -n ingress-nginx -l app.kubernetes.io/component=controller -o name | head -1)
kubectl exec -n ingress-nginx $POD -- cat /etc/nginx/nginx.conf | grep -A4 orders-api
#   → you will SEE a server{} / location{} block naming orders-api

# 6. PROOF it routes (fake the Host header, no DNS needed):
IP=$(minikube ip 2>/dev/null || kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -H "Host: shop.com" http://$IP/
#   → "hello from orders-api"   ← request went LB → nginx → orders-api pod

# 7. Clean up.
kubectl delete ingress orders-basic -n orders
kubectl delete namespace orders
```

What you verified: the controller **exists** (step 1), your Ingress YAML **turned into a real nginx.conf** (step 5), and a request with the right `Host` header **reached your pod through the proxy** (step 6).

---

## Practice exercises

### Exercise 1 — easy
Deploy two `hashicorp/http-echo` apps (`orders-api` saying "orders" and `payments-api` saying "payments") in namespace `orders`. Write ONE Ingress on host `shop.com` that sends `/api/orders` to the first and `/api/payments` to the second. Test both with `curl -H "Host: shop.com" http://<ip>/api/orders` and `/api/payments`. Which field made the two paths go to different Services?

### Exercise 2 — medium
Take the same two apps but route them by **hostname** instead of path: `orders.shop.com` → orders-api and `payments.shop.com` → payments-api, both at path `/`. Test with `curl -H "Host: orders.shop.com" http://<ip>/`. Then run `kubectl exec` into the controller pod and `grep` the two `server_name` blocks in nginx.conf. Explain in one sentence how nginx tells the two hostnames apart even though both hit the same IP.

### Exercise 3 — hard (production simulation)
On a cluster reachable from the internet (or using a local ACME server like Pebble), install cert-manager, create a `letsencrypt-staging` ClusterIssuer, and add TLS to your `shop.com` Ingress with the `cert-manager.io/cluster-issuer` annotation and a `secretName: shop-tls`. Watch cert-manager create a `Certificate` object, solve the HTTP-01 challenge, and populate the Secret (`kubectl get secret shop-tls -o yaml`). Then curl `https://` and confirm TLS is terminated at nginx (your app logs show plain HTTP, no TLS). Write down: which component held the private key, and which component never saw it.

---

## Mental model checkpoint

Answer from memory:

1. What is the difference between an Ingress **resource** and an Ingress **controller**, and which one actually moves traffic?
2. At which OSI layer does Ingress operate, and why does that let it do host/path routing that a LoadBalancer Service cannot?
3. Why does one Ingress + one controller cost less than three LoadBalancer Services?
4. Where does TLS termination physically happen, and does your `orders-api` pod ever see encrypted bytes?
5. What does `ingressClassName` decide, and what breaks if you omit it?
6. In which namespace must a TLS Secret live relative to the Ingress that uses it?
7. What does cert-manager automate, and what single annotation triggers it?

---

## Quick reference card

| Field / concept | What it does | Key detail |
|---|---|---|
| `kind: Ingress` | HTTP/HTTPS routing rules | Just data; does nothing without a controller |
| Ingress controller | The pod (nginx) that reads Ingress + proxies | Install once, cluster-wide |
| `ingressClassName` | Which controller owns this Ingress | Omit it → nobody acts on it |
| `rules[].host` | Match on HTTP `Host:` header | Enables host-based routing |
| `paths[].path` + `pathType` | Match on URL path | `Prefix`, `Exact`, or ImplementationSpecific |
| `backend.service` | Which Service to forward to | Must be in the same namespace |
| `tls[].secretName` | TLS cert+key Secret | Type `kubernetes.io/tls`, same namespace |
| `cert-manager.io/cluster-issuer` | Auto-provision + renew cert | Read by cert-manager, not nginx |
| `rewrite-target` annotation | Rewrite path before proxying | nginx-specific; other controllers differ |
| Ingress vs LoadBalancer | L7 smart router vs L4 raw door | Ingress multiplexes many apps behind one LB |

---

## When would I use this at work?

1. **Consolidating public endpoints.** Your team has 8 microservices each behind a LoadBalancer costing money. You put them all behind one Ingress with path/host rules and cut 8 cloud LBs down to 1, saving real budget and giving users clean URLs.

2. **Adding HTTPS without touching app code.** Security says every endpoint must be HTTPS. Instead of adding TLS to eight Node.js apps, you terminate TLS once at the Ingress with cert-manager, and every app keeps speaking plain HTTP internally.

3. **Blue/green and canary at the edge.** You use ingress-nginx canary annotations to send 5% of `/api/orders` traffic to `orders-api:v2` while 95% stays on `v1`, all by editing Ingress annotations — no changes to the apps themselves.

---

## Connected topics

- **Study before:**
  - **Topic 35 — Services**: Ingress forwards to Services; you must understand ClusterIP and Endpoints first.
  - **Topic 40 — Kubernetes Networking in Depth**: how pods get IPs and how kube-proxy routes, the layer beneath Ingress.
  - **Topic 36 — ConfigMaps and Secrets**: the TLS Secret that holds your certificate lives here.
- **Study after:**
  - **Topic 42 — Network Policies**: lock down which pods the Ingress controller (and everything else) may reach.
  - **Topic 49 — Helm Basics**: how you actually install ingress-nginx and cert-manager in practice.
  - **Topic 53 — RBAC**: the controller needs permissions to watch Ingress objects cluster-wide.
