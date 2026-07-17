# 130 — Design a Task Scheduler
## Category: LLD Case Study

---

## What is this?

A **task scheduler** is the thing that runs your code *later*, not *now*. You hand it a job and a time — "send this email in 10 minutes," "regenerate the sitemap every hour," "retry this failed payment with backoff" — and it takes responsibility for firing that job at exactly the right moment.

Think of it as an **airport control tower**. Dozens of planes (tasks) want to take off. Each one has a scheduled slot. The tower keeps them ordered by who's next, holds the rest on the tarmac, and only clears a plane for takeoff when its slot arrives — and only if there's a free runway (a free worker). Nobody takes off early. Nobody takes off if all runways are busy.

This is the low-level design (LLD) behind everything from `setTimeout`, to cron, to Celery, to your CI pipeline's job runner.

---

## Why does it matter?

Almost every non-trivial backend needs deferred work. If you don't understand how a scheduler is built, you'll reach for the wrong tool and get burned:

- **You'll block a request thread doing work that should have been deferred.** The user clicks "checkout," and you send the receipt email *inside the request* — now checkout takes 3 seconds because your SMTP server is slow. A scheduler moves that off the hot path.
- **You'll busy-loop and melt a CPU core.** The naive scheduler scans a list of a million tasks every millisecond asking "is anyone due yet?" A real one sleeps until the *next* task is due and wakes up exactly then.
- **You'll double-run jobs** the day you scale to two servers, because nothing stops both from grabbing the same task. (That's the distributed extension, and it's the interview's favorite follow-up.)
- **You'll have no retry story.** A task fails, and it's just... gone. No backoff, no dead-letter, no visibility.

**Interview angle:** "Design a task scheduler" (or "job scheduler," "in-memory cron") tests whether you can combine three data-structure/concurrency ideas cleanly: a **priority queue** (what runs next?), a **producer-consumer worker pool** (who runs it, and how many at once?), and a **retry policy** (what if it fails?). It's a favorite because a weak candidate scans a list and a strong one reaches for a heap.

**Real-work angle:** You will configure or debug one of these — BullMQ, Agenda, Quartz, Celery, Sidekiq, Kubernetes CronJobs, AWS EventBridge Scheduler. Knowing the internals is the difference between "the job didn't run and I don't know why" and "the job didn't run because its `nextRunTime` was recomputed against the wrong clock."

---

## The core idea — explained simply

### The Airport Control Tower Analogy

Picture a small regional airport.

**Planes waiting to take off** are your **tasks**. Each has a departure time (`nextRunTime`).

**The tower's flight strip board** is a **priority queue ordered by departure time**. The board is not sorted by when the plane arrived or its size — it's sorted so the *earliest departure is always on top*. The controller only ever has to look at the top strip to know who's next.

**The runways** are your **worker pool**. There are only a fixed number (say 3). A plane can only depart if a runway is free. If all runways are busy, the next plane holds — even if its slot has arrived.

**The controller** is the **scheduler loop**. He does one simple thing on repeat: look at the top strip. Is that plane's departure time now or in the past? If yes and a runway is free, clear it for takeoff. If its time is still in the future, he doesn't pace around asking every plane every second — he **sets an alarm for exactly the top plane's departure time and takes a nap.** If a new plane with an *earlier* slot shows up, the alarm gets reset earlier.

**A recurring flight** (the 9am, 10am, 11am shuttle) is the neat trick: the moment the 9am shuttle departs, the controller writes a *new* strip for the same flight at 10am and slots it back onto the board. The flight reschedules itself.

**A rejected takeoff** (engine warning) is a **failure + retry**: the plane taxis back, waits a bit longer each time (backoff), and tries again — up to a limit, after which it's towed to the hangar (dead-letter).

| Analogy | Technical concept |
|---------|-------------------|
| Plane waiting to depart | A `Task` |
| Departure time | `task.nextRunTime` |
| Flight strip board (earliest on top) | **Min-heap / priority queue** keyed on `nextRunTime` |
| Looking only at the top strip | `peek()` in O(1) |
| The controller on repeat | The **scheduler loop** (`tick`) |
| Setting an alarm, then napping | Sleeping until the next due time (not busy-looping) |
| The runways (fixed number) | The **worker pool** / concurrency limit |
| Clearing a plane for takeoff | Dispatching a task to a worker (producer → consumer) |
| The 9/10/11am shuttle | A **recurring task** re-inserting itself |
| Rejected takeoff, taxi back, try later | **Retry with backoff** |
| Towed to the hangar | **Dead-letter** after max attempts |

Everything below is this analogy made precise, in runnable JavaScript.

---

## Key concepts inside this topic

### 1. Requirements & clarifying questions

Before writing a line of code, pin down scope. In an interview, *say these out loud* — it's half the grade.

**Functional requirements:**

