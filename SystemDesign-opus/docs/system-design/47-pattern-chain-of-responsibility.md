# 47 — Chain of Responsibility Pattern

## Category: LLD Patterns

---

## What is this?

**Chain of Responsibility (CoR)** lets you pass a request along a line of handler objects. Each handler looks at the request and decides one of two things: *"I'll deal with this"* or *"not mine — pass it down the line."* The sender has no idea which handler will end up doing the work, and doesn't care.

Think of a **support ticket**. You email support. An L1 agent reads it — password reset? Handled. Billing dispute over ₹50,000? *Not my call* — escalate to L2. L2 can't authorise a refund that large either — escalate to the Manager. The customer sent **one** email; they never had to know the org chart.

---

## Why does it matter?

Without CoR you end up with the same corrosive pattern every time: one giant function that knows about every possible handler.

```javascript
// The thing CoR exists to kill:
function handleRequest(req) {
  if (req.amount <= 1000)        return teamLead.approve(req);
  else if (req.amount <= 10000)  return manager.approve(req);
  else if (req.amount <= 100000) return director.approve(req);
  else                           return cfo.approve(req);
}
```

That function is **coupled to every approver in the company.** Add a "VP" tier between manager and director and you edit this function. Change the CFO's limit and you edit this function. It knows the org chart, the limits, and the order — three things that change independently.

**Interview angle:** CoR is a top-5 behavioural pattern in LLD rounds, and it has the single best real-world hook in the whole catalogue: **Express middleware is Chain of Responsibility.** If you can say "`app.use((req, res, next) => ...)` — `next()` *is* the chain link" and then implement a mini middleware engine on the whiteboard, you've demonstrated pattern knowledge *and* framework understanding in one breath. Interviewers love that.

**Real-work angle:** Every request pipeline you've ever touched is CoR — Express/Koa middleware, Redux middleware, ASP.NET pipelines, Netty handlers, servlet filters, logging appenders, event bubbling in the DOM, approval workflows, spam filter chains.

---

## The core idea — explained simply

### The Support Desk Analogy

You raise a ticket: *"I was charged twice, please refund ₹50,000."*

```
You ──▶ [L1 Agent] ──▶ [L2 Engineer] ──▶ [Support Manager] ──▶ [Director]
         can refund      can refund         can refund           can refund
         up to ₹1,000    up to ₹10,000      up to ₹100,000       any amount
```

- **L1** reads it. ₹50,000 > her ₹1,000 limit. She doesn't argue, doesn't apologise, doesn't invent a policy. She **passes it to the next person in the chain.**
- **L2** — ₹50,000 > ₹10,000. Passes it on.
- **Manager** — ₹50,000 ≤ ₹100,000. **Handled.** The chain stops here. The Director never hears about it.

Three properties fall out of this, and they *are* the pattern:

1. **You (the sender) know only the first handler.** You emailed support@. You did not need the Manager's phone number.
2. **Each handler knows only its successor.** L1 knows "if I can't do it, give it to L2." She doesn't know the Director exists.
3. **The chain is data, not code.** HR can reorder the escalation ladder without rewriting L1's job description.

| Support desk | Chain of Responsibility | In code |
|---|---|---|
| The ticket | **The request** | `{ amount: 50000, reason: '...' }` |
| L1 / L2 / Manager / Director | **Concrete handlers** | `class TeamLead extends Approver` |
| "my refund limit" | The handler's **guard condition** | `if (req.amount <= this.limit)` |
| "escalate to my boss" | The **`next` link** | `this.next.handle(req)` |
| The escalation ladder | **The chain** | `l1.setNext(l2).setNext(mgr)` |
| Nobody in the company can approve it | Request **falls off the end** | `next === null` → must be handled! |
| support@ inbox | The **client's single entry point** | `chain.handle(req)` |

**The one thing beginners get wrong:** they think CoR means "exactly one handler handles it." It doesn't. There are **two flavours**, and both are legitimate:

- **Flavour A — "first match wins":** the chain stops at whoever handles it. (Approvals, error handlers, spam filters.)
- **Flavour B — "everybody runs":** every handler does its bit and then explicitly passes control on. (Express middleware, logging pipelines, validation chains.)

Same structure. The difference is whether a handler *returns* or *calls next*. We build both below.

---

## Key concepts inside this topic

### 1. The problem it solves — the painful version first

