# 88 — Containers and Orchestration — Docker and Kubernetes Concepts
## Category: HLD Components

---

## What is this?

A **container** is a normal program running on a normal Linux machine — but wearing a blindfold and a leash. The blindfold (**namespaces**) limits what the process can *see*: its own filesystem, its own process list, its own network interface. The leash (**cgroups**) limits what it can *use*: how much CPU and memory it's allowed to consume. That's it. There is no little computer inside your computer.

**Orchestration** (Kubernetes, ECS, Nomad) is the layer above: when you have 200 of those containers spread across 20 machines, something has to decide *which machine runs what*, restart the ones that die, replace them one-by-one during a deploy, and give them a stable address that other services can call. That "something" is the orchestrator.

Think of containers as **shipping containers** and Kubernetes as the **port**: cranes, schedules, and a manifest that says where every box belongs.

---

## Why does it matter?

**If you don't understand containers**, you'll design systems that quietly assume a machine is a pet — that a server has a name, keeps its files, holds sessions in memory, and lives for months. Then you deploy to a container platform, pods get rescheduled at 3am, and half your users get logged out.

**The design consequence is the whole point of this doc:** containers give you a cheap, identical, disposable unit of compute. That makes horizontal scaling (see [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md)) actually practical instead of theoretical. But the price of "disposable" is that **your application must be stateless**. That single constraint reshapes your architecture: sessions move to Redis, uploads move to blob storage, logs move to stdout.

**In interviews:** almost every HLD answer ends with "…and we run N stateless instances of this service behind a load balancer." The interviewer will poke at that: *How do you deploy without downtime? What happens when an instance dies mid-request? How do you scale it? Where does session state live?* Those are container questions.

**At work:** you'll write Dockerfiles, argue about probes, get paged for `CrashLoopBackOff` and `OOMKilled`, and — most importantly — occasionally be the person who says "we have three services and two engineers; we do not need Kubernetes."

---

## The core idea — explained simply

### The Apartment Building vs the Gated Villa Analogy

A **virtual machine** is a **gated villa**. It has its own foundation, its own plumbing, its own water heater, its own everything. Building one takes months (a VM boots a whole guest operating system — kernel, init, drivers, filesystem — which is *seconds to minutes*). It's extremely private: nothing you do inside villa A can affect villa B, because they share nothing but the land.

A **container** is an **apartment in a big building**. Your apartment has its own front door, its own furniture, its own address on the mailbox — and from inside, it genuinely feels like your own home. But you share the building's foundation, water main, and electrical supply with 200 other apartments. That shared infrastructure is **the host's Linux kernel**.

Because you're not pouring a new foundation, moving in takes *minutes, not months* — that's the millisecond container startup. Because you share, you can fit hundreds of apartments on the land where you'd fit three villas — that's density. And because you share, a burst pipe in the building's main *does* affect everyone — that's the weaker isolation boundary (a kernel exploit escapes the container; it does not escape the VM as easily).

**Mapping the analogy back:**

| Analogy | Technical reality |
|---|---|
| The building's foundation & plumbing | The **host Linux kernel** — shared by all containers |
| Your apartment's walls, door, mailbox | **Namespaces** — PID, mount, network, user, UTS, IPC. What the process can SEE |
| Your metered electricity & water quota | **cgroups** — CPU shares, memory limit. What the process can USE |
| The furniture you moved in with | The **image** — your app plus its entire userland (libc, node binary, node_modules) |
| A gated villa | A **VM** — its own kernel, own OS, own boot sequence |
| The building manager assigning apartments | The **orchestrator** (Kubernetes scheduler) |

The one sentence to remember: **a container is a process, not a machine.** Run `ps aux` on the host and you will literally see your Node process sitting there among the host's other processes. It just can't see them back.

### The comparison table you should be able to draw

