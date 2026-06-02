# Docker & Kubernetes — Complete Master Plan
## Deep Mastery Curriculum: Zero → Backend Engineer Level

---

## Phases Overview

| Phase | Focus | Topics | Goal |
|-------|-------|--------|------|
| Docker Phase 1 | Foundations | 01–05 | Understand what containers ARE at the kernel level |
| Docker Phase 2 | Images & Dockerfiles | 06–11 | Build production-grade images for Node.js |
| Docker Phase 3 | Running Containers | 12–17 | Master every runtime flag, networking, volumes |
| Docker Phase 4 | Docker Compose | 18–22 | Orchestrate multi-container dev environments |
| Docker Phase 5 | Docker in Production | 23–27 | Security, CI/CD, registries, resource management |
| Kubernetes Phase 1 | Foundations | 28–32 | Understand K8s architecture and the reconciliation loop |
| Kubernetes Phase 2 | Core Objects | 33–38 | Pods, Deployments, Services, ConfigMaps, Namespaces |
| Kubernetes Phase 3 | Storage & Networking | 39–42 | PVs, Ingress, Network Policies, CNI |
| Kubernetes Phase 4 | Operations | 43–48 | Health checks, autoscaling, rolling updates, Jobs |
| Kubernetes Phase 5 | Production K8s | 49–54 | Helm, secrets, observability, CI/CD, RBAC |

---

## Complete Curriculum — All 54 Topics

### DOCKER PHASE 1 — Foundations

| # | Topic | One-line Description |
|---|-------|---------------------|
| 01 | What Is a Container | VM vs container, what isolation actually means, why the industry moved to containers |
| 02 | Linux Foundations of Containers | Namespaces (6 types), cgroups, overlay filesystem — the kernel trinity that makes Docker possible |
| 03 | Docker Architecture | Docker CLI → Docker daemon → containerd → runc — the full component chain and how they communicate |
| 04 | Images and Layers | What an image actually is on disk, layers as filesystem diffs, content-addressable storage, pull mechanics |
| 05 | Your First Container | docker run step-by-step trace, interactive vs detached, container lifecycle states |

### DOCKER PHASE 2 — Images and Dockerfiles

| # | Topic | One-line Description |
|---|-------|---------------------|
| 06 | Dockerfile in Depth | Every instruction (FROM, RUN, COPY, ADD, ENV, ARG, WORKDIR, EXPOSE, CMD, ENTRYPOINT, USER, VOLUME, LABEL, HEALTHCHECK) with exact semantics |
| 07 | CMD vs ENTRYPOINT | The real difference, override behavior, exec form vs shell form, combining them correctly |
| 08 | Layer Caching in Depth | What breaks cache, correct instruction ordering, cache busting intentionally, cache math |
| 09 | Multi-Stage Builds | Why they exist, build stage vs runtime stage, dramatic image size reduction, dependency separation |
| 10 | .dockerignore | How it affects build context size and speed, patterns, what to always exclude |
| 11 | Production Dockerfile for Node.js | Multi-stage, non-root user, correct COPY order, health check, signal handling, complete template |

### DOCKER PHASE 3 — Running Containers

| # | Topic | One-line Description |
|---|-------|---------------------|
| 12 | docker run in Depth | Every important flag (-d, -p, -v, -e, --name, --network, --restart, --memory, --cpus, --rm) with kernel implications |
| 13 | Networking in Docker | Bridge, host, none, custom networks, container-to-container DNS, veth pairs, iptables rules |
| 14 | Volumes in Depth | Bind mounts vs named volumes vs tmpfs, host filesystem mapping, data persistence and lifecycle |
| 15 | Environment Variables and Secrets | -e flag, --env-file, 12-factor app config, why not to bake secrets into images |
| 16 | Container Lifecycle | Created, running, paused, stopped, exited, dead — what each state means at the kernel level |
| 17 | Logging | docker logs, log drivers (json-file, syslog, fluentd), structured logging, reading logs in production |

