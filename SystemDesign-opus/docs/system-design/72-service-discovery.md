# 72 — Service Discovery — How Services Find Each Other
## Category: HLD Components

---

## What is this?

**Service discovery** is the mechanism that answers one question, continuously and correctly: *"I want to talk to the `orders` service — what IP address and port do I send this request to, right now, that will actually be answered by a healthy instance?"*

Think of it as the **phone book of your architecture** — except the phone numbers change every few minutes. People move house constantly (deploys), extra people show up during dinner rush (autoscaling), and some people quietly die without telling anyone (crashes). A paper phone book would be wrong within an hour. Service discovery is a *living* phone book that keeps itself up to date.

---

## Why does it matter?

In a monolith, calling another module looks like this:

```javascript
// Monolith. The "address" of orderService is a memory pointer.
// It cannot move. It cannot be unhealthy. It cannot be on another machine.
const order = await orderService.create({ userId, items });
```

There is no network, no address, no failure mode beyond a thrown exception. Splitting that monolith into services (recall from [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md)) replaces that pointer with a network call — and now you have a brand-new problem you never had before: **where is it?**

Your first instinct is to write this:

```javascript
// DO NOT DO THIS.
const order = await fetch('http://10.0.1.42:3000/orders', { method: 'POST', ... });
```

That IP is a lie waiting to happen. In a cloud environment:

- **Every deploy replaces instances.** New containers, new IPs. `10.0.1.42` is now a payments pod.
- **Autoscaling adds and removes instances constantly.** At 9am you have 3 orders instances, at noon you have 40, at 3am you have 2. Their addresses were never in your config file.
- **Containers get ephemeral ports.** Orchestrators often map a container to a random host port so multiple instances can share a machine. Port `3000` might actually be `32781` today.
- **Hardware fails.** A node dies, its pods vanish, replacements appear elsewhere.

**Interview angle:** "How does Service A find Service B?" is a follow-up in *almost every* microservices design round. The strong answer names client-side vs server-side discovery, the registry, health checks, and the staleness window. The weak answer is "uh, we put the URL in an env variable."

**Real-work angle:** If you deploy to Kubernetes, you use service discovery every single day whether you understand it or not. Understanding it is the difference between debugging a 5-minute outage and staring blankly at `ECONNREFUSED`.

---

## The core idea — explained simply

### The Hotel Concierge Analogy

You are staying in a huge hotel. You want a plumber.

**Option A — the hardcoded way.** Someone once told you "the plumber is in room 412." So you walk to room 412. But the plumber checked out yesterday and room 412 now holds a honeymooning couple. You have just paged a stranger at 2am. This is `http://10.0.1.42:3000`.

**Option B — the registry.** The hotel keeps a **front-desk ledger**. Every tradesperson who checks in writes their name and room number in it: "Plumber — room 209." "Plumber — room 517." "Electrician — room 118." When they check out, they cross their line out.

Now you have two ways to use that ledger:

1. **Client-side discovery:** You walk to the front desk yourself, read the ledger, see *two* plumbers, and pick one (maybe the one on your floor). You then walk to their room directly. You did the lookup and the choosing.
2. **Server-side discovery:** You just call the front desk and say "send me a plumber." The concierge reads the ledger, picks one, and forwards your request. You never saw the ledger. You only ever needed one phone number: the front desk.

**And the hard problem:** what if a plumber has a heart attack in his room? He never crossed his name out. The ledger still says "Plumber — room 209." The hotel's answer is a **heartbeat**: every tradesperson must phone the front desk every 30 seconds to say "still alive." If the desk hasn't heard from room 209 in 90 seconds, it strikes the line out. Between the heart attack and the strike-out, **the ledger is lying to you** — and you will knock on a dead man's door. That window is unavoidable, and it is why your code needs retries.

| Analogy piece | Technical concept |
|---|---|
| The hotel | Your cluster / VPC |
| A tradesperson | A service instance (pod, container, VM) |
| Room number | IP address + port |
| The front-desk ledger | The **service registry** |
| Writing your name on check-in | **Registration** |
| Crossing it out on check-out | **Deregistration** (graceful shutdown) |
| Phoning the desk every 30s | **Heartbeat** |
| Striking out a silent line after 90s | **TTL expiry / health check eviction** |
| You reading the ledger yourself | **Client-side discovery** |
| The concierge forwarding for you | **Server-side discovery** |
| Knocking on a dead man's door | The **staleness window** — why you need retries + circuit breakers |
| "Send me a plumber" — one number, forever | A **stable virtual IP / DNS name** |

---

## Key concepts inside this topic

### 1. The service registry — the live database of instances

A **service registry** is a key-value store whose keys are service names and whose values are lists of healthy instances:

```
orders   → [ 10.0.1.42:32781, 10.0.3.19:32104, 10.0.2.77:31900 ]
payments → [ 10.0.4.11:30022 ]
search   → [ 10.0.1.90:31555, 10.0.5.61:31555 ]
```

It must be **highly available** (if the registry is down, nothing can find anything — it is a spectacular single point of failure) and it must be **fresh** (a list that is 10 minutes old is worse than useless).

