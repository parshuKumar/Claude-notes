# 115 — Plugin and middleware architecture

## What is this?

Plugin and middleware architecture is a design pattern where your core application exposes well-defined extension points, and separate, independently-written pieces of code "plug into" those points to add behavior — without ever touching the core code itself. Think of a power strip: the wall socket (your core app) doesn't know or care what gets plugged into it — a lamp, a charger, a router — it just provides a standard interface, and each plugged-in device adds its own functionality. Express middleware, Fastify plugins, and webpack loaders are all real-world examples of this pattern in the Node.js ecosystem.

## Why does it matter for backend development?

Backend systems grow. A server that starts as "handle login and return JSON" eventually needs logging, rate limiting, authentication, caching, metrics, and feature flags — and different teams or environments need different combinations of these. Hard-coding all of that into one giant `server.js` makes the file unmaintainable and untestable. A plugin/middleware architecture lets you register features as independent, composable units — each one testable in isolation, each one optional, each one ordered explicitly. This is exactly how Express, Fastify, Koa, and most production Node.js frameworks are built internally, and it's the same pattern you'll reach for whenever you build an internal framework, a CLI tool, or a library that other developers on your team need to extend.

---

## Syntax / API

```js
// A minimal middleware-style pipeline — the same shape Express uses internally

// The pipeline stores an ordered list of middleware functions
class Pipeline {
  constructor() {
    this.middlewares = []; // array of (context, next) => void functions
  }

  // use() registers a new middleware — this is the "plug in" step
  use(middlewareFn) {
    this.middlewares.push(middlewareFn); // add to the end of the chain
    return this; // return `this` so calls can be chained: pipeline.use(a).use(b)
  }

  // run() executes the chain for a given context (e.g. a request)
  async run(context) {
    let index = -1; // tracks which middleware we are currently on

    // dispatch(i) calls middleware at position i, then hands control to i+1
    const dispatch = async (i) => {
      if (i <= index) throw new Error('next() called multiple times'); // safety check
      index = i; // move the cursor forward
      const middlewareFn = this.middlewares[i]; // grab the current middleware
      if (!middlewareFn) return; // no more middleware — chain is done
      // call it, passing a `next` function that continues the chain
      await middlewareFn(context, () => dispatch(i + 1));
    };

    await dispatch(0); // start the chain at the first middleware
    return context; // return the (possibly mutated) context
  }
}

module.exports = { Pipeline };
```

---

## How it works — line by line

- `this.middlewares = []` — the pipeline is nothing more than an array. Every plugin you "register" is just a function pushed into this array. There is no magic — the order in the array is the order things run.
- `use(middlewareFn)` — this is the registration hook. Anyone can call `pipeline.use(fn)` to add behavior, without ever seeing or editing the pipeline's internals. This is the "plug" part of plugin architecture.
- `run(context)` — this kicks off processing for one request (or one job, one event — whatever your "context" represents).
- `index` — a cursor that remembers how far along the chain we are. It exists purely to catch a common bug: calling `next()` twice by accident.
- `dispatch(i)` — a recursive function. It looks up the middleware at position `i`, runs it, and gives that middleware a `next` callback that — when called — recursively runs `dispatch(i + 1)`. This is exactly how Express's `req, res, next` chain works under the hood.
- If a middleware never calls `next()`, the chain simply stops there — later middlewares never run. That's intentional: it lets a middleware "short-circuit" the pipeline (for example, an auth middleware that rejects the request and never lets it reach the route handler).
- Because `dispatch` is `async` and each middleware is `await`ed, middlewares can do asynchronous work (database calls, network requests) before calling `next()`, and the pipeline correctly waits for them.

---

## Example 1 — basic

```js
// File: pipeline-demo.js
const { Pipeline } = require('./pipeline'); // the class we defined above

const pipeline = new Pipeline(); // create an empty pipeline

// Plugin 1: logs the request as it enters
pipeline.use(async (context, next) => {
  console.log(`[LOG] → incoming request for ${context.path}`); // before next()
  await next(); // pass control to the next middleware
  console.log(`[LOG] ← response sent for ${context.path}`); // after downstream finished
});

// Plugin 2: attaches a timestamp to the context
pipeline.use(async (context, next) => {
  context.receivedAt = Date.now(); // mutate the shared context object
  await next(); // continue the chain
});

// Plugin 3: the "final" handler — does not call next() because it's the last stop
pipeline.use(async (context) => {
  context.responseBody = `Hello, request received at ${context.receivedAt}`;
});

// Run the pipeline with a fake request context
pipeline.run({ path: '/users' }).then((finalContext) => {
  console.log('Final context:', finalContext); // inspect the mutated object
});

// Output order:
// [LOG] → incoming request for /users
// [LOG] ← response sent for /users
// Final context: { path: '/users', receivedAt: 171..., responseBody: 'Hello, request received at ...' }
```

---

## Example 2 — real world backend use case