| | Virtual Machine | Container |
|---|---|---|
| **What it is** | Emulated hardware + full guest OS | An isolated process on the host kernel |
| **Boot / start time** | 30s – several minutes | 50ms – 2s (it's just `exec`) |
| **Image size** | 1–20 GB (contains an OS) | 20 MB – 500 MB (contains a userland) |
| **Isolation strength** | Strong — hardware-level, separate kernel | Weaker — kernel is shared, kernel bug = escape |
| **Density per host** | ~5–20 per beefy host | ~100–1000 per beefy host |
| **Overhead** | Hypervisor + guest kernel + guest OS RAM | Near zero (~a few MB) |
| **Best for** | Hostile multi-tenancy, different OS kernels | Packaging & scaling your own services |

(Note: "serverless" platforms like AWS Lambda use micro-VMs such as Firecracker to get *both* — VM-grade isolation with ~125ms starts. See [89 — Serverless Architecture](./89-serverless-architecture.md).)

---

## Key concepts inside this topic

### 1. The design problem containers actually solve: "works on my machine"

Before containers, deploying meant: SSH to the server, hope it has Node 18 and not Node 14, hope `libssl` is the right version, run `npm install` and pray the transitive dependency tree resolves the same way it did on your laptop. Your test environment and your production environment were *similar*. They were never *identical*, and the gap is where outages live.

A container **image** bundles your application **and its entire userland** — the exact Node binary, the exact glibc, the exact `node_modules`, the exact system certificates:

```dockerfile
# Multi-stage: build with the full toolchain, ship only what runs.
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev          # lockfile-exact deps — reproducible, not "whatever npm feels like today"
COPY . .

FROM node:20-slim
WORKDIR /app
COPY --from=build /app ./
ENV NODE_ENV=production
USER node                      # don't run as root inside the container
EXPOSE 3000
CMD ["node", "server.js"]
```

The output of `docker build` is an immutable, content-addressed artifact: `myapp@sha256:9f2c…`. The **artifact you tested in CI is byte-for-byte the artifact running in production.** Not "a rebuild of it." *It.*

That unlocks **immutable infrastructure**: you never SSH in and patch a running server. To change anything, you build a new image and replace the container. Servers become disposable, and "configuration drift" (that one box in the fleet that someone hand-tweaked in 2021 and nobody remembers why) stops existing.

### 2. Containers make horizontal scaling real — but only for stateless apps

Recall from [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md) that horizontal scaling means *adding more identical boxes* instead of buying a bigger box. Horizontal scaling needs a unit that is **cheap, identical, and disposable**. Containers are exactly that unit.

But "disposable" has teeth. **A container can be killed at any moment** — the node is drained for a kernel patch, a cheaper spot instance is reclaimed, the autoscaler scales in, a deploy rolls, the process breaches its memory limit. If you didn't design for that, killing it destroys something valuable.

**This constraint shapes your architecture. It is the most important thing in this doc:**

| If you're tempted to store… | …on the container's local disk / memory | Put it here instead | Why |
|---|---|---|---|
| Logged-in user sessions | `express-session` MemoryStore | **Redis** (or a signed JWT) | Pod dies → every user logged out. Also: request 2 hits a different pod that never saw the session |
| User-uploaded files | `/app/uploads` | **S3 / GCS blob storage** | The file vanishes with the pod, and only 1 of your 10 pods ever had it |
| Application logs | `/var/log/app.log` | **stdout / stderr** | The orchestrator collects stdout and ships it to Loki/CloudWatch/ELK. A file inside a dead pod is unreadable |
| Cached computations | in-process `Map` | **Redis** (or accept per-pod cache) | Fine as a *pure* cache if a miss is cheap; never as a source of truth |
| Background job state | a `setInterval` in the web process | **A queue** (SQS, BullMQ) | A pod restart mid-job silently loses work; a queue redelivers it |
| Scheduled cron work | `node-cron` in every pod | **A CronJob / leader-elected job** | 10 pods = your "daily email" sends 10 times |

Here's the bad-vs-good in code:

```js
// BAD — stateful. Works perfectly on your laptop with one process.
// Breaks the moment there are 2 pods, or 1 pod that restarts.
import session from 'express-session';
const uploads = new Map();                     // lives in this pod's RAM only

app.use(session({ secret: 'x', resave: false, saveUninitialized: false }));
// ^ default MemoryStore: session exists in exactly one pod's heap

app.post('/upload', (req, res) => {
  fs.writeFileSync(`/app/uploads/${req.file.name}`, req.file.buf); // this pod's disk
  res.sendStatus(201);
});
```

```js
// GOOD — stateless. The pod holds nothing that can't be recreated.
import session from 'express-session';
import { RedisStore } from 'connect-redis';
import { createClient } from 'redis';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const redis = createClient({ url: process.env.REDIS_URL }); // config from env, not baked in
await redis.connect();
const s3 = new S3Client({});

app.use(session({
  store: new RedisStore({ client: redis }),   // any pod can serve any user's next request
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));

app.post('/upload', async (req, res) => {
  await s3.send(new PutObjectCommand({
    Bucket: process.env.UPLOAD_BUCKET,
    Key: `uploads/${req.file.name}`,
    Body: req.file.buf,
  }));
  res.sendStatus(201);
});

// Logs go to stdout. The platform handles collection, rotation, shipping.
console.log(JSON.stringify({ level: 'info', msg: 'upload complete', reqId: req.id }));
```

The good version can be killed at any instant, and the worst thing that happens is one in-flight request retries.

### 3. Why you need an orchestrator at all

One container on one machine: `docker run`. Done. You do not need Kubernetes.

Now: **40 services × 5 replicas = 200 containers, across 20 machines.** Ask yourself who does the following, at 3am, without you:

1. **Placement / scheduling** — this pod needs 500m CPU and 1Gi RAM; which of the 20 nodes has room? (Bin-packing.)
2. **Self-healing** — the process crashed, or the node caught fire. Restart it. Somewhere else, if needed.
3. **Deployment** — replace 5 running v1 containers with 5 v2 containers, a few at a time, never dropping below capacity, and roll back automatically if the new ones don't come up healthy.
4. **Service discovery** — pod IPs change constantly. What address does the `orders` service use to call `payments`? (See [72 — Service Discovery](./72-service-discovery.md).)
5. **Scaling** — traffic tripled at noon. Add pods. It's 3am and traffic is nothing. Remove them.
6. **Config & secrets** — the same image runs in dev, staging, and prod with different DB URLs and different credentials.

Kubernetes is a system whose entire job is those six things. If you only have three services on two VMs, a bash script and a load balancer do all six adequately — and that is a legitimate, defensible answer in an interview.

### 4. The Kubernetes objects that matter for *design*

You don't need to memorize YAML. You need to know **which problem each object exists to solve**, because in an HLD interview you'll be describing the *shape*, not the syntax.

#### Pod — the disposable unit

The smallest deployable thing: one or more containers that share a network namespace (same IP, they can talk over `localhost`) and can share volumes. In practice it's usually **one app container**, sometimes plus a sidecar (a logging shipper, a service-mesh proxy).

The design-relevant facts:
- **A pod is ephemeral.** It is never repaired. It is deleted and a *new* one is created.
- **A new pod gets a NEW IP address.** Every time.
- **Cattle, not pets.** You do not name it, nurse it, or SSH into it to fix things.

This is precisely why **you can never address a pod by IP**. Any design where service A hardcodes the IP of service B's instance is wrong on Kubernetes, and wrong in general.

#### Deployment / ReplicaSet — declarative desired state and the reconciliation loop

You do not tell Kubernetes *"start a container."* You tell it **"I want 5 healthy pods running image `myapp:v2`"** and go home. That statement is the **desired state**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 5                       # DESIRED state. Not an instruction — a fact you are asserting.
  selector:
    matchLabels: { app: orders-api }
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0             # never dip below 5 healthy pods during a deploy
      maxSurge: 2                   # may briefly run 7
  template:
    metadata:
      labels: { app: orders-api }
    spec:
      containers:
        - name: api
          image: registry.example.com/orders-api:v2   # pin a tag/digest, never :latest
          ports: [{ containerPort: 3000 }]
          envFrom:
            - configMapRef: { name: orders-config }
            - secretRef:    { name: orders-secrets }
          resources:
            requests: { cpu: "250m", memory: "256Mi" }  # scheduler uses this to place the pod
            limits:   { cpu: "1000m", memory: "512Mi" } # kernel enforces this
```

**This is the deep concept — say it out loud in the interview: the reconciliation loop (control loop).** A controller runs forever, doing:

```
loop forever:
    desired = read spec from etcd        # "5 pods of orders-api:v2"
    actual  = observe the cluster        # "4 pods, one of them still v1"
    diff    = desired - actual
    take the smallest action that shrinks diff
```

Nobody "restarts the crashed pod." A pod dies → *actual* becomes 4 → the loop notices the diff → it creates one. Nobody "does a deploy." You change `image: v1` → `image: v2` → *desired* changed → the loop grinds *actual* toward it, a few pods at a time, respecting `maxUnavailable`. Rollback (`kubectl rollout undo`) is the same mechanic in reverse: it just re-declares the old ReplicaSet as desired.

This idea — **converge on declared state, continuously, rather than execute imperative steps** — is *the* architectural pattern of Kubernetes, Terraform, ArgoCD, and most modern infra. It's what makes the system self-healing: the loop doesn't care *why* actual drifted, only that it did. It ties directly to [135 — Zero-Downtime Deployments](./135-zero-downtime-deployments.md).

#### Service — the answer to unstable pod IPs

Pods have unstable IPs. So you never talk to a pod. You talk to a **Service**: a stable virtual IP plus a stable DNS name, which load-balances across all **Ready** pods matching a label selector.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders-api                  # → DNS: orders-api.default.svc.cluster.local
spec:
  type: ClusterIP                   # stable virtual IP, reachable only inside the cluster
  selector: { app: orders-api }     # membership by LABEL, not by IP list
  ports:
    - port: 80                      # what callers dial
      targetPort: 3000              # what the container listens on
```

Your Node code then does exactly what you'd hope:

```js
// No IPs. No registry client. No Consul. Just DNS the cluster resolves for you.
const res = await fetch(`http://orders-api/orders/${id}`);
```

This is service discovery, built in (compare with the general patterns in [72 — Service Discovery](./72-service-discovery.md)). The magic word is **Ready**: a pod that fails its readiness probe is silently removed from the Service's endpoint list and receives no traffic — without being killed.

| Service type | What it gives you | Use when |
|---|---|---|
| **ClusterIP** (default) | Internal-only virtual IP + DNS | Service-to-service calls. 95% of your services |
| **NodePort** | Opens the same high port on every node's IP | Rarely, by hand. Mostly a building block |
| **LoadBalancer** | Provisions a real cloud LB (an AWS NLB/ELB) | Exposing something to the internet — but one LB per service gets expensive fast |

#### Ingress — L7 routing into the cluster

You do not want 30 cloud load balancers (and 30 bills, and 30 TLS certs). You want **one** entry point that does HTTP-aware routing by host and path. That's the Ingress: an L7 reverse proxy (nginx, Traefik, Envoy) inside the cluster, in front of your Services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: public-edge
spec:
  tls:
    - hosts: [api.example.com]
      secretName: example-tls        # TLS terminates HERE; internal traffic is plain HTTP
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend: { service: { name: orders-api, port: { number: 80 } } }
          - path: /users
            pathType: Prefix
            backend: { service: { name: users-api,  port: { number: 80 } } }
```

One public IP, one certificate, path-based fan-out to many services. This is the natural front door for a microservices deployment ([71 — Microservices vs Monolith](./71-microservices-vs-monolith.md)) and functionally an API-gateway-lite.

#### ConfigMap / Secret — one image, every environment

A cardinal rule (from the **12-factor app** methodology): **config lives in the environment, never in the image.** If your DB URL is baked into the image, you need a different image per environment — and now the artifact you tested is *not* the artifact you shipped, which destroys the entire benefit from section 1.

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: orders-config }
data:
  LOG_LEVEL: "info"
  PAYMENTS_URL: "http://payments-api"
  MAX_CART_ITEMS: "50"