Two ways instances get *into* it:

**a) Self-registration.** The instance registers itself on boot and heartbeats until it dies.

```javascript
// self-register.js — the instance knows about the registry and talks to it directly.
import http from 'node:http';
import { setInterval } from 'node:timers';

const REGISTRY = 'http://registry.internal:8500';
const SERVICE = 'orders';
const INSTANCE_ID = `orders-${process.env.HOSTNAME}-${process.pid}`;
const PORT = Number(process.env.PORT);
const ADDRESS = process.env.POD_IP; // injected by the platform

async function register() {
  await fetch(`${REGISTRY}/v1/register`, {
    method: 'PUT',
    body: JSON.stringify({
      id: INSTANCE_ID,
      name: SERVICE,
      address: ADDRESS,
      port: PORT,
      // TTL is the contract: "evict me if you don't hear from me within 30s"
      check: { ttl: '30s', deregisterCriticalAfter: '90s' },
    }),
  });
}

// The heartbeat is what proves we are still alive. It must be sent from the
// SAME event loop that serves traffic — if the loop is blocked, heartbeats stop,
// and stopping is exactly the right signal.
function startHeartbeat() {
  return setInterval(() => {
    fetch(`${REGISTRY}/v1/check/pass/${INSTANCE_ID}`, { method: 'PUT' })
      .catch((err) => console.error('heartbeat failed', err.message));
  }, 10_000); // heartbeat at 1/3 of the TTL — survives two lost packets
}

async function deregister() {
  await fetch(`${REGISTRY}/v1/deregister/${INSTANCE_ID}`, { method: 'PUT' });
}

const server = http.createServer(/* ... your app ... */);
server.listen(PORT, register);
const hb = startHeartbeat();

// Graceful shutdown: leave the registry FIRST, then drain, then exit.
// If you exit first, the registry keeps sending you traffic for up to 90 seconds.
for (const sig of ['SIGTERM', 'SIGINT']) {
  process.on(sig, async () => {
    clearInterval(hb);
    await deregister();
    await new Promise((r) => setTimeout(r, 5_000)); // let in-flight LBs notice
    server.close(() => process.exit(0));
  });
}
```

**b) Third-party registration.** The instance knows *nothing* about the registry. A separate agent — a sidecar container, a daemon on the host, or the orchestrator itself — watches what is running and registers it. This is what Kubernetes does: your Node.js app has no registry code at all. It is cleaner (no discovery library in your app) but it means the registrar is one more moving part.

| | Self-registration | Third-party registration |
|---|---|---|
| Who calls the registry | The app itself | An agent / the platform |
| App code coupling | High — you import a client library | Zero |
| Language support | Need a library per language | Language-agnostic |
| Knows app-level health | Yes (it *is* the app) | Only what it can probe |
| Example | Netflix Eureka client | Kubernetes, Consul + Registrator |

### 2. Deregistration, heartbeats, and the corpse problem

Graceful shutdown is easy: catch `SIGTERM`, call deregister, drain, exit. That is the happy path.

**The hard case is the crash.** A process gets OOM-killed. A node loses power. A network cable is unplugged. In none of those cases does anything call `deregister()`. The registry still lists a dead instance, and it will happily hand that address to callers.

The only defence is **heartbeats + TTL expiry**: the registry evicts any instance it hasn't heard from within the TTL. Now do the arithmetic on the window during which the registry is lying:

```
Heartbeat interval:      10s   (instance says "I'm alive" every 10s)
TTL:                     30s   (registry evicts after 30s of silence)

Worst case: the instance dies 1ms after sending a heartbeat.
  → Registry believes it is healthy for the remaining ~10s of that interval
  → PLUS the 30s TTL must elapse
  → Worst-case staleness ≈ 40 seconds

Plus: clients often CACHE the registry list for 30s to avoid hammering it.
  → Real-world worst case ≈ 40s + 30s = ~70 seconds of routing to a corpse.
```

**Seventy seconds is an eternity at 5,000 requests/second.** You cannot fix this by tuning the TTL down to 2 seconds — you'd get false evictions on every GC pause and every brief network blip, and healthy instances would flap in and out of the registry.

So accept it: **the registry will sometimes be wrong.** The system tolerates that in the *client*, not in the registry:

- **Retries** — if a connection is refused, immediately try the next instance in the list. `ECONNREFUSED` on a TCP connect is the fastest, cheapest possible failure signal; use it.
- **Circuit breakers** — after N failures against an endpoint, stop calling it entirely for a cooldown period. See [73 — Circuit Breaker](./73-circuit-breaker.md).
- **Passive health checking** — the caller notes which addresses just failed and quarantines them locally, *without* waiting for the registry to catch up.

