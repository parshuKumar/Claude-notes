# 38 — Class Implementing Interface

## What is this?

When a class uses the `implements` keyword, it is making a **compile-time promise**: "this class will provide every property and method the interface declares." TypeScript then verifies that promise — if the class is missing anything, it's a compile error.

```ts
interface IUserService {
  findById(userId: number): Promise<User | null>;
  createUser(dto: CreateUserDto): Promise<User>;
}

class UserService implements IUserService {
  // TypeScript checks: does UserService have findById and createUser with matching signatures?
  async findById(userId: number): Promise<User | null> { /* ... */ return null; }
  async createUser(dto: CreateUserDto): Promise<User>  { /* ... */ return {} as User; }
}
```

`implements` adds **zero runtime behaviour** — it is erased during compilation. It is a pure type-level check.

## Why does it matter?

In a backend application you often have:
- **Multiple implementations of the same concept** — a `CacheService` backed by Redis in production and an in-memory map in tests
- **Interchangeable adapters** — `EmailService` backed by SendGrid, Mailgun, or SMTP
- **Dependency injection** — a controller receives `IUserRepository`, not `PostgresUserRepository`; swap the implementation without touching the consumer

`implements` is the mechanism that makes TypeScript enforce these contracts at the definition site. Without it, you discover mismatches only at the call site (or at runtime in JS).

A second use: **self-documenting intent**. Writing `class UserService implements IUserService` tells every reader exactly what shape this class is expected to expose — no need to guess from the method list.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no enforcement. The interface is a comment at best:
// "UserRepository should have findById, save, delete" — but nothing stops you from:
class UserRepository {
  async getUserById(id) { /* ... */ }  // wrong name — no error
  // save and delete simply missing — discovered at runtime
}

// DI container injects it — crashes at runtime when consumer calls .findById()
container.register("UserRepository", UserRepository);
```

```ts
// TypeScript — mistake caught at definition time:
interface IUserRepository {
  findById(userId: number): Promise<User | null>;
  save(user: User): Promise<User>;
  delete(userId: number): Promise<void>;
}

class UserRepository implements IUserRepository {
  async getUserById(id: number): Promise<User | null> { return null; } // ← wrong name
  // ❌ Error: Class 'UserRepository' incorrectly implements interface 'IUserRepository'.
  //    Property 'findById' is missing in type 'UserRepository' but required in type 'IUserRepository'.
  //    Property 'save' is missing ...
  //    Property 'delete' is missing ...
}

// ✅ Correct:
class UserRepository implements IUserRepository {
  async findById(userId: number): Promise<User | null> { /* ... */ return null; }
  async save(user: User): Promise<User>                { /* ... */ return user; }
  async delete(userId: number): Promise<void>          { /* ... */ }
}
```

---

## Syntax

```ts
// ── Single interface ───────────────────────────────────────────────────────
class Foo implements IFoo { /* ... */ }

// ── Multiple interfaces (a class can implement many) ──────────────────────
class Foo implements IFoo, IBar, IBaz { /* ... */ }

// ── Combined with extends ──────────────────────────────────────────────────
class Foo extends BaseClass implements IFoo, IBar { /* ... */ }

// ── Abstract class implementing an interface ───────────────────────────────
abstract class BaseRepo implements IRepository<User, number> {
  abstract findById(id: number): Promise<User | null>;  // still abstract — concrete subclass must implement
  async delete(id: number): Promise<void> { /* ... */ } // concrete — shared
}

// ── Class satisfying a readonly interface property ─────────────────────────
interface IConfig { readonly port: number; }
class Config implements IConfig {
  readonly port = 3000;      // ✅ readonly satisfies readonly
  // port = 3000;            // ✅ also works — mutable satisfies readonly (structural typing)
}
```

---

## How it works — concept by concept

### Concept 1 — `implements` checks shape, not implementation

TypeScript only verifies that the class has the right properties and method signatures. It does not check logic or behaviour:

```ts
interface IHashService {
  hash(plainText: string): Promise<string>;
  verify(plainText: string, hash: string): Promise<boolean>;
}

