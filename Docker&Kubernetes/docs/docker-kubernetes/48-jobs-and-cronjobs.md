# 48 — Jobs and CronJobs
## Section: Kubernetes in Practice

---

## ELI5 — The Simple Analogy

Most workers in a company have a job that **never ends** — the receptionist answers the phone all day, every day, forever. That's a Deployment: a pod that's *supposed* to keep running. If it stops, something is wrong, so Kubernetes restarts it.

But some work is **"do this one thing, then you're done."** Like "shred these 500 old files." You hire a temp, they shred the files, and then they **go home** — and that's *success*, not failure. If a normal boss (a Deployment) saw the temp leave, they'd panic and hire them again in an endless loop. You need a different kind of boss who understands "finished = good." That boss is a **Job**: it runs a pod until the task **completes successfully**, then leaves it done.

And a **CronJob** is just a **calendar with an alarm clock** stapled to that idea: "every night at 2am, hire a temp to shred yesterday's files." It doesn't run anything itself — it just *creates a Job* on a schedule, over and over, like an alarm that fires a new temp each night.

So: **Deployment** = runs forever. **Job** = runs once until it succeeds, then stops. **CronJob** = fires a new Job on a schedule.

---

## The underlying mechanism (Job controller + etcd + a time loop, no direct kernel primitive)

Jobs and CronJobs are controllers in `kube-controller-manager` (Topic 29), driven by the reconciliation loop (Topic 30). The thing that makes them *different* from a Deployment is what "desired state" means and how they treat a **finished** pod.

**The pod `restartPolicy` is the crux.** A Deployment's pods use `restartPolicy: Always` — the kubelet restarts the container whenever it exits, *even on exit code 0*, because a long-running server should never exit. A Job's pods use `restartPolicy: OnFailure` or `Never`. This tells the kubelet: **exit code 0 means "done, leave it,"** and only a *non-zero* exit means "failed." That one field is the difference between "restart forever" and "run to completion."

**The Job controller counts successes.** It watches its pods in etcd. Its desired state is `completions` successful pod exits. Each time a pod exits 0, it increments `status.succeeded`. When `succeeded == completions`, the Job is marked `Complete` and the controller stops creating pods. If a pod exits non-zero, the controller creates a **replacement** pod (a retry) — but only up to `backoffLimit` times, after which the Job is marked `Failed`.

**The CronJob controller is a clock.** It runs a loop that, roughly every 10 seconds, reads the CronJob's `schedule` (standard cron syntax), compares it to the current time, and when a scheduled time has arrived that it hasn't handled yet, it **creates a fresh Job object** in etcd. It never runs containers itself — it only mints Jobs. The Job controller then does the actual work as above. This is why a CronJob is "an alarm clock that presses the Job button."

There's no kernel magic — it's etcd objects, exit codes read by the kubelet, and two controllers counting and clock-watching.

---

## What is this?

A **Job** runs one or more pods until a specified number of them **complete successfully** (exit 0), then stops — the right tool for one-off tasks like a database migration or a bulk data fix.

A **CronJob** creates a Job automatically on a **cron schedule** (e.g. every night at 2am) — the right tool for recurring maintenance like nightly cleanups, report generation, or backups.

---

## Why does it matter for a backend developer?

You constantly have work that **shouldn't run forever**:

- Run a **database migration** before `orders-api:1.5.0` starts serving.
- **Reindex** search data after a bulk import.
- **Nightly**: delete cancelled orders older than 90 days, expire stale carts, roll up yesterday's metrics.
- **Weekly**: send a digest email, prune old audit logs.

If you don't understand Jobs, you'll do these the wrong, dangerous ways:

- **Run a migration as a Deployment** → it "succeeds," exits 0, the Deployment restarts it, it runs the migration **again and again in a crash-loop**, possibly corrupting data or hammering the DB.
- **Run a nightly cleanup via a `setInterval()` inside `orders-api`** → now every one of your 6 API replicas runs the cleanup at 2am *simultaneously*, six times, racing each other, while also making your request-serving pods do heavy batch work and blow their memory limits.
- **Have no retry/timeout controls** → a flaky migration either never retries or retries infinitely, and a hung cleanup runs for 9 hours holding DB locks with nothing to stop it.