```javascript
// A client that assumes the registry is stale, because it is.
async function callWithFailover(instances, path, { attempts = 3 } = {}) {
  const candidates = shuffle([...instances]); // spread load, avoid thundering herd
  let lastErr;

  for (let i = 0; i < Math.min(attempts, candidates.length); i++) {
    const { address, port } = candidates[i];
    try {
      const res = await fetch(`http://${address}:${port}${path}`, {
        signal: AbortSignal.timeout(1_000), // never wait forever on a dead host
      });
      if (res.status >= 500) throw new Error(`upstream ${res.status}`);
      return res;
    } catch (err) {
      // Connection refused / timeout => the registry handed us a corpse.
      // Do NOT surface this to the user. Try the next one.
      lastErr = err;
      localQuarantine.add(`${address}:${port}`, 30_000);
    }
  }
  throw new Error(`all instances failed: ${lastErr.message}`);
}
```

### 3. Client-side discovery

The client fetches the **full list** of instances from the registry and does the load balancing **itself**. Classic example: **Netflix Eureka** (the registry) + **Ribbon** (the client-side load balancer library).

```javascript
// A client-side discovery load balancer, in about 40 lines.
class DiscoveryClient {
  constructor(registryUrl, { refreshMs = 30_000 } = {}) {
    this.registryUrl = registryUrl;
    this.cache = new Map(); // serviceName -> instances[]
    this.cursor = new Map(); // serviceName -> round-robin index
    // Poll the registry in the background. Requests are served from CACHE,
    // so a slow/down registry does not add latency to the hot path.
    setInterval(() => this.refreshAll(), refreshMs).unref();
  }

  async refreshAll() {
    for (const name of this.cache.keys()) {
      try {
        const res = await fetch(`${this.registryUrl}/v1/health/service/${name}?passing=true`);
        this.cache.set(name, await res.json());
      } catch {
        // Registry unreachable: KEEP the stale list. A stale list is far better
        // than an empty one — this is the AP choice, and it is the right one here.
      }
    }
  }

  pick(serviceName) {
    const instances = this.cache.get(serviceName) ?? [];
    const healthy = instances.filter((i) => !localQuarantine.has(`${i.address}:${i.port}`));
    if (healthy.length === 0) throw new Error(`no healthy instances of ${serviceName}`);

    // The whole POINT of client-side: we can be smart here.
    // Prefer an instance in our own availability zone — saves ~1-2ms and cross-AZ egress cost.
    const local = healthy.filter((i) => i.zone === process.env.AZ);
    const pool = local.length > 0 ? local : healthy;

    const n = (this.cursor.get(serviceName) ?? 0) % pool.length;
    this.cursor.set(serviceName, n + 1);
    return pool[n];
  }
}
```

**Pros:** one less network hop (you call the target directly). The client can be *smart* — zone-aware routing, least-connections, consistent hashing to keep a user pinned to a cache-warm instance.

**Cons:** you must reimplement this logic **in every language your company uses**. A Node client, a Java client, a Go client, a Python client — four libraries, four sets of bugs, four upgrade cycles. And your application is now coupled to the registry's API.

### 4. Server-side discovery

The client calls **one stable address** and something in the middle does the lookup and forwarding. That "something" is a load balancer, a virtual IP, or a DNS name backed by one.

```javascript
// The entire client-side implementation of server-side discovery:
const res = await fetch('http://orders/api/v1/orders', { method: 'POST', body });
// That's it. `orders` resolves to a stable virtual IP that never changes.
// No registry client, no library, no cache, no load balancing code.
```

The router (Kubernetes Service + kube-proxy, an AWS ALB, an NGINX/Envoy edge proxy) is the one that talks to the registry and knows the real pod IPs.

**Pros:** clients are dumb and simple — nothing to reimplement per language. Load balancing lives in one place, run by one team.

**Cons:** an extra network hop on every call. And the LB is another highly-available thing you must run, monitor, and scale (though on Kubernetes and AWS this is managed for you, which is precisely why it won).

### 5. DNS-based discovery — tempting, and mostly a trap

"Why not just use DNS? Return multiple A records for `orders.internal` and let the client pick one." It sounds perfect. It is the oldest service discovery system on earth. Here is why it burns people:

- **Clients cache DNS aggressively and ignore your TTL.** The JVM historically cached DNS lookups **forever** by default (`networkaddress.cache.ttl = -1` under a security manager). Node.js does not cache in-process by default, but the OS resolver, `systemd-resolved`, and every sidecar in between might. You set a 5-second TTL, you scale down, and some client keeps hammering a dead IP for an hour.
- **DNS carries no health information.** An A record says "this IP exists." It does not say "this instance has a working database connection" or "this instance is at 100% CPU." The registry knows; DNS discards it.
- **DNS carries no load information.** Every instance looks identical. You get round-robin at best, and only if the resolver bothers to rotate.
- **No port information in A records.** SRV records carry ports, but support for SRV in HTTP clients is patchy at best.
- **Failover is slow.** Removing an instance means waiting out every cache in the chain.

DNS is fine as the *stable name* pointing at a load balancer (server-side discovery). It is a bad *instance list*.

### 6. How Kubernetes actually does it

This is the industry default, so know it cold. Kubernetes is a **server-side discovery** system with a **third-party registrar** (the platform itself), and you write zero discovery code.

**a) The `Service` object** gives you two stable things: a **ClusterIP** (a virtual IP that never changes for the life of the Service) and a **DNS name**:

```
orders.default.svc.cluster.local
  │       │      │        └── cluster domain
  │       │      └── "this is a service"
  │       └── namespace
  └── service name