---
apiVersion: v1
kind: Secret
metadata: { name: orders-secrets }
type: Opaque
stringData:
  DATABASE_URL: "postgres://app:s3cr3t@db.internal:5432/orders"
  SESSION_SECRET: "…"
```

```js
// The app reads only from process.env — it has no idea it's in Kubernetes.
// The same code runs under `docker run -e`, under Fargate, or on your laptop.
const config = {
  logLevel:     process.env.LOG_LEVEL ?? 'info',
  paymentsUrl:  process.env.PAYMENTS_URL,
  databaseUrl:  requireEnv('DATABASE_URL'),   // fail FAST at boot if missing…
};
function requireEnv(k) {                      // …never discover it's missing at 2am under load
  const v = process.env[k];
  if (!v) throw new Error(`Missing required env var: ${k}`);
  return v;
}
```

Honest caveat: a Kubernetes `Secret` is only **base64-encoded**, not encrypted, in etcd by default. Real secret management means enabling encryption-at-rest, or using AWS Secrets Manager / Vault / External Secrets. Say that in an interview and you'll sound like someone who has actually run this.

#### HorizontalPodAutoscaler — and what it cannot fix

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders-api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders-api }
  minReplicas: 3
  maxReplicas: 40
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 65 }  # % of the *request*, not the limit
```