class BrokenHashService implements IHashService {
  // TypeScript is satisfied — shape is correct:
  async hash(plainText: string): Promise<string>                { return plainText; } // ← not actually hashing!
  async verify(plainText: string, hash: string): Promise<boolean> { return true; }    // ← always true!
}
// ✅ Compiles fine — `implements` only checks shape
```

The contract is structural, not behavioural. Tests are the mechanism for checking behaviour.

### Concept 2 — A class can implement multiple interfaces

A class has single inheritance (`extends` one class only) but can implement as many interfaces as needed:

```ts
interface IReadable<T>   { findById(id: number): Promise<T | null>; findAll(): Promise<T[]>; }
interface IWritable<T>   { save(entity: T): Promise<T>; delete(id: number): Promise<void>; }
interface ISearchable<T> { search(query: string): Promise<T[]>; }
interface IAuditable     { lastModifiedBy: string; lastModifiedAt: Date; }

// Implements all four:
class ProductRepository implements IReadable<Product>, IWritable<Product>, ISearchable<Product>, IAuditable {
  lastModifiedBy: string = "system";
  lastModifiedAt: Date   = new Date();

  async findById(id: number): Promise<Product | null>  { /* ... */ return null; }
  async findAll():             Promise<Product[]>       { /* ... */ return []; }
  async save(product: Product): Promise<Product>        { /* ... */ return product; }
  async delete(id: number):    Promise<void>            { /* ... */ }
  async search(query: string): Promise<Product[]>       { /* ... */ return []; }
}
```

### Concept 3 — Implementing interface properties

Interface properties can be `readonly` or mutable. A class can satisfy either with a mutable or readonly property — structural typing is used:

```ts
interface IService {
  readonly serviceName: string;
  status: "active" | "inactive";
}

class AuthService implements IService {
  // readonly property — satisfies 'readonly serviceName':
  readonly serviceName = "AuthService";  // ✅

  // mutable property — satisfies 'status':
  status: "active" | "inactive" = "active";  // ✅
}

// TypeScript allows a mutable class property to satisfy a readonly interface property:
class FlexService implements IService {
  serviceName = "FlexService";  // mutable — still satisfies readonly in the interface
  status: "active" | "inactive" = "active";
}

// But you cannot widen a readonly interface property to mutable through the interface type:
const svc: IService = new FlexService();
svc.serviceName = "other";  // ❌ Error — IService.serviceName is readonly in the interface type
(svc as FlexService).serviceName = "other"; // ✅ — accessing through the concrete mutable type
```

### Concept 4 — The `implements` clause does not give you the interface's properties

A very common mistake: writing `implements` and expecting the properties to be declared automatically. They are not:

```ts
interface IEntity {
  id: number;
  createdAt: Date;
}

// ❌ Common mistake — implements does NOT add properties to the class:
class User implements IEntity {
  // Error: Property 'id' is missing. Property 'createdAt' is missing.
  name: string = "";
}

// ✅ You must declare them yourself:
class User implements IEntity {
  id:        number = 0;
  createdAt: Date   = new Date();
  name:      string = "";
}
```

### Concept 5 — Abstract class + interface together

An abstract class can implement an interface and leave some (or all) methods abstract — forcing concrete subclasses to supply the implementation:

```ts
interface INotificationService {
  send(userId: number, title: string, body: string): Promise<void>;
  sendBatch(userIds: number[], title: string, body: string): Promise<void>;
  readonly channelName: string;
}

// Abstract class satisfies part of the interface, leaves the rest abstract:
abstract class BaseNotificationService implements INotificationService {
  abstract readonly channelName: string;
  abstract send(userId: number, title: string, body: string): Promise<void>;

  // Concrete implementation of sendBatch — built on top of send():
  async sendBatch(userIds: number[], title: string, body: string): Promise<void> {
    await Promise.all(userIds.map(id => this.send(id, title, body)));
  }
}

// Concrete subclass only needs to implement the two remaining abstract members:
class PushNotificationService extends BaseNotificationService {
  readonly channelName = "push";