```js
// File: src/core/appFramework.js
// A tiny extensible backend framework: plugins register hooks that run
// at specific lifecycle points — "onRequest", "onError", "onShutdown".
// This is the exact shape Fastify's plugin system uses in production.

class AppFramework {
  constructor() {
    this.hooks = {
      onRequest: [], // ran before the route handler
      onResponse: [], // ran after the route handler
      onError: [], // ran when a handler throws
    };
    this.routes = new Map(); // path -> handler function
    this.plugins = new Set(); // track which plugins were already registered
  }

  // register() is the public "plugin registration" API
  register(plugin, options = {}) {
    if (this.plugins.has(plugin)) return this; // prevent double-registration
    this.plugins.add(plugin); // mark plugin as installed
    plugin(this, options); // hand the app instance to the plugin so it can hook in
    return this; // allow chaining: app.register(a).register(b)
  }

  // addHook() lets a plugin subscribe to a lifecycle event
  addHook(hookName, fn) {
    if (!this.hooks[hookName]) throw new Error(`Unknown hook: ${hookName}`); // fail fast on typos
    this.hooks[hookName].push(fn); // append the hook function
  }

  // route() lets a plugin (or the app itself) register a route handler
  route(path, handlerFn) {
    this.routes.set(path, handlerFn); // store handler by path
  }

  // handleRequest() simulates receiving an HTTP request and running the full lifecycle
  async handleRequest(requestBody) {
    const context = { requestBody, userId: null, statusCode: 200 }; // shared per-request state

    try {
      for (const hook of this.hooks.onRequest) await hook(context); // run all onRequest hooks in order

      const handler = this.routes.get(requestBody.path); // find the matching route
      if (!handler) throw new Error(`No route for ${requestBody.path}`); // 404-style failure
      context.result = await handler(context); // run the actual business logic

      for (const hook of this.hooks.onResponse) await hook(context); // run all onResponse hooks
      return context;
    } catch (err) {
      context.error = err; // attach the error to context
      for (const hook of this.hooks.onError) await hook(context); // let error plugins react (logging, alerts)
      return context;
    }
  }
}

// ── A plugin: authentication ────────────────────────────────────────────────
function authPlugin(app) {
  app.addHook('onRequest', async (context) => {
    const authToken = context.requestBody.authToken; // pull token off the incoming request
    if (!authToken) throw new Error('Missing authToken'); // reject unauthenticated requests
    context.userId = authToken === 'valid-token-123' ? 'user_42' : null; // fake token check
    if (!context.userId) throw new Error('Invalid authToken');
  });
}

// ── A plugin: request logging ───────────────────────────────────────────────
function loggingPlugin(app) {
  app.addHook('onError', async (context) => {
    console.error(`[ERROR] ${context.requestBody.path} — ${context.error.message}`); // log failures
  });
  app.addHook('onResponse', async (context) => {
    console.log(`[OK] ${context.requestBody.path} — user ${context.userId}`); // log successes
  });
}

// ── Wiring it all together ──────────────────────────────────────────────────
const app = new AppFramework();
app.register(authPlugin); // plug in auth without touching core code
app.register(loggingPlugin); // plug in logging independently

app.route('/profile', async (context) => {
  return { userId: context.userId, message: 'Welcome back!' }; // the actual route logic
});

app.handleRequest({ path: '/profile', authToken: 'valid-token-123' }).then((result) => {
  console.log('Result:', result.result); // → { userId: 'user_42', message: 'Welcome back!' }
});

module.exports = { AppFramework };
```

---

## Common mistakes

### Mistake 1 — Forgetting to call `next()` (the chain silently stops)

```js
// ❌ WRONG — middleware does its work but never calls next()
// Every middleware registered AFTER this one will simply never run,
// and the request hangs forever with no response.
pipeline.use(async (context, next) => {
  context.userId = 'user_42'; // does useful work
  // next() is never called — chain dies here
});

// ✅ CORRECT — always call next() unless you intentionally want to stop the chain
pipeline.use(async (context, next) => {
  context.userId = 'user_42';
  await next(); // hand control to the next middleware in line
});
```

### Mistake 2 — Mutating shared state without namespacing (plugins collide)

```js
// ❌ WRONG — two unrelated plugins both write to context.data,
// so the second plugin silently overwrites the first plugin's data
function cachePlugin(app) {
  app.addHook('onRequest', async (context) => { context.data = { cacheHit: true }; });
}
function metricsPlugin(app) {
  app.addHook('onRequest', async (context) => { context.data = { durationMs: 12 }; });
  // cacheHit is now gone — overwritten, not merged
}

// ✅ CORRECT — each plugin gets its own namespaced slot on the context
function cachePlugin(app) {
  app.addHook('onRequest', async (context) => {
    context.cache = { cacheHit: true }; // dedicated key — no collision
  });
}
function metricsPlugin(app) {
  app.addHook('onRequest', async (context) => {
    context.metrics = { durationMs: 12 }; // dedicated key — no collision
  });
}
```