Inside the same namespace, `http://orders` is enough — the resolver appends the rest.
```

**b) The `EndpointSlice` object** is the actual registry. The Kubernetes control plane watches every pod matching the Service's label selector, and lists the IPs of the ones that are **Ready** (readiness probe passing — see below):

```
Service: orders (ClusterIP 10.96.14.7)
  selector: app=orders
       │
       ▼
EndpointSlice: orders-x7k2m
  10.244.1.13:3000  ready=true
  10.244.2.41:3000  ready=true
  10.244.3.88:3000  ready=false   ← readiness probe failing, gets NO traffic
```

**c) kube-proxy** watches EndpointSlices and programs the Linux kernel — `iptables` rules (or IPVS, which scales better past a few thousand services) — so that any packet sent to `10.96.14.7:80` is DNAT'd to one of the ready pod IPs. **There is no proxy process in the data path.** The kernel rewrites the destination address. That is why the "extra hop" of server-side discovery is nearly free here.

**d) Headless Services.** Set `clusterIP: None` and Kubernetes gives you **no** virtual IP — instead, DNS returns the **raw pod IPs** as multiple A records. You use this when you *need* to address individual pods: databases with a leader (Postgres, Kafka, Cassandra), or gRPC clients that want to hold a persistent connection to every backend and load balance across them at the *request* level.

```javascript
// Headless service: resolve the raw pod IPs yourself, then do client-side balancing.
import { promises as dns } from 'node:dns';

const pods = await dns.resolve4('orders-headless.default.svc.cluster.local');
// → ['10.244.1.13', '10.244.2.41']  — the actual pods, no virtual IP
```

### 7. Dedicated registries: Consul, etcd, ZooKeeper — and the CAP choice

Recall from [09 — CAP Theorem] that when a network partition happens, a distributed system must choose: stay **Consistent** (refuse to answer if you might be wrong) or stay **Available** (answer anyway, possibly with stale data). You cannot have both during a partition.

**Consul, etcd, and ZooKeeper are CP.** They use consensus protocols (Raft for Consul and etcd; ZAB for ZooKeeper). If the cluster loses quorum — say 2 of 3 nodes are unreachable — the survivors **stop accepting writes**, and depending on configuration may refuse reads too. Your registry has gone dark rather than risk telling you something false. (Kubernetes stores all state in etcd, so this is your registry too, whether you knew it or not.)

**Netflix's Eureka is deliberately AP.** Its designers argued: *for a service registry specifically, a possibly-stale list is vastly more useful than no list at all.* Eureka servers replicate best-effort, don't do consensus, and under partition enter "self-preservation mode" — they **stop evicting instances** on the theory that a mass heartbeat failure is more likely a network problem than 200 simultaneous server deaths. Better to keep serving a list that includes some dead instances (your retries will handle those) than to empty the registry and take down the whole fleet.

**This is a fantastic interview talking point.** The argument goes: *"For most stateful systems I want CP — I don't want two nodes both thinking they're the leader. But for service discovery, I lean AP. Stale routing data degrades gracefully because clients already retry and circuit-break. An unavailable registry, by contrast, is a total outage — nothing can find anything. The cost of being slightly wrong is much lower than the cost of being silent."*

| Registry | Consistency | Consensus | Typical use |
|---|---|---|---|
| **etcd** | CP | Raft | Kubernetes' backing store |
| **Consul** | CP | Raft | Multi-datacenter discovery + KV + mesh |
| **ZooKeeper** | CP | ZAB | Kafka, HBase, older Hadoop stacks |
| **Eureka** | AP | none (best-effort replication) | Netflix's JVM microservices |

### 8. Service mesh — moving discovery out of your code entirely

A **service mesh** (Istio, Linkerd) injects a **sidecar proxy** — a second container, usually Envoy — into every pod. Your Node.js app makes a plain, boring HTTP call to `http://orders`. Iptables rules transparently redirect that packet to the sidecar on `localhost`, and the sidecar does *everything*:

- service discovery (it subscribes to the control plane for live endpoints)
- load balancing (least-request, zone-aware, consistent hash)
- retries, timeouts, circuit breaking
- **mTLS** — mutual TLS between every pair of services, with certificates rotated automatically. Your app speaks plaintext HTTP; the wire is encrypted.
- telemetry — golden metrics (latency, traffic, errors, saturation) for every call, for free, with no instrumentation in your code. See [80 — Monitoring and Observability](./80-monitoring-and-observability.md).
- traffic shifting — "send 1% of traffic to v2" is a config change, not a deploy.

Your application code becomes *pure business logic*. Every cross-cutting concern moves into infrastructure.

**The trade-off is enormous operational complexity.** You now run a control plane. You have doubled your container count. Every request pays two extra proxy hops (out through your sidecar, in through theirs) — typically +1 to +3ms p50, but the p99 tail is where meshes bite. Debugging gets harder: "is this 503 from my app, my sidecar, their sidecar, or their app?" And you need someone on the team who genuinely understands Envoy config.