The HPA is itself a control loop: observe the metric, compare to target, adjust replica count. It can scale on CPU, memory, or **custom metrics** — and custom metrics are usually the better signal: *queue depth*, *requests per second*, *p95 latency*. CPU is a lazy proxy; a Node app blocked on database I/O sits at 8% CPU while timing out.

**Now the part interviewers are actually listening for. The HPA cannot fix a bottleneck that is not in the pods.** If your Postgres is at 100% CPU and connections are saturated, adding pods **makes it worse**: each new pod opens its own connection pool, deepens the DB's queue, and increases contention. You scale the stateless tier *into* the wall harder.

```
Traffic ↑ → latency ↑ → CPU ↑ → HPA adds pods → more DB connections
   → DB slower → latency ↑ → CPU ↑ → HPA adds MORE pods → DB falls over
```

Fixing that requires read replicas, caching, a connection **pooler** like PgBouncer, or query work — not replicas. Same for a downstream third-party API with a rate limit: more pods just get you rate-limited faster. **Autoscaling amplifies whatever your real bottleneck is.**

#### StatefulSet + PersistentVolume — for the things that can't be stateless

Some things genuinely have identity and durable disk: databases, Kafka brokers, Elasticsearch nodes. A **StatefulSet** gives you what a Deployment deliberately denies:

- **Stable, ordinal names**: `kafka-0`, `kafka-1`, `kafka-2` — and `kafka-0` is *always* `kafka-0` after a restart.
- **Stable network identity** via a headless Service (`kafka-0.kafka.default.svc…`) — pets, deliberately.
- **Its own PersistentVolumeClaim**, which follows the pod across reschedules: `kafka-0` reattaches to *its own* disk.
- **Ordered** rollout and scale-down (N-1 before N).

A **PersistentVolume** is a real disk (an EBS volume, say) whose lifecycle is independent of any pod.

**The honest advice, and you should say this out loud:** *running your production database inside Kubernetes is a serious undertaking.* Backups, PITR, failover, version upgrades, storage performance, and the fact that a bad node drain can take down your primary — these are the hard parts, and Kubernetes solves none of them for you. Operators (CloudNativePG, Strimzi) help, but they're software you now have to operate too. **For most teams, the right answer is a managed database — RDS, Aurora, Cloud SQL, Confluent — with the stateless tier in Kubernetes.** An interviewer who has been on-call will nod hard at that.

### 5. Probes and resource limits — the two things everyone gets wrong

**Three probes, three different jobs:**

