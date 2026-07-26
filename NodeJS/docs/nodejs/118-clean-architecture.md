# 118 — Clean architecture in Node

## What is this?

Clean architecture is a way of organizing a codebase into layers — **controller → use case → domain → infrastructure** — where each layer only knows about the layer directly inside it, never the other way around. Think of a restaurant: the waiter (controller) takes your order and never touches food; the chef (use case) follows a recipe (domain rules) to decide what to cook; the recipe itself doesn't care whether the kitchen has a gas stove or an induction one (infrastructure) — that detail is swappable. The core rule, called the **dependency rule**, is that dependencies always point inward, toward business logic, never outward toward frameworks or databases.

## Why does it matter for backend development?

Real backend apps grow past a few routes fast — a `userController.js` that starts at 30 lines ends up 600 lines deep with SQL queries, validation, email sending, and Express `req`/`res` all tangled together. When that happens, changing the database or writing a unit test becomes painful because business logic is welded to Express and to a specific ORM. Clean architecture matters because it lets you: (1) unit-test business rules with zero database or HTTP server running, (2) swap Postgres for MongoDB, or Express for Fastify, by only touching the infrastructure layer, and (3) onboard new engineers faster because "where does the logic that calculates a discount live?" always has one answer — the domain layer. Backend developers reach for this once an app has multiple teams, multiple entry points (REST + cron jobs + message queue consumers), or a codebase that needs to outlive its first framework choice.

---

## Syntax / API

```js
// ── The four layers, as folders, from outside-in ────────────────────────────
// src/
//   controllers/   → talks HTTP (req, res), knows nothing about SQL
//   use-cases/      → orchestrates one action ("register a user")
//   domain/         → pure business rules and entities, zero dependencies
//   infrastructure/ → database clients, email providers, external APIs

// domain/user.entity.js
// Pure JS class — no require('express'), no require('pg'), nothing external
class User {
  // constructor validates the invariant: every User MUST have a valid email
  constructor({ userId, email, passwordHash }) {
    if (!email.includes('@')) throw new Error('Invalid email'); // domain rule
    this.userId = userId;
    this.email = email;
    this.passwordHash = passwordHash;
  }
}

module.exports = { User }; // domain has nothing to import — it is the center

// use-cases/register-user.usecase.js
// Depends ONLY on the domain and on an abstract "port" (interface), never on infra directly
class RegisterUserUseCase {
  // userRepository is injected — it is a PORT (interface), the concrete DB class is not known here
  constructor({ userRepository, hasher }) {
    this.userRepository = userRepository; // abstraction, not "new PostgresUserRepository()"
    this.hasher = hasher;                 // abstraction for password hashing
  }

  // execute() is the single entry point — one use case, one job
  async execute({ email, plainPassword }) {
    const passwordHash = await this.hasher.hash(plainPassword); // delegate hashing to a port
    const user = new User({ userId: cryptoRandomId(), email, passwordHash }); // build domain entity
    await this.userRepository.save(user); // persist via the port, not a concrete driver
    return user;
  }
}

module.exports = { RegisterUserUseCase };
```

---

## How it works — line by line

Picture a request coming in to `POST /users`. It flows through the layers strictly in one direction:

1. **Controller** receives the raw HTTP `req`/`res` from Express. Its only job is to pull data out of `req.body`, call a use case, and turn the result back into an HTTP response (`res.json`, status codes). It never contains a business rule like "password must be 8 characters" — that belongs deeper.
2. **Use case** (also called "interactor" or "application service") represents one thing the app can do — `RegisterUserUseCase`, `TransferFundsUseCase`. It coordinates the steps: validate input shape, build a domain entity, ask a repository to save it, maybe trigger a side effect like sending a welcome email. It never imports Express or a SQL driver directly.
3. **Domain** is the innermost layer — plain classes and functions representing business concepts (`User`, `Order`, `Money`) and the rules that must always hold true (an `Order` cannot ship with zero items). It has **zero external dependencies** — no `require('express')`, no `require('pg')`, not even `require('./infrastructure/...')`. This is what makes it trivially testable and reusable.
4. **Infrastructure** is the outermost layer — the actual database client, email provider SDK, file system access, HTTP clients to third-party APIs. It **implements** the ports (interfaces) that use cases declare they need, e.g. a `PostgresUserRepository` implements the `save(user)` method that `RegisterUserUseCase` expects.

The **dependency rule** ties it together: source code dependencies can only point inward. Controllers may depend on use cases. Use cases may depend on domain. Nothing inward-facing may depend on anything outward-facing — domain never imports infrastructure, and use cases only depend on infrastructure through an abstract interface (a port), never a concrete class. The concrete implementation gets handed in from outside — this is the same **dependency injection** idea from Topic 116, just applied at the architecture level rather than the class level.