Without Jobs/CronJobs you will:
- Crash-loop one-off tasks by modeling them as long-running services.
- Duplicate scheduled work across replicas.
- Have no clean way to bound retries (`backoffLimit`) or runtime (`activeDeadlineSeconds`).
- Not know how to prevent overlapping runs (`concurrencyPolicy`) when last night's cleanup is still running.

This is core operational plumbing for any real backend on Kubernetes.

---

## The physical reality

When a Job runs, here's what **actually exists** in the cluster.

**A Job object owns pods, and tracks a success count:**

```
$ kubectl get job db-migrate -n orders
NAME         COMPLETIONS   DURATION   AGE
db-migrate   1/1           14s        2m
#            │
#            └─ succeeded / completions. "1/1" = it's done.

$ kubectl get pods -n orders -l job-name=db-migrate
NAME               READY   STATUS      RESTARTS   AGE
db-migrate-7q4mk   0/1     Completed   0          2m
#                          │
#                          └─ NOT "Running", NOT "Error" — "Completed". The pod exited 0 and is LEFT there
#                             (so you can read its logs) rather than restarted.
```

The Job's status in etcd literally records the counts and completion time:

```
$ kubectl get job db-migrate -n orders -o jsonpath='{.status}'
{"succeeded":1,"completionTime":"2026-07-12T02:00:14Z","conditions":[{"type":"Complete","status":"True"}]}
```

**A CronJob keeps a history of the Jobs it spawned:**

```
$ kubectl get cronjob nightly-cleanup -n orders
NAME              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
nightly-cleanup   0 2 * * *     False     0        8h              5d
#                 │                       │        │
│                 │                       │        └─ last time it fired a Job
│                 │                       └─ how many Jobs are running right now
│                 └─ cron schedule: 02:00 every day

$ kubectl get jobs -n orders
NAME                         COMPLETIONS   AGE
nightly-cleanup-28234560     1/1           3d    ← kept in history (successfulJobsHistoryLimit)
nightly-cleanup-28235280     1/1           2d
nightly-cleanup-28236000     1/1           1d    ← the numeric suffix is the scheduled minute-since-epoch
```

Each spawned Job in turn owns its own pod. So the ownership chain is: **CronJob → Job → Pod**, and each layer is a real object in etcd. A completed Job's pod sticks around (as `Completed`) precisely so you can `kubectl logs` it to see what happened — until TTL or history limits clean it up.

---

## How it works — step by step

### A Job running a DB migration (`completions: 1`, `backoffLimit: 4`)

1. **You apply the Job.** The API server writes a Job object to etcd. Desired state: 1 successful completion.
2. **The Job controller creates a pod.** The pod runs `npm run migrate`. Its `restartPolicy` is `Never`.
3. **kubelet runs the container and watches its exit code.** This is the whole game — the exit code is the verdict.
4. **Case A — exit 0 (success).** The kubelet marks the pod `Completed` and does **not** restart it (because `restartPolicy: Never` + exit 0). The Job controller sees the success, sets `status.succeeded = 1`. Since `1 == completions`, it marks the Job `Complete`. Done. The `Completed` pod is left so you can read its logs.
5. **Case B — exit 1 (migration threw).** The kubelet marks the pod `Failed`. The Job controller counts one failure and creates a **new** pod to retry — but waits an exponentially increasing backoff (10s, 20s, 40s...) first. It will retry up to `backoffLimit: 4` times.
6. **Case B continues — 5th failure.** Once failures exceed `backoffLimit`, the Job controller stops retrying and marks the Job `Failed` with reason `BackoffLimitExceeded`. No more pods. Now a human (or the CI pipeline) must intervene — which is correct; you don't want a broken migration retrying forever.
7. **Case C — it hangs.** If a pod runs longer than `activeDeadlineSeconds`, the controller **kills** the Job and its pods with reason `DeadlineExceeded`, regardless of retries. This bounds total runtime so a stuck migration holding DB locks can't run all night.

### A CronJob firing that Job nightly (`schedule: "0 2 * * *"`)