- Schedule a task to run **once** at a future time.
- Schedule a task to run on a **recurring interval** (every N ms — a simplified cron).
- Tasks have a **priority** (tie-breaker when two are due at the same instant).
- A **pool of workers** executes due tasks, with a cap on how many run at once.
- **Retry on failure** with exponential backoff, up to a max number of attempts.
- **Cancel** a scheduled task before it runs.

**Non-functional requirements:**

- Finding the next task to run should be **cheap** — O(log n) inserts, O(1) "who's next," not an O(n) scan every tick.
- The scheduler must **not busy-wait** — it sleeps until the next task is due.
- Deterministic and **testable** — the clock must be injectable so tests don't depend on wall-clock sleeps.

**Clarifying questions to ask the interviewer** (each one changes the design):

- **Persistence?** "Do tasks survive a process restart, or is in-memory fine?" (In-memory here; persistence is an extension.)
- **Single-node or distributed?** "One process, or many scheduler nodes that mustn't double-run a task?" (Single-node here; distributed is the big extension via a lock — recall topics 83/84.)
- **Max concurrency?** "How many tasks may run simultaneously?" (Bounded worker pool.)
- **Missed-execution policy?** "If the process was down when a recurring task was due, do we run it once on recovery, skip it, or fire every missed occurrence?" (We'll run-once-on-catch-up; state the policy explicitly.)
- **Task duration?** "Are tasks short (ms) or long (minutes)?" (Affects whether the worker pool needs cancellation/timeouts.)

### 2. Identify the core objects (nouns)

Read the requirements and underline the nouns. Those become classes.

| Object | Responsibility |
|--------|----------------|
| `Task` | One unit of work: an id, the function to run (`execute()`), a priority, its `nextRunTime`, its recurring `intervalMs` (or null), and retry state (`attempts`, `maxAttempts`, `status`). |
| `TaskQueue` (a **MinHeap**) | Holds all scheduled tasks ordered so the earliest `nextRunTime` is always on top. Owns *ordering*. |
| `Scheduler` | The brain. Accepts `schedule()`/`cancel()`, runs the loop, decides what's due, dispatches, reschedules recurring tasks, and handles retries. |
| `WorkerPool` | A bounded concurrency limiter. Owns *how many tasks run at once*. |
| `RetryPolicy` | Given the attempt number, returns the backoff delay and whether to keep trying. Owns *retry math*. |
| `TriggerStrategy` | Computes the *next* run time for a task (one-off returns null → done; recurring returns now + interval). Owns *recurrence*. |

**Enum:**

```javascript
const TaskStatus = Object.freeze({
  SCHEDULED: 'SCHEDULED',   // in the queue, waiting for its time
  RUNNING:   'RUNNING',     // handed to a worker, executing now
  COMPLETED: 'COMPLETED',   // finished successfully
  FAILED:    'FAILED',      // exhausted all retries → dead-letter
  CANCELLED: 'CANCELLED',   // cancelled before it ran
});
```

### 3. Behaviours (verbs) + who owns them

The trap in LLD is putting every verb on one god-class. Distribute ownership:

- **schedule(task)** → `Scheduler`. It stamps the task `SCHEDULED` and pushes it onto the `TaskQueue`.
- **peek the earliest / pop the earliest** → `TaskQueue` (the heap). It, and only it, knows the ordering.
- **decide what's due** → `Scheduler`. On each `tick(now)`, it pops every task whose `nextRunTime <= now`.
- **run at most N at once** → `WorkerPool`. The scheduler *asks* the pool to run a task; the pool decides whether there's capacity or the task must wait.
- **reschedule a recurring task** → the `Task`'s own `TriggerStrategy` computes the next time; the scheduler re-inserts it. **The task owns its own trigger config.**
- **retry on failure** → `RetryPolicy` computes the backoff; the scheduler re-inserts the task at `now + backoff`, or dead-letters it.

Notice the clean split: **the queue owns ordering, the pool owns concurrency, the task owns its trigger/retry config, the scheduler orchestrates.** That's the whole design.

### 4. The core structures

This is the meat. Four ideas.

**(a) A priority queue / min-heap ordered by `nextRunTime`.**

Why not just a list? Imagine 1,000,000 scheduled tasks. On every tick you need "which task is due next?" With a plain array you'd scan all million entries every tick — O(n) per tick, and you tick constantly. That's the busy-loop that melts a CPU.

A **binary min-heap** gives you:

- `peek()` → the earliest task in **O(1)** (it's always the root).
- `push(task)` → insert in **O(log n)** (bubble up).
- `pop()` → remove the earliest in **O(log n)** (bubble down).

So the scheduler answers "who's next?" instantly and inserts/removes cheaply. A heap is a complete binary tree stored in a flat array: for node at index `i`, its children are at `2i+1` and `2i+2`, its parent at `Math.floor((i-1)/2)`. The invariant: every parent's key ≤ both children's keys. The key here is a tuple `(nextRunTime, priority)` — earliest time wins; on a tie, higher priority wins.

**(b) The scheduler loop.**

The loop is embarrassingly simple once the heap exists:

```
peek the top task
  is it due (nextRunTime <= now)?
    yes → pop it and dispatch to the worker pool
    no  → sleep until top.nextRunTime (or until a new, earlier task is added)
```

The **critical testability lesson**: do *not* call `Date.now()` and `setTimeout` directly in the core logic. Inject a **clock** and drive time with a `tick(now)` method. In production the clock is the real wall clock; in tests you *advance the clock by hand* and assert deterministically — no flaky "await a real 50ms" sleeps. This is the single most valuable idea in the whole document for real engineering: **time is a dependency; inject it.**

**(c) Recurring tasks.**

After a recurring task runs, the scheduler asks its `TriggerStrategy` for the next time (`now + intervalMs`), stamps the task `SCHEDULED` again, and pushes it back onto the heap. **The task reschedules itself.** One task object cycles through the queue forever until cancelled. This self-re-insertion is the elegant core of cron.

**(d) The worker pool — producer/consumer.**

Here the scheduler is the **producer**: when a task becomes due, it hands the task off to run. The **workers are consumers**: they pull work and execute it. (Recall [51 — Producer-Consumer](./51-pattern-producer-consumer.md) and [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md).) A **bounded pool** caps how many tasks run at once — if 100 tasks come due in the same tick but the pool size is 3, only 3 run and the other 97 wait for a free slot.

Even single-threaded JavaScript needs this. Node runs one thread, but tasks are `async` (they `await` I/O). Without a limit, 100 due tasks would fire 100 concurrent DB queries and exhaust your connection pool. The pool is a **concurrency limiter**: at most N in-flight promises; when one settles, the next waiting task starts.

**(e) Retries with backoff.**

On failure, the `RetryPolicy` decides: if attempts remain, re-schedule the task at `now + backoff(attempt)` where backoff grows exponentially (e.g. 100ms, 200ms, 400ms, 800ms). Exponential backoff avoids hammering a struggling downstream. When attempts are exhausted, mark `FAILED` and **dead-letter** it (recall [67 — Message Queues](./67-message-queues.md): a dead-letter queue is where poison messages go so they stop blocking healthy work). Backoff is best paired with *jitter* (a small random offset) so a thousand tasks that failed together don't all retry at the exact same instant — a "thundering herd."

---

## Visual / Diagram description

```
                          schedule(task) / cancel(id)
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────┐
        │                    SCHEDULER                        │
        │                                                     │
        │   tick(now):                                        │
        │     while heap not empty AND peek.nextRunTime<=now: │
        │        task = heap.pop()                            │
        │        pool.submit(() => run(task))       ───────┐  │
        │     nextWake = heap.peek()?.nextRunTime           │  │
        └───────────────┬─────────────────────────────────┼──┘
                        │ owns ORDERING                    │ producer
                        ▼                                  ▼  hands off due tasks
      ┌──────────────────────────────┐        ┌───────────────────────────┐
      │   TaskQueue  (MIN-HEAP)       │        │   WORKER POOL (size = 3)   │
      │   keyed on nextRunTime        │        │   owns CONCURRENCY         │
      │                               │        │                            │
      │            [t=100]  ← root    │        │  ┌──────┐ ┌──────┐ ┌──────┐│
      │           /       \           │        │  │ W1   │ │ W2   │ │ W3   ││ consumers
      │       [t=150]    [t=200]      │        │  │RUNNING│ │RUNNING│ │ idle ││
      │       /                       │        │  └──────┘ └──────┘ └──────┘│
      │   [t=400]                     │        │  waiting: [tX] [tY] ...     │
      └──────────────────────────────┘        └────────────┬──────────────┘
                        ▲                                    │
                        │  recurring: re-insert(now+interval)│ on finish
                        │  retry:     re-insert(now+backoff) │
                        └────────────────────────────────────┘
                                   (self-rescheduling)
```

**How to read it:** `schedule()` drops a task into the **min-heap** (left), which keeps the earliest `nextRunTime` at the root so the scheduler's `peek()` is O(1). The **scheduler loop** (top) pops every due task and hands it to the **worker pool** (right) — this is the producer→consumer boundary. The pool runs at most 3 at once; extras wait. When a worker finishes, two arrows flow *back* into the heap: a **recurring** task re-inserts itself at `now + interval`, and a **failed** task re-inserts at `now + backoff`. Cancelled and exhausted tasks leave the system.

---

## Real world examples

### BullMQ (Node.js + Redis) — representative

BullMQ is the de-facto job queue for Node. Conceptually it maps exactly onto this design: jobs have a `delay` (nextRunTime), a `priority`, an `attempts` count with a `backoff` strategy (`exponential`/`fixed`), and repeatable jobs (cron/every) that re-schedule themselves. The big difference is where the "heap" lives: BullMQ stores the ordered set in **Redis sorted sets** (a `ZSET` scored by timestamp — Redis's version of our min-heap) so *multiple* worker processes share one queue. Workers `BRPOPLPUSH` jobs off, giving the bounded-consumer behavior across machines.

### Quartz Scheduler (Java) — representative

Quartz is the classic enterprise scheduler. It separates a **Job** (the work) from a **Trigger** (when to fire) — precisely our `Task` vs `TriggerStrategy` split. Triggers come in flavors (SimpleTrigger for intervals, CronTrigger for cron expressions), a thread pool bounds concurrency, and a `misfire` policy answers the missed-execution question we raised in clarifying questions. Clustered Quartz uses a **database row lock** so only one node fires a given trigger — the distributed extension made real.

### Linux `cron` / systemd timers — factual

The original. `crond` reads crontab entries, computes each job's next run time, sleeps until the earliest, wakes, forks the due jobs, and recomputes. No priority queue in classic cron (it wakes every minute), but systemd timers and `anacron` add missed-execution catch-up — the same policy question. It's the proof that this pattern is 50 years old and everywhere.

---

## Trade-offs

**Min-heap vs. sorted list vs. scan-every-tick:**

| Approach | peek next | insert | Verdict |
|----------|-----------|--------|---------|
| Scan a list every tick | O(n) | O(1) | Simple but melts CPU at scale — avoid |
| Sorted array | O(1) | O(n) (shift) | Good peek, painful insert |
| **Binary min-heap** | **O(1)** | **O(log n)** | **The sweet spot** for many scheduled tasks |
| Redis ZSET / timing wheel | O(log n) / O(1) | O(log n) / O(1) | For distributed or millions of timers |

**Injected clock vs. real `setTimeout` in core logic:**

| | Injected clock + `tick()` | Real wall-clock sleeps |
|---|---|---|
| Testability | Deterministic, instant tests | Flaky, slow (real sleeps) |
| Production wiring | One thin adapter drives `tick` | Direct, less code |
| Debuggability | Replay time by hand | Heisenbugs |

**Bounded pool vs. unbounded:**

| | Bounded worker pool | Unbounded (fire all) |
|---|---|---|
| Downstream safety | Protects DBs/APIs | Can exhaust connection pools |
| Latency of a burst | Some tasks wait | All start immediately |
| Predictable load | Yes | No |

**The sweet spot:** a **min-heap** keyed on `(nextRunTime, priority)`, driven by an **injected clock** through a `tick()` loop, dispatching to a **bounded worker pool**, with **exponential-backoff-plus-jitter** retries and a **dead-letter** terminus. That combination is small, testable, and scales from a toy to (with a shared store + lock) a distributed cron service.

---

## Common interview questions on this topic

### Q1: "Why a heap instead of just sorting a list of tasks?"

**Hint:** A full sort is O(n log n) and you'd redo it on every insert. A heap keeps the *minimum* reachable in O(1) and supports insert/extract-min in O(log n) *incrementally* — you never re-sort the whole thing. You don't need the tasks fully ordered; you only ever need the *next* one, and a heap is the data structure that gives you exactly "the next one, cheaply."

### Q2: "How does the loop avoid busy-waiting? Walk me through the sleep."

**Hint:** Peek the root. If it's due, pop and dispatch. If it's *not* due, compute `delay = root.nextRunTime - now` and sleep for exactly that long (in production, `setTimeout(delay)`). Crucially, if `schedule()` inserts a task with an *earlier* time while you're asleep, you must **wake early and recompute** — otherwise the new task fires late. In the deterministic model we sidestep sleeping entirely by advancing an injected clock and calling `tick(now)`.

### Q3: "How does a recurring task work internally — where does the recurrence live?"

**Hint:** It's self-rescheduling. When the task finishes, if it has an `intervalMs`, the scheduler asks its `TriggerStrategy` for the next time (`now + intervalMs`), stamps it `SCHEDULED`, and pushes the *same task object* back onto the heap. The recurrence config lives on the task (Strategy pattern), not in the scheduler — so adding cron later means adding a new strategy, not touching the loop.

### Q4: "Two servers now run the scheduler. How do you stop them both running the same task?"

**Hint:** This is the distributed follow-up. In-memory heaps don't share state, so you need (1) a **shared store** for tasks (Redis ZSET / a DB table) and (2) a way to ensure exactly one node claims each due task — either **leader election** (recall 83) so one node schedules, or a **distributed lock / atomic claim** (recall 84) where a node does an atomic "SELECT ... FOR UPDATE SKIP LOCKED" or Redis `SET key val NX` before running. Whoever wins the lock runs it; the rest skip.

### Q5: "A task keeps failing. Walk me through retry and the dead-letter path."

**Hint:** On failure, increment `attempts`. If `attempts < maxAttempts`, the `RetryPolicy` returns `backoff = base * 2^(attempts-1)` (plus jitter), and the scheduler re-inserts the task at `now + backoff` — so retries space out exponentially instead of hammering. When `attempts` hits `maxAttempts`, stop retrying, set status `FAILED`, and push it to a **dead-letter** list for human inspection so a poison task can't retry forever (recall 67).

---

## Practice exercise

### Build a "reminder" scheduler and prove it deterministically

Starting from the code below (or from scratch), build a scheduler and write a **deterministic test** — no real sleeps — that proves all four behaviors:

1. Schedule three one-off tasks at t=300, t=100, t=200. Advance an injected clock in steps and assert they execute in the order **100, 200, 300** (proves heap ordering, not insertion order).
2. Schedule a recurring task with `intervalMs=50` starting at t=50. Advance the clock to t=200 and assert it ran at 50, 100, 150, 200 (proves self-rescheduling).
3. Schedule a task that throws on its first two runs and succeeds on the third, with `maxAttempts=3` and base backoff 100ms. Assert it retries at the backed-off times and finally COMPLETES (proves retry + backoff).
4. Set the worker pool size to 1, make each task take "time" (resolve on a later tick), schedule two due at the same instant, and assert they run **sequentially**, not concurrently (proves the concurrency limit).

Produce: a single `.js` file that runs with `node`, prints an execution log with timestamps, and ends by asserting the log matches expectations. Aim for ~30 minutes. The goal is to feel how injecting the clock makes time-based code *testable*.

---

## Quick reference cheat sheet

- **Task = a Command.** Model each job as an object with an `execute()` method (recall 43). The scheduler stores and invokes commands without knowing what they do.
- **Min-heap keyed on `nextRunTime`** — O(1) peek "who's next," O(log n) insert/extract. Never scan a list every tick.
- **Tie-break by priority** — when two tasks share a `nextRunTime`, the higher priority runs first.
- **The loop:** peek → if due, pop & dispatch → else sleep until the top's time (wake early if an earlier task arrives).
- **Inject the clock.** Drive time with `tick(now)`, not `Date.now()`. Testable, deterministic, no flaky sleeps.
- **Recurring = self-rescheduling.** After running, re-insert at `now + intervalMs`. The task cycles the heap forever.
- **Scheduler is the producer, workers are consumers** (recall 51). The pool bounds concurrency (recall 52).
- **Bounded pool** — at most N in-flight promises; extras wait. Protects downstream connection pools.
- **Retry with exponential backoff + jitter** — `base * 2^(n-1)` so retries space out and don't thunder.
- **Dead-letter** after `maxAttempts` — mark FAILED, park it, stop retrying poison tasks (recall 67).
- **Cancel** by id — flip status to CANCELLED; the loop skips cancelled tasks when popped (lazy deletion).
- **Distributed = shared store + lock.** Multiple nodes need a ZSET/DB plus leader election or a claim-lock so no task double-runs (recall 83/84).
- **Missed-execution policy** — decide up front: catch-up once, skip, or fire every missed occurrence.
- **Patterns in play:** Command (task), Producer-Consumer (dispatch), Strategy (trigger + retry).

---

## Full JavaScript implementation

Runnable with `node scheduler.js`. Every core piece from the spec is here: `TaskStatus` enum, a `Task` modeled as a **Command**, a real `MinHeap`, a bounded `WorkerPool`, `RetryPolicy`/`TriggerStrategy` strategies, the `Scheduler` with an **injected clock**, and a **deterministic** `main()` that advances the clock by hand.

```javascript
'use strict';

// ---------- Enum (Object.freeze so it can't be mutated) ----------
const TaskStatus = Object.freeze({
  SCHEDULED: 'SCHEDULED',
  RUNNING: 'RUNNING',
  COMPLETED: 'COMPLETED',
  FAILED: 'FAILED',
  CANCELLED: 'CANCELLED',
});

// ---------- Strategy: how to retry ----------
// Owns the retry math. Swap it out for a different backoff without touching
// the scheduler. Exponential backoff + a touch of deterministic "jitter".
class RetryPolicy {
  constructor({ maxAttempts = 3, baseDelayMs = 100 } = {}) {
    this.maxAttempts = maxAttempts;
    this.baseDelayMs = baseDelayMs;
  }
  shouldRetry(attempts) {
    return attempts < this.maxAttempts;
  }
  // attempts already incremented: 1 -> base, 2 -> base*2, 3 -> base*4 ...
  backoffMs(attempts) {
    return this.baseDelayMs * Math.pow(2, attempts - 1);
  }
}

// ---------- Strategy: when does this task run next? ----------
// One-off returns null (done forever). Recurring returns now + interval.
// Adding cron later = a new strategy here, scheduler untouched.
class TriggerStrategy {
  constructor(intervalMs = null) {
    this.intervalMs = intervalMs; // null => one-off
  }
  isRecurring() {
    return this.intervalMs !== null;
  }
  nextRunTime(now) {
    return this.isRecurring() ? now + this.intervalMs : null;
  }
}

// ---------- Task: modeled as a COMMAND (recall 43) ----------
// A task IS a command: an object that packages some work behind execute().
// The scheduler invokes execute() without knowing or caring what it does.
let _autoId = 0;
class Task {
  constructor({ name, run, nextRunTime, priority = 0, trigger = new TriggerStrategy() }) {
    this.id = `task-${++_autoId}`;
    this.name = name;
    this._run = run; // async () => any ; the actual work
    this.priority = priority; // higher = runs first on a nextRunTime tie
    this.nextRunTime = nextRunTime;
    this.trigger = trigger;
    this.attempts = 0;
    this.status = TaskStatus.SCHEDULED;
  }
  // The Command interface. Returns whatever run() returns; may throw.
  execute() {
    return this._run();
  }
}

// ---------- MinHeap: the priority queue keyed on (nextRunTime, priority) ----------
// Complete binary tree in a flat array. parent(i)=(i-1)/2, children=2i+1,2i+2.
// Invariant: every parent's key <= its children's keys, so the MIN is the root.
class MinHeap {
  constructor() {
    this._arr = [];
  }
  size() {
    return this._arr.length;
  }
  peek() {
    return this._arr[0]; // O(1): earliest task is always the root
  }
  // Ordering: earliest nextRunTime first; tie broken by HIGHER priority.
  _less(a, b) {
    if (a.nextRunTime !== b.nextRunTime) return a.nextRunTime < b.nextRunTime;
    return a.priority > b.priority;
  }
  push(task) {
    this._arr.push(task);
    this._bubbleUp(this._arr.length - 1); // O(log n)
  }
  pop() {
    const arr = this._arr;
    if (arr.length === 0) return undefined;
    const top = arr[0];
    const last = arr.pop();
    if (arr.length > 0) {
      arr[0] = last;
      this._bubbleDown(0); // O(log n)
    }
    return top;
  }
  _bubbleUp(i) {
    const arr = this._arr;
    while (i > 0) {
      const parent = (i - 1) >> 1;
      if (this._less(arr[i], arr[parent])) {
        [arr[i], arr[parent]] = [arr[parent], arr[i]];
        i = parent;
      } else break;
    }
  }
  _bubbleDown(i) {
    const arr = this._arr;
    const n = arr.length;
    while (true) {
      let smallest = i;
      const l = 2 * i + 1;
      const r = 2 * i + 2;
      if (l < n && this._less(arr[l], arr[smallest])) smallest = l;
      if (r < n && this._less(arr[r], arr[smallest])) smallest = r;
      if (smallest === i) break;
      [arr[i], arr[smallest]] = [arr[smallest], arr[i]];
      i = smallest;
    }
  }
}

// ---------- WorkerPool: bounded concurrency limiter (producer-consumer) ----------
// The scheduler PRODUCES due tasks; this pool of "workers" CONSUMES them,
// running at most `size` at once. Extra work waits in a FIFO backlog.
// (Recall 51 producer-consumer, 52 concurrency.) Single-threaded JS still
// needs this: it caps in-flight async work so we don't overwhelm downstream.
class WorkerPool {
  constructor(size = 3) {
    this.size = size;
    this.active = 0;
    this.backlog = []; // queued thunks: () => Promise
  }
  // Submit a unit of work. Runs now if a slot is free, else waits.
  submit(thunk) {
    if (this.active < this.size) {
      this._run(thunk);
    } else {
      this.backlog.push(thunk);
    }
  }
  _run(thunk) {
    this.active++;
    // When it settles (success OR failure), free the slot and pull backlog.
    Promise.resolve()
      .then(thunk)
      .catch(() => {}) // errors are handled by the caller's own wrapper
      .finally(() => {
        this.active--;
        if (this.backlog.length > 0) {
          this._run(this.backlog.shift());
        }
      });
  }
  idle() {
    return this.active === 0 && this.backlog.length === 0;
  }
}

// ---------- Clock: injected dependency (the testability lesson) ----------
// Core logic NEVER calls Date.now(). It reads clock.now(). In production you
// pass a RealClock; in tests a ManualClock you advance by hand -> deterministic.
class ManualClock {
  constructor(start = 0) {
    this._t = start;
  }
  now() {
    return this._t;
  }
  set(t) {
    this._t = t;
  }
  advance(dt) {
    this._t += dt;
  }
}

// ---------- Scheduler: orchestrates everything ----------
class Scheduler {
  constructor({ clock, pool, retryPolicy, log = console.log } = {}) {
    this.clock = clock;
    this.pool = pool;
    this.retryPolicy = retryPolicy;
    this.log = log;
    this.heap = new MinHeap();
    this.cancelled = new Set(); // ids marked cancelled -> skipped lazily
    this.deadLetter = []; // tasks that exhausted retries
    this.tasksById = new Map();
  }

  schedule(task) {
    task.status = TaskStatus.SCHEDULED;
    this.tasksById.set(task.id, task);
    this.heap.push(task);
    this.log(`[t=${this.clock.now()}] scheduled ${task.name} (${task.id}) for t=${task.nextRunTime}`);
    return task.id;
  }

  cancel(taskId) {
    // Lazy deletion: mark cancelled, skip it when it surfaces at the heap top.
    this.cancelled.add(taskId);
    const task = this.tasksById.get(taskId);
    if (task) task.status = TaskStatus.CANCELLED;
    this.log(`[t=${this.clock.now()}] cancelled ${taskId}`);
  }

  // Time until the next task is due (for a real event loop to sleep on).
  // null => nothing scheduled. <=0 => something is due right now.
  msUntilNextDue() {
    const top = this.heap.peek();
    if (!top) return null;
    return top.nextRunTime - this.clock.now();
  }

  // THE LOOP. Pops every task due at `now` and dispatches it, respecting
  // the worker-pool concurrency limit. Deterministic: caller controls `now`.
  tick(now) {
    this.clock.set(now);
    while (this.heap.size() > 0 && this.heap.peek().nextRunTime <= now) {
      const task = this.heap.pop();
      if (this.cancelled.has(task.id)) {
        this.log(`[t=${now}] skip ${task.name} (cancelled)`);
        continue;
      }
      this._dispatch(task, now);
    }
  }

  // Hand one due task to the pool (producer -> consumer boundary).
  _dispatch(task, now) {
    task.status = TaskStatus.RUNNING;
    this.pool.submit(async () => {
      const startedAt = this.clock.now();
      try {
        const result = await task.execute(); // invoke the Command
        this.log(`[t=${startedAt}] RAN ${task.name} attempt#${task.attempts + 1} -> ${result}`);
        this._onSuccess(task);
      } catch (err) {
        this.log(`[t=${startedAt}] FAILED ${task.name} attempt#${task.attempts + 1}: ${err.message}`);
        this._onFailure(task);
      }
    });
  }

  _onSuccess(task) {
    task.attempts = 0; // reset per successful run
    if (task.trigger.isRecurring()) {
      // Recurring: reschedule self at now + interval.
      const next = task.trigger.nextRunTime(this.clock.now());
      task.nextRunTime = next;
      task.status = TaskStatus.SCHEDULED;
      this.heap.push(task);
      this.log(`      -> ${task.name} reschedules for t=${next}`);
    } else {
      task.status = TaskStatus.COMPLETED;
    }
  }

  _onFailure(task) {
    task.attempts++;
    if (this.retryPolicy.shouldRetry(task.attempts)) {
      const backoff = this.retryPolicy.backoffMs(task.attempts);
      task.nextRunTime = this.clock.now() + backoff;
      task.status = TaskStatus.SCHEDULED;
      this.heap.push(task);
      this.log(`      -> retry ${task.name} in ${backoff}ms (at t=${task.nextRunTime})`);
    } else {
      task.status = TaskStatus.FAILED;
      this.deadLetter.push(task);
      this.log(`      -> DEAD-LETTER ${task.name} after ${task.attempts} attempts`);
    }
  }
}