---

## Example 1 — basic

```js
// domain/order.entity.js
// Pure domain entity — describes what an Order IS and what rules it must obey
class Order {
  constructor({ orderId, items }) {
    if (items.length === 0) throw new Error('Order must have at least one item'); // domain rule
    this.orderId = orderId;
    this.items = items;
    this.total = items.reduce((sum, item) => sum + item.price * item.qty, 0); // computed by domain, not controller
  }
}

module.exports = { Order };

// use-cases/create-order.usecase.js
// Orchestrates the "create an order" action using only the domain + a repository PORT
const { Order } = require('../domain/order.entity');

class CreateOrderUseCase {
  // orderRepository is injected — could be in-memory today, Postgres tomorrow
  constructor({ orderRepository }) {
    this.orderRepository = orderRepository;
  }

  async execute({ items }) {
    const orderId = `order_${Date.now()}`;   // generate an id (could be uuid in real code)
    const order = new Order({ orderId, items }); // domain validates and computes total
    await this.orderRepository.save(order);  // persistence delegated to the port
    return order;                            // return the created entity
  }
}

module.exports = { CreateOrderUseCase };

// infrastructure/in-memory-order.repository.js
// A concrete adapter — implements the shape the use case expects: save(order)
class InMemoryOrderRepository {
  constructor() {
    this.orders = new Map(); // fake "database" — just a Map in RAM
  }

  async save(order) {
    this.orders.set(order.orderId, order); // store by id
    return order;
  }
}

module.exports = { InMemoryOrderRepository };

// Wiring it together (this is the "composition root" — the ONE place layers meet)
const { CreateOrderUseCase } = require('./use-cases/create-order.usecase');
const { InMemoryOrderRepository } = require('./infrastructure/in-memory-order.repository');

const orderRepository = new InMemoryOrderRepository();          // pick a concrete implementation
const createOrder = new CreateOrderUseCase({ orderRepository }); // inject it into the use case

createOrder.execute({ items: [{ price: 10, qty: 2 }] })
  .then((order) => console.log('Created:', order)); // → Created: Order { orderId: ..., total: 20 }
```

---

## Example 2 — real world backend use case

```js
// domain/user.entity.js
// The domain layer — pure business rules, no framework or database code allowed here
class User {
  constructor({ userId, email, passwordHash, createdAt = new Date() }) {
    if (!email.includes('@')) throw new Error('Invalid email format');       // domain rule
    if (!passwordHash) throw new Error('Password hash is required');         // domain rule
    this.userId = userId;
    this.email = email;
    this.passwordHash = passwordHash;
    this.createdAt = createdAt;
  }
}

module.exports = { User };

// use-cases/register-user.usecase.js
// Depends only on domain + ports (interfaces), injected from the composition root
const { User } = require('../domain/user.entity');
const { randomUUID } = require('crypto'); // Node built-in, allowed even in use-case layer

class RegisterUserUseCase {
  // userRepository and hasher are PORTS — abstract shapes, not concrete DB/bcrypt classes
  constructor({ userRepository, hasher, emailService }) {
    this.userRepository = userRepository;
    this.hasher = hasher;
    this.emailService = emailService;
  }

  async execute({ email, plainPassword }) {
    const existingUser = await this.userRepository.findByEmail(email); // check via port
    if (existingUser) throw new Error('Email already registered');      // use-case level rule

    const passwordHash = await this.hasher.hash(plainPassword);   // delegate hashing algorithm choice
    const user = new User({ userId: randomUUID(), email, passwordHash }); // build valid domain entity

    await this.userRepository.save(user);              // persist through the port
    await this.emailService.sendWelcomeEmail(user.email); // side effect, also through a port

    return user; // controller decides how to shape the HTTP response from this
  }
}

module.exports = { RegisterUserUseCase };

// infrastructure/postgres-user.repository.js
// Concrete adapter — the ONLY place that knows about SQL and the pg driver
class PostgresUserRepository {
  constructor({ dbConnection }) {
    this.dbConnection = dbConnection; // injected pg Pool or client
  }

  async findByEmail(email) {
    const result = await this.dbConnection.query(
      'SELECT * FROM users WHERE email = $1', [email]
    );
    return result.rows[0] || null; // returns raw row or null — mapping kept simple here
  }

  async save(user) {
    await this.dbConnection.query(
      'INSERT INTO users (user_id, email, password_hash, created_at) VALUES ($1, $2, $3, $4)',
      [user.userId, user.email, user.passwordHash, user.createdAt]
    );
  }
}

module.exports = { PostgresUserRepository };

// infrastructure/bcrypt-hasher.js
// Concrete adapter for the "hasher" port — swappable for argon2 later without touching the use case
const bcrypt = require('bcrypt');

class BcryptHasher {
  async hash(plainPassword) {
    return bcrypt.hash(plainPassword, 10); // 10 salt rounds
  }
}

module.exports = { BcryptHasher };

// controllers/user.controller.js
// Talks HTTP only — pulls data out of req, calls the use case, shapes the res
class UserController {
  constructor({ registerUserUseCase }) {
    this.registerUserUseCase = registerUserUseCase; // injected use case, controller doesn't build it
  }

  // Express handler — bound so `this` stays correct when passed to router.post
  register = async (req, res) => {
    try {
      const { email, password } = req.body;                      // raw HTTP input
      const user = await this.registerUserUseCase.execute({       // delegate ALL logic
        email,
        plainPassword: password,
      });
      res.status(201).json({ userId: user.userId, email: user.email }); // HTTP-shaped response
    } catch (error) {
      res.status(400).json({ error: error.message }); // controller only translates errors to HTTP codes
    }
  };
}

module.exports = { UserController };

// app.js — the composition root: the ONE file allowed to know about every layer at once
const express = require('express');
const { Pool } = require('pg');
const { PostgresUserRepository } = require('./infrastructure/postgres-user.repository');
const { BcryptHasher } = require('./infrastructure/bcrypt-hasher');
const { RegisterUserUseCase } = require('./use-cases/register-user.usecase');
const { UserController } = require('./controllers/user.controller');

const dbConnection = new Pool({ connectionString: process.env.DATABASE_URL }); // real infra

// Wire concrete implementations into the abstractions the use case expects
const userRepository = new PostgresUserRepository({ dbConnection });
const hasher = new BcryptHasher();
const emailService = { sendWelcomeEmail: async (email) => console.log(`Welcome email → ${email}`) };

const registerUserUseCase = new RegisterUserUseCase({ userRepository, hasher, emailService });
const userController = new UserController({ registerUserUseCase });

const app = express();
app.use(express.json());
app.post('/users', userController.register); // controller is the only layer that touches Express

app.listen(3000, () => console.log('Server on http://localhost:3000'));
```