**Adopt a mesh when** you have dozens of services across multiple languages and you need mTLS or uniform observability as a compliance/scale requirement. **Do not adopt one** for six services in one language — you will spend more time on the mesh than on your product.

### 9. Health checks: liveness vs readiness

These two are constantly confused and the confusion causes real outages. The distinction is simple:

| | **Liveness** | **Readiness** |
|---|---|---|
| Asks | "Is the process alive / not wedged?" | "Can I serve traffic *right now*?" |
| On failure | **RESTARTS the container** | **REMOVES you from the load balancer** — does not kill you |
| Should check | Almost nothing. Is the event loop responsive? | Dependencies: DB connected? Cache warm? Migrations done? Not overloaded? |
| Failure is | Terminal — only a restart can fix it | Usually temporary — you may recover |

```javascript
import express from 'express';
const app = express();

let isShuttingDown = false;

// LIVENESS: deliberately dumb. If this responds at all, the event loop is turning.
// Checking the database here is a CLASSIC, DEVASTATING mistake: if the DB has a
// 30-second blip, every pod fails liveness, every pod gets RESTARTED, and now you
// have a cluster-wide restart storm on top of a database problem. Never do it.
app.get('/healthz', (req, res) => res.status(200).send('ok'));

// READINESS: this is where dependency checks belong. Failing it takes you OUT of
// rotation without killing you. When the DB comes back, you rejoin automatically.
app.get('/readyz', async (req, res) => {
  if (isShuttingDown) return res.status(503).send('shutting down');

  const checks = {
    db: await db.ping().then(() => true).catch(() => false),
    cache: cache.isConnected(),
    // Shed load rather than pile up: if the queue is deep, ask the LB to route
    // elsewhere until we catch up. Self-defence, not death.
    notOverloaded: app.locals.inflight < 200,
  };

  const ok = Object.values(checks).every(Boolean);
  res.status(ok ? 200 : 503).json(checks);
});

// The shutdown dance that prevents dropped requests:
process.on('SIGTERM', async () => {
  isShuttingDown = true;      // 1. readiness starts failing IMMEDIATELY
  await sleep(10_000);        // 2. wait for the LB/kube-proxy to notice and stop
                              //    sending us new traffic (propagation is NOT instant)
  server.close();             // 3. now finish in-flight requests and exit
});
```

**The classic disaster:** you put a database check in the liveness probe. The database has a brief hiccup. Every pod in the fleet fails liveness simultaneously. Kubernetes restarts all of them. They come up, still can't reach the database, fail liveness again, restart again — a **CrashLoopBackOff storm**. A 30-second database blip becomes a 30-minute total outage, caused entirely by the health check.

**Rule:** liveness answers "is this process irrecoverably wedged?" — and the honest answer is almost always "no." Keep it trivial.

---

## Visual / Diagram description

### Diagram 1: Client-side vs server-side discovery, side by side

```
        CLIENT-SIDE DISCOVERY                    SERVER-SIDE DISCOVERY
        (Eureka + Ribbon)                        (Kubernetes Service, AWS ALB)

   ┌──────────────────┐                      ┌──────────────────┐
   │   API Gateway    │                      │   API Gateway    │
   │  ┌────────────┐  │                      │                  │
   │  │ Discovery  │  │  1. "give me the     │  (no discovery   │
   │  │  Client +  │──┼──── list of         │   code at all)   │
   │  │    LB      │  │     orders"          │                  │
   │  └─────┬──────┘  │                      └────────┬─────────┘
   └────────┼─────────┘                               │
            │                                          │ 1. POST http://orders/…
            ▼                                          │   (one stable address,
   ┌──────────────────┐                                │    forever)
   │ SERVICE REGISTRY │                                ▼
   │    (Eureka)      │                      ┌──────────────────┐
   │                  │                      │  LOAD BALANCER   │
   │ orders →         │  2. returns          │  / ClusterIP VIP │
   │  10.0.1.4:3000   │ ─── ALL 3 ───▶       │   10.96.14.7     │
   │  10.0.2.9:3000   │     instances        └────────┬─────────┘
   │  10.0.3.7:3000   │                               │  2. LB consults registry
   └──────────────────┘                               │     & picks ONE instance
            ▲                                         │     (client never sees it)
            │ heartbeats                              ▼
   ┌────────┴─────────────────────┐        ┌──────────────────────┐
   │                              │        │  ┌────────────────┐  │
   │  3. client PICKS one itself  │        │  │ orders 10.0.1.4│  │
   │     and calls it DIRECTLY    │        │  └────────────────┘  │
   │                              │        │  ┌────────────────┐  │
   │  ┌────────────┐              │        │  │ orders 10.0.2.9│◀─┼── forwarded
   │  │orders      │◀─── direct   │        │  └────────────────┘  │
   │  │10.0.2.9    │     call     │        │  ┌────────────────┐  │
   │  └────────────┘              │        │  │ orders 10.0.3.7│  │
   │  ┌────────────┐              │        │  └────────────────┘  │
   │  │orders      │              │        └──────────────────────┘
   │  │10.0.3.7    │              │
   │  └────────────┘              │         HOPS: 2 (client→LB→instance)
   └──────────────────────────────┘         CLIENT CODE: none
                                            LANGUAGES: all, free
    HOPS: 1 (client→instance)
    CLIENT CODE: a library, per language
    LANGUAGES: one library each
```