  async send(userId: number, title: string, body: string): Promise<void> {
    await pushGateway.send({ userId, title, body });
  }
  // sendBatch is inherited from BaseNotificationService — no reimplementation needed ✅
}
```

### Concept 6 — Using the interface type for dependency injection

The power of `implements` is consumed at the call site — accept the interface type, not the concrete class:

```ts
interface ICacheService {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, ttlMs?: number): Promise<void>;
  del(key: string): Promise<void>;
}

class RedisCacheService implements ICacheService {
  constructor(private readonly redisClient: RedisClient) {}
  async get<T>(key: string): Promise<T | null>            { return this.redisClient.get(key); }
  async set<T>(key: string, value: T, ttlMs = 300_000): Promise<void> { await this.redisClient.set(key, value, ttlMs); }
  async del(key: string): Promise<void>                   { await this.redisClient.del(key); }
}

class InMemoryCacheService implements ICacheService {
  private readonly store = new Map<string, { value: unknown; expiresAt: number }>();

  async get<T>(key: string): Promise<T | null> {
    const entry = this.store.get(key);
    if (!entry || Date.now() > entry.expiresAt) return null;
    return entry.value as T;
  }
  async set<T>(key: string, value: T, ttlMs = 300_000): Promise<void> {
    this.store.set(key, { value, expiresAt: Date.now() + ttlMs });
  }
  async del(key: string): Promise<void> { this.store.delete(key); }
}

// Consumer accepts the interface — works with either implementation:
class UserService {
  constructor(
    private readonly userRepo: IUserRepository,
    private readonly cache:    ICacheService,  // ← interface type, not RedisCacheService
  ) {}

  async findById(userId: number): Promise<User | null> {
    const key    = `user:${userId}`;
    const cached = await this.cache.get<User>(key);
    if (cached) return cached;
    const user = await this.userRepo.findById(userId);
    if (user) await this.cache.set(key, user, 60_000);
    return user;
  }
}

// Production:
const userSvc = new UserService(new PostgresUserRepo(db), new RedisCacheService(redisClient));

// Tests:
const userSvc = new UserService(new InMemoryUserRepo(), new InMemoryCacheService());
```

### Concept 7 — When to use abstract class vs interface

```
Does your class need shared state or shared method bodies?
  YES → abstract class (+ optionally implements an interface too)
  NO  → interface

Will there be multiple unrelated implementors?
  YES → interface (e.g. a Logger interface implemented by ConsoleLogger, FileLogger, CloudLogger)
  NO  → either works

Do the implementors share an inheritance relationship?
  YES → abstract class
  NO  → interface

Do you need to check instanceof at runtime?
  YES → abstract class (exists at runtime)
  NO  → interface (erased at runtime)
```

---

## Example 1 — basic

```ts
// Pluggable storage adapters via interface — swap without touching consumers

interface IStorageAdapter {
  read(key: string): Promise<string | null>;
  write(key: string, value: string): Promise<void>;
  delete(key: string): Promise<void>;
  list(prefix: string): Promise<string[]>;
  readonly adapterName: string;
}

// Adapter 1 — local filesystem:
import { readFile, writeFile, unlink, readdir } from "fs/promises";
import { join } from "path";

class LocalFileStorageAdapter implements IStorageAdapter {
  readonly adapterName = "LocalFile";

  constructor(private readonly basePath: string) {}

  async read(key: string): Promise<string | null> {
    try {
      return await readFile(join(this.basePath, key), "utf-8");
    } catch {
      return null;
    }
  }

  async write(key: string, value: string): Promise<void> {
    await writeFile(join(this.basePath, key), value, "utf-8");
  }

  async delete(key: string): Promise<void> {
    await unlink(join(this.basePath, key));
  }

  async list(prefix: string): Promise<string[]> {
    const entries = await readdir(this.basePath);
    return entries.filter(e => e.startsWith(prefix));
  }
}

// Adapter 2 — in-memory (used in tests):
class InMemoryStorageAdapter implements IStorageAdapter {
  readonly adapterName = "InMemory";
  private readonly store = new Map<string, string>();