```javascript
// ============ BAD — the god-function that knows everyone ============
class ExpenseService {
  approve(request) {
    // Coupled to: every approver, every limit, and the ORDER of approvers.
    if (request.amount <= 1000) {
      console.log(`TeamLead approved ₹${request.amount}`);
      return true;
    } else if (request.amount <= 10000) {
      console.log(`Manager approved ₹${request.amount}`);
      return true;
    } else if (request.amount <= 100000) {
      console.log(`Director approved ₹${request.amount}`);
      return true;
    } else if (request.amount <= 1000000) {
      console.log(`CFO approved ₹${request.amount}`);
      return true;
    }
    return false; // silently rejected. Why? Who knows.
  }
}
```

What's wrong:

| Symptom | Root cause |
|---|---|
| Adding a "VP" tier means editing `approve()` | The dispatcher knows every handler → **Open/Closed violation** |
| The order of approvers is hard-coded in `else if` order | Chain order is **code**, not configuration |
| The engineering-expenses chain and the marketing chain differ → copy-paste the whole method | You cannot **reuse one handler in two chains** |
| You cannot unit-test "does Manager approve ₹5,000?" | Handlers aren't objects, they're branches |
| `return false` at the end | The **unhandled** case is an afterthought |

### 2. The structure — participants

```
Handler (abstract)  → declares handle(request); holds `next`; provides setNext()
ConcreteHandler     → either handles the request, or delegates to `next`
Client              → builds the chain, then calls handle() on the FIRST handler only
```

```
┌──────────────────────────────────────────┐
│              Handler  (abstract)          │
│                                          │
│  # next : Handler | null                 │
│                                          │
│  + setNext(h: Handler) : Handler  ◀──────┼── returns the handler you PASSED IN,
│  + handle(req) : any                     │   so chains build fluently
│  # passToNext(req) : any                 │
└────────────────────▲─────────────────────┘
                     │ extends
     ┌───────────────┼──────────────┬────────────────┐
     │               │              │                │
┌────┴─────┐   ┌─────┴─────┐  ┌─────┴──────┐  ┌──────┴──────┐
│ TeamLead │──▶│  Manager  │─▶│  Director  │─▶│     CFO     │──▶ null
│ limit 1k │   │ limit 10k │  │ limit 100k │  │ limit 1000k │
└──────────┘   └───────────┘  └────────────┘  └─────────────┘
     ▲
     │ client calls handle() HERE, and only here
  ┌──┴────┐
  │Client │
  └───────┘
```

The `setNext()` trick — returning the handler you were *given*, not `this` — is what makes chains read like a sentence:

```javascript
teamLead.setNext(manager).setNext(director).setNext(cfo);
//       └─ returns manager ─┘└─ returns director ─┘
// The chain reads left-to-right in the exact order requests flow. Build it once, read it forever.
```

### 3. Example (a) — the approval chain ("first match wins")

```javascript
// ============================================================================
// 47 — Chain of Responsibility: expense approval chain
// Run with: node approval-chain.js
// ============================================================================

class UnhandledRequestError extends Error {
  constructor(request) {
    super(`No handler in the chain could approve ₹${request.amount}`);
    this.name = 'UnhandledRequestError';
    this.request = request;
  }
}

// --- The abstract Handler -----------------------------------------------
class Approver {
  constructor(name, limit) {
    this.name = name;
    this.limit = limit;
    this.next = null;
  }

  // Returns the NEXT handler (not `this`) so calls can be chained fluently.
  setNext(handler) {
    this.next = handler;
    return handler;
  }

  handle(request) {
    if (this.canApprove(request)) {
      return this.approve(request);
    }
    return this.passToNext(request);
  }

  // The single place the "fell off the end" case is handled. Loudly.
  passToNext(request) {
    if (!this.next) throw new UnhandledRequestError(request);
    return this.next.handle(request);
  }

  // Subclasses override these two. Default: approve anything within my limit.
  canApprove(request) { return request.amount <= this.limit; }
  approve(request) {
    return { approvedBy: this.name, amount: request.amount };
  }
}

// --- Concrete handlers --------------------------------------------------
class TeamLead extends Approver {
  constructor() { super('TeamLead', 1_000); }
}

class Manager extends Approver {
  constructor() { super('Manager', 10_000); }
}

class Director extends Approver {
  constructor() { super('Director', 100_000); }
  // Extra rule that ONLY the Director has. Note: no other class changed.
  canApprove(request) {
    return super.canApprove(request) && request.category !== 'legal';
  }
}

class CFO extends Approver {
  constructor() { super('CFO', Infinity); }
}

// --- Client: builds the chain, then knows only its head -----------------
function buildApprovalChain() {
  const lead = new TeamLead();
  lead.setNext(new Manager()).setNext(new Director()).setNext(new CFO());
  return lead; // the client only ever holds the HEAD
}

function demoApprovals() {
  const chain = buildApprovalChain();
  const requests = [
    { amount: 500,     category: 'travel' },
    { amount: 7_500,   category: 'travel' },
    { amount: 60_000,  category: 'hardware' },
    { amount: 60_000,  category: 'legal' },     // Director refuses -> escalates to CFO
    { amount: 5_000_000, category: 'travel' },  // CFO has Infinity limit -> approves
  ];

  for (const req of requests) {
    try {
      const result = chain.handle(req);
      console.log(`₹${req.amount} (${req.category}) -> approved by ${result.approvedBy}`);
    } catch (err) {
      console.log(`₹${req.amount} (${req.category}) -> ✘ ${err.message}`);
    }
  }
}

demoApprovals();
// ₹500 (travel) -> approved by TeamLead
// ₹7500 (travel) -> approved by Manager
// ₹60000 (hardware) -> approved by Director
// ₹60000 (legal) -> approved by CFO        <-- Director's extra rule pushed it up
// ₹5000000 (travel) -> approved by CFO
```