**What this shows.** On the left, the *client* is the intelligent party: it holds the list, it makes the choice, it connects directly. One hop, maximum control, but that intelligence must exist in every language you write. On the right, the client is deliberately stupid: it knows one address and nothing else. Two hops, but the intelligence lives in one shared piece of infrastructure and every language gets it for free.

### Diagram 2: The registration and heartbeat lifecycle

```
  INSTANCE                        REGISTRY                        CALLER
     │                               │                               │
     │──── 1. REGISTER ─────────────▶│                               │
     │      (id, ip, port, ttl=30s)  │ orders → [10.0.2.9] ✓         │
     │                               │                               │
     │──── 2. heartbeat (t=10s) ────▶│ ✓ alive                       │
     │──── 2. heartbeat (t=20s) ────▶│ ✓ alive                       │
     │                               │◀──── 3. "who is orders?" ─────│
     │                               │───── [10.0.2.9] ─────────────▶│
     │◀───────────────── 4. request ─┼───────────────────────────────│
     │───────────────── 200 OK ──────┼──────────────────────────────▶│
     │                               │                               │
     ╳  5. PROCESS CRASHES (t=25s)   │                               │
     ╳     no deregister call ever   │ orders → [10.0.2.9] ✓ (LIE)   │
     ╳     happens                   │                               │
     ╳                               │◀──── "who is orders?" ────────│
     ╳                               │───── [10.0.2.9] ────────────▶│  ← stale!
     ╳◀──────────────── request ─────┼───────────────────────────────│
     ╳     ECONNREFUSED ─────────────┼──────────────────────────────▶│
     ╳                               │                        ┌──────┴──────┐
     ╳                               │                        │ RETRY on    │
     ╳                               │                        │ next        │
     ╳                               │                        │ instance,   │
     ╳   ◀── THE LYING WINDOW ──▶    │                        │ quarantine  │
     ╳       ~10s (missed beat)      │                        │ this one    │
     ╳     + 30s (TTL must expire)   │                        └─────────────┘
     ╳     + up to 30s client cache  │
     ╳     ≈ up to 70 seconds        │
     ╳                               │
     ╳   6. TTL EXPIRES (t=55s) ────▶│ orders → [] ✗ evicted
```

**What this shows.** Steps 1–4 are the happy path. Step 5 is the one that matters: after a crash, **nothing tells the registry**. The registry keeps confidently handing out a dead address for tens of seconds. No amount of registry cleverness closes that window — only the *caller's* retry and circuit-breaker logic makes the system survive it. Draw this diagram in an interview and you will have made the key point.

---

## Real world examples

### 1. Netflix — Eureka, and choosing AP on purpose

Netflix built **Eureka** for their AWS-based JVM microservices, and their design write-ups are explicit that they chose **availability over consistency**. Eureka servers replicate registry state to each other on a best-effort basis with no consensus protocol. Under a partition, Eureka enters **self-preservation mode**: when the rate of received heartbeats drops below an expected threshold, it *stops evicting instances entirely*, on the reasoning that a sudden mass heartbeat failure almost certainly means the network is broken, not that 300 servers died at once. Clients (Ribbon) hold a locally cached instance list and do their own load balancing, so even a completely unreachable Eureka doesn't stop traffic — clients just keep using the last list they had. The whole system is built on the premise that **stale is survivable and silent is fatal.**

### 2. Kubernetes — server-side discovery as a platform primitive

Kubernetes made service discovery so invisible that most engineers never think about it. You write `fetch('http://orders')`. CoreDNS resolves `orders.default.svc.cluster.local` to a ClusterIP. kube-proxy has already programmed iptables/IPVS rules on every node so packets to that VIP are DNAT'd to a ready pod IP. The control plane keeps EndpointSlices in sync with pod readiness, and etcd (a CP store) holds the truth. Your app contains **zero lines of discovery code** — the registrar is the platform, and the load balancer is the kernel.

### 3. HashiCorp Consul — discovery plus mesh, across datacenters

Consul runs an agent on every node. Services register with their local agent (or the agent registers them), and the agent runs configurable health checks — HTTP, TCP, script, or TTL. Consul then exposes the registry through **two interfaces at once**: a DNS interface (`orders.service.consul` — so legacy apps that only speak DNS still work) and a rich HTTP API for clients that want health metadata and tags. Its server cluster is Raft-based and therefore CP. Consul Connect layers a service mesh (Envoy sidecars, mTLS, intentions-based authorization) on top of that same registry — which is why Consul is common in hybrid estates that aren't all-in on Kubernetes, especially those spanning multiple datacenters.

---

## Trade-offs