### DOCKER PHASE 4 — Docker Compose

| # | Topic | One-line Description |
|---|-------|---------------------|
| 18 | Docker Compose in Depth | What it is, docker-compose.yml structure, services/networks/volumes top-level keys |
| 19 | Compose File Deep Dive | Every important field: image, build, ports, volumes, environment, depends_on, healthcheck, restart, networks |
| 20 | Multi-Container Apps | Node.js + Postgres + Redis in Compose, service discovery by name, startup order and health dependencies |
| 21 | Compose for Development | Hot reload with bind mounts, environment overrides, docker-compose.override.yml, dev vs prod configs |
| 22 | Compose Commands | up, down, build, logs, exec, ps, restart — when to use each, flags that matter |

### DOCKER PHASE 5 — Docker in Production

| # | Topic | One-line Description |
|---|-------|---------------------|
| 23 | Image Security | Vulnerability scanning, non-root users, read-only filesystems, minimal base images (alpine, distroless) |
| 24 | Docker Registry | Pushing/pulling from Docker Hub, private registries, image tagging strategy (semver, SHA, latest) |
| 25 | Resource Limits | --memory, --cpus, what happens when limits are exceeded, OOMKilled, CPU throttling at the cgroup level |
| 26 | Health Checks | HEALTHCHECK instruction, --health-cmd, health states, what Docker does with unhealthy containers |
| 27 | Docker in CI/CD | Building images in GitHub Actions, layer caching in CI, pushing to registry on merge, automation patterns |

### KUBERNETES PHASE 1 — Foundations

| # | Topic | One-line Description |
|---|-------|---------------------|
| 28 | What Is Kubernetes and Why | The problem Docker Compose doesn't solve at scale, orchestration need, self-healing, declarative state |
| 29 | Kubernetes Architecture | Control plane (API server, etcd, scheduler, controller manager) vs worker nodes (kubelet, kube-proxy, CRI) |
| 30 | The Reconciliation Loop | Desired state vs actual state, how controllers watch and reconcile, the core concept of all K8s behavior |
| 31 | kubectl Basics | Contexts, namespaces, basic verbs (get, describe, apply, delete, logs, exec), output formats |
| 32 | YAML and Manifests | apiVersion, kind, metadata, spec — the structure of every manifest, how the API server processes them |

### KUBERNETES PHASE 2 — Core Objects

| # | Topic | One-line Description |
|---|-------|---------------------|
| 33 | Pods in Depth | Smallest deployable unit, why not just containers, multi-container pods, init containers, pod lifecycle phases |
| 34 | Deployments | Managing pods at scale, replicas, rolling updates, rollback, the ReplicaSet underneath, revision history |
| 35 | Services | ClusterIP vs NodePort vs LoadBalancer, kube-proxy routing (iptables/IPVS), DNS for services |
| 36 | ConfigMaps and Secrets | Externalizing config, mounting as files vs env vars, how Secrets are stored in etcd, update propagation |
| 37 | Namespaces | Isolation boundaries, resource quotas, naming conventions, when to use multiple namespaces |
| 38 | Labels and Selectors | How Kubernetes connects objects together, label strategy for teams, matchLabels mechanics |

### KUBERNETES PHASE 3 — Storage and Networking

| # | Topic | One-line Description |
|---|-------|---------------------|
| 39 | Persistent Volumes and Claims | PV, PVC, StorageClass, dynamic provisioning, reclaim policies, what happens when a pod restarts |
| 40 | Kubernetes Networking in Depth | How pods get IPs, CNI plugins, pod-to-pod routing across nodes, service IP routing, kube-proxy modes |
| 41 | Ingress | Ingress controllers (nginx), routing rules, TLS termination, path-based and host-based routing |
| 42 | Network Policies | Restricting pod-to-pod traffic, ingress/egress rules, default deny, zero-trust networking |

### KUBERNETES PHASE 4 — Operations