Notice what adding a **VP** tier costs you: one new class, one line in `buildApprovalChain()`. `TeamLead`, `Manager`, `Director`, `CFO` are untouched. That is the Open/Closed Principle ([17](./15-solid-open-closed.md)) paying rent.

### 4. Example (b) — the processing pipeline ("everybody runs")

Now the second flavour. Here **no handler "wins"** — each one does its job and passes the request onward. Any handler can *stop* the chain by not calling next (that's how auth rejects you).

```javascript
// ============================================================================
// A request pipeline: auth -> validate -> rate-limit -> log -> handle
// Same pattern, different intent: EVERY handler runs, in order.
// ============================================================================

class PipelineHandler {
  constructor() { this.next = null; }

  setNext(handler) {
    this.next = handler;
    return handler;
  }

  // Subclasses override this.
  async handle(_ctx) { throw new Error('handle() not implemented'); }

  // Explicitly pass control down. Falling off the end is fine here —
  // it just means the pipeline completed.
  async passToNext(ctx) {
    if (this.next) return this.next.handle(ctx);
    return ctx;
  }
}

class AuthHandler extends PipelineHandler {
  async handle(ctx) {
    if (!ctx.headers.authorization) {
      ctx.response = { status: 401, body: 'Unauthorized' };
      return ctx;                 // STOP the chain. Nothing downstream runs.
    }
    ctx.user = { id: 'u_42', role: 'admin' };   // enrich, then continue
    ctx.trace.push('auth');
    return this.passToNext(ctx);
  }
}

class ValidationHandler extends PipelineHandler {
  async handle(ctx) {
    if (!ctx.body?.title) {
      ctx.response = { status: 400, body: 'title is required' };
      return ctx;                 // STOP
    }
    ctx.trace.push('validate');
    return this.passToNext(ctx);
  }
}

class RateLimitHandler extends PipelineHandler {
  constructor(maxPerWindow = 3) {
    super();
    this.max = maxPerWindow;
    this.counts = new Map();      // userId -> hits in the current window
  }
  async handle(ctx) {
    const hits = (this.counts.get(ctx.user.id) ?? 0) + 1;
    this.counts.set(ctx.user.id, hits);
    if (hits > this.max) {
      ctx.response = { status: 429, body: 'Too Many Requests' };
      return ctx;                 // STOP
    }
    ctx.trace.push(`rate-limit(${hits}/${this.max})`);
    return this.passToNext(ctx);
  }
}

class LoggingHandler extends PipelineHandler {
  async handle(ctx) {
    const start = Date.now();
    ctx.trace.push('log:start');
    const result = await this.passToNext(ctx);   // run the REST of the chain first
    result.trace.push(`log:end(${Date.now() - start}ms)`);
    return result;                               // then do work on the way BACK
  }
}

class BusinessHandler extends PipelineHandler {
  async handle(ctx) {
    ctx.trace.push('handle');
    ctx.response = { status: 201, body: { id: 'post_1', title: ctx.body.title } };
    return this.passToNext(ctx);   // next is null -> pipeline ends
  }
}

async function demoPipeline() {
  const auth = new AuthHandler();
  auth.setNext(new ValidationHandler())
      .setNext(new RateLimitHandler(2))
      .setNext(new LoggingHandler())
      .setNext(new BusinessHandler());

  const makeCtx = (overrides = {}) => ({
    headers: { authorization: 'Bearer abc' },
    body: { title: 'Hello CoR' },
    trace: [],
    response: null,
    ...overrides,
  });

  console.log(await auth.handle(makeCtx()));                          // 201, full trace
  console.log(await auth.handle(makeCtx()));                          // 201, 2/2
  console.log(await auth.handle(makeCtx()));                          // 429 — stopped at rate-limit
  console.log(await auth.handle(makeCtx({ headers: {} })));           // 401 — stopped at auth
  console.log(await auth.handle(makeCtx({ body: {} })));              // 400 — stopped at validate
}

demoPipeline();
```

Two things worth staring at:

- **`LoggingHandler` awaits `passToNext()` and then keeps working.** That's the "onion" shape — code before `next` runs on the way in, code after `next` runs on the way out. That is *exactly* how Express timing/logging middleware works, and it's why the pattern maps so cleanly onto middleware.
- **Order is behaviour.** Put `RateLimitHandler` before `AuthHandler` and it crashes on `ctx.user.id` — the user doesn't exist yet. Chain order is a real coupling, and we'll name it as a risk below.

### 5. A real Node.js ecosystem example — Express middleware IS this pattern

You have written Chain of Responsibility hundreds of times without knowing it:

```javascript
import express from 'express';
const app = express();

// Each of these is a HANDLER. `next` is the LINK to the next handler.
app.use((req, res, next) => {              // AuthHandler
  if (!req.headers.authorization) return res.status(401).send('Unauthorized');
  req.user = { id: 'u_42' };
  next();                                   // ◀── "not mine to finish — pass it on"
});

app.use((req, res, next) => {              // LoggingHandler (the onion shape)
  const start = Date.now();
  res.on('finish', () => console.log(`${req.method} ${req.url} ${Date.now() - start}ms`));
  next();
});

app.use(express.json());                   // ValidationHandler (parses + may 400)

app.post('/posts', (req, res) => {         // BusinessHandler — the chain ENDS here
  res.status(201).json({ id: 'post_1', title: req.body.title });
  // note: no next() call. This handler HANDLED it. The chain stops.
});

// The 404 handler: what "falling off the end of the chain" looks like in Express.
app.use((req, res) => res.status(404).send('Not Found'));
```

The mapping is one-to-one:

| Express | Chain of Responsibility |
|---|---|
| `app.use(fn)` | `handler.setNext(fn)` — appending to the chain |
| The `next` argument | The `this.next.handle(...)` link |
| Calling `next()` | Delegating to the successor |
| **Not** calling `next()` (you sent a response) | **Handling** the request; chain stops |
| `next(err)` | Skipping to the error-handling sub-chain |
| The final `app.use` 404 handler | The "fell off the end, nobody handled it" case |
| Middleware order in the file | The chain's link order |

Koa is the same idea with an explicit onion: `async (ctx, next) => { before(); await next(); after(); }`.

**Now build the engine yourself.** This is the whiteboard moment. ~20 lines:

```javascript
// ============ A mini Express-style middleware engine ============
class MiniApp {
  constructor() { this.stack = []; }

  use(fn) { this.stack.push(fn); return this; }   // append a link

  // Build the chain lazily by index. dispatch(i) IS the `next` handed to
  // middleware i — calling it runs middleware i+1. That's the whole trick.
  async run(req, res) {
    const dispatch = async (i) => {
      const fn = this.stack[i];
      if (!fn) return res.end?.(404, 'Not Found');    // fell off the end
      let called = false;
      const next = async (err) => {
        if (called) throw new Error('next() called twice in the same middleware');
        called = true;
        if (err) throw err;
        return dispatch(i + 1);                       // ◀── the chain link
      };
      return fn(req, res, next);
    };
    return dispatch(0);
  }
}

// --- it really runs ---
const app = new MiniApp();
app.use((req, res, next) => { req.trace = ['auth']; next(); })
   .use(async (req, res, next) => { const t = Date.now(); await next(); req.trace.push(`${Date.now() - t}ms`); })
   .use((req, res, next) => { req.body ? next() : res.end(400, 'bad body'); })
   .use((req, res) => res.end(201, `created: ${req.body.title}`));

const res = { end: (status, body) => console.log(status, body, '| trace:', req.trace) };
const req = { body: { title: 'Hello' } };
app.run(req, res);   // 201 created: Hello | trace: [ 'auth', '0ms' ]
```

That `dispatch(i)` closure is the *entire* insight: **`next` is not a magic keyword — it's just a function that invokes the next handler.** Express's real router is ~200 more lines of path-matching and error routing on top of exactly this.

### 6. When NOT to use it / how it's abused

| Situation | Verdict |
|---|---|
| 2 handlers, fixed order, will never change | **Don't.** Two function calls. The chain is ceremony. |
| The dispatch is a pure lookup (`type → handler`) with no fallthrough | **Don't.** That's a `Map`, and it's O(1). A chain is O(n) and answers a question you never asked. |
| You need to *guarantee* someone handles it | Use CoR but **throw** at the end of the chain (as we do). Never `return false` quietly. |
| Handlers must run in a fixed sequence, all of them | Yes — the pipeline flavour. |
| A request might match zero, one, or several handlers, and the set changes | Yes — this is CoR's home turf. |
| 30 middleware deep, and a request mysteriously 200s with an empty body | **This is the abuse.** Someone forgot `next()`. Debugging is now archaeology. |

**The three real risks, named:**

1. **Falling off the end unhandled.** The chain is *permitted* to not handle a request. That's a feature for middleware and a catastrophe for approvals. **Fix:** end every "must be handled" chain with either a throw (our `UnhandledRequestError`) or a mandatory catch-all terminal handler (Express's 404 middleware). Decide which, explicitly.