  async read(key: string): Promise<string | null>  { return this.store.get(key) ?? null; }
  async write(key: string, value: string): Promise<void> { this.store.set(key, value); }
  async delete(key: string): Promise<void>         { this.store.delete(key); }
  async list(prefix: string): Promise<string[]>    { return [...this.store.keys()].filter(k => k.startsWith(prefix)); }
}

// Consumer accepts the interface — works with any adapter:
class ConfigStore {
  constructor(private readonly storage: IStorageAdapter) {}

  async get<T>(key: string): Promise<T | null> {
    const raw = await this.storage.read(key);
    return raw ? (JSON.parse(raw) as T) : null;
  }

  async set<T>(key: string, value: T): Promise<void> {
    await this.storage.write(key, JSON.stringify(value));
  }

  async keys(prefix = ""): Promise<string[]> {
    return this.storage.list(prefix);
  }
}

// Production:
const configStore = new ConfigStore(new LocalFileStorageAdapter("/etc/myapp/config"));

// Tests:
const configStore = new ConfigStore(new InMemoryStorageAdapter());
await configStore.set("database.port", 5432);
const port = await configStore.get<number>("database.port"); // 5432
```

---

## Example 2 — real world backend use case

```ts
// Full DI-wired auth system — every dependency injected via interface

// ── Interfaces (the contracts) ─────────────────────────────────────────────

interface IUserRepository {
  findById(userId: number): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<User>;
}

interface ITokenRepository {
  store(token: RefreshToken): Promise<void>;
  find(tokenValue: string): Promise<RefreshToken | null>;
  revoke(tokenValue: string): Promise<void>;
  revokeAllForUser(userId: number): Promise<void>;
}

interface IPasswordService {
  hash(plainPassword: string): Promise<string>;
  verify(plainPassword: string, hash: string): Promise<boolean>;
}

interface ITokenService {
  signAccessToken(payload: JwtPayload): string;
  signRefreshToken(userId: number): string;
  verifyAccessToken(token: string): JwtPayload;
}

interface ILogger {
  info(message: string, meta?: Record<string, unknown>): void;
  warn(message: string, meta?: Record<string, unknown>): void;
  error(message: string, meta?: Record<string, unknown>): void;
}

// ── Concrete implementations ───────────────────────────────────────────────

class PostgresUserRepository implements IUserRepository {
  constructor(private readonly db: DatabasePool) {}

  async findById(userId: number): Promise<User | null> {
    const [row] = await this.db.query<User>("SELECT * FROM users WHERE id = $1", [userId]);
    return row ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const [row] = await this.db.query<User>(
      "SELECT * FROM users WHERE email = $1",
      [email.toLowerCase()],
    );
    return row ?? null;
  }

  async save(user: User): Promise<User> {
    const [saved] = await this.db.query<User>(
      `INSERT INTO users (id, email, password_hash, role, created_at, updated_at)
       VALUES ($1,$2,$3,$4,$5,$6)
       ON CONFLICT (id) DO UPDATE SET email=EXCLUDED.email, updated_at=now()
       RETURNING *`,
      [user.id, user.email, user.passwordHash, user.role, user.createdAt, user.updatedAt],
    );
    return saved;
  }
}

class BcryptPasswordService implements IPasswordService {
  constructor(private readonly rounds: number = 12) {}

  async hash(plainPassword: string): Promise<string> {
    return bcrypt.hash(plainPassword, this.rounds);
  }

  async verify(plainPassword: string, hash: string): Promise<boolean> {
    return bcrypt.compare(plainPassword, hash);
  }
}

class JwtTokenService implements ITokenService {
  constructor(
    private readonly accessSecret:  string,
    private readonly refreshSecret: string,
    private readonly accessTtlSecs: number = 900,     // 15 min
  ) {}

  signAccessToken(payload: JwtPayload): string {
    return jwt.sign(payload, this.accessSecret, { expiresIn: this.accessTtlSecs });
  }

  signRefreshToken(userId: number): string {
    return jwt.sign({ sub: userId, jti: randomUUID() }, this.refreshSecret, { expiresIn: "30d" });
  }

