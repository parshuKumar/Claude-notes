# 19 — Implementing Interfaces in Classes

## What is this?

The `implements` keyword on a class declaration tells TypeScript: **"this class must satisfy this interface's contract."** If the class is missing any required property or method, or if a method has the wrong signature, TypeScript raises a compile error — before you ever run the code.

```ts
class MyClass implements MyInterface {
  // Must provide every property and method declared in MyInterface
}
```

A class can implement multiple interfaces at once:

```ts
class MyClass implements InterfaceA, InterfaceB, InterfaceC { }
```

`implements` is purely a **compile-time check**. It doesn't change the JavaScript output at all — it just makes TypeScript verify the class's structure against the interface.

## Why does it matter?

JavaScript classes have no contract enforcement. You can define a `UserRepository` class that's supposed to have a `findByEmail` method, forget to implement it, and only find out at runtime when the method call throws. In large codebases with multiple team members, this is a constant source of subtle bugs.

With `implements`:
- TypeScript verifies the class matches the interface at the point of declaration — not at the call site
- You can swap implementations (in-memory → PostgreSQL → Redis) and TypeScript guarantees every implementation fulfils the same contract
- You document architectural intent: "this class is the concrete implementation of *this* abstraction"
- Dependency injection becomes type-safe: inject `UserRepository` (interface) into a service, and any implementing class is accepted

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no contract enforcement, find bugs at runtime
class InMemoryUserRepo {
  constructor() {
    this.users = [];
  }

  // Forgot to implement findByEmail — no warning
  findById(id) {
    return this.users.find(u => u.id === id) || null;
  }
}

class UserService {
  constructor(repo) {
    this.repo = repo;
  }

  async getUserByEmail(email) {
    // This crashes at runtime — findByEmail doesn't exist
    return this.repo.findByEmail(email);
  }
}
```

```ts
// TypeScript — contract enforced at compile time
interface UserRepository {
  findById(id: number): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(data: CreateUserBody): Promise<User>;
  update(id: number, data: UpdateUserBody): Promise<User | null>;
  delete(id: number): Promise<boolean>;
}

// ❌ Missing findByEmail — TypeScript errors immediately:
class InMemoryUserRepo implements UserRepository {
  private users: User[] = [];

  async findById(id: number): Promise<User | null> {
    return this.users.find(u => u.id === id) ?? null;
  }
  // ❌ Class 'InMemoryUserRepo' incorrectly implements interface 'UserRepository'
  //    Property 'findByEmail' is missing
}

// ✅ All methods implemented:
class InMemoryUserRepo implements UserRepository {
  private users: User[] = [];

  async findById(id: number): Promise<User | null> { /* ... */ return null; }
  async findByEmail(email: string): Promise<User | null> { /* ... */ return null; }
  async create(data: CreateUserBody): Promise<User> { /* ... */ return {} as User; }
  async update(id: number, data: UpdateUserBody): Promise<User | null> { /* ... */ return null; }
  async delete(id: number): Promise<boolean> { /* ... */ return false; }
}
```

---

## Syntax

```ts
// ── SINGLE implements ──────────────────────────────────────────────────────
class PostgresUserRepository implements UserRepository {
  // must satisfy all of UserRepository
}

// ── MULTIPLE implements ─────────────────────────────────────────────────────
class UserService implements IUserService, IHealthCheckable, IDisposable {
  // must satisfy all three interfaces
}

// ── implements + extends ────────────────────────────────────────────────────
class AdminUserService extends BaseService implements IUserService, IAuditLogger {
  // inherits from BaseService AND must satisfy both interfaces
}

// ── Class property implementing an interface property ─────────────────────
interface Config {
  readonly host: string;
  port: number;
}

class AppConfig implements Config {
  readonly host: string = "localhost";   // public — interface props are always public
  port: number = 3000;
}

// ── Class with constructor implementing interface ──────────────────────────
interface Logger {
  info(msg: string): void;
  error(msg: string): void;
  warn(msg: string): void;
}

class ConsoleLogger implements Logger {
  constructor(private readonly prefix: string) {}