---

## Common mistakes

### Mistake 1 — Fat controllers holding business logic

```js
// ❌ WRONG — validation, hashing, and SQL all live inside the Express handler
app.post('/users', async (req, res) => {
  const { email, password } = req.body;
  if (!email.includes('@')) return res.status(400).json({ error: 'Invalid email' }); // business rule leaked into controller
  const passwordHash = await bcrypt.hash(password, 10);                              // infra call leaked into controller
  await dbConnection.query('INSERT INTO users VALUES ($1, $2)', [email, passwordHash]); // SQL leaked into controller
  res.status(201).json({ email });
});

// ✅ CORRECT — controller only talks HTTP, everything else lives in the use case
app.post('/users', async (req, res) => {
  try {
    const user = await registerUserUseCase.execute({ // one call, all logic hidden behind it
      email: req.body.email,
      plainPassword: req.body.password,
    });
    res.status(201).json({ userId: user.userId });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```

### Mistake 2 — Domain layer importing infrastructure directly

```js
// ❌ WRONG — domain/order.entity.js requiring a database client breaks the dependency rule
const { dbConnection } = require('../infrastructure/db'); // domain now depends on infra — inward rule violated

class Order {
  async save() {
    await dbConnection.query('INSERT INTO orders ...'); // domain entity should never know HOW it's persisted
  }
}

// ✅ CORRECT — domain stays pure; persistence is the repository's job, called from the use case
class Order {
  constructor({ orderId, items }) {
    if (items.length === 0) throw new Error('Order must have at least one item'); // only business rules here
    this.orderId = orderId;
    this.items = items;
  }
}
// Saving happens in use-cases/create-order.usecase.js via orderRepository.save(order)
```

### Mistake 3 — Use case depending on a concrete class instead of a port

```js
// ❌ WRONG — use case imports and instantiates the concrete Postgres repository directly
const { PostgresUserRepository } = require('../infrastructure/postgres-user.repository');

class RegisterUserUseCase {
  constructor() {
    this.userRepository = new PostgresUserRepository({ dbConnection }); // hard-coded — can't swap or mock
  }
  async execute({ email }) { /* ... */ }
}
// Now unit testing this use case requires a real Postgres connection

// ✅ CORRECT — the concrete class is injected from outside (composition root), use case only knows the shape
class RegisterUserUseCase {
  constructor({ userRepository }) { // any object with findByEmail() and save() will do
    this.userRepository = userRepository;
  }
  async execute({ email }) { /* ... */ }
}
// In tests: new RegisterUserUseCase({ userRepository: fakeInMemoryRepository })
```

---

## Practice exercises

### Exercise 1 — easy

