# 116 — Dependency injection in Node

## What is this?

Dependency Injection (DI) is a design technique where an object receives the things it depends on — a database client, a logger, an API client — from the outside, instead of creating them itself. Think of a restaurant kitchen: the chef doesn't grow the vegetables or raise the cattle; ingredients are delivered already prepared, so the chef just cooks, and any supplier can be swapped without rewriting the recipe. In code, this means a `UserService` never calls `new PgClient()` internally — a `dbConnection` is handed to it, usually through its constructor.

## Why does it matter for backend development?

Backend services talk to databases, caches, email providers, payment gateways, and third-party APIs — things that are slow, stateful, or expensive to spin up inside a test. If a class builds its own dependencies internally, you cannot test it without a real database, and you cannot swap Postgres for a mock without editing the class itself. DI decouples "what a class needs" from "how that thing is built," so a real `dbConnection` is injected in production and a fake one is injected in tests — no code inside the class changes. This is the foundation the Repository pattern (Topic 117) and Clean Architecture (Topic 118) are both built on, and it is what makes large Express/Fastify codebases testable at scale.

---

## Syntax / API

```js
// ── 1. Manual DI — no library, just constructor parameters ─────────────────
class UserService {
  // Dependencies arrive as constructor arguments, never built inside the class
  constructor(userRepository, logger) {
    this.userRepository = userRepository; // store the injected repository
    this.logger = logger;                 // store the injected logger
  }

  async getUser(userId) {
    this.logger.info(`Fetching user ${userId}`); // use the injected logger
    return this.userRepository.findById(userId); // use the injected repository
  }
}

// Wiring happens once, at the app's "composition root" (usually server.js)
const userRepository = require('./userRepository');            // concrete implementation
const logger = require('./logger');                             // concrete implementation
const userService = new UserService(userRepository, logger);    // manual injection

// ── 2. awilix — a real DI container library for Node ───────────────────────
const { createContainer, asClass, asValue } = require('awilix');

const container = createContainer();                 // create an empty container

container.register({
  dbConnection:   asValue(require('./db')),                       // an already-built value
  logger:         asClass(require('./Logger')).singleton(),       // one shared instance app-wide
  userRepository: asClass(require('./UserRepository')).scoped(),  // one instance per request
  userService:    asClass(require('./UserService')).scoped(),     // one instance per request
});

// awilix resolves by matching constructor PARAMETER NAMES to registered names
const resolvedUserService = container.resolve('userService');

// ── 3. inversify-lite — a minimal token-based container, no decorators ─────
const TOKENS = { Logger: Symbol('Logger'), UserRepo: Symbol('UserRepo') }; // unique identity tokens

class LiteContainer {
  constructor() {
    this.registry = new Map(); // maps a token → a factory function
  }
  register(token, factory) {
    this.registry.set(token, factory); // store how to build this dependency
    return this;                       // allow chaining .register().register()
  }
  resolve(token) {
    const factory = this.registry.get(token);          // look up the builder
    if (!factory) throw new Error(`No registration for ${String(token)}`); // fail loudly
    return factory(this);                               // build it, passing the container in
  }
}
```

---

## How it works — line by line

- **Manual DI** is just a constructor (or function) that accepts dependencies as arguments rather than instantiating them with `new` or `require()` inline. There is no library involved — the "container" is a plain JavaScript file where you build objects in the right order and pass them along.
- **awilix** is a real DI container. You `register()` each dependency once, giving it a name and a "resolver" (`asValue` for a plain object, `asClass` for a class awilix should instantiate, `asFunction` for a factory function). When you call `container.resolve('userService')`, awilix reads the constructor's parameter names and automatically supplies each matching registered dependency — this is called **auto-wiring**.
- **Lifetimes** control how often a new instance is created: `.singleton()` builds it once and reuses it forever (good for a `dbConnection` pool or a `logger`); `.scoped()` builds a fresh instance per "scope" — in a web app, that scope is usually one incoming HTTP request (good for anything holding request-specific data like an `authToken`); `.transient()` (the default) builds a brand-new instance every single time it is resolved.
- **inversify-lite** is the manual, no-decorator version of what full IoC frameworks like InversifyJS do: instead of registering by string name, you register by a unique `Symbol` token, which avoids name collisions across a large codebase. The container just holds a map from token to a factory function, and `resolve()` calls that factory, handing it the container so it can resolve its own nested dependencies.
- In every flavor, the actual **wiring** (deciding which concrete implementation goes where) happens in exactly one place — the composition root — so business logic classes never know or care where their dependencies came from.

---

## Example 1 — basic