  info(msg: string): void  { console.log(`[INFO]  [${this.prefix}] ${msg}`); }
  error(msg: string): void { console.error(`[ERROR] [${this.prefix}] ${msg}`); }
  warn(msg: string): void  { console.warn(`[WARN]  [${this.prefix}] ${msg}`); }
}
```

---

## How it works — rule by rule

### `implements` is a check, not inheritance

`implements` does **not** copy anything from the interface into the class. The interface has no runtime existence — it's erased at compile time. `implements` only makes TypeScript verify the class's shape against the interface:

```ts
interface Greeter {
  greet(name: string): string;
}

class UserGreeter implements Greeter {
  // TypeScript checks: does UserGreeter have greet(name: string): string ?
  // If yes → ✅. If no → ❌ compile error.
  greet(name: string): string {
    return `Hello, ${name}!`;
  }
}

// At runtime, the compiled JavaScript has no mention of 'Greeter':
// class UserGreeter {
//   greet(name) { return `Hello, ${name}!`; }
// }
```

### Interface members must be public

Interface properties and methods are implicitly public. A class must implement them as `public` (which is the default). You cannot fulfill an interface contract with a `private` or `protected` member:

```ts
interface UserRepository {
  findById(id: number): Promise<User | null>;
}

class BadRepo implements UserRepository {
  private async findById(id: number): Promise<User | null> { return null; }
  // ❌ Property 'findById' in type 'BadRepo' is not assignable to 'UserRepository'
  //    because it is private
}

class GoodRepo implements UserRepository {
  async findById(id: number): Promise<User | null> { return null; } // ✅ public (default)
}
```

### A class can have MORE than the interface requires

`implements` only checks that the interface's requirements are satisfied. The class can freely add extra properties and methods beyond the interface:

```ts
interface UserRepository {
  findById(id: number): Promise<User | null>;
  create(data: CreateUserBody): Promise<User>;
}

class PostgresUserRepository implements UserRepository {
  // ✅ Required by interface:
  async findById(id: number): Promise<User | null> { /* ... */ return null; }
  async create(data: CreateUserBody): Promise<User> { /* ... */ return {} as User; }

  // ✅ Extra — not in interface, but fine:
  async findManyByIds(ids: number[]): Promise<User[]> { /* ... */ return []; }
  async truncate(): Promise<void> { /* ... */ }
  get connectionPool() { return this.pool; }

  private pool: unknown = null;
}
```

### Implementing multiple interfaces

```ts
interface IUserService {
  getUser(id: number): Promise<User>;
  createUser(data: CreateUserBody): Promise<User>;
}

interface IHealthCheckable {
  healthCheck(): Promise<{ status: "ok" | "degraded"; message: string }>;
}

interface IDisposable {
  dispose(): Promise<void>;
}

class UserService implements IUserService, IHealthCheckable, IDisposable {
  constructor(private readonly repo: UserRepository) {}

  async getUser(id: number): Promise<User> {
    const user = await this.repo.findById(id);
    if (!user) throw new Error(`User ${id} not found`);
    return user;
  }

  async createUser(data: CreateUserBody): Promise<User> {
    return this.repo.create(data);
  }

  async healthCheck(): Promise<{ status: "ok" | "degraded"; message: string }> {
    try {
      await this.repo.findById(0);  // test DB connectivity
      return { status: "ok", message: "UserService healthy" };
    } catch {
      return { status: "degraded", message: "Database unreachable" };
    }
  }

  async dispose(): Promise<void> {
    // clean up connections, timers, etc.
  }
}
```

### `implements` + `extends` together

A class can inherit from a base class AND implement interfaces simultaneously:

```ts
abstract class BaseService {
  protected readonly logger: Logger;

  constructor(logger: Logger) {
    this.logger = logger;
  }

  protected logOperation(op: string): void {
    this.logger.info(`Operation: ${op}`);
  }
}

interface IUserService {
  getUser(id: number): Promise<User>;
  createUser(data: CreateUserBody): Promise<User>;
}