// ============================================================
//  main() — DETERMINISTIC demo. No real sleeps: we advance a
//  ManualClock by hand and call tick() at each step.
// ============================================================
async function main() {
  const clock = new ManualClock(0);
  const pool = new WorkerPool(2); // at most 2 tasks run at once
  const retryPolicy = new RetryPolicy({ maxAttempts: 3, baseDelayMs: 100 });
  const sched = new Scheduler({ clock, pool, retryPolicy });

  // (1) A one-off task due at t=100.
  sched.schedule(
    new Task({
      name: 'send-welcome-email',
      nextRunTime: 100,
      priority: 1,
      run: async () => 'email sent',
    })
  );

  // (2) A recurring task every 50ms, first due at t=50.
  sched.schedule(
    new Task({
      name: 'heartbeat',
      nextRunTime: 50,
      run: async () => 'ping',
      trigger: new TriggerStrategy(50), // recurring
    })
  );

  // (3) A task that fails its first TWO runs, then succeeds. First due t=60.
  let flaky = 0;
  sched.schedule(
    new Task({
      name: 'charge-card',
      nextRunTime: 60,
      priority: 5, // high priority tie-breaker
      run: async () => {
        flaky++;
        if (flaky <= 2) throw new Error(`gateway timeout #${flaky}`);
        return 'charged';
      },
    })
  );

  // Drive time deterministically. Each step = "the event loop woke up at t".
  // A real runner would sleep msUntilNextDue() between ticks; we just advance.
  const steps = [50, 60, 100, 150, 160, 200, 260, 300, 360, 400];
  for (const t of steps) {
    sched.tick(t);
    // Let the microtask queue drain so async task bodies + pool backlog run
    // before we advance the clock again (keeps output ordered & deterministic).
    await Promise.resolve();
    await Promise.resolve();
  }

  console.log('\n--- final state ---');
  console.log('dead-letter count:', sched.deadLetter.length);
  console.log('heap still holds (recurring):', sched.heap.size(), 'task(s)');
}