1. **You apply the CronJob.** It goes to etcd. Nothing runs yet — it's just a schedule + a Job template.
2. **The CronJob controller ticks (~every 10s).** It reads the current time and the `schedule`. Most ticks: "not 2am, nothing to do."
3. **2:00:00 arrives.** The controller computes that a scheduled time has passed that it hasn't yet handled, and **creates a Job** named `nightly-cleanup-<epoch-minute>`. It applies `concurrencyPolicy` here (see below).
4. **The Job controller takes over.** It runs the cleanup pod exactly like the Job trace above (completions, backoff, deadline all apply to *this* Job).
5. **Next night, another tick at 2am creates another Job.** Over time you accumulate a history of Job objects.
6. **History is trimmed.** `successfulJobsHistoryLimit` (default 3) and `failedJobsHistoryLimit` (default 1) cause the controller to garbage-collect old finished Jobs so etcd doesn't fill with thousands of them.
7. **Overlap handling.** If it's 2am again but *last night's* cleanup Job is somehow still running, `concurrencyPolicy` decides: `Allow` (run both), `Forbid` (skip the new one), or `Replace` (kill the old, start the new).

**The key insight:** the exit code is the truth (step 3–4), `backoffLimit` bounds *retries*, `activeDeadlineSeconds` bounds *time*, and the CronJob layer is purely a scheduler that manufactures Jobs.

---

## Exact syntax breakdown

### Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  namespace: orders
spec:
  backoffLimit: 4
  activeDeadlineSeconds: 600
  completions: 1
  parallelism: 1
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: orders-api:1.5.0
          command: ["npm", "run", "migrate"]
```

```
apiVersion: batch/v1           ← Jobs/CronJobs live in the "batch" API group, not "apps"
kind: Job
spec:
  backoffLimit: 4
  │            │
  │            └─ how many times to RETRY a failing pod before giving up (marking Job Failed).
  │               Counts pod failures, not container restarts. Default 6.
  │
  activeDeadlineSeconds: 600
  │                     │
  │                     └─ hard wall-clock limit (10 min). Exceed it → Job killed with DeadlineExceeded,
  │                        even mid-retry. Bounds a hung migration from running forever.
  │
  completions: 1
  │           │
  │           └─ how many pods must exit 0 for the Job to be Complete. 1 = a single task.
  │              Set to N for "run this exact task N times" (e.g. process 100 items).
  │
  parallelism: 1
  │           │
  │           └─ how many pods may run AT ONCE. With completions:100, parallelism:5 = a worker
  │              pool of 5 chewing through 100 completions. Default 1.
  │
  ttlSecondsAfterFinished: 3600
  │                       │
  │                       └─ auto-delete the Job (and its pods) 1h after it finishes, so completed
  │                          Jobs don't pile up. Without it, they linger until you delete them.
  │
  template:
    spec:
      restartPolicy: Never     ← MUST be Never or OnFailure for a Job (NOT Always).
      │                           Never = failed pod is left, controller makes a NEW pod to retry.
      │                           OnFailure = kubelet restarts the SAME pod's container in place.
      containers:
        - name: migrate
          command: ["npm","run","migrate"]   ← exit 0 = success, non-zero = failure. This is the verdict.
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
  namespace: orders
spec:
  schedule: "0 2 * * *"
  timeZone: "America/New_York"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cleanup
              image: orders-api:1.5.0
              command: ["node", "scripts/cleanup-old-orders.js"]
```

```
kind: CronJob
spec:
  schedule: "0 2 * * *"
  │          │ │ │ │ │
  │          │ │ │ │ └─ day of week (0-6, Sun=0)   *=every
  │          │ │ │ └─── month (1-12)               *=every
  │          │ │ └───── day of month (1-31)        *=every
  │          │ └─────── hour (0-23)          -> 2 = 2 AM
  │          └───────── minute (0-59)        -> 0 = on the hour
  │          (so: "at minute 0 of hour 2, every day, every month, every weekday" = 2:00 AM daily)
  │
  timeZone: "America/New_York"   ← interpret the schedule in THIS tz (else UTC). Avoids "why did it run at 9pm?"
  │
  concurrencyPolicy: Forbid
  │                 │
  │                 └─ what to do if the previous run is still going when the next is due:
  │                    Allow   = run them concurrently (default)
  │                    Forbid  = SKIP the new run until the old finishes (safest for DB cleanups)
  │                    Replace = kill the old run, start the new one
  │
  startingDeadlineSeconds: 300
  │                       │
  │                       └─ if the controller missed the scheduled time (e.g. control-plane was down),
  │                          still start within 300s of the deadline; miss by more → skip that run.
  │
  successfulJobsHistoryLimit: 3  ← keep the last 3 successful Jobs (for log inspection); GC older ones
  failedJobsHistoryLimit: 1      ← keep the last 1 failed Job
  │
  jobTemplate:                   ← the Job that gets minted each tick. Everything under here is a Job spec.
    spec:
      ...                        ← backoffLimit/activeDeadlineSeconds/etc apply to EACH spawned Job