// Inherits from BaseService AND implements IUserService:
class UserService extends BaseService implements IUserService {
  constructor(
    private readonly repo: UserRepository,
    logger: Logger,
  ) {
    super(logger);  // call BaseService constructor
  }

  async getUser(id: number): Promise<User> {
    this.logOperation(`getUser(${id})`);  // from BaseService
    const user = await this.repo.findById(id);
    if (!user) throw new Error(`User ${id} not found`);
    return user;
  }

  async createUser(data: CreateUserBody): Promise<User> {
    this.logOperation("createUser");
    return this.repo.create(data);
  }
}
```

### Using the interface type, not the class type

This is the core payoff: once you `implements` an interface, you should **inject and consume the interface type**, not the concrete class. This is how you achieve loose coupling:

```ts
// ❌ TIGHTLY COUPLED — controller depends on the concrete class:
class UserController {
  constructor(private readonly service: UserService) {}
  // Locked to UserService — can't swap in MockUserService for tests
}

// ✅ LOOSELY COUPLED — controller depends on the interface:
class UserController {
  constructor(private readonly service: IUserService) {}
  // Accepts UserService, MockUserService, or any future implementation
}

// Wiring:
const repo = new InMemoryUserRepository();
const service = new UserService(repo, new ConsoleLogger("UserService"));
const controller = new UserController(service);  // ✅ UserService satisfies IUserService
```

---

## Example 1 — basic

```ts
// Logger interface with two concrete implementations

interface Logger {
  info(msg: string, meta?: Record<string, unknown>): void;
  warn(msg: string, meta?: Record<string, unknown>): void;
  error(msg: string, meta?: Record<string, unknown>): void;
  debug(msg: string, meta?: Record<string, unknown>): void;
}

// ── Console implementation ─────────────────────────────────────────────────
class ConsoleLogger implements Logger {
  constructor(private readonly context: string) {}

  private format(level: string, msg: string, meta?: Record<string, unknown>): string {
    const ts = new Date().toISOString();
    const metaStr = meta ? ` ${JSON.stringify(meta)}` : "";
    return `${ts} [${level}] [${this.context}] ${msg}${metaStr}`;
  }

  info(msg: string, meta?: Record<string, unknown>): void {
    console.log(this.format("INFO", msg, meta));
  }
  warn(msg: string, meta?: Record<string, unknown>): void {
    console.warn(this.format("WARN", msg, meta));
  }
  error(msg: string, meta?: Record<string, unknown>): void {
    console.error(this.format("ERROR", msg, meta));
  }
  debug(msg: string, meta?: Record<string, unknown>): void {
    console.debug(this.format("DEBUG", msg, meta));
  }
}

// ── Silent implementation (for tests) ─────────────────────────────────────
class SilentLogger implements Logger {
  readonly logs: Array<{ level: string; msg: string; meta?: Record<string, unknown> }> = [];

  info(msg: string, meta?: Record<string, unknown>): void  { this.logs.push({ level: "info", msg, meta }); }
  warn(msg: string, meta?: Record<string, unknown>): void  { this.logs.push({ level: "warn", msg, meta }); }
  error(msg: string, meta?: Record<string, unknown>): void { this.logs.push({ level: "error", msg, meta }); }
  debug(msg: string, meta?: Record<string, unknown>): void { this.logs.push({ level: "debug", msg, meta }); }
}

// Consumer accepts the interface — works with both implementations:
class OrderProcessor {
  constructor(private readonly logger: Logger) {}  // Logger interface, not ConsoleLogger

  processOrder(orderId: number): void {
    this.logger.info("Processing order", { orderId });
    // ... processing logic ...
    this.logger.info("Order processed", { orderId });
  }
}

// Production:
const processor = new OrderProcessor(new ConsoleLogger("OrderProcessor"));
// Testing:
const silentLogger = new SilentLogger();
const testProcessor = new OrderProcessor(silentLogger);
testProcessor.processOrder(42);
console.log(silentLogger.logs); // inspect what was logged in tests
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response } from 'express';

// ── Full repository + service pattern with implements ──────────────────────