2. **Chain-order coupling.** `RateLimitHandler` needs `ctx.user`, which only exists after `AuthHandler`. That dependency is invisible — it lives in the *ordering*, not in any type signature. **Fix:** make handlers assert their preconditions (`if (!ctx.user) throw new Error('RateLimit requires Auth upstream')`) so a mis-ordered chain fails at the first request, loudly, instead of silently rate-limiting `undefined`.

3. **Debugging a long chain.** With 20 handlers, "why did my request 403?" means stepping through 20 frames. **Fix:** the `ctx.trace` array in our pipeline example. Push a breadcrumb in every handler. In production, that becomes a span per handler in your tracing tool. Cheap, and it turns a 20-frame debug session into reading one array.

### 7. Related patterns and how they differ — the #1 interview follow-up

**CoR vs Decorator — structurally near-identical, different intent.**

Both wrap objects in a linked sequence, each holding a reference to the next. But:

| | **Decorator** ([35](./35-pattern-decorator.md)) | **Chain of Responsibility** (this doc) |
|---|---|---|
| **What's in the chain?** | Objects that all implement the **same interface as the thing they wrap** | Handlers implementing a **Handler** interface — different from the request's type |
| **Does every link run?** | **Yes, always.** A decorator *must* call the wrapped object; that's its contract. | **Not necessarily.** A handler may stop the chain, or may not handle it at all. |
| **Purpose** | **Add behaviour** to one object (compression + encryption + buffering on a stream) | **Find/route** a handler for a request (or run a pipeline) |
| **Can a link short-circuit?** | No — that would break the wrapped object's contract | **Yes** — that's the whole point (401 stops everything) |
| **The result** | An enhanced object you keep and call many times | A processed request; the chain is infrastructure |