| Probe | Question it asks | Failure action | Getting it wrong |
|---|---|---|---|
| **liveness** | "Is this process wedged beyond saving?" | **Kills and restarts the container** | Too aggressive → healthy-but-slow pods get killed → cascading restart loop → outage |
| **readiness** | "Can this pod serve traffic *right now*?" | **Removes it from the Service's endpoints.** Does NOT kill it | Missing → traffic hits a pod still warming up → 502s on every deploy |
| **startup** | "Has the app finished booting yet?" | Suppresses liveness/readiness until it passes | Missing on a slow-booting app → liveness kills it mid-boot → CrashLoopBackOff forever |

**This distinction is the single most common production mistake.** Liveness failure is violent — it restarts you. Readiness failure is gentle — it just takes you out of the rotation until you recover.

The classic self-inflicted outage: someone points liveness at `/health`, and `/health` checks the database. The database has a 30-second hiccup. Every pod's liveness probe fails simultaneously. Kubernetes **restarts your entire fleet at once**, all of them slam the recovering database with cold connection pools on boot, and a 30-second blip becomes a 20-minute outage. **Liveness must check only the process itself. Dependency checks belong in readiness.**

```js
// /healthz — LIVENESS. "Is my event loop alive?" That's ALL it may check.
// If this fails, K8s will KILL me. Never make that depend on a downstream service.
app.get('/healthz', (_req, res) => res.status(200).send('ok'));

// /readyz — READINESS. "Should I receive traffic right now?"
// Failing here only removes me from the load balancer. Safe to check dependencies.
let shuttingDown = false;
app.get('/readyz', async (_req, res) => {
  if (shuttingDown) return res.status(503).send('draining');   // deploy: stop new traffic first
  try {
    await db.query('SELECT 1');
    return res.status(200).send('ready');
  } catch {
    return res.status(503).send('db unavailable');             // pulled out, NOT killed
  }
});

// Graceful shutdown — the other half of zero-downtime deploys.
// On SIGTERM: fail readiness, let the LB stop routing to us, THEN drain in-flight requests.
process.on('SIGTERM', async () => {
  shuttingDown = true;
  await new Promise((r) => setTimeout(r, 5000)); // let endpoint removal propagate
  server.close(async () => { await db.end(); process.exit(0); });
});
```

```yaml
          startupProbe:                     # slow boot? buy up to 60s before liveness engages
            httpGet: { path: /healthz, port: 3000 }
            failureThreshold: 30
            periodSeconds: 2
          livenessProbe:
            httpGet: { path: /healthz, port: 3000 }   # process-only check
            periodSeconds: 10
            failureThreshold: 3               # generous — killing is destructive
          readinessProbe:
            httpGet: { path: /readyz, port: 3000 }    # may check dependencies
            periodSeconds: 5
            failureThreshold: 2               # strict — removal is cheap and reversible
```

**Requests vs limits** — also routinely confused:

| | `requests` | `limits` |
|---|---|---|
| **Who uses it** | The **scheduler**, to decide which node has room | The **kernel** (cgroups), to enforce a ceiling at runtime |
| **CPU over-budget** | — | Process is **throttled** — it just runs slower. Not killed |
| **Memory over-budget** | — | Process is **OOMKilled** — hard kill, no warning, no cleanup |

CPU is *compressible* (you can give a process less and it merely slows down). Memory is *incompressible* (you cannot give a process less RAM than it needs; you can only kill it). This asymmetry has a practical rule: **always set a memory limit; be careful with a hard CPU limit.** A CPU limit set too low silently throttles you into latency hell (p99 spikes with 20% CPU usage — a classic, maddening bug), while an unset memory limit lets one leaky pod evict its neighbors off the node.

---

## Visual / Diagram description

### The full request path through a Kubernetes cluster