| Approach | Pros | Cons | Use when |
|---|---|---|---|
| **Hardcoded IPs / config** | Trivial. Zero infrastructure. | Breaks on every deploy and every scale event. Not viable in the cloud. | Never, beyond a local demo |
| **DNS-based** | Universal. Every language speaks it. No client library. | Aggressive caching ignores TTLs; no health or load info; slow failover; no ports in A records. | As a stable name in front of an LB — not as an instance list |
| **Client-side (Eureka + Ribbon)** | One hop. Smart, zone-aware, consistent-hash balancing. Survives a dead registry via cached lists. | A discovery library per language. App coupled to the registry. Every client is a place bugs can hide. | Single-language shop; latency-critical; you need locality routing |
| **Server-side (K8s Service, ALB)** | Dumb clients. Language-agnostic. Balancing logic in one place, one team. | Extra hop. The LB is another HA system to run (usually managed). | The default for almost everyone |
| **Service mesh (Istio, Linkerd)** | Discovery, retries, mTLS, and telemetry with zero app code. Uniform across all languages. | Big operational burden. Doubled containers. Two extra proxy hops. Hard to debug. | Dozens of services, multiple languages, mTLS/observability mandates |

**Consistency of the registry itself:**

| Choice | You get | You give up |
|---|---|---|
| **CP** (etcd, Consul, ZooKeeper) | Never a split-brain view; correct leader election | Under partition, the registry may refuse to answer — nothing can find anything |
| **AP** (Eureka) | The registry always answers | The answer may include dead instances — clients MUST retry |

**Rule of thumb:** Use whatever your platform gives you. On Kubernetes, that means **Services + readiness probes** — do not bolt Eureka onto it. Reach for a mesh only when the pain of doing retries, mTLS, and observability by hand in four different languages is *demonstrably* worse than the pain of running Envoy. And whichever you pick, **write your clients as if the registry is lying to you**, because for a few seconds after every crash, it is.

---

## Common interview questions on this topic

### Q1: "Service A needs to call Service B. How does A find B?"
**Hint:** Do not say "environment variable." Say: B's instances **register** with a service registry on startup and heartbeat to stay listed. Then pick a mode — either A queries the registry and load balances itself (**client-side**, e.g. Eureka + Ribbon), or A calls one stable virtual IP / DNS name and a load balancer does the lookup and forwarding (**server-side**, e.g. a Kubernetes Service). Name the default: on Kubernetes it's server-side, and the app writes no discovery code at all.

### Q2: "An instance crashes hard — power cut, no graceful shutdown. What happens to callers?"
**Hint:** This is the question they actually want to hear you answer. Nothing deregisters it, so the registry keeps handing out its address until the **TTL expires** — worst case heartbeat interval + TTL + client cache TTL, easily 30–70 seconds. During that window callers get `ECONNREFUSED`. The registry *cannot* fix this; the **client** must: retry on the next instance immediately, quarantine the failing address locally, and trip a **circuit breaker** ([73](./73-circuit-breaker.md)). Tightening the TTL to 2s doesn't help — you'd get false evictions on every GC pause and healthy instances would flap.

### Q3: "What's the difference between a liveness probe and a readiness probe?"
**Hint:** Liveness = "is the process wedged?" — failing it **restarts the container**. Readiness = "can I serve traffic right now?" — failing it **removes you from the load balancer without killing you**. Then deliver the punchline: never check your database in a liveness probe. A brief DB outage would fail liveness on every pod at once, restart the entire fleet, and turn a 30-second blip into a CrashLoopBackOff storm. Dependency checks belong in **readiness** only.

### Q4: "Should a service registry be CP or AP? Why?"
**Hint:** Recall CAP (topic 09): under a partition you pick one. etcd/Consul/ZooKeeper chose **CP** — they'd rather refuse to answer than risk being wrong. Eureka deliberately chose **AP** — a stale list beats no list. Argue the AP side for discovery specifically: stale routing data **degrades gracefully** because clients already retry and circuit-break, whereas an unavailable registry is a **total outage** — nothing can find anything. The cost of being slightly wrong is far below the cost of being silent. (Note the irony that Kubernetes stores its registry in etcd, a CP store — that's a deliberate correctness trade for the control plane.)

### Q5: "Why not just use DNS for service discovery?"
**Hint:** Because DNS records carry no health and no load information — an A record says an IP exists, not that it works. And clients cache aggressively and routinely ignore TTLs (the JVM historically cached forever; OS resolvers and sidecars cache too), so removing a dead instance can take far longer than your TTL suggests. DNS is great as the *stable name* in front of a load balancer. It is a bad *live instance list*.

### Q6: "What does a service mesh give you that a plain registry doesn't?"
**Hint:** It moves discovery, load balancing, retries, timeouts, circuit breaking, **mTLS**, and telemetry out of your application and into a **sidecar proxy** — uniformly, across every language, with zero app code. The price is enormous operational complexity: a control plane, double the containers, two extra proxy hops per request, and much harder debugging. Worth it at dozens of polyglot services with security/observability mandates; badly not worth it for six services in one language.

---

## Practice exercise

### Build a Working Service Registry (~35 min)

Build a tiny but *honest* service registry in Node.js — one that actually exhibits the staleness window — and prove to yourself that the client is what saves you.