The interview one-liner: **"Decorator adds behaviour and always delegates; CoR routes a request and may stop. Same links, opposite promises."**

| Pattern | Relationship to CoR |
|---|---|
| [35 — Decorator](./35-pattern-decorator.md) | Same linked structure. See table above. The #1 follow-up. |
| [43 — Command](./43-pattern-command.md) | The *request* travelling the chain is often a Command object. CoR routes it; Command encapsulates it. |
| [48 — Mediator](./48-pattern-mediator.md) | The opposite topology: CoR is a **line** (each knows only its successor); Mediator is a **hub** (everyone knows one centre). |
| [38 — Composite](./38-pattern-composite.md) | Combine them: a request bubbles *up* a Composite tree, each parent a potential handler. This is literally DOM event bubbling. |
| [46 — State](./46-pattern-state.md) | Both are behavioural and both delegate. State delegates to **one** object that swaps itself; CoR delegates **along a line**. |

---

## Visual / Diagram description

### Diagram 1: Flow through the chain (first-match-wins)

```
   CLIENT
     │  handle({ amount: 60000, category: 'legal' })
     ▼
 ┌──────────────┐  60000 > 1000?      ┌──────────────┐  60000 > 10000?
 │  TeamLead    │  YES, not mine      │   Manager    │  YES, not mine
 │  limit 1,000 │ ─────────────────▶  │ limit 10,000 │ ─────────────────┐
 └──────────────┘   passToNext()      └──────────────┘   passToNext()   │
                                                                        │
     ┌──────────────────────────────────────────────────────────────────┘
     ▼
 ┌────────────────┐  60000 <= 100000 BUT category === 'legal'   ┌────────────┐
 │   Director     │  → my extra rule says no                    │    CFO     │
 │ limit 100,000  │ ──────────────────────────────────────────▶ │ limit ∞    │
 │ (no legal)     │              passToNext()                   └─────┬──────┘
 └────────────────┘                                                   │
                                                                      │ HANDLED ✔
                                                          ┌───────────▼──────────┐
                                                          │ { approvedBy: 'CFO' } │
                                                          └───────────┬──────────┘
                                                                      │
     ◀────────────────── returns straight back to CLIENT ─────────────┘
                (each frame just returns its child's result upward)

   If CFO had also declined and next === null:
       throw new UnhandledRequestError   ◀── the "fell off the end" case, made LOUD
```