  verifyAccessToken(token: string): JwtPayload {
    return jwt.verify(token, this.accessSecret) as JwtPayload;
  }
}

// ── Service — accepts only interfaces, never concrete classes ──────────────

class AuthService {
  constructor(
    private readonly userRepo:     IUserRepository,    // ← interface
    private readonly tokenRepo:    ITokenRepository,   // ← interface
    private readonly passwordSvc:  IPasswordService,   // ← interface
    private readonly tokenSvc:     ITokenService,      // ← interface
    private readonly logger:       ILogger,            // ← interface
  ) {}

  async login(email: string, password: string): Promise<AuthTokens> {
    const user = await this.userRepo.findByEmail(email);
    if (!user) {
      this.logger.warn("Login failed — user not found", { email });
      throw new UnauthorizedError("Invalid credentials");
    }

    const valid = await this.passwordSvc.verify(password, user.passwordHash);
    if (!valid) {
      this.logger.warn("Login failed — wrong password", { userId: user.id });
      throw new UnauthorizedError("Invalid credentials");
    }

    const accessToken  = this.tokenSvc.signAccessToken({ sub: user.id, role: user.role });
    const refreshToken = this.tokenSvc.signRefreshToken(user.id);

    await this.tokenRepo.store({ userId: user.id, token: refreshToken, createdAt: new Date() });
    this.logger.info("Login successful", { userId: user.id });

    return { accessToken, refreshToken };
  }

  async logout(refreshToken: string): Promise<void> {
    await this.tokenRepo.revoke(refreshToken);
  }

  async logoutAll(userId: number): Promise<void> {
    await this.tokenRepo.revokeAllForUser(userId);
    this.logger.info("All sessions revoked", { userId });
  }
}

// ── Wiring — production ────────────────────────────────────────────────────

const authService = new AuthService(
  new PostgresUserRepository(db),
  new PostgresTokenRepository(db),
  new BcryptPasswordService(12),
  new JwtTokenService(process.env.JWT_ACCESS_SECRET!, process.env.JWT_REFRESH_SECRET!),
  new ConsoleLogger(),
);

// ── Wiring — tests (all fakes) ─────────────────────────────────────────────

class InMemoryUserRepository implements IUserRepository {
  private readonly users = new Map<number, User>();
  private readonly byEmail = new Map<string, User>();

  async findById(userId: number): Promise<User | null>    { return this.users.get(userId) ?? null; }
  async findByEmail(email: string): Promise<User | null>  { return this.byEmail.get(email) ?? null; }
  async save(user: User): Promise<User>                   {
    this.users.set(user.id, user);
    this.byEmail.set(user.email, user);
    return user;
  }
}

class NoOpLogger implements ILogger {
  info() {}
  warn() {}
  error() {}
}

const testAuthService = new AuthService(
  new InMemoryUserRepository(),
  new InMemoryTokenRepository(),
  new BcryptPasswordService(4),   // fast rounds for tests
  new JwtTokenService("test-secret", "test-refresh-secret"),
  new NoOpLogger(),
);
```

---

## Common mistakes

### Mistake 1 — Expecting `implements` to add properties automatically

```ts
interface ITimestamped {
  createdAt: Date;
  updatedAt: Date;
}

// ❌ Common mistake:
class OrderEntity implements ITimestamped {
  id:     number = 0;
  status: string = "pending";
  // Error: Property 'createdAt' is missing
  // Error: Property 'updatedAt' is missing
}

// ✅ You must explicitly declare them:
class OrderEntity implements ITimestamped {
  id:        number = 0;
  status:    string = "pending";
  createdAt: Date   = new Date();  // ← must be here
  updatedAt: Date   = new Date();  // ← must be here
}
```

### Mistake 2 — Injecting the concrete class instead of the interface

```ts
// ❌ Tightly coupled — cannot swap for tests or alternative implementations:
class OrderService {
  constructor(
    private readonly repo: PostgresOrderRepository,  // ← concrete class
  ) {}
}