interface UserRepository {
  findById(id: number): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findAll(page: number, pageSize: number): Promise<{ users: User[]; total: number }>;
  create(data: CreateUserBody): Promise<User>;
  update(id: number, data: UpdateUserBody): Promise<User | null>;
  delete(id: number): Promise<boolean>;
}

interface IUserService {
  getUser(id: number): Promise<User>;
  getUserByEmail(email: string): Promise<User | null>;
  listUsers(page: number, pageSize: number): Promise<{ users: User[]; total: number }>;
  createUser(data: CreateUserBody): Promise<User>;
  updateUser(id: number, data: UpdateUserBody): Promise<User>;
  deleteUser(id: number): Promise<void>;
}

interface IHealthCheckable {
  healthCheck(): Promise<{ healthy: boolean; service: string; message: string }>;
}

// ── Repository implementation ──────────────────────────────────────────────
class InMemoryUserRepository implements UserRepository {
  private store: User[] = [];

  async findById(id: number): Promise<User | null> {
    return this.store.find(u => u.id === id) ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.store.find(u => u.email === email) ?? null;
  }

  async findAll(page: number, pageSize: number): Promise<{ users: User[]; total: number }> {
    const start = (page - 1) * pageSize;
    return {
      users: this.store.slice(start, start + pageSize),
      total: this.store.length,
    };
  }

  async create(data: CreateUserBody): Promise<User> {
    const user: User = {
      id: Date.now(),
      email: data.email,
      name: data.name,
      role: data.role ?? "viewer",
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    this.store.push(user);
    return user;
  }

  async update(id: number, data: UpdateUserBody): Promise<User | null> {
    const user = this.store.find(u => u.id === id);
    if (!user) return null;
    Object.assign(user, data, { updatedAt: new Date() });
    return user;
  }

  async delete(id: number): Promise<boolean> {
    const index = this.store.findIndex(u => u.id === id);
    if (index === -1) return false;
    this.store.splice(index, 1);
    return true;
  }
}

// ── Service implementation ─────────────────────────────────────────────────
class UserService implements IUserService, IHealthCheckable {
  constructor(
    private readonly repo: UserRepository,   // interface type — not the concrete class
    private readonly logger: Logger,
  ) {}

  async getUser(id: number): Promise<User> {
    this.logger.debug("getUser called", { id });
    const user = await this.repo.findById(id);
    if (!user) {
      this.logger.warn("User not found", { id });
      throw new Error(`User ${id} not found`);
    }
    return user;
  }

  async getUserByEmail(email: string): Promise<User | null> {
    return this.repo.findByEmail(email);
  }

  async listUsers(page: number, pageSize: number): Promise<{ users: User[]; total: number }> {
    this.logger.debug("listUsers called", { page, pageSize });
    return this.repo.findAll(page, pageSize);
  }

  async createUser(data: CreateUserBody): Promise<User> {
    this.logger.info("Creating user", { email: data.email });
    const existing = await this.repo.findByEmail(data.email);
    if (existing) throw new Error(`Email ${data.email} already registered`);
    const user = await this.repo.create(data);
    this.logger.info("User created", { userId: user.id });
    return user;
  }

  async updateUser(id: number, data: UpdateUserBody): Promise<User> {
    this.logger.info("Updating user", { id });
    const user = await this.repo.update(id, data);
    if (!user) throw new Error(`User ${id} not found`);
    return user;
  }

  async deleteUser(id: number): Promise<void> {
    this.logger.info("Deleting user", { id });
    const deleted = await this.repo.delete(id);
    if (!deleted) throw new Error(`User ${id} not found`);
  }

  async healthCheck(): Promise<{ healthy: boolean; service: string; message: string }> {
    try {
      await this.repo.findById(-1);  // test the repo is responsive
      return { healthy: true, service: "UserService", message: "OK" };
    } catch (err) {
      return { healthy: false, service: "UserService", message: (err as Error).message };
    }
  }
}

// ── Controller using only interface types ──────────────────────────────────
class UserController {
  constructor(
    private readonly service: IUserService,  // interface — not UserService class
  ) {}