**What it shows:** the client called exactly one method on exactly one object (`TeamLead`). It has no reference to the CFO, no knowledge that a Director exists, and no idea an escalation happened. Adding a VP between Manager and Director changes *one line* in the chain-building function and nothing else.

### Diagram 2: The onion — how `next()` works in middleware

```
       REQUEST IN                                          RESPONSE OUT
           │                                                     ▲
           ▼                                                     │
 ┌─────────────────────────────────────────────────────────────────────┐
 │ LOGGING            const t = Date.now();                            │
 │                    await next();  ─────┐         ┌── log(elapsed)   │
 │  ┌──────────────────────────────────────────────────────────────┐   │
 │  │ AUTH           if (!token) return 401  ◀── SHORT CIRCUIT      │   │
 │  │                req.user = ...                                 │   │
 │  │                await next(); ────┐    ┌───── (nothing after)  │   │
 │  │  ┌────────────────────────────────────────────────────────┐   │   │
 │  │  │ RATE LIMIT   if (hits > max) return 429  ◀── SHORT CIRC │   │   │
 │  │  │              await next(); ──┐    ┌──── (nothing after)  │   │   │
 │  │  │  ┌───────────────────────────────────────────────────┐  │   │   │
 │  │  │  │ BUSINESS HANDLER                                   │  │   │   │
 │  │  │  │   res.status(201).json({...})    ◀── the CORE      │  │   │   │
 │  │  │  │   (does NOT call next → chain ends here)           │  │   │   │
 │  │  │  └───────────────────────────────────────────────────┘  │   │   │
 │  │  └────────────────────────────────────────────────────────┘   │   │
 │  └──────────────────────────────────────────────────────────────┘   │
 └─────────────────────────────────────────────────────────────────────┘

  Code BEFORE `await next()`  runs on the way IN  (auth, validate, rate-limit)
  Code AFTER  `await next()`  runs on the way OUT (timing, response headers, metrics)
  A handler that RETURNS without calling next()  → short-circuits everything inside it
  Nothing matched at the core                    → falls off the end → 404
```

**What it shows:** why `LoggingHandler` can measure the duration of everything downstream — it holds the stack frame open across `await next()`. And why a missing `next()` call is such a nasty bug: the request just *stops*, mid-onion, with no error.

---

## Real world examples

### 1. Express.js — the middleware stack

Express's router keeps an array of `{ path, method, handler }` layers and walks it with an index-based `next()` closure — architecturally the same as the `MiniApp` above. Everything you use is a link in that chain: `express.json()` (body parsing), `cors()`, `helmet()` (security headers), `morgan()` (logging), `passport.authenticate()` (auth). Error-handling middleware is a *parallel* sub-chain: Express detects a 4-argument signature `(err, req, res, next)` and only routes to those when something calls `next(err)`. That's CoR with two chains and a switch between them.

### 2. Koa — the explicit onion

Koa made the "on the way in / on the way out" shape first-class: `app.use(async (ctx, next) => { /* in */ await next(); /* out */ })`. Its `koa-compose` function is ~15 lines and is the cleanest published implementation of Chain of Responsibility in the JS ecosystem — worth reading in full.