// If you want to test OrderService, you must create a real PostgresOrderRepository
// (with a real database connection) — impossible in unit tests.

// ✅ Depend on the interface — loosely coupled:
interface IOrderRepository {
  findById(id: number): Promise<Order | null>;
  save(order: Order): Promise<Order>;
}

class OrderService {
  constructor(
    private readonly repo: IOrderRepository,  // ← interface
  ) {}
}

// Now test with a fake:
class FakeOrderRepo implements IOrderRepository {
  async findById(id: number): Promise<Order | null> { return null; }
  async save(order: Order): Promise<Order>          { return order; }
}

const svc = new OrderService(new FakeOrderRepo()); // ✅ no database needed
```

### Mistake 3 — Implementing the interface on the wrong side of the hierarchy

```ts
// ❌ Implementing a wide interface on a narrow base — subclasses inherit
//    concrete implementations they shouldn't have:
interface IFullRepository<T> {
  findById(id: number): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: number): Promise<void>;
  bulkInsert(entities: T[]): Promise<void>;
  truncate(): Promise<void>;  // ← dangerous operation
}

abstract class BaseRepository<T> implements IFullRepository<T> {
  abstract findById(id: number): Promise<T | null>;
  abstract findAll(): Promise<T[]>;
  abstract save(entity: T): Promise<T>;
  abstract delete(id: number): Promise<void>;
  abstract bulkInsert(entities: T[]): Promise<void>;
  abstract truncate(): Promise<void>;  // ← every subclass must implement this dangerous method
}

// ✅ Use Interface Segregation — split the interface:
interface IReadRepository<T>  { findById(id: number): Promise<T | null>; findAll(): Promise<T[]>; }
interface IWriteRepository<T> { save(entity: T): Promise<T>; delete(id: number): Promise<void>; }
interface IAdminRepository<T> extends IReadRepository<T>, IWriteRepository<T> {
  bulkInsert(entities: T[]): Promise<void>;
  truncate(): Promise<void>;
}

// Consumer only asks for what it needs:
class UserService {
  constructor(private readonly repo: IReadRepository<User>) {} // read-only — safe
}
class AdminUserService {
  constructor(private readonly repo: IAdminRepository<User>) {} // admin — explicit
}
```

---

## Practice exercises

### Exercise 1 — easy

Define these two interfaces and implement them on a single class:

```ts
interface IReadable {
  findById(id: number): Promise<Record<string, unknown> | null>;
  findAll(limit?: number): Promise<Record<string, unknown>[]>;
}

interface IWritable {
  create(data: Record<string, unknown>): Promise<Record<string, unknown>>;
  update(id: number, data: Partial<Record<string, unknown>>): Promise<Record<string, unknown> | null>;
  remove(id: number): Promise<boolean>;
}
```

1. Implement both on `InMemoryStore` — backed by a `Map<number, Record<string, unknown>>` with an auto-incrementing `id`.
2. Implement `IReadable` only on `ReadOnlyView` — wrap an existing `InMemoryStore` and expose only reads.
3. Show that `ReadOnlyView` cannot be used where `IWritable` is expected.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a pluggable rate-limiter system. Define:

```ts
interface IRateLimiter {
  readonly strategy:   string;
  check(key: string):  Promise<RateLimitResult>;
  reset(key: string):  Promise<void>;
  resetAll():          Promise<void>;
}

interface RateLimitResult {
  allowed:       boolean;
  remaining:     number;
  resetAtMs:     number;
  retryAfterMs:  number | null;
}
```

Implement these two concrete classes:

1. `SlidingWindowRateLimiter implements IRateLimiter` — tracks timestamps of requests in a sliding window; constructor takes `(maxRequests: number, windowMs: number)`
2. `TokenBucketRateLimiter implements IRateLimiter` — refills tokens at a fixed rate; constructor takes `(capacity: number, refillRatePerSec: number)`

Then write a `RateLimitMiddleware` that accepts `IRateLimiter` and produces an Express-style middleware function. Show it working with both implementations.

```ts
// Write your code here
```

### Exercise 3 — hard

Design and fully implement a **plugin system** for an HTTP request pipeline. Each plugin hooks into the pipeline at defined stages:

```ts
interface IRequestPlugin {
  readonly pluginName:    string;
  readonly priority:      number;  // lower number = runs first (0 = highest priority)
  onRequest?(ctx: RequestContext):  Promise<RequestContext | null>; // null = abort
  onResponse?(ctx: ResponseContext): Promise<ResponseContext>;
  onError?(ctx: ErrorContext):       Promise<ErrorContext | void>;
}