### Mistake 3 — Registering plugins in the wrong order and assuming order doesn't matter

```js
// ❌ WRONG — the logging plugin reads context.userId before the auth
// plugin has had a chance to set it, because it was registered first
app.register(loggingPlugin); // runs onRequest hooks first — reads userId (still null!)
app.register(authPlugin);    // sets userId, but too late for logging's onRequest hook

// ✅ CORRECT — register plugins in the order their hooks must actually run
app.register(authPlugin);    // sets context.userId FIRST
app.register(loggingPlugin); // now safely reads context.userId
// Rule of thumb: auth/parsing plugins go first, logging/metrics plugins go last
```

---

## Practice exercises

### Exercise 1 — easy

Build a simple `EventPipeline` class with a `use(fn)` method to register handler functions and a `run(payload)` method that calls every registered handler, in registration order, passing the same `payload` object to each one. Register three handlers: one that adds a `processedAt` timestamp field to the payload, one that logs the payload to the console, and one that adds a `status: 'done'` field. Call `run({ orderId: 'order_101' })` and print the final payload.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `MiddlewareChain` class (like the `Pipeline` shown in Syntax/API) that supports `next()`-style chaining for an async request pipeline. Register three middlewares:
1. A `parseAuth` middleware that reads `context.headers.apiKey` and sets `context.isAuthenticated` to `true` if it equals `'secret-key-99'`, otherwise sets it to `false` and does **not** call `next()` (short-circuits the chain).
2. A `rateLimiter` middleware that only runs if `context.isAuthenticated` is `true`, and rejects (throws) if `context.requestCount` (a number you pass in) is greater than `5`.
3. A `handler` middleware that sets `context.response = 'Request handled successfully'`.

Test it with two different contexts: one with a valid `apiKey` and `requestCount: 2`, and one with an invalid `apiKey`. Log the final context for both and confirm the short-circuit behavior works.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a full `PluginHost` system that supports:
1. `pluginHost.register(pluginFn, options)` — installs a plugin, passing it a restricted API object (not the whole host) with only `addHook(hookName, fn)` and `getConfig(key)` available to the plugin.
2. Three lifecycle hooks: `beforeSave`, `afterSave`, `onValidationError`.
3. A `saveRecord(record)` method that: runs all `beforeSave` hooks (which may mutate the record, e.g. add a `createdAt` timestamp), then validates that the record has a non-empty `userId` field (if missing, run all `onValidationError` hooks and abort), then "saves" it (just push it into an in-memory array), then runs all `afterSave` hooks.
4. Write at least two plugins: a `timestampPlugin` that adds `createdAt` in `beforeSave`, and an `auditLogPlugin` that logs `"Saved record for {userId}"` in `afterSave`.
5. Demonstrate that calling `saveRecord({})` (missing `userId`) correctly triggers the validation-error path instead of saving.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE CONCEPT
  Core app exposes extension points ("hooks" / "use()")
  Plugins/middleware register functions against those points
  Core app never needs to know what a specific plugin does

BUILDING BLOCKS
  registry array/map    → where(this.middlewares = [] / this.hooks = {})
  use(fn) / register(fn)→ the public API to add a plugin/middleware
  next()                → hands control to the next item in the chain
  context object         → shared, mutable state passed through the chain

CHAIN BEHAVIOR
  next() called          → chain continues to the next middleware
  next() NOT called       → chain stops here (intentional short-circuit)
  next() called twice     → bug — throw an error to catch it early
  error thrown mid-chain  → should be caught and routed to an onError hook

DESIGN RULES
  - Namespace what each plugin writes onto the shared context
    (context.auth, context.metrics — never a shared flat context.data)
  - Registration ORDER matters — auth/parsing first, logging/metrics last
  - Give plugins a restricted API surface, not the whole app instance
    (principle of least privilege — prevents accidental coupling)
  - Prevent double-registration with a Set/Map keyed by plugin reference

REAL NODE.JS EXAMPLES OF THIS PATTERN
  Express      → app.use(middleware) — req, res, next chain
  Fastify      → fastify.register(plugin) — encapsulated plugin contexts
  Koa          → ctx, next — same recursive dispatch shape
  webpack      → compiler.hooks.emit.tap('MyPlugin', fn) — tapable hooks
  Passport.js  → strategies registered as plugins for auth

WHEN TO REACH FOR THIS PATTERN
  - You're building a framework/library others will extend
  - You need optional, swappable features (dev vs prod behavior)
  - Different teams own different cross-cutting concerns
    (auth team owns auth plugin, observability team owns logging plugin)
```

---

## Connected topics

- **53 — Express.js fundamentals** — Express's `app.use()` and `req, res, next` chain is a production implementation of exactly this pattern.
- **55 — Express middleware in depth** — dives deeper into middleware ordering, error-handling middleware, and the built-in/third-party/custom middleware categories referenced here.
- **116 — Dependency injection in Node** — DI containers are a complementary pattern for wiring plugins/services together without hard-coded imports, often used alongside plugin systems.