| # | Topic | One-line Description |
|---|-------|---------------------|
| 43 | Health Checks in Kubernetes | Liveness vs readiness vs startup probes, exact difference, how misconfiguration kills your app |
| 44 | Resource Requests and Limits | CPU/memory requests vs limits, scheduler decisions, OOMKilled, CPU throttling, QoS classes |
| 45 | Horizontal Pod Autoscaler | Scaling on CPU/memory, custom metrics, min/max replicas, scaling behavior, cooldown periods |
| 46 | Rolling Updates and Rollbacks | maxSurge, maxUnavailable, zero-downtime deploys, rollback mechanics, revision history |
| 47 | DaemonSets and StatefulSets | When to use instead of Deployments, logging agents, databases, stable network identity, ordered startup |
| 48 | Jobs and CronJobs | One-time tasks, scheduled tasks, backoffLimit, completions, parallelism, real use cases |

### KUBERNETES PHASE 5 — Production Kubernetes

| # | Topic | One-line Description |
|---|-------|---------------------|
| 49 | Helm Basics | What Helm is, charts, values.yaml, templating, installing/upgrading releases, chart repositories |
| 50 | Secrets Management at Scale | K8s Secrets limitations, HashiCorp Vault concept, External Secrets Operator, sealed secrets |
| 51 | Observability in Kubernetes | Metrics (Prometheus), logs (EFK/Loki), traces (Jaeger/OTEL), instrumenting a Node.js app |
| 52 | CI/CD with Kubernetes | GitOps concept (ArgoCD/Flux), deploying via kubectl in GitHub Actions, image update strategies |
| 53 | RBAC | Role, ClusterRole, RoleBinding, ClusterRoleBinding, service accounts, least privilege for CI/CD |
| 54 | Multi-Environment Strategy | Dev/staging/prod namespaces vs clusters, Helm values per environment, promotion workflows |

---

## Complete File List

```
docs/docker-kubernetes/
├── 00-master-plan.md
├── 01-what-is-a-container.md
├── 02-linux-foundations-of-containers.md
├── 03-docker-architecture.md
├── 04-images-and-layers.md
├── 05-your-first-container.md
├── 06-dockerfile-in-depth.md
├── 07-cmd-vs-entrypoint.md
├── 08-layer-caching-in-depth.md
├── 09-multi-stage-builds.md
├── 10-dockerignore.md
├── 11-production-dockerfile-for-nodejs.md
├── 12-docker-run-in-depth.md
├── 13-networking-in-docker.md
├── 14-volumes-in-depth.md
├── 15-environment-variables-and-secrets.md
├── 16-container-lifecycle.md
├── 17-logging.md
├── 18-docker-compose-in-depth.md
├── 19-compose-file-deep-dive.md
├── 20-multi-container-apps.md
├── 21-compose-for-development.md
├── 22-compose-commands.md
├── 23-image-security.md
├── 24-docker-registry.md
├── 25-resource-limits.md
├── 26-health-checks.md
├── 27-docker-in-cicd.md
├── 28-what-is-kubernetes-and-why.md
├── 29-kubernetes-architecture.md
├── 30-the-reconciliation-loop.md
├── 31-kubectl-basics.md
├── 32-yaml-and-manifests.md
├── 33-pods-in-depth.md
├── 34-deployments.md
├── 35-services.md
├── 36-configmaps-and-secrets.md
├── 37-namespaces.md
├── 38-labels-and-selectors.md
├── 39-persistent-volumes-and-claims.md
├── 40-kubernetes-networking-in-depth.md
├── 41-ingress.md
├── 42-network-policies.md
├── 43-health-checks-in-kubernetes.md
├── 44-resource-requests-and-limits.md
├── 45-horizontal-pod-autoscaler.md
├── 46-rolling-updates-and-rollbacks.md
├── 47-daemonsets-and-statefulsets.md
├── 48-jobs-and-cronjobs.md
├── 49-helm-basics.md
├── 50-secrets-management-at-scale.md
├── 51-observability-in-kubernetes.md
├── 52-cicd-with-kubernetes.md
├── 53-rbac.md
└── 54-multi-environment-strategy.md
```