main();
```

**What the demo proves when you run it:**

- **Ordering** — even though `send-welcome-email` (t=100) was scheduled *before* `heartbeat` (t=50), the heap fires `heartbeat` first. Order is by time, not insertion.
- **Recurrence** — `heartbeat` runs at 50, then reschedules itself for 100, 150, 200... You'll see the `reschedules for t=...` lines, and at the end the heap still holds it.
- **Retry + backoff** — `charge-card` fails at t≈60 and t≈160 (retry after 100ms), then succeeds at t≈360 (retry after 200ms). Watch the `retry ... in 100ms` / `in 200ms` lines: the delays double.
- **Concurrency limit** — with `WorkerPool(2)`, if three tasks ever come due in one tick, only two start immediately and the third waits for a free slot (the backlog).

---

## Design patterns used and WHY

| Pattern | Where | Why it's the right call |
|---------|-------|-------------------------|
| **Command** (recall [43](./43-pattern-command.md)) | `Task` with `execute()` | A task *is* a command — it packages arbitrary work behind a uniform `execute()` interface. The scheduler stores, queues, retries, and invokes tasks without knowing what any of them actually do. That decoupling is exactly why the scheduler can be generic. |
| **Producer-Consumer** (recall [51](./51-pattern-producer-consumer.md)) | Scheduler → `WorkerPool` | The scheduler *produces* due tasks; the pool of workers *consumes* and runs them. The pool is the bounded buffer between them — it decouples "how fast tasks come due" from "how fast we can run them," and caps concurrency (recall [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md)) so a burst can't overwhelm downstream. |
| **Strategy** | `TriggerStrategy`, `RetryPolicy` | *When* a task recurs and *how* it backs off are algorithms that vary independently of the scheduler. Making them injectable objects means adding cron (a new `TriggerStrategy`) or a different backoff (a new `RetryPolicy`) requires zero changes to the loop. |
| **Dependency Injection** | The `clock` | Time is a dependency. Injecting a clock turns time-based code — the hardest thing to test — into something you drive by hand. This is the single most transferable lesson here. |

**The concurrency angle (recall 52):** even though Node is single-threaded, tasks are `async` and interleave via the event loop. The `WorkerPool` is a **concurrency limiter** over in-flight promises — the async-world equivalent of a fixed thread pool — so "run at most N at once" holds without real threads.

---

## Extensions the interviewer will ask for

The mark of a good design is how gracefully it absorbs "now add X." Each of these slots into the existing seams:

- **"Add cron expressions."** Add a `CronTrigger extends TriggerStrategy` whose `nextRunTime(now)` parses a cron string (`"0 * * * *"`) and returns the next matching instant. The scheduler already asks the trigger for the next time — nothing else changes. This is why trigger was a Strategy.

- **"Persist tasks so they survive a restart."** Introduce a `TaskStore` (SQL table or Redis ZSET scored by `nextRunTime`). `schedule()` writes through to the store; on boot, the scheduler loads all `SCHEDULED` tasks back into the heap. The heap becomes an in-memory index over durable state. (The runnable functions themselves need a registry — you persist a *task type name* + args, not a closure.)

- **"Make it distributed — N nodes must not double-run a task."** This is the big one. Move the source of truth to a shared store (Redis/DB). Then either (a) **leader election** (recall [83 — Leader Election]) so exactly one node owns the schedule and dispatches, or (b) an **atomic claim / distributed lock** (recall [84 — Distributed Locking]) where each node does `SELECT ... FOR UPDATE SKIP LOCKED` or Redis `SET id node NX PX ttl` before running a due task — whoever wins runs it, the rest skip. The single-node design becomes a **distributed cron / job-queue service** (BullMQ, Quartz cluster, Celery beat) essentially by swapping the in-memory heap for a shared, locked one.

- **"Add task dependencies / DAGs."** Give each task a `dependsOn: [taskId...]`. A task only becomes *eligible* (pushed onto the heap) once all its dependencies are `COMPLETED`. Keep a waiting map keyed by dependency; on each completion, decrement dependents' counters and schedule any that hit zero. The heap/loop is unchanged — you've only added an eligibility gate in front of it.

- **"Priority-based preemption."** Today priority is only a tie-breaker for tasks due at the same instant. True preemption (a high-priority task should bump a running low-priority one) needs cancellable/cooperative task execution — pass an `AbortSignal` into `execute()`, and when a high-priority task arrives with no free worker, signal the lowest-priority running task to yield and requeue it. This is a real complexity jump; call it out as such.

Each extension lands on an existing seam — a new Strategy, a store behind the same interface, a lock around the same dispatch, a gate before the same heap. That's the payoff of separating *ordering* (heap), *concurrency* (pool), and *trigger/retry config* (task).

---

## Connected topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Previous** | [51 — Producer-Consumer](./51-pattern-producer-consumer.md) | The scheduler→worker-pool dispatch IS producer-consumer; read it first to see the pattern in isolation. |
| **Next** | [67 — Message Queues](./67-message-queues.md) | Scale this scheduler out and you get a distributed job queue with delayed messages, retries, and dead-letter queues — the very features here, made durable. |
| **Related** | [43 — Command](./43-pattern-command.md) | A `Task` is a Command object with `execute()`; this is that pattern applied. |
| **Related** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) | The bounded worker pool and "at most N in-flight" limiter are concurrency control — the theory behind the pool. |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (nouns → behaviours → class diagram → code → patterns → extensions) used to structure this whole case study. |