interface RequestContext {
  readonly requestId: string;
  readonly method:    string;
  readonly path:      string;
  readonly headers:   Record<string, string>;
  body:               unknown;
  metadata:           Record<string, unknown>;
}

interface ResponseContext {
  readonly requestId: string;
  statusCode:         number;
  body:               unknown;
  headers:            Record<string, string>;
}

interface ErrorContext {
  readonly requestId: string;
  error:              Error;
  statusCode:         number;
}
```

Implement these three plugins:

1. `AuthPlugin implements IRequestPlugin` (priority 0) — reads `Authorization: Bearer <token>` header, decodes it (use a stub), attaches `userId` and `userRole` to `ctx.metadata`. If token missing/invalid, returns `null` (abort).

2. `RequestLoggingPlugin implements IRequestPlugin` (priority 10) — logs `[requestId] METHOD /path` on `onRequest`, logs `[requestId] STATUS durationMs` on `onResponse`, logs `[requestId] ERROR message` on `onError`.

3. `RateLimitPlugin implements IRequestPlugin` (priority 5) — uses `IRateLimiter` (injected via constructor). If rate limit exceeded, aborts by returning `null` with a `429` response attached to metadata.

Then build a `PluginPipeline` class (does not implement any interface — it orchestrates them) that:
- Takes `IRequestPlugin[]` in constructor
- Sorts by `priority` ascending
- Has `process(ctx: RequestContext): Promise<ResponseContext>` — runs plugins in order
- Calls `onRequest` hooks in priority order, `onResponse` hooks in reverse priority order (like middleware), `onError` hooks if anything throws

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Single interface:
class Foo implements IFoo { /* must provide all IFoo members */ }

// Multiple interfaces:
class Foo implements IFoo, IBar, IBaz { /* must provide all of each */ }

// Combined with extends:
class Foo extends Base implements IFoo { /* must provide all IFoo members not in Base */ }

// Abstract class implementing interface (leaves some abstract for subclass):
abstract class AbstractFoo implements IFoo {
  abstract requiredByIFoo(): void;    // still abstract — concrete subclass must implement
  sharedByIFoo(): void { /* ... */ }  // concrete — subclass inherits
}

// Interface type for DI (the key pattern):
// Define:   interface ICacheService { get, set, del }
// Implement: class RedisCacheService implements ICacheService { ... }
// Consume:   constructor(private readonly cache: ICacheService) {}
// Test:      class FakeCacheService implements ICacheService { ... }
```

| Rule | Detail |
|---|---|
| `implements` is compile-time only | Zero runtime cost, erased to JS |
| Does NOT auto-declare properties | You must declare them in the class body |
| A class can implement many interfaces | `implements A, B, C` |
| `extends` and `implements` can coexist | `extends Base implements A, B` |
| Abstract class can partially implement | Leave some members abstract for subclasses |
| Mutable property satisfies readonly interface | Structural compatibility |
| Accept interface type, not concrete class | The core DI pattern |

---

## Connected topics

- **14 — Interfaces** — the shape contract that `implements` enforces.
- **15 — Type vs Interface** — when to use a type alias vs an interface for the contract.
- **19 — Implementing interfaces (earlier intro)** — the first look at `implements` syntax.
- **33 — Classes in TypeScript** — full class syntax; `implements` in context.
- **37 — Abstract classes** — use with `implements` to share partial implementation across subclasses.
- **39 — Static members** — `static` properties are NOT part of an `implements` contract (only instance members are checked).
- **18 — Extending interfaces** — composing interfaces before implementing them on a class.