---

## Progress Tracker

### Docker Phase 1 — Foundations
- [ ] 01 — What Is a Container
- [ ] 02 — Linux Foundations of Containers
- [ ] 03 — Docker Architecture
- [ ] 04 — Images and Layers
- [ ] 05 — Your First Container

### Docker Phase 2 — Images and Dockerfiles
- [ ] 06 — Dockerfile in Depth
- [ ] 07 — CMD vs ENTRYPOINT
- [ ] 08 — Layer Caching in Depth
- [ ] 09 — Multi-Stage Builds
- [ ] 10 — .dockerignore
- [ ] 11 — Production Dockerfile for Node.js

### Docker Phase 3 — Running Containers
- [ ] 12 — docker run in Depth
- [ ] 13 — Networking in Docker
- [ ] 14 — Volumes in Depth
- [ ] 15 — Environment Variables and Secrets
- [ ] 16 — Container Lifecycle
- [ ] 17 — Logging

### Docker Phase 4 — Docker Compose
- [ ] 18 — Docker Compose in Depth
- [ ] 19 — Compose File Deep Dive
- [ ] 20 — Multi-Container Apps
- [ ] 21 — Compose for Development
- [ ] 22 — Compose Commands

### Docker Phase 5 — Docker in Production
- [ ] 23 — Image Security
- [ ] 24 — Docker Registry
- [ ] 25 — Resource Limits
- [ ] 26 — Health Checks
- [ ] 27 — Docker in CI/CD

### Kubernetes Phase 1 — Foundations
- [ ] 28 — What Is Kubernetes and Why
- [ ] 29 — Kubernetes Architecture
- [ ] 30 — The Reconciliation Loop
- [ ] 31 — kubectl Basics
- [ ] 32 — YAML and Manifests

### Kubernetes Phase 2 — Core Objects
- [ ] 33 — Pods in Depth
- [ ] 34 — Deployments
- [ ] 35 — Services
- [ ] 36 — ConfigMaps and Secrets
- [ ] 37 — Namespaces
- [ ] 38 — Labels and Selectors

### Kubernetes Phase 3 — Storage and Networking
- [ ] 39 — Persistent Volumes and Claims
- [ ] 40 — Kubernetes Networking in Depth
- [ ] 41 — Ingress
- [ ] 42 — Network Policies

### Kubernetes Phase 4 — Operations
- [ ] 43 — Health Checks in Kubernetes
- [ ] 44 — Resource Requests and Limits
- [ ] 45 — Horizontal Pod Autoscaler
- [ ] 46 — Rolling Updates and Rollbacks
- [ ] 47 — DaemonSets and StatefulSets
- [ ] 48 — Jobs and CronJobs

### Kubernetes Phase 5 — Production Kubernetes
- [ ] 49 — Helm Basics
- [ ] 50 — Secrets Management at Scale
- [ ] 51 — Observability in Kubernetes
- [ ] 52 — CI/CD with Kubernetes
- [ ] 53 — RBAC
- [ ] 54 — Multi-Environment Strategy

---

## How to Use This Plan

| Command | Action |
|---------|--------|
| **START** | Begin Topic 01 |
| **NEXT** | Move to the next topic after exercises |
| **HINT** | Get one nudge for a stuck exercise |
| **PROGRESS** | Show the checklist with completion status |
| **REDO [topic#]** | Regenerate a specific topic |

---

## Notes

- Each topic produces one complete markdown file following the mandatory template
- All exercises require a real machine with Docker (and later Kubernetes) installed
- Topics build on each other — do them in order
- Every topic references back to previous topics explicitly
- Target audience: Node.js backend developer moving toward platform/DevOps confidence