  async getUser(req: Request, res: Response): Promise<void> {
    try {
      const user = await this.service.getUser(parseInt(req.params.id, 10));
      res.json({ success: true, data: user });
    } catch (err) {
      res.status(404).json({ success: false, error: (err as Error).message });
    }
  }

  async createUser(req: Request, res: Response): Promise<void> {
    try {
      const user = await this.service.createUser(req.body as CreateUserBody);
      res.status(201).json({ success: true, data: user });
    } catch (err) {
      res.status(400).json({ success: false, error: (err as Error).message });
    }
  }
}

// ── Wiring — dependency injection by hand ─────────────────────────────────
const logger = new ConsoleLogger("App");
const userRepo = new InMemoryUserRepository();
const userService = new UserService(userRepo, logger);
const userController = new UserController(userService);
// To swap to Postgres: replace InMemoryUserRepository with PostgresUserRepository
// Nothing else changes — the interfaces are the seams.
```

---

## Common mistakes

### Mistake 1 — Implementing with private methods (breaks the public contract)

```ts
interface EventEmitter {
  on(event: string, handler: () => void): void;
  off(event: string, handler: () => void): void;
  emit(event: string): void;
}

class BrokenEmitter implements EventEmitter {
  private on(event: string, handler: () => void): void { }  // ❌ private!
  // Error: Property 'on' in type 'BrokenEmitter' is not assignable to the same
  // property in base type 'EventEmitter' — it is private

  off(event: string, handler: () => void): void { }
  emit(event: string): void { }
}

// ✅ Interface members must be public (the default):
class FixedEmitter implements EventEmitter {
  on(event: string, handler: () => void): void { }   // public — fine
  off(event: string, handler: () => void): void { }
  emit(event: string): void { }
}
```

### Mistake 2 — Injecting the class type instead of the interface type

```ts
interface IEmailService {
  sendWelcomeEmail(to: string): Promise<void>;
}

class SmtpEmailService implements IEmailService {
  async sendWelcomeEmail(to: string): Promise<void> { /* ... */ }
}

// ❌ BAD — locked to the concrete class:
class UserService {
  constructor(private readonly emailService: SmtpEmailService) {}
  // Can't inject MockEmailService in tests — it's not SmtpEmailService
}

// ✅ GOOD — accept the interface:
class UserService {
  constructor(private readonly emailService: IEmailService) {}
  // Accepts SmtpEmailService, MockEmailService, SendGridEmailService, anything
}
```

### Mistake 3 — Forgetting that `implements` doesn't add the interface's types to `this`

```ts
interface Serializable {
  serialize(): string;
}

class User implements Serializable {
  constructor(public name: string) {}