```
                          Internet
                              │
                              ▼
                   ┌──────────────────────┐
                   │  Cloud Load Balancer │   one public IP, one TLS cert
                   └──────────┬───────────┘
                              ▼
                   ┌──────────────────────┐
                   │  INGRESS (L7 proxy)  │   TLS terminates here
                   │  api.example.com     │   /orders → orders-api
                   │                      │   /users  → users-api
                   └───┬──────────────┬───┘
                       ▼              ▼
          ┌───────────────────┐  ┌──────────────────┐
          │ SERVICE           │  │ SERVICE          │   stable virtual IP + DNS
          │ orders-api :80    │  │ users-api :80    │   LB across READY pods only
          └─────┬───────┬─────┘  └────────┬─────────┘
                │       │                 │
     ┌──────────┘       └──────┐          └──────────┐
     ▼                         ▼                     ▼
┌──────────────┐    ┌──────────────┐        ┌──────────────┐
│ NODE 1       │    │ NODE 2       │        │ NODE 3       │
│ ┌──────────┐ │    │ ┌──────────┐ │        │ ┌──────────┐ │
│ │Pod orders│ │    │ │Pod orders│ │        │ │Pod users │ │
│ │10.1.0.7  │ │    │ │10.1.1.3  │ │        │ │10.1.2.9  │ │  ← IPs change on
│ └──────────┘ │    │ └──────────┘ │        │ └──────────┘ │    every restart
│ ┌──────────┐ │    │ ┌──────────┐ │        │ ┌──────────┐ │
│ │Pod users │ │    │ │Pod orders│ │        │ │Pod users │ │
│ └──────────┘ │    │ └──────────┘ │        │ └──────────┘ │
│   kubelet    │    │   kubelet    │        │   kubelet    │
└──────▲───────┘    └──────▲───────┘        └──────▲───────┘
       │                   │                       │
       │  "run these pods" │   +  "here's my state"│
       └───────────┬───────┴───────────────────────┘
                   ▼
        ┌──────────────────────────────────┐
        │        CONTROL PLANE             │
        │  ┌────────────────────────────┐  │
        │  │ API Server  ◄──► etcd      │  │  etcd = the DESIRED state
        │  ├────────────────────────────┤  │
        │  │ Scheduler                  │  │  which node fits this pod?
        │  ├────────────────────────────┤  │
        │  │ Controller Manager         │  │  the RECONCILIATION LOOPS
        │  │   Deployment ctrl          │  │
        │  │   ReplicaSet ctrl  ────────┼──┼─► desired=5, actual=4 → create 1
        │  │   HPA ctrl                 │  │
        │  └────────────────────────────┘  │
        └──────────────────────────────────┘

  ↕ Everything runs FOREVER:  observe actual → compare to desired → act.
    Nothing is ever "done." That is the whole design.
```

**What the diagram shows.** Traffic enters at one public load balancer, hits the Ingress, which reads the HTTP `Host` and path and routes to the right **Service**. The Service is a stable name; behind it the actual pods are shuffling constantly — different IPs, different nodes, appearing and disappearing. Below, the **control plane** never stops: etcd stores what you *declared*, the kubelets report what *is*, and the controllers close the gap. Draw this on a whiteboard and you have covered 80% of what an interviewer wants on the deployment layer.

### The reconciliation loop during a rolling deploy (maxUnavailable: 0, maxSurge: 2)

```
t0   desired: 5 × v2     actual: [v1 v1 v1 v1 v1]              healthy: 5
t1   controller surges:  [v1 v1 v1 v1 v1 v2 v2]                v2s NOT Ready yet → no traffic
t2   v2 pods pass readiness → Service adds them                healthy: 7
t3   controller kills 2 v1 (SIGTERM → drain → exit)            [v1 v1 v1 v2 v2]
t4   …repeat…                                                  [v1 v2 v2 v2 v2]
t5   converged:          actual: [v2 v2 v2 v2 v2]              healthy: 5   ✓

  Capacity NEVER drops below 5.  A v2 that fails readiness is never given traffic,
  and the rollout STALLS instead of continuing — a broken deploy stops itself.
```

---

## Real world examples

### Google — where all of this came from

Google has run essentially everything in containers for well over a decade, on an internal cluster manager called **Borg**; the famous claim from their 2015 Borg paper is that they start *billions* of containers per week. Linux **cgroups** themselves were originally contributed to the kernel by Google engineers — the isolation primitives exist because Google needed them to bin-pack many workloads onto shared machines at high utilization. Kubernetes was created by Googlers as an open-source system informed by that Borg experience, which is why its core is a set of reconciliation controllers rather than a deployment script runner.

### Netflix — containers without adopting Kubernetes

Netflix runs a large containerized workload, but for years the platform underneath it was **Titus**, their own container management system, built to integrate tightly with the AWS primitives and the Netflix ecosystem they already had (Spinnaker for delivery, Eureka for discovery, their own IAM/VPC integration). The lesson is a good one for interviews: *containers* are the durable idea; *Kubernetes* is one implementation of the orchestration layer, and a large org with strong platform engineering can rationally choose something else.

### A typical high-growth SaaS company (representative architecture)

The pattern you'll meet most often in the wild, and the one you should propose by default:

- **Stateless tier in Kubernetes** (EKS/GKE): API services, background workers, all running from immutable images, autoscaled by HPA on queue depth and RPS.
- **State outside Kubernetes, fully managed**: Postgres on RDS/Aurora, Redis on ElastiCache, Kafka on MSK/Confluent, files in S3.
- **Ingress + cert-manager** for one TLS front door; **ArgoCD** for GitOps (which is, again, a reconciliation loop — this time between a Git repo and the cluster).

This split — *stateless in the orchestrator, stateful managed by the cloud* — is the single most common and most defensible architecture in 2020s infrastructure.

---

## Trade-offs

### Containers vs VMs vs bare metal

| | Pros | Cons |
|---|---|---|
| **Containers** | Fast start; tiny; dense; identical artifact dev→prod; cheap unit for horizontal scaling | Shared kernel = weaker isolation; forces statelessness (a cost if you fight it); a new layer to learn and debug |
| **VMs** | Strong isolation; run any OS; mature tooling | Slow boot; heavy; low density; images drift |
| **Bare metal** | Max performance; no virtualization tax | Zero elasticity; you are back to pets |