```

Manually triggering a CronJob's Job right now (great for testing a migration/cleanup on demand):

```
kubectl create job --from=cronjob/nightly-cleanup manual-run-1 -n orders
│              │    │                             │
│              │    │                             └─ a name for this one-off Job
│              │    └─ clone the pod template from this CronJob
│              └─ create a Job immediately (don't wait for 2am)
└─ kubectl
```

---

## Example 1 — basic

A one-off database migration Job for `orders-api`, every line commented.

```yaml
# db-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate-1-5-0          # include the version so each migration is a distinct Job
  namespace: orders
spec:
  backoffLimit: 3                 # retry a failed migration up to 3 times, then stop and alert a human
  activeDeadlineSeconds: 300      # if it runs > 5 min, kill it (a migration this small shouldn't hang)
  ttlSecondsAfterFinished: 1800   # clean up this Job 30 min after it finishes
  template:
    spec:
      restartPolicy: Never        # exit 0 = done; on failure, make a fresh pod (don't loop the same one)
      containers:
        - name: migrate
          image: orders-api:1.5.0 # same image as the app, so migrations match the code
          command: ["npm", "run", "migrate"]   # e.g. runs node-pg-migrate / knex migrate:latest
          env:
            - name: DATABASE_URL  # point at the primary from Topic 47
              value: "postgres://app:secret@postgres-0.postgres.orders.svc.cluster.local:5432/orders"
```

```bash
# Run the migration.
kubectl apply -f db-migrate.yaml

# Watch it. It should go Running -> Completed (not restart, not crash-loop).
kubectl get job db-migrate-1-5-0 -n orders -w
# NAME               COMPLETIONS   DURATION   AGE
# db-migrate-1-5-0   0/1           5s         5s
# db-migrate-1-5-0   1/1           11s        11s     ← 1/1 = success

# Read what the migration actually did (the Completed pod's logs are still there).
kubectl logs -n orders -l job-name=db-migrate-1-5-0
# > Migrating 1 file: 013_add_discount_column.sql ... done

# If it had FAILED, you'd see:
#   db-migrate-1-5-0   0/1   ...  and pods in Error, then reason BackoffLimitExceeded after 3 tries.
kubectl describe job db-migrate-1-5-0 -n orders   # shows conditions: Complete or Failed + reason
```

The critical contrast: model this as a Deployment and exit-0 would trigger a restart, re-running the migration forever. As a Job, exit-0 means **done** and Kubernetes leaves it alone.

---

## Example 2 — production scenario

**The situation.** `orders-api` accumulates junk: carts abandoned > 7 days, `CANCELLED` orders older than 90 days, and expired idempotency keys in Redis. Left alone, the `orders` table grows to tens of millions of dead rows, queries slow down, and Postgres storage bloats. You need a **nightly cleanup** at 2am that is safe, bounded, and never overlaps with itself.

**The wrong way** (that a lot of teams ship first): a `setInterval` inside `orders-api`.

```js
// DON'T DO THIS inside your API server
setInterval(runCleanup, 24 * 60 * 60 * 1000);
```

With 6 `orders-api` replicas, this runs **6 times at once**, each deleting the same rows and racing on locks, while stealing CPU/memory from request-serving pods — occasionally getting them OOMKilled (Topic 44) mid-request. It's a latent outage.

**The right way** — a CronJob, isolated from the serving path:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
  namespace: orders
spec:
  schedule: "0 2 * * *"                 # 2:00 AM
  timeZone: "America/New_York"          # business timezone, not UTC surprises
  concurrencyPolicy: Forbid             # if last night's run is somehow still going, SKIP tonight's —
                                        # never run two deleters against the same tables at once
  startingDeadlineSeconds: 600          # if the control plane was down at 2am, still run within 10 min
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2                   # a transient DB blip can retry twice
      activeDeadlineSeconds: 1800       # HARD 30-min cap — a runaway delete can't hold locks all night
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cleanup
              image: orders-api:1.5.0   # reuse the app image; the script shares the same DB models
              command: ["node", "scripts/cleanup.js"]
              env:
                - name: DATABASE_URL
                  value: "postgres://app:secret@postgres-0.postgres.orders.svc.cluster.local:5432/orders"
              resources:                # its own limits — batch work can't hurt the API pods (Topic 44)
                requests: { cpu: "250m", memory: "256Mi" }
                limits:   { cpu: "1",    memory: "512Mi" }
```

Why each choice is load-bearing:

- **`concurrencyPolicy: Forbid`** — if a huge cleanup one night runs past 2am the next, the new run is *skipped* instead of double-deleting. For a destructive task, this prevents lock contention and duplicate work.
- **`activeDeadlineSeconds: 1800`** — a cleanup that hits a bad query plan and starts a table scan can't run for 9 hours holding row locks and blocking `orders-api` writes. It's killed at 30 min and alerts you.
- **`backoffLimit: 2`** — a momentary DB failover gets two retries, but a genuinely broken script fails fast rather than pounding the DB all night.
- **Own resource limits** — the batch job runs in its own pod with its own budget; it literally cannot OOM a request-serving replica.
- **Reusing `orders-api:1.5.0`** — the cleanup script imports the same ORM models/migrations, so it can't drift from the app's schema.

Test it on demand before trusting the schedule:

```bash
kubectl create job --from=cronjob/nightly-cleanup cleanup-test -n orders
kubectl logs -n orders -l job-name=cleanup-test -f
# > Deleted 4,127 abandoned carts; 18,904 cancelled orders > 90d; 220,113 expired idempotency keys.
```

**The payoff:** maintenance runs exactly once, at the right time, in an isolated pod with bounded retries and runtime, and it can never overlap or starve your live API. That's the difference between "a cron script we hope works" and production-grade operations.

---

## Common mistakes

**Mistake 1 — `restartPolicy: Always` on a Job (or modeling a task as a Deployment).**

```
$ kubectl apply -f migrate-job.yaml
The Job "db-migrate" is invalid: spec.template.spec.restartPolicy: Unsupported value:
"Always": supported values: "OnFailure", "Never"
```

Or, if you used a Deployment instead of a Job, no error — but the migration succeeds, exits 0, and:

```
$ kubectl get pods -n orders
db-migrate-xxxx   0/1   CrashLoopBackOff   7   6m    ← "crash looping" on SUCCESS, re-running the migration
```

Root cause: `Always` means "restart even on exit 0," which is correct for servers and catastrophic for one-shot tasks. Right: use `kind: Job` with `restartPolicy: Never` or `OnFailure`.

**Mistake 2 — No `activeDeadlineSeconds`, so a hung Job runs forever.**

Your migration deadlocks against a long-running transaction and just sits there:

```
$ kubectl get job db-migrate -n orders
NAME         COMPLETIONS   DURATION   AGE
db-migrate   0/1           4h37m      4h37m    ← still "running," holding a lock, blocking deploys
```

Root cause: with no time bound, a stuck pod runs until you notice. Right: set `activeDeadlineSeconds` so it's killed with `DeadlineExceeded` and you get alerted instead of a silent 4-hour stall.

**Mistake 3 — `concurrencyPolicy: Allow` (default) for a job that must not overlap.**

A backup CronJob runs every 30 min, but one run takes 40 min:

```
$ kubectl get jobs -n orders
backup-28236000   0/1   Running   # started 12:00, still going
backup-28236030   0/1   Running   # started 12:30 — now TWO backups write the same target, corrupting it
```

Root cause: the default `Allow` lets runs stack up. Right: `concurrencyPolicy: Forbid` (skip until the last finishes) or `Replace` (cancel the old one) for anything that shares a resource.

**Mistake 4 — CronJob schedule in UTC when you meant local time.**

```yaml
schedule: "0 2 * * *"   # you meant 2am your time; with no timeZone this is 2am UTC = 9pm EST
```

Your "nightly" cleanup runs during your evening traffic peak. Root cause: without `timeZone`, cron is interpreted in **UTC**. Right: set `timeZone: "America/New_York"` (or your zone) explicitly.

**Mistake 5 — Completed Jobs pile up and clutter (or fill etcd).**

```
$ kubectl get jobs -n orders | wc -l
1,847    ← months of finished migration/cleanup Jobs never cleaned up
```

Root cause: finished Jobs are **not** auto-deleted unless you tell them to. Right: set `ttlSecondsAfterFinished` on Jobs, and rely on `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` on CronJobs to trim their spawned Jobs.

---

## Hands-on proof

Run these on any cluster right now.

```bash
kubectl create namespace orders 2>/dev/null

# ---------- Job: runs to completion, then STOPS (does not restart on success) ----------
cat <<'EOF' | kubectl apply -n orders -f -
apiVersion: batch/v1
kind: Job
metadata: { name: onetime }
spec:
  backoffLimit: 2
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox:1.36
          command: ["sh","-c","echo 'doing one-time work'; sleep 3; echo done; exit 0"]
EOF

# 1. Watch it reach Completed (1/1) and then do nothing further — no restart despite exit 0.
kubectl get job onetime -n orders -w   # Ctrl-C once COMPLETIONS shows 1/1
kubectl get pods -n orders -l job-name=onetime   # STATUS = Completed (not Running, not CrashLoop)
kubectl logs -n orders -l job-name=onetime       # the pod's logs are still readable

# ---------- Job that FAILS: prove backoffLimit bounds the retries ----------
cat <<'EOF' | kubectl apply -n orders -f -
apiVersion: batch/v1
kind: Job
metadata: { name: flaky }
spec:
  backoffLimit: 2                       # allow 2 retries, then give up
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox:1.36
          command: ["sh","-c","echo trying; exit 1"]   # always fails
EOF

# 2. Watch it create a few pods (retries) then mark the Job Failed with BackoffLimitExceeded.
sleep 60
kubectl get pods -n orders -l job-name=flaky          # several Error pods (initial + retries)
kubectl describe job flaky -n orders | grep -A2 Conditions   # Failed / BackoffLimitExceeded

# ---------- CronJob: fires a Job on a schedule ----------
cat <<'EOF' | kubectl apply -n orders -f -
apiVersion: batch/v1
kind: CronJob
metadata: { name: ticktock }
spec:
  schedule: "*/1 * * * *"               # every minute (for the demo)
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: tick
              image: busybox:1.36
              command: ["sh","-c","date; echo cleanup ran"]
EOF

# 3. Trigger one immediately instead of waiting (proves manual runs work).
kubectl create job --from=cronjob/ticktock manual-now -n orders
kubectl logs -n orders -l job-name=manual-now

# 4. Wait ~2 minutes and see the CronJob spawn Jobs on its own, and history trimmed to 2.
sleep 130
kubectl get cronjob ticktock -n orders     # LAST SCHEDULE populated
kubectl get jobs -n orders                 # ticktock-<epoch> Jobs; at most 2 successful kept

# Clean up.
kubectl delete namespace orders
```

What you verified: a Job reaches **Completed and stops** on exit 0 (step 1), a failing Job **retries up to `backoffLimit` then Fails** (step 2), a CronJob **mints Jobs on a schedule** and you can **trigger one manually** (steps 3–4), with history auto-trimmed.

---

## Practice exercises

### Exercise 1 — easy
Create a Job that runs `busybox` with `command: ["sh","-c","echo migrating; exit 0"]`. Watch it with `kubectl get job -w` until `1/1`. Then run `kubectl get pods -l job-name=<job>` and note the pod STATUS. Explain in one sentence why the pod is `Completed` rather than restarted, referencing `restartPolicy`.

### Exercise 2 — medium
Create a Job with `completions: 6` and `parallelism: 2`, running `busybox` that sleeps 5 seconds and exits 0. Watch `kubectl get pods -l job-name=<job> -w`. How many pods run at the same time, and how many total pods run before the Job is Complete? Explain what `completions` and `parallelism` each controlled.

### Exercise 3 — hard (production simulation)
Build the nightly cleanup as a CronJob scheduled `*/2 * * * *` (every 2 min for testing) whose pod sleeps 200 seconds (longer than the interval) and exits 0. Set `concurrencyPolicy: Forbid`. Watch across several minutes and prove that a second Job is **skipped** while the first is still running (only one `Running` at a time). Then set `activeDeadlineSeconds: 60` and observe a run get killed with `DeadlineExceeded`. Write down: (a) what `Forbid` prevented, and (b) why `activeDeadlineSeconds` is essential for a destructive cleanup that could hold DB locks.

---

## Mental model checkpoint

Answer from memory:

1. What field distinguishes a Job's pod from a Deployment's pod, and what does each value mean?
2. What determines whether a Job pod counts as success vs failure?
3. What does `backoffLimit` bound, and what does `activeDeadlineSeconds` bound — and how do they differ?
4. What is the ownership chain from a CronJob down to a running container?
5. What do the three `concurrencyPolicy` values do, and when would you pick `Forbid`?
6. Why does a completed Job's pod stick around, and how do you stop them from piling up?
7. Why should a nightly cleanup be a CronJob rather than a `setInterval` inside `orders-api`?

---

## Quick reference card

| Field / command | What it does | Key detail |
|---|---|---|
| `kind: Job` | Run pods until N complete successfully | `apiVersion: batch/v1` |
| `restartPolicy` | `Never` / `OnFailure` for Jobs | `Always` is rejected; it's for Deployments |
| `completions` | How many successful exits needed | Default 1 |
| `parallelism` | How many pods run at once | Worker-pool with `completions` |
| `backoffLimit` | Max retries before Job = Failed | Default 6; bounds *retries* |
| `activeDeadlineSeconds` | Hard wall-clock limit | Bounds *time*; kills with `DeadlineExceeded` |
| `ttlSecondsAfterFinished` | Auto-delete Job after it ends | Stops finished Jobs piling up |
| `kind: CronJob` | Create a Job on a cron schedule | Only mints Jobs; runs nothing itself |
| `schedule` | Cron string `min hr dom mon dow` | UTC unless `timeZone` set |
| `concurrencyPolicy` | Overlap behavior | `Allow`/`Forbid`/`Replace` |
| `successfulJobsHistoryLimit` | Kept finished Jobs | Default 3 (failed: 1) |
| `kubectl create job --from=cronjob/X` | Fire a CronJob's Job now | Test before trusting the schedule |

---

## When would I use this at work?

1. **Database migrations on deploy.** Run `npm run migrate` as a Job (often a Helm pre-upgrade hook, Topic 49) *before* the new `orders-api` pods roll out, so the schema is ready. The Job's exit code gates the deploy: migration fails → deploy aborts.

2. **Scheduled maintenance.** Nightly cleanup of stale orders/carts, weekly audit-log pruning, hourly cache warm-ups, or periodic report generation — each a CronJob with `concurrencyPolicy` and `activeDeadlineSeconds` so it's safe and bounded, isolated from your request-serving pods.

3. **One-off operational fixes.** A backfill after a bug ("recompute totals for orders created between X and Y") runs as a single Job with `backoffLimit` and a deadline — no code in the app, no crash-loop risk, full logs retained for the audit trail.

---

## Connected topics

- **Study before:**
  - **Topic 33 — Pods in Depth**: Jobs and CronJobs ultimately run pods; `restartPolicy` behavior is a pod concept.
  - **Topic 34 — Deployments**: the "runs forever" model these are the "runs to completion" counterpart to.
  - **Topic 30 — The Reconciliation Loop**: the controller behavior counting completions and ticking the schedule.
- **Study after:**
  - **Topic 44 — Resource Requests and Limits**: giving batch Jobs their own budget so they can't starve serving pods.
  - **Topic 47 — DaemonSets and StatefulSets**: the `postgres-0.postgres` endpoint your migration/cleanup Jobs connect to.
  - **Topic 49 — Helm Basics**: running migration Jobs as pre-install/pre-upgrade hooks in a chart.
  - **Topic 52 — CI/CD with Kubernetes**: wiring migration Jobs into the deploy pipeline and gating on their success.