### 3. The DOM — event bubbling

A click on a `<button>` inside a `<div>` inside `<body>` fires the handler on the button, then the div, then body, then document. Each ancestor is a handler in the chain. `event.stopPropagation()` is a handler saying **"I handled it — stop the chain."** That's CoR walking up a Composite tree ([38](./38-pattern-composite.md)).

### 4. Redux middleware

`applyMiddleware(thunk, logger, apiMiddleware)` composes middleware into a chain where each has the signature `store => next => action => ...`. Every action passes through every link; each can transform, log, swallow, or forward it. Same pattern, different domain (actions instead of HTTP requests).

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| **Sender is decoupled from receiver** | The client knows only the head of the chain |
| **Open/Closed** | New handler = new class + one line of wiring; existing handlers untouched |
| **Single Responsibility** | Each handler does one thing (auth, or validation, or logging) |
| **The chain is data** | Reorder, insert, remove, or build different chains per route/tenant at runtime |
| **Handlers are reusable** | The same `RateLimitHandler` sits in five different chains |
| **Trivially testable** | One handler + a fake `next` = a complete unit test |

| Cons | The price you pay |
|---|---|
| **No delivery guarantee** | A request can fall off the end unhandled. You must design for it. |
| **Order is invisible coupling** | Rate-limit before auth = crash. No type checks this. |
| **Hard to debug** | 20 links deep, "which one swallowed my request?" needs tracing/breadcrumbs |
| **Runtime cost** | O(n) walk + n stack frames per request. Irrelevant at n=10, real at n=500 |
| **Easy to abuse** | Forgetting `next()` hangs the request silently — the classic Express bug |
| **Indirection** | Reading the wiring code is now required to understand what a request does |

**The sweet spot:** Use CoR when you have **3+ independent handlers**, the **set or order can change**, and it's genuinely fine (or genuinely designed for) that a request might be handled by any of them, several of them, or none. If the mapping is a fixed lookup (`event.type → handler`), use a `Map`. If you have exactly two steps that will never change, just call them. And **always** decide, in writing, what happens when the request reaches the end: throw, or a terminal catch-all. Never `undefined`.

---

## Common interview questions on this topic

### Q1: "Give me a real-world example of Chain of Responsibility."
**Hint:** Lead with **Express middleware** — `app.use((req, res, next) => ...)`, where `next()` is the link and *not* calling `next()` means "I handled it." Then note the two flavours: middleware is the "everybody runs" flavour; an approval ladder or an exception handler is the "first match wins" flavour. Bonus: DOM event bubbling, with `stopPropagation()` as the chain-stopper.

### Q2: "How is this different from Decorator? They look identical."
**Hint:** Structure is the same (linked wrappers); the *promise* is opposite. A **Decorator must always delegate** — it adds behaviour to an object while preserving its interface. A **CoR handler may stop the chain** — it's routing a request, and short-circuiting is the feature. Also: decorators share the interface of the object they wrap; handlers share a `Handler` interface that has nothing to do with the request's type.

### Q3: "What if no handler handles the request?"
**Hint:** Name it as a **design decision**, not an accident. Two answers: (a) **throw** at the end of the chain (`UnhandledRequestError`) when the domain requires an owner — expense approvals must be approved or explicitly denied; (b) install a **terminal catch-all** handler that always handles — that's Express's 404 middleware. What you must *never* do is return `undefined`/`false` and let the caller guess.

### Q4: "Why does `setNext()` return the next handler instead of `this`?"
**Hint:** So chains build fluently in reading order: `a.setNext(b).setNext(c)` wires a→b→c. Returning `this` would give you `a.setNext(b).setNext(c)` meaning "a's next is b, then a's next is c" — silently overwriting the link. It's a small API decision that prevents a real bug.

### Q5: "Your chain has 20 handlers and one request returns a 403. How do you debug it?"
**Hint:** Breadcrumbs. Push the handler name onto a `ctx.trace` array (or open an OpenTelemetry span) in every handler. Then the response's trace *is* the answer — you see exactly where the chain stopped. Say the general principle: **in a decoupled chain, observability is not optional, it's the thing you traded for the decoupling.**

### Q6: "Implement Express's `next()` from scratch."
**Hint:** An array of middleware plus an index-based recursive dispatcher. `dispatch(i)` looks up `stack[i]`, and the `next` it hands to that middleware is a closure calling `dispatch(i + 1)`. Guard against double-`next()` with a `called` flag. Terminal case: `stack[i]` is undefined → 404. That's the whole engine in ~15 lines (see the `MiniApp` above).