**What to produce — three files:**

**1. `registry.js`** — an Express server with:
- `PUT /register` — body `{ id, name, address, port, ttlMs }`. Store in an in-memory `Map`.
- `PUT /heartbeat/:id` — refresh that instance's `lastSeen` timestamp.
- `DELETE /deregister/:id` — remove it immediately.
- `GET /services/:name` — return only instances whose `lastSeen` is within their TTL.
- A background sweeper (`setInterval`, every 5s) that **evicts** expired instances and logs `EVICTED <id> after <n>ms of silence`.

**2. `service.js`** — a worker that takes a port from `process.argv`, serves `GET /work` returning `{ instance: <its id> }`, self-registers with a **10-second TTL**, and heartbeats every **3 seconds**. Handle `SIGTERM` by deregistering *before* exiting.

**3. `client.js`** — every 500ms: fetch the instance list (cache it for 5 seconds — this is the client cache that widens the staleness window), pick one round-robin, call `GET /work`, and log which instance answered. On failure, **retry the next instance** and quarantine the failed address for 15 seconds.

**Then run the experiment and record the numbers:**
1. Start the registry and **three** services on ports 4001, 4002, 4003. Start the client. You should see requests spread across all three.
2. **Graceful stop:** `kill -TERM` one service. How many client requests fail? (Should be **zero** — it deregistered first.)
3. **Hard crash:** `kill -9` another service. Now count: **how many client requests hit the dead instance before the registry evicts it?** Time it with a stopwatch.
4. Compare your measured window against the theory: `heartbeat interval (3s) + TTL (10s) + client cache (5s)` ≈ 18s worst case.
5. Now **comment out the retry logic in the client.** Re-run step 3. Watch a fraction of your requests die outright — and understand, in your bones, why retries are not optional.

Write down the number of failed requests **with** and **without** retries. That single comparison is the whole lesson of this topic.

---

## Quick reference cheat sheet

- **The problem:** in the cloud, instance IPs and ports change on every deploy, every autoscale, and every crash. You can never hardcode an address.
- **Service registry:** the live map of `service name → healthy instances right now`. It is critical infrastructure and must be highly available.
- **Self-registration:** the instance calls the registry on boot and heartbeats. Simple, but needs a client library per language.
- **Third-party registration:** an agent or the platform registers it (this is Kubernetes). Zero app code — the preferred model.
- **Heartbeat + TTL** is the ONLY defence against a crashed instance that never deregisters. Heartbeat at roughly **1/3 of the TTL** so you survive a couple of lost packets.
- **The staleness window is real:** heartbeat interval + TTL + client cache can leave the registry lying to you for **30–70 seconds**. You cannot tune it away.
- **Therefore: clients MUST retry** on the next instance and use circuit breakers ([73](./73-circuit-breaker.md)). Discovery alone is never enough.
- **Client-side discovery:** client gets the full list and balances itself. One hop, smart routing, but a library per language.
- **Server-side discovery:** client calls one stable VIP/DNS name; the LB does the lookup. Dumb clients, language-agnostic, one extra hop. **The modern default.**
- **DNS is a trap as an instance list:** clients cache and ignore TTLs, and DNS carries no health or load info. Fine as a stable name in front of an LB.
- **Kubernetes:** `Service` = stable ClusterIP + DNS (`orders.default.svc.cluster.local`); `EndpointSlice` = the ready pod IPs; `kube-proxy` = iptables/IPVS rules doing the forwarding in the kernel. **Headless service** (`clusterIP: None`) returns raw pod IPs.
- **CP registries** (etcd, Consul, ZooKeeper) may refuse to answer under partition. **AP registries** (Eureka) always answer, possibly with stale data. For discovery, AP is a defensible and often better choice — tie it to CAP (topic 09).
- **Service mesh** (Istio, Linkerd): sidecar proxies absorb discovery, LB, retries, mTLS, and telemetry out of your code. Massive operational cost — earn it before you adopt it.
- **Liveness** = "is the process alive?" → failing it **RESTARTS** the container. Keep it trivial.
- **Readiness** = "can I serve traffic now?" → failing it **REMOVES you from the LB** without killing you. All dependency checks go here.
- **Never put a DB check in liveness.** One DB blip → every pod fails liveness → fleet-wide restart storm → CrashLoopBackOff.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) — splitting the monolith is exactly what creates the "where is it?" problem |
| **Next** | [73 — Circuit Breaker](./73-circuit-breaker.md) — what your client does when discovery hands it a dead address |
| **Related** | [55 — Load Balancing](./55-load-balancing.md) — server-side discovery IS a load balancer with a live registry behind it |
| **Related** | [88 — Containers and Orchestration](./88-containers-and-orchestration.md) — how Kubernetes Services, EndpointSlices, and kube-proxy fit together |
| **Related** | [54 — DNS Deep Dive](./54-dns-deep-dive.md) — why DNS caching and TTLs make DNS a poor instance list |
| **Related** | [80 — Monitoring and Observability](./80-monitoring-and-observability.md) — health checks, golden signals, and what a mesh gives you for free |