Build a tiny clean-architecture slice for "adding a product to a wishlist":
1. A `WishlistItem` domain entity (in `domain/wishlist-item.entity.js`) that takes `{ userId, productId }` and throws if either is missing.
2. An `AddToWishlistUseCase` (in `use-cases/`) that receives an injected `wishlistRepository` with a `save(item)` method, builds a `WishlistItem`, and saves it.
3. An `InMemoryWishlistRepository` (in `infrastructure/`) that stores items in an array.
4. Wire the three together in one script and log the saved item.

```js
// Write your code here
```

---

### Exercise 2 — medium

Extend the Example 2 registration flow into a **login** flow following the same layers:
1. Domain: reuse the existing `User` entity (assume it already exists).
2. Use case: `LoginUserUseCase` that receives an injected `userRepository` (with `findByEmail`) and an injected `hasher` (with a `compare(plainPassword, passwordHash)` method), finds the user by email, compares the password, and throws `'Invalid credentials'` if it doesn't match, otherwise returns the user.
3. Infrastructure: write a `FakeHasher` (for testing) whose `compare()` just does `plainPassword === 'correct-password'`, and an `InMemoryUserRepository` with `findByEmail()` returning a hardcoded user.
4. Controller: a plain function `loginController(req, res)` (no real Express needed — you can fake `req`/`res` objects) that calls the use case and returns either a 200 with the user's id or a 401 with the error message.
5. Prove it works by calling the controller twice — once with the correct password, once with a wrong one — and logging both responses.

```js
// Write your code here
```

---

### Exercise 3 — hard

Design and build a small **order-fulfillment** slice with four layers end to end, enforcing the dependency rule strictly:
1. Domain: an `Order` entity with `items` (array of `{ productId, qty, price }`) and a computed `total`; it must throw if `qty` or `price` is negative for any item. Add a domain method `applyDiscount(percentage)` that returns a new total without mutating the original total (pure calculation).
2. Use case: `PlaceOrderUseCase` that depends on an injected `orderRepository` (`save`), an injected `inventoryService` port (`reserveStock(productId, qty)` — assume it may throw `'Out of stock'`), and an injected `notificationService` port (`notifyOrderPlaced(order)`). It must: build the `Order`, call `reserveStock` for every item BEFORE saving, save the order only if all stock reservations succeed, then notify — and if any reservation fails partway through, it must not save the order at all.
3. Infrastructure: build fake in-memory implementations of `inventoryService` (tracks stock counts in a `Map` and throws when it hits zero) and `notificationService` (just logs), plus an `InMemoryOrderRepository`.
4. Composition root: wire everything together and test two scenarios — an order that succeeds fully, and an order where one item is out of stock (verify the order was NOT saved in that case by checking the repository afterward).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THE FOUR LAYERS (outside → in)
  controllers/      → HTTP only: req/res, status codes, no business logic
  use-cases/        → orchestrates ONE action, depends on domain + ports only
  domain/           → pure business entities and rules, ZERO external deps
  infrastructure/    → concrete DB clients, email/SMS SDKs, external APIs

THE DEPENDENCY RULE
  Dependencies point INWARD only:
    controller → use case → domain
                     ↓
              infrastructure (via an injected PORT/interface, never direct import)
  Domain NEVER imports anything from use-cases, controllers, or infrastructure

PORTS vs ADAPTERS
  Port      = the shape a use case expects (e.g. { save, findByEmail })
  Adapter   = the concrete class implementing that shape (PostgresUserRepository)
  Use case depends on the PORT shape, adapter is injected from outside

COMPOSITION ROOT
  The ONE file (usually app.js / server.js / a DI container) allowed to
  import every layer and wire concrete adapters into use cases

WHY IT PAYS OFF
  - Unit test use cases with fake repositories — no real DB needed
  - Swap Postgres → MongoDB, or Express → Fastify, without touching domain/use-cases
  - Business rule bugs are found in domain/, not scattered across route handlers

WHEN TO SKIP IT
  - Tiny scripts, single-file CLIs, throwaway prototypes — the ceremony isn't worth it
  - Reach for it once an app has real business rules, multiple entry points,
    or is expected to survive a framework/database change

RED FLAGS THAT YOU'VE BROKEN THE RULE
  - domain/*.js has a `require('pg')`, `require('express')`, or `require('axios')`
  - use-cases/*.js does `new PostgresXRepository()` instead of receiving it via constructor
  - Route handler has SQL queries or bcrypt calls directly in it
```

---

## Connected topics

- **116 — Dependency injection in Node** — clean architecture's dependency rule is enforced in practice by injecting concrete infrastructure classes into use cases via their constructors.
- **117 — Repository pattern in Node** — the repository pattern IS the "port" that sits between the use case and infrastructure layers in this architecture.
- **60 — API versioning and structure** — covers the controller/service/router folder conventions that clean architecture's controller and use-case layers build directly on top of.