### Kubernetes vs a managed container platform

| | Kubernetes (EKS/GKE/self-hosted) | Managed platform (ECS/Fargate, Cloud Run, Render, Fly.io) |
|---|---|---|
| **Power** | Enormous. Any topology, any workload, any extension (CRDs, operators) | Constrained to the paved path |
| **Portability** | Real. The same manifests run on any cloud | You are married to the vendor's model |
| **Complexity** | Very high. Networking (CNI), RBAC, admission control, upgrades, storage classes, DNS, service mesh… | Low. Push an image, set env vars, done |
| **Who operates it** | You need a **platform team** (realistically 2+ engineers, permanently) | The cloud provider |
| **Cost at small scale** | Control plane fees + always-on nodes + the salaries of the people debugging it | Often scale-to-zero, pay per request |
| **Time to first deploy** | Days to weeks | An afternoon |

**The honest take, and a genuinely strong interview answer:** Kubernetes is the right call when you have **many services, many teams, real scale, and a platform team to own it** — its complexity buys you a uniform, portable, self-healing substrate that lets 200 engineers ship independently. It is very often a **costly distraction for a small team running three services**: you inherit a permanent operational tax (upgrades, CVEs, CNI mysteries, YAML sprawl) in exchange for flexibility you will not use. That team should ship containers to ECS/Fargate, Cloud Run, or Render, and spend the saved months on the product.

**Rule of thumb:** *Containerize immediately — it's nearly free and it forces the statelessness that makes everything else possible. Orchestrate with the simplest thing that works, and adopt Kubernetes only when the pain of not having it is concrete and nameable.* Knowing when **not** to reach for Kubernetes is a senior signal; reaching for it reflexively is a junior one.

---

## Common interview questions on this topic

### Q1: "Is a container just a lightweight VM?"

**Hint:** No — and the difference is the whole point. A VM virtualizes *hardware* and boots its own guest kernel; a container is an ordinary process on the **host's** kernel, isolated by **namespaces** (what it can see: PID, mount, network, user) and constrained by **cgroups** (what it can use: CPU, memory). That's why a container starts in milliseconds — there's nothing to boot, it's just `exec` — and why you can pack hundreds on one host. The trade-off is a weaker isolation boundary: a kernel vulnerability is a container escape, which is exactly why hostile multi-tenant platforms (AWS Lambda) use micro-VMs like Firecracker to get both.

### Q2: "Your service is slow. The HPA scales from 5 pods to 30 and it gets *worse*. What happened?"

**Hint:** The bottleneck isn't in the pods. Almost certainly the database (or a rate-limited downstream). Each new pod opens its own connection pool, so 30 pods × 20 connections = 600 connections into a Postgres that comfortably handles ~200 — deeper queues, more lock contention, more context switching, slower queries, higher latency, *more* CPU in the pods, so the HPA adds *more* pods. A textbook feedback loop. The fixes are all outside the HPA: a connection pooler (PgBouncer), read replicas, caching, query optimization — plus a sane `maxReplicas` so autoscaling can't burn your database down. **Autoscaling amplifies your real bottleneck; it doesn't find it.**

### Q3: "What's the difference between a liveness and a readiness probe, and what happens if you get them backwards?"

**Hint:** Liveness failure **kills and restarts** the container; readiness failure only **removes it from the Service's endpoints** (no traffic, but it stays alive to recover). Backwards or careless usage causes real outages: point liveness at a `/health` endpoint that checks the DB, let the DB hiccup for 30 seconds, and Kubernetes restarts *every pod simultaneously* — turning a blip into an outage as your entire fleet cold-starts against a recovering database. Rule: **liveness checks only the process; dependency checks go in readiness.** Add a startup probe for slow-booting apps so liveness doesn't kill them mid-boot.

### Q4: "Why can't I just store user sessions in memory in my pods?"

**Hint:** Because a pod is disposable and there's more than one of them. Request 1 hits pod A and creates a session in pod A's heap; request 2 hits pod B, which has never heard of it. And a deploy, node drain, spot reclaim, or OOM kill wipes the session entirely. Session affinity ("sticky sessions") is a band-aid that just relocates the failure — the pod still dies eventually, and it wrecks your load distribution. Externalize it: **Redis** (or a signed, stateless JWT). Same logic for uploads (→ S3), logs (→ stdout), and cron (→ a CronJob, not `setInterval` in every replica).

### Q5: "We're 4 engineers with 3 services. Should we use Kubernetes?"

**Hint:** Almost certainly not — and saying so confidently is a *strong* signal, not a weak one. Kubernetes' complexity (CNI, RBAC, upgrades, storage classes, YAML sprawl, a permanent on-call surface) pays for itself when many teams need a uniform, portable, self-healing substrate. With 4 engineers it's a second full-time job you didn't hire for. **Containerize anyway** — that part is cheap and forces good stateless design — then run those same images on ECS/Fargate, Cloud Run, or Render. Migrating to Kubernetes later is easy *because* you containerized; that's the option value you're buying. Adopt it when the pain is concrete and nameable, not preemptively.