```js
// File: src/notifications/NotificationService.js
// A service that sends welcome emails — depends on an "email client" abstraction

class NotificationService {
  // emailClient is injected — could be a real SMTP client or a fake for tests
  constructor(emailClient) {
    this.emailClient = emailClient; // store the injected dependency
  }

  async sendWelcomeEmail(userId, emailAddress) {
    const subject = 'Welcome!';                       // build the email subject
    const body = `Thanks for joining, user ${userId}`; // build the email body
    await this.emailClient.send(emailAddress, subject, body); // delegate to injected client
    return { userId, emailAddress, status: 'sent' };   // return a simple result
  }
}

// ── A real implementation, used in production ───────────────────────────────
class SmtpEmailClient {
  async send(to, subject, body) {
    console.log(`[SMTP] Sending to ${to}: ${subject} — ${body}`); // pretend to send via SMTP
  }
}

// ── A fake implementation, used only in tests ───────────────────────────────
class FakeEmailClient {
  constructor() {
    this.sentEmails = []; // record every "sent" email instead of really sending it
  }
  async send(to, subject, body) {
    this.sentEmails.push({ to, subject, body }); // just push into memory, no network call
  }
}

// ── Production wiring ────────────────────────────────────────────────────────
const productionService = new NotificationService(new SmtpEmailClient()); // real client injected

// ── Test wiring — SAME class, ZERO code changes inside NotificationService ─
const fakeClient = new FakeEmailClient();
const testService = new NotificationService(fakeClient); // fake client injected instead

testService.sendWelcomeEmail('user_42', 'test@example.com').then(() => {
  console.log(fakeClient.sentEmails); // → [{ to, subject, body }] — verifiable in a test, no real email sent
});

module.exports = { NotificationService, SmtpEmailClient, FakeEmailClient };
```

---

## Example 2 — real world backend use case

```js
// File: src/container.js
// Composition root for an Express app — wires database, repository, service, controller
// using awilix with request-scoped containers so each request gets isolated state.

const { createContainer, asClass, asValue, asFunction } = require('awilix');
const dbConnection = require('./db');       // real pg/mongo connection pool (Topic 90/91)
const Logger = require('./Logger');         // simple logger class
const UserRepository = require('./repositories/UserRepository'); // Topic 117 preview
const UserService = require('./services/UserService');
const UserController = require('./controllers/UserController');

// The root container holds everything that is safe to share across the whole app
const rootContainer = createContainer();

rootContainer.register({
  dbConnection:   asValue(dbConnection),                        // shared connection pool
  logger:         asClass(Logger).singleton(),                  // one logger for the app
  userRepository: asClass(UserRepository).scoped(),              // fresh per request
  userService:    asClass(UserService).scoped(),                 // fresh per request
  userController: asClass(UserController).scoped(),              // fresh per request
});

module.exports = rootContainer;

// File: src/middleware/scopedContainer.js
// Express middleware — creates one request-scoped child container per incoming request
const rootContainer = require('../container');

function scopedContainerMiddleware(requestBody, req, res, next) {
  req.container = rootContainer.createScope(); // isolated scope, one per HTTP request
  req.container.register({
    authToken: asValue(req.headers.authorization), // request-specific data, safe in this scope
  });
  next(); // hand off to the next middleware/route handler
}

module.exports = scopedContainerMiddleware;

// File: src/routes/userRoutes.js
// Route handler resolves what it needs from the request's own scope — never a global

const express = require('express');
const router = express.Router();

router.get('/users/:userId', async (req, res) => {
  const userController = req.container.resolve('userController'); // scoped instance for THIS request
  const userId = req.params.userId;                                 // route param
  const result = await userController.getUser(userId);              // delegate to controller
  res.json(result);                                                  // send JSON response
});

module.exports = router;
```

---

## Common mistakes

### Mistake 1 — Hardcoding a concrete dependency inside the class

```js
// ❌ WRONG — UserService builds its own database client, tightly coupled to Postgres
class UserService {
  constructor() {
    const { Client } = require('pg');           // hardcoded concrete dependency
    this.dbConnection = new Client({ host: 'prod-db' }); // impossible to swap in a test
  }
  async getUser(userId) {
    return this.dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  }
}
// Testing this requires a real Postgres server running — slow and fragile

// ✅ CORRECT — dbConnection is injected, the class doesn't know or care what it is
class UserService {
  constructor(dbConnection) {
    this.dbConnection = dbConnection; // injected — could be real pg, or a fake in a test
  }
  async getUser(userId) {
    return this.dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  }
}
// Tests inject a fake { query: async () => [...] } — no real database needed
```

### Mistake 2 — Injecting the whole container (service locator anti-pattern)

```js
// ❌ WRONG — the class receives the entire container and pulls what it wants at runtime
class UserService {
  constructor(container) {
    this.container = container; // hides the real dependencies — unclear what this class needs
  }
  async getUser(userId) {
    const dbConnection = this.container.resolve('dbConnection'); // dependency discovered too late
    return dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  }
}
// You cannot tell from the constructor what UserService actually depends on

// ✅ CORRECT — dependencies are explicit constructor parameters
class UserService {
  constructor(dbConnection, logger) {
    this.dbConnection = dbConnection; // explicit — you can see exactly what's required
    this.logger = logger;             // explicit — easy to inject a fake in a test
  }
  async getUser(userId) {
    this.logger.info(`Fetching ${userId}`);
    return this.dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  }
}
// awilix (or manual wiring) auto-supplies these by matching parameter names
```

### Mistake 3 — Using the wrong lifetime for request-specific data