  serialize(): string {
    return JSON.stringify({ name: this.name });
  }
}

// The interface only checks the class shape — it provides no default implementation.
// If you call a method that doesn't exist yet, TypeScript errors immediately:
class Product implements Serializable {
  constructor(public sku: string) {}
  // ❌ Class 'Product' incorrectly implements interface 'Serializable'
  //    Property 'serialize' is missing in type 'Product'
  // The interface doesn't give you serialize() — you must write it yourself.
}
```

---

## Practice exercises

### Exercise 1 — easy

Define a `Cache<T>` interface with:
- `get(key: string): T | null`
- `set(key: string, value: T, ttlMs: number): void`
- `delete(key: string): void`
- `has(key: string): boolean`
- `clear(): void`
- `size(): number`

Write two classes that implement `Cache<T>`:
1. `InMemoryCache<T>` — stores entries with expiry in a `Map<string, { value: T; expiresAt: number }>`
2. `NoOpCache<T>` — a silent do-nothing implementation (get always returns null, all writes are no-ops — useful for disabling caching in tests)

Write a function `getCachedUser(cache: Cache<User>, userId: string): User | null` that uses the `Cache<User>` interface — not either concrete class.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a notification system with the following interfaces:

```
NotificationPayload — type: "email" | "sms" | "push", recipient: string, subject?: string, message: string

INotificationSender interface:
  - send(payload: NotificationPayload): Promise<{ sent: boolean; messageId: string }>
  - supports(type: NotificationPayload["type"]): boolean

INotificationLogger interface:
  - logSent(payload: NotificationPayload, messageId: string): void
  - logFailed(payload: NotificationPayload, error: string): void
  - getSentCount(): number

INotificationService interface:
  - dispatch(payload: NotificationPayload): Promise<void>
  - dispatchMany(payloads: NotificationPayload[]): Promise<{ sent: number; failed: number }>
```

Implement:
1. `ConsoleSender implements INotificationSender` — logs to console, supports all types
2. `InMemoryNotificationLogger implements INotificationLogger` — stores sent logs in an array
3. `NotificationService implements INotificationService` — takes `INotificationSender` and `INotificationLogger` in constructor; `dispatch` calls send, logs result; `dispatchMany` runs all in parallel

```ts
// Write your code here
```

### Exercise 3 — hard

Build a plugin-based HTTP middleware system:

Interfaces:
```
IMiddleware — execute(ctx: RequestContext, next: () => Promise<void>): Promise<void>
IMiddlewareRegistry — register(name: string, middleware: IMiddleware): void; get(name: string): IMiddleware | undefined; getAll(): IMiddleware[]
IRequestPipeline — use(middleware: IMiddleware): IRequestPipeline (fluent); run(ctx: RequestContext): Promise<void>
```

Where `RequestContext` is:
```
{ requestId: string; method: string; path: string; headers: Record<string, string>; body: unknown; userId?: number; startTime: number; logs: string[] }
```

Implement:
1. `MiddlewareRegistry implements IMiddlewareRegistry`
2. `RequestPipeline implements IRequestPipeline` — `run` chains all middlewares calling each one's `execute`, passing a `next` that calls the next middleware in the chain (classic onion/chain-of-responsibility pattern)

Write four concrete middleware classes implementing `IMiddleware`:
- `RequestLogger` — pushes `"[LOG] METHOD PATH"` into `ctx.logs`
- `RequestTimer` — pushes elapsed time into `ctx.logs` after `next()` resolves
- `AuthMiddleware` — reads `ctx.headers["authorization"]`, sets `ctx.userId = 99` if valid, short-circuits (doesn't call `next()`) with a log entry if missing
- `BodyParser` — parses `ctx.body` if it's a string (JSON.parse), stores result back on `ctx.body`, calls `next()`

Build a pipeline with all four, run it twice — once with auth header, once without — and log `ctx.logs` each time.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Meaning |
|--------|---------|
| `class C implements I` | C must satisfy all of I's public contract |
| `class C implements I, J, K` | C must satisfy all three interfaces |
| `class C extends B implements I` | Inherits from B, must also satisfy I |
| Interface props → must be public | Can't fulfill with `private` or `protected` |
| Class can have more than interface | Extra methods/props are fine |
| `implements` is compile-time only | No runtime cost, no JS output change |
| Accept interface type, not class | `constructor(service: IUserService)` not `UserService` |

| Rule | Notes |
|------|-------|
| `implements` = verification only | Interface provides no default implementation |
| Missing method → immediate error | Caught at the class declaration, not at the call site |
| Wrong method signature → error | Parameter types and return type must match |
| Inject interface, not class | Key to swappable implementations and testability |
| Abstract class vs interface | Abstract class can provide default implementations; interface cannot |
| `readonly` in interface | Class property must also be `readonly` |

## Connected topics

- **14 — Interfaces** — the contract being implemented.
- **18 — Extending interfaces** — the interface hierarchy that classes implement.
- **15 — type vs interface** — why `implements` is idiomatic with `interface` but technically works with object-shape `type` aliases too.
- **33 — Classes in TypeScript** — `private`, `protected`, `public`, `abstract` — the full class system.
- **34 — Abstract classes** — abstract classes provide partial implementations; interfaces provide none.
- **35 — Access modifiers** — `private`, `protected`, `public`, `readonly` in the context of `implements`.