---

## Practice exercise

### "Make the Pet Stateless" — refactor and deploy design (~35 min)

You've inherited a Node/Express service, `photo-api`, currently running as a single long-lived process on one EC2 box. It:

1. Uses `express-session` with the default in-memory store.
2. Saves uploaded photos to `/var/app/uploads/` and serves them from there.
3. Writes logs to `/var/app/logs/app.log` with rotation via a cron entry.
4. Runs a `node-cron` job at 02:00 that emails a daily digest to every user.
5. Caches the user's profile in a module-level `Map` and treats it as the source of truth after the first read (writes update the Map, and a background flusher persists it to Postgres every 60 seconds).
6. Reads its DB password from a `config.prod.json` file committed to the repo.

**Produce these four things:**

**(a) A stateless-migration table.** One row per problem above: *what breaks when this runs as 6 pods that can be killed at any time*, and *where the state should actually live*. Be specific about the failure — "the daily digest email is sent 6 times" is a good answer; "it breaks" is not.

**(b) The Dockerfile** — multi-stage, non-root user, `npm ci`, production-only deps, no secrets baked in.

**(c) The Kubernetes objects you'd need**, with a one-line justification each: Deployment (how many replicas, what requests/limits, and *why those numbers*), Service (which type, and why), Ingress (what host/path), ConfigMap vs Secret (what goes in each — be careful about #6), HPA (which metric, and why *not* CPU if the service is I/O-bound on S3 uploads).

**(d) Write the two health endpoints in JavaScript** — `/healthz` (liveness) and `/readyz` (readiness) — and, in two sentences, explain what would go wrong if you swapped their contents. Then add a `SIGTERM` handler that drains correctly.

**Stretch:** the digest job must run *exactly once* across 6 pods. Give two ways to guarantee that, and name the trade-off of each. (Hint: one involves a Kubernetes object; the other involves Redis and the words "distributed lock.")

---

## Quick reference cheat sheet

- **A container is a process, not a machine.** Namespaces = what it can SEE. cgroups = what it can USE. Shared host kernel = millisecond starts, weaker isolation.
- **The image bundles the app AND its userland**, so the artifact you tested *is* the artifact you deploy. That's what makes **immutable infrastructure** possible — never patch a running server, replace it.
- **Containers make horizontal scaling real** by giving you a cheap, identical, disposable unit — *provided the app is stateless*.
- **The stateless constraint shapes your architecture:** sessions → Redis/JWT, files → S3, logs → stdout, cron → CronJob, jobs → a queue. Nothing valuable on local disk or in local memory.
- **Pods are cattle, not pets** — ephemeral, and they get a **new IP every time**. Never address a pod by IP.
- **The reconciliation loop is THE idea**: declare desired state; a controller observes actual and continuously converges. Self-healing and rolling deploys are the same mechanism, not two features.
- **Service** = stable virtual IP + DNS name, load-balancing across **Ready** pods. ClusterIP (internal, 95% of cases) / NodePort / LoadBalancer (one real cloud LB — expensive per service).
- **Ingress** = one L7 front door: host/path routing + TLS termination, so you don't buy 30 load balancers.
- **ConfigMap/Secret** externalize config so **one image runs in every environment** (12-factor). K8s Secrets are only base64 by default — use real secret management.
- **HPA cannot fix a bottleneck outside the pods.** If the DB is the constraint, more pods make it **worse**. Prefer custom metrics (queue depth, RPS) over CPU for I/O-bound services.
- **Liveness failure RESTARTS. Readiness failure only pulls out of the load balancer.** Liveness must check the process only — never a dependency, or one DB hiccup restarts your whole fleet.
- **Requests** are for the scheduler (placement); **limits** are enforced by the kernel. Over CPU limit → **throttled**. Over memory limit → **OOMKilled**.
- **StatefulSet + PersistentVolume** for things with identity and disk — but a **managed database is usually the right answer**; running production Postgres in K8s is a serious undertaking.
- **Know when NOT to use Kubernetes.** Three services and four engineers? Containerize, then ship to ECS/Fargate, Cloud Run, or Render. Adopt K8s when the pain is concrete and you have a platform team.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [89 — Serverless Architecture](./89-serverless-architecture.md) — the next rung up the abstraction ladder: don't manage the container either, just the function (and note that Lambda uses micro-VMs to get VM isolation with container-like startup) |
| **Next** | [135 — Zero-Downtime Deployments](./135-zero-downtime-deployments.md) — rolling updates, readiness gates, graceful SIGTERM draining, blue-green and canary; the reconciliation loop is what makes all of it work |
| **Related** | [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md) — containers are what make "just add another identical instance" a cheap, real operation instead of a slide-deck aspiration |
| **Related** | [72 — Service Discovery](./72-service-discovery.md) — pods have unstable IPs, so a Kubernetes Service *is* service discovery: a stable DNS name resolving to the currently-Ready endpoints |
| **Related** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) — orchestration is the substrate that makes running 40 independently deployed services operationally survivable |