---

## Practice exercise

### Build a content-moderation chain (30–40 min)

You're moderating user-submitted comments. Build **both flavours** of CoR against the same domain.

**Part A — the "everybody runs" pipeline (20 min).**
Build a pipeline where each handler enriches a `ctx = { comment, user, flags: [], score: 0, blocked: false, trace: [] }`:

1. `ProfanityHandler` — scans for banned words; pushes `'profanity'` to `flags`, adds 40 to `score`.
2. `SpamLinkHandler` — counts URLs; more than 2 → flag `'spam'`, add 30.
3. `NewAccountHandler` — if `user.ageInDays < 7`, add 20 to `score`.
4. `RateLimitHandler` — more than 5 comments in the last minute → **short-circuit**: set `blocked = true` and *do not* call next.
5. `ScoringHandler` (terminal) — `score >= 50` → `ctx.decision = 'REJECT'`, `>= 25` → `'REVIEW'`, else `'APPROVE'`.

Every handler must push its name to `ctx.trace`.

**Part B — the "first match wins" chain (10 min).**
Now build an **appeal** chain: a rejected comment is appealed. `AutoModerator` (can only overturn score < 30) → `HumanModerator` (< 60) → `TrustSafetyLead` (anything). If nobody can overturn it, **throw** `UnhandledRequestError`.

**Part C — prove the risks are real (10 min).**
1. Deliberately put `RateLimitHandler` **first**, before the handlers that populate `ctx.user`. Show what breaks. Then add a precondition assert to that handler so it fails loudly instead.
2. Remove the terminal `ScoringHandler` and show a comment falling off the end with `ctx.decision === undefined`. Fix it by throwing.
3. Print `ctx.trace` for a rate-limited comment and show it stops early.

**Deliverable:** one runnable `moderation.js` with a `main()` that runs 4 comments (clean, profane, spammy-from-new-account, rate-limited) through both chains and prints the trace for each.

---

## Quick reference cheat sheet

- **Chain of Responsibility** = pass a request along a line of handlers until one handles it (or all of them process it).
- **The sender knows only the head** of the chain. Each handler knows only its **successor**. Nobody knows the whole chain.
- **`setNext(h)` returns `h`, not `this`** — that's what makes `a.setNext(b).setNext(c)` build a→b→c fluently.
- **Two flavours:** *first-match-wins* (approvals, error handlers — a handler returns and stops) and *everybody-runs* (middleware, pipelines — each handler calls `next`).
- **Express middleware IS this pattern.** `next()` is the link; not calling it means "I handled it"; the final `app.use` 404 is "fell off the end."
- **The onion:** code before `await next()` runs on the way in; code after runs on the way out. That's how timing middleware works.
- **`next` is not magic** — it's a closure that calls `dispatch(i + 1)` over an array. You can write the engine in 15 lines.
- **Design the end of the chain explicitly:** throw an `UnhandledRequestError`, or install a terminal catch-all. Never return `undefined`.
- **Chain order is invisible coupling** — rate-limit before auth crashes. Make handlers assert their preconditions.
- **Long chains need breadcrumbs** — a `ctx.trace` array (or a tracing span per handler) turns debugging from archaeology into reading a list.
- **Adding a handler = one class + one wiring line.** Existing handlers never change. That's the Open/Closed payoff.
- **CoR vs Decorator:** same links, opposite promises. Decorator always delegates and adds behaviour; CoR may short-circuit and routes a request.
- **CoR vs Mediator:** CoR is a **line**, Mediator is a **hub**.
- **Don't use it** for a fixed 2-step sequence, or when the dispatch is a plain `type → handler` lookup (use a `Map`, it's O(1)).

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [46 — State Pattern](./46-pattern-state.md) — the other "delegate instead of branching" behavioural pattern |
| **Next** | [48 — Mediator Pattern](./48-pattern-mediator.md) — the hub topology; CoR is a line, Mediator is a star |
| **Related** | [35 — Decorator Pattern](./35-pattern-decorator.md) — structurally near-identical, different intent; the #1 interview follow-up |
| **Related** | [43 — Command Pattern](./43-pattern-command.md) — the request travelling down a chain is often a Command object |
| **Related** | [38 — Composite Pattern](./38-pattern-composite.md) — bubble a request up a Composite tree and you have DOM event propagation |