```js
// ❌ WRONG — RequestContext registered as a singleton, shared across ALL requests
container.register({
  requestContext: asClass(RequestContext).singleton(), // built ONCE, reused forever
});
// Request A sets authToken = 'tokenA', then Request B (concurrent) overwrites it with 'tokenB'
// Request A's later code now sees Request B's authToken — a serious data leak

// ✅ CORRECT — scoped lifetime creates a fresh instance per request
container.register({
  requestContext: asClass(RequestContext).scoped(), // built fresh for each container.createScope()
});
// Each request gets req.container = rootContainer.createScope();
// Its requestContext.authToken never leaks into any other request's scope
```

---

## Practice exercises

### Exercise 1 — easy

Build a `Logger` class with an `info(message)` method that prefixes and prints messages (e.g. `[LOG] message`). Build an `OrderService` class whose constructor takes a `logger` as its only dependency, with a method `placeOrder(userId, itemId)` that logs `"Order placed by <userId> for <itemId>"` via the injected logger and returns `{ userId, itemId, status: 'placed' }`.

Then write a `FakeLogger` class that stores every message in an array instead of printing it (`this.messages = []`). Instantiate `OrderService` twice — once with the real `Logger`, once with `FakeLogger` — call `placeOrder` on both, and print `fakeLogger.messages` to prove the fake captured the call without touching the console.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a hand-rolled DI container object with two methods: `register(name, factory, options)` where `options.singleton` is a boolean, and `resolve(name)`. When `singleton` is true, the container must build the dependency only once and return the same cached instance on every future `resolve()` call; otherwise it builds a new instance every time.

Register a `Database` class (fake — a `connect()` method that logs once, and a `query()` method returning a hardcoded array) as a singleton, a `Logger` class as a singleton, and a `UserRepository` class (constructor takes `database` and `logger`, has a `findById(userId)` method that calls `database.query()` and logs via `logger`) as non-singleton. Resolve `UserRepository` twice and prove with `===` that both resolutions received the exact same `Database` instance, while confirming two separate `UserRepository` instances were created.

```js
// Write your code here
```

---

### Exercise 3 — hard

Using `awilix` (or your own `LiteContainer` pattern from this doc), build a small backend wiring with these registrations: `dbConnection` (singleton — a fake object with a `query()` method), `requestContext` (scoped — holds `authToken` and `sessionId` passed in per request), `userRepository` (scoped — depends on `dbConnection`, has `findById(userId)`), `userService` (scoped — depends on `userRepository` and `requestContext`, has `getCurrentUser()` that reads `userId` off of `requestContext`).

Simulate two "concurrent requests" by creating two scopes (`container.createScope()`), registering a different `authToken`/`sessionId` on each scope's `requestContext`, resolving `userService` from each scope, and proving: (1) each scope's `userService.getCurrentUser()` sees only its own `authToken`/`sessionId` — no leakage between them, and (2) both scopes' resolved `userRepository` instances share the identical `dbConnection` singleton via `===`. Finally, write one more test-style call that resolves `userService` with a manually-injected mock `userRepository` (no container involved) to demonstrate the testability benefit directly.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT DI IS
  A class receives its dependencies from outside (constructor/function args)
  instead of building them internally with `new` or `require()` inline.

WHY IT MATTERS
  Swappable implementations (real dbConnection vs fake in tests)
  Decoupled business logic from concrete infrastructure
  No real DB / network calls needed to unit test a service

THREE FLAVORS
  Manual DI        → plain constructor params, wired by hand at startup
  awilix           → real container, auto-wires by matching param NAMES
  inversify-lite   → manual container keyed by Symbol TOKENS, no decorators

AWILIX RESOLVERS
  asValue(x)     → register an already-built value/object
  asClass(Cls)   → register a class, awilix calls `new Cls(...)`
  asFunction(fn) → register a factory function

LIFETIMES
  singleton   → built once, shared everywhere        (dbConnection, logger)
  scoped      → built once PER request/scope           (requestContext, authToken)
  transient   → built fresh on every single resolve()  (default, stateless helpers)

COMMON MISTAKES
  Hardcoding `new PgClient()` inside a service   → tight coupling, untestable
  Injecting the whole container (service locator) → hides real dependencies
  Wrong lifetime (singleton for per-request data) → data leaks across requests

COMPOSITION ROOT
  Wiring happens ONCE, in one file (container.js / server.js)
  Business logic classes never call require() for their own dependencies

TESTABILITY WIN
  new UserService(fakeRepository, fakeLogger)
  → no real DB, no real network, fully deterministic unit tests
```

---

## Connected topics

- **117 — Repository pattern in Node** — the repository is exactly the kind of dependency you inject into a service; DI is what makes swapping a real repository for a mock possible.
- **118 — Clean architecture in Node** — clean architecture's dependency rule (inner layers never depend on outer layers) is enforced in practice through dependency injection at the boundaries.
- **85 — Mocking and stubbing** — DI is the mechanism that lets `jest.mock()`/manual fakes replace real dependencies (databases, email clients, APIs) without touching the class under test.
