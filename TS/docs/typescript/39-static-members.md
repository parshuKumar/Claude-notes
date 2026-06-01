# 39 — Static Members in TypeScript

## What is this?

A **static member** belongs to the **class itself**, not to any instance. You access it through the class name, not through `this` or a created object.

TypeScript adds full type checking to static properties, static methods, static getters/setters, and static blocks — everything that JavaScript's `static` keyword supports, plus a typed layer on top.

```ts
class RequestCounter {
  static count: number = 0;           // static property — shared across all instances
  static increment(): void {          // static method
    RequestCounter.count++;
  }
}

RequestCounter.increment();
RequestCounter.count;  // 1 — accessed on the class, not on an instance
```

## Why does it matter?

Static members serve four distinct backend purposes:

1. **Shared state** — a connection pool size, a global request counter, a registry of handlers
2. **Factory methods** — `User.fromDto(dto)`, `ApiError.notFound()`, `Money.usd(cents)` — controlled construction
3. **Constants** — `HttpStatus.NOT_FOUND`, `UserRole.ADMIN` — named values tied to a class
4. **Singletons** — one shared instance of a service, config, or logger

Without TypeScript, statics are untyped and undocumented — you discover them by reading the source. With TypeScript, the type system enforces access, prevents wrong assignments, and makes autocomplete work.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — statics work but are untyped, undocumented:
class DatabasePool {
  static #instance = null;      // private static — JS only
  static maxConnections = 10;   // no type — could accidentally be set to "ten"

  static getInstance(config) {
    if (!DatabasePool.#instance) {
      DatabasePool.#instance = new DatabasePool(config);
    }
    return DatabasePool.#instance;  // returns unknown type
  }
}

DatabasePool.maxConnections = "ten";       // ✅ JavaScript allows this — silent bug
const pool = DatabasePool.getInstance();   // return type unknown — no autocomplete
```

```ts
// TypeScript — statics are typed, enforced, documented:
class DatabasePool {
  private static instance: DatabasePool | null = null;
  static readonly maxConnections: number = 10;

  private constructor(private readonly config: DbConfig) {}

  static getInstance(config: DbConfig): DatabasePool {
    if (!DatabasePool.instance) {
      DatabasePool.instance = new DatabasePool(config);
    }
    return DatabasePool.instance;  // return type: DatabasePool — full autocomplete
  }
}

DatabasePool.maxConnections = 5;  // ❌ Error: readonly
const pool = DatabasePool.getInstance(config); // type: DatabasePool ✅
new DatabasePool(config);         // ❌ Error: constructor is private
```

---

## Syntax

```ts
class MyClass {

  // ── Static property (typed) ───────────────────────────────────────────
  static count: number = 0;
  static readonly VERSION: string = "1.0.0";
  private static instance: MyClass | null = null;
  protected static registry: Map<string, MyClass> = new Map();

  // ── Static method ─────────────────────────────────────────────────────
  static create(name: string): MyClass { return new MyClass(name); }
  private static validate(name: string): boolean { return name.length > 0; }

  // ── Static getter / setter ────────────────────────────────────────────
  static get instanceCount(): number { return MyClass.count; }
  static set debugMode(value: boolean) { /* ... */ }

  // ── Static block (ES2022 / TS 4.4+) ──────────────────────────────────
  // Runs once when the class is first evaluated — for complex static init:
  static {
    MyClass.count = parseInt(process.env.INITIAL_COUNT ?? "0", 10);
    console.log("MyClass static block executed");
  }

  // ── Instance members (for comparison) ────────────────────────────────
  name: string;
  constructor(name: string) {
    this.name = name;
    MyClass.count++;       // ✅ instance method CAN access static via class name
  }

  getVersion(): string {
    return MyClass.VERSION; // ✅ accessing static from instance method
  }
}

// Access static members via the class name:
MyClass.count;
MyClass.VERSION;
MyClass.create("service");
MyClass.instanceCount;
```

---

## How it works — concept by concept

### Concept 1 — Static properties are shared across all instances

A static property has **one copy** — on the class itself. All instances share it:

```ts
class ApiRateLimiter {
  static requestCount: number = 0;
  static readonly windowMs: number = 60_000;

  constructor(public readonly clientId: string) {
    ApiRateLimiter.requestCount++;  // every new limiter increments the shared counter
  }

  static reset(): void {
    ApiRateLimiter.requestCount = 0;
  }
}

const limiterA = new ApiRateLimiter("client_1");
const limiterB = new ApiRateLimiter("client_2");
const limiterC = new ApiRateLimiter("client_3");

ApiRateLimiter.requestCount;  // 3 — shared across all three instances

limiterA.requestCount; // ❌ Error: Property 'requestCount' does not exist on type 'ApiRateLimiter'
                       // Static properties are NOT on instances — only on the class
```

### Concept 2 — Static members cannot use `T` from a generic class

Generic type parameters belong to *instances*, not to the class itself. Static members cannot reference the class-level generic `T`:

```ts
class Repository<T> {
  static count: number = 0;          // ✅ fine — not using T
  static create<T>(): Repository<T>  // ✅ fine — introduces its own T
    { return new Repository<T>(); }

  // ❌ Cannot use the class-level T in a static member:
  static defaultItem: T = {} as T;           // Error: Static members cannot reference class type parameters
  static findAll(): T[]  { return []; }      // Error: same
}

// Solution: define a new type parameter on the static method itself:
class Repository<T> {
  static empty<U>(): Repository<U> { return new Repository<U>(); }
  // U is independent of the class T — the static method gets its own
}
```

### Concept 3 — Private constructor + static factory = controlled construction

Making the constructor `private` forces all creation through static factory methods. This pattern lets you validate, normalize, or pool instances:

```ts
class UserId {
  private constructor(
    public readonly value: number,
  ) {}

  static from(value: unknown): UserId {
    if (typeof value !== "number" || !Number.isInteger(value) || value <= 0) {
      throw new TypeError(`Invalid userId: ${value}`);
    }
    return new UserId(value);
  }

  static fromString(value: string): UserId {
    return UserId.from(parseInt(value, 10));
  }

  equals(other: UserId): boolean { return this.value === other.value; }
  toString(): string { return String(this.value); }
}

UserId.from(42);       // ✅ UserId { value: 42 }
UserId.from(-1);       // ❌ throws TypeError
UserId.fromString("7"); // ✅ UserId { value: 7 }
new UserId(42);        // ❌ Error: constructor is private
```

### Concept 4 — Singleton pattern with private static instance

```ts
class AppConfig {
  private static _instance: AppConfig | null = null;

  readonly databaseUrl:  string;
  readonly jwtSecret:    string;
  readonly port:         number;
  readonly environment:  "development" | "staging" | "production";

  private constructor() {
    // Called only once — private, only accessible through getInstance():
    this.databaseUrl  = requireEnv("DATABASE_URL");
    this.jwtSecret    = requireEnv("JWT_SECRET");
    this.port         = parseInt(process.env.PORT ?? "3000", 10);
    this.environment  = (process.env.NODE_ENV ?? "development") as "development" | "staging" | "production";
  }

  static getInstance(): AppConfig {
    if (!AppConfig._instance) {
      AppConfig._instance = new AppConfig();
    }
    return AppConfig._instance;
  }

  // For testing — reset the singleton so tests start fresh:
  static _resetForTest(): void {
    AppConfig._instance = null;
  }
}

const config1 = AppConfig.getInstance();
const config2 = AppConfig.getInstance();
config1 === config2;  // true — same object
new AppConfig();      // ❌ Error: private constructor
```

### Concept 5 — Static factory methods (named constructors)

Multiple static factories give you expressive, named construction paths:

```ts
class ApiError extends Error {
  private constructor(
    public readonly statusCode: number,
    message: string,
    public readonly code: string,
  ) {
    super(message);
    this.name = "ApiError";
  }

  // Named factories — each sets the right code and status automatically:
  static badRequest(message: string):  ApiError { return new ApiError(400, message, "BAD_REQUEST"); }
  static unauthorized(message = "Unauthorized"): ApiError { return new ApiError(401, message, "UNAUTHORIZED"); }
  static forbidden(message = "Forbidden"):       ApiError { return new ApiError(403, message, "FORBIDDEN"); }
  static notFound(resource: string):  ApiError { return new ApiError(404, `${resource} not found`, "NOT_FOUND"); }
  static conflict(message: string):   ApiError { return new ApiError(409, message, "CONFLICT"); }
  static internal(message = "Internal server error"): ApiError { return new ApiError(500, message, "INTERNAL"); }

  toResponse(): { statusCode: number; message: string; code: string } {
    return { statusCode: this.statusCode, message: this.message, code: this.code };
  }
}

throw ApiError.notFound("User");        // ApiError: User not found (404)
throw ApiError.unauthorized();          // ApiError: Unauthorized (401)
throw ApiError.badRequest("Invalid email format");
```

### Concept 6 — Static constants / enum-like named values

```ts
class HttpStatus {
  // Cannot be instantiated — all members are static:
  private constructor() {}

  static readonly OK:                    200 = 200;
  static readonly CREATED:               201 = 201;
  static readonly NO_CONTENT:            204 = 204;
  static readonly BAD_REQUEST:           400 = 400;
  static readonly UNAUTHORIZED:          401 = 401;
  static readonly FORBIDDEN:             403 = 403;
  static readonly NOT_FOUND:             404 = 404;
  static readonly CONFLICT:              409 = 409;
  static readonly UNPROCESSABLE_ENTITY:  422 = 422;
  static readonly INTERNAL_SERVER_ERROR: 500 = 500;

  static isSuccess(code: number): boolean  { return code >= 200 && code < 300; }
  static isClientError(code: number): boolean { return code >= 400 && code < 500; }
  static isServerError(code: number): boolean { return code >= 500; }
}

res.status(HttpStatus.CREATED).json(user);         // 201
HttpStatus.isSuccess(HttpStatus.OK);               // true
HttpStatus.isClientError(HttpStatus.NOT_FOUND);    // true
```

### Concept 7 — Static blocks for complex initialization

Static blocks (TypeScript 4.4+ / ES2022) run once when the class is first loaded — useful for initialization that needs try/catch or multi-step logic:

```ts
class FeatureFlags {
  static readonly flags: ReadonlyMap<string, boolean>;
  static readonly loadedAt: Date;

  static {
    const rawFlags: Record<string, boolean> = {};
    try {
      const envFlags = process.env.FEATURE_FLAGS;
      if (envFlags) Object.assign(rawFlags, JSON.parse(envFlags));
    } catch {
      console.warn("Failed to parse FEATURE_FLAGS — using defaults");
    }
    FeatureFlags.flags    = new Map(Object.entries(rawFlags));
    FeatureFlags.loadedAt = new Date();
  }

  static isEnabled(flag: string): boolean {
    return FeatureFlags.flags.get(flag) ?? false;
  }
}

FeatureFlags.isEnabled("new_checkout_flow");  // true / false
```

### Concept 8 — `typeof ClassName` for typing the class itself (not instances)

When you pass a class as a value, use `typeof ClassName` as the type:

```ts
class ServiceBase {
  constructor(public readonly name: string) {}
  static create(name: string): ServiceBase { return new ServiceBase(name); }
}

// typeof ServiceBase = the constructor type (the class itself)
// ServiceBase        = the instance type

function registerService(ServiceClass: typeof ServiceBase, name: string): ServiceBase {
  return ServiceClass.create(name);  // calls the static factory
}

registerService(ServiceBase, "auth");    // ✅
registerService(new ServiceBase("x"), "auth"); // ❌ Error: instance, not class
```

---

## Example 1 — basic

```ts
// Request ID generator — static counter + static factory

class RequestId {
  private static counter: number = 0;
  private static readonly prefix: string = "req";

  readonly id:        string;
  readonly sequence:  number;
  readonly createdAt: Date;

  private constructor(sequence: number) {
    this.sequence  = sequence;
    this.createdAt = new Date();
    this.id        = `${RequestId.prefix}_${sequence.toString().padStart(8, "0")}`;
  }

  static generate(): RequestId {
    return new RequestId(++RequestId.counter);
  }

  static get totalGenerated(): number {
    return RequestId.counter;
  }

  static reset(): void {
    RequestId.counter = 0;
  }

  toString(): string { return this.id; }
}

const id1 = RequestId.generate(); // req_00000001
const id2 = RequestId.generate(); // req_00000002
const id3 = RequestId.generate(); // req_00000003

RequestId.totalGenerated;  // 3
id1.id;                    // "req_00000001"
id1.sequence;              // 1

new RequestId(1);          // ❌ Error: private constructor

// Registering event types with a static registry:
class EventRegistry {
  private static readonly handlers: Map<string, Set<(payload: unknown) => void>> = new Map();

  static on(eventType: string, handler: (payload: unknown) => void): void {
    if (!EventRegistry.handlers.has(eventType)) {
      EventRegistry.handlers.set(eventType, new Set());
    }
    EventRegistry.handlers.get(eventType)!.add(handler);
  }

  static emit(eventType: string, payload: unknown): void {
    EventRegistry.handlers.get(eventType)?.forEach(h => h(payload));
  }

  static off(eventType: string, handler: (payload: unknown) => void): void {
    EventRegistry.handlers.get(eventType)?.delete(handler);
  }

  static get registeredEvents(): string[] {
    return [...EventRegistry.handlers.keys()];
  }

  private constructor() {}  // prevent instantiation — utility class
}

EventRegistry.on("user.created", payload => console.log("New user:", payload));
EventRegistry.emit("user.created", { userId: 1, email: "dev@api.dev.io" });
EventRegistry.registeredEvents;  // ["user.created"]
```

---

## Example 2 — real world backend use case

```ts
// Connection pool singleton + typed HTTP status constants + service factory registry

// ── Part 1: Singleton connection pool ─────────────────────────────────────

interface PoolConfig {
  host:            string;
  port:            number;
  database:        string;
  user:            string;
  password:        string;
  maxConnections?: number;
  idleTimeoutMs?:  number;
}

class ConnectionPool {
  private static _instance: ConnectionPool | null = null;

  readonly maxConnections: number;
  readonly idleTimeoutMs:  number;
  private activeCount:     number = 0;
  private readonly config: PoolConfig;

  private constructor(config: PoolConfig) {
    this.config         = config;
    this.maxConnections = config.maxConnections ?? 10;
    this.idleTimeoutMs  = config.idleTimeoutMs  ?? 30_000;
  }

  static initialize(config: PoolConfig): ConnectionPool {
    if (ConnectionPool._instance) {
      throw new Error("ConnectionPool already initialized — call getInstance() instead");
    }
    ConnectionPool._instance = new ConnectionPool(config);
    return ConnectionPool._instance;
  }

  static getInstance(): ConnectionPool {
    if (!ConnectionPool._instance) {
      throw new Error("ConnectionPool not initialized — call initialize(config) first");
    }
    return ConnectionPool._instance;
  }

  static get isInitialized(): boolean {
    return ConnectionPool._instance !== null;
  }

  static _resetForTest(): void {
    ConnectionPool._instance = null;
  }

  async query<T>(sql: string, params: unknown[] = []): Promise<T[]> {
    this.activeCount++;
    try {
      // actual pool query — omitted
      return [] as T[];
    } finally {
      this.activeCount--;
    }
  }

  get stats(): { active: number; max: number; host: string } {
    return { active: this.activeCount, max: this.maxConnections, host: this.config.host };
  }
}

// ── Part 2: HttpStatus constants class ────────────────────────────────────

class HttpStatus {
  private constructor() {}

  static readonly OK                    = 200 as const;
  static readonly CREATED               = 201 as const;
  static readonly ACCEPTED              = 202 as const;
  static readonly NO_CONTENT            = 204 as const;
  static readonly BAD_REQUEST           = 400 as const;
  static readonly UNAUTHORIZED          = 401 as const;
  static readonly FORBIDDEN             = 403 as const;
  static readonly NOT_FOUND             = 404 as const;
  static readonly CONFLICT              = 409 as const;
  static readonly UNPROCESSABLE_ENTITY  = 422 as const;
  static readonly TOO_MANY_REQUESTS     = 429 as const;
  static readonly INTERNAL_SERVER_ERROR = 500 as const;
  static readonly SERVICE_UNAVAILABLE   = 503 as const;

  static isSuccess(code: number): boolean        { return code >= 200 && code < 300; }
  static isRedirect(code: number): boolean       { return code >= 300 && code < 400; }
  static isClientError(code: number): boolean    { return code >= 400 && code < 500; }
  static isServerError(code: number): boolean    { return code >= 500; }

  static text(code: number): string {
    const map: Record<number, string> = {
      200: "OK", 201: "Created", 204: "No Content",
      400: "Bad Request", 401: "Unauthorized", 403: "Forbidden",
      404: "Not Found", 409: "Conflict", 422: "Unprocessable Entity",
      429: "Too Many Requests", 500: "Internal Server Error", 503: "Service Unavailable",
    };
    return map[code] ?? "Unknown";
  }
}

// ── Part 3: Service factory registry ──────────────────────────────────────

type ServiceFactory<T> = () => T;

class ServiceLocator {
  private static readonly factories = new Map<string, ServiceFactory<unknown>>();
  private static readonly singletons = new Map<string, unknown>();

  private constructor() {}

  static register<T>(key: string, factory: ServiceFactory<T>, asSingleton = true): void {
    ServiceLocator.factories.set(key, factory as ServiceFactory<unknown>);
    if (!asSingleton) ServiceLocator.singletons.delete(key);
  }

  static resolve<T>(key: string): T {
    if (ServiceLocator.singletons.has(key)) {
      return ServiceLocator.singletons.get(key) as T;
    }
    const factory = ServiceLocator.factories.get(key);
    if (!factory) throw new Error(`Service not registered: ${key}`);
    const instance = factory() as T;
    ServiceLocator.singletons.set(key, instance);
    return instance;
  }

  static get registeredKeys(): string[] {
    return [...ServiceLocator.factories.keys()];
  }

  static _resetForTest(): void {
    ServiceLocator.factories.clear();
    ServiceLocator.singletons.clear();
  }
}

// ── Wiring ─────────────────────────────────────────────────────────────────

// App startup:
ConnectionPool.initialize({
  host: "localhost", port: 5432, database: "appdb",
  user: "app", password: process.env.DB_PASSWORD!,
});

ServiceLocator.register("UserRepository", () => new UserRepository(ConnectionPool.getInstance()));
ServiceLocator.register("UserService",    () => new UserService(ServiceLocator.resolve("UserRepository")));

// Usage anywhere in the app:
const userService = ServiceLocator.resolve<UserService>("UserService");
res.status(HttpStatus.CREATED).json(await userService.createUser(req.body, requestId));
```

---

## Common mistakes

### Mistake 1 — Accessing a static member through an instance

```ts
class SessionStore {
  static readonly maxSessions: number = 1000;
  static activeCount: number = 0;

  constructor() { SessionStore.activeCount++; }
}

const store = new SessionStore();

store.maxSessions;    // ❌ Error: Property 'maxSessions' does not exist on type 'SessionStore'
store.activeCount;    // ❌ Error: Property 'activeCount' does not exist on type 'SessionStore'

SessionStore.maxSessions;  // ✅ 1000
SessionStore.activeCount;  // ✅ 1
```

### Mistake 2 — Using the generic type parameter `T` in a static member

```ts
class Queue<T> {
  private items: T[] = [];

  // ❌ Cannot reference class-level T in static context:
  static defaultQueue: Queue<T> = new Queue<T>();  // Error: cannot reference type parameter T

  // ✅ Introduce a new type parameter on the static method:
  static empty<U>(): Queue<U> { return new Queue<U>(); }

  // ✅ Or use a concrete type:
  static emptyStringQueue(): Queue<string> { return new Queue<string>(); }
}
```

### Mistake 3 — Mutating a `readonly` static from outside the class

```ts
class AppConfig {
  static readonly MAX_RETRIES: number = 3;
  static readonly BASE_URL: string = "https://api.dev.io";
}

AppConfig.MAX_RETRIES = 10;  // ❌ Error: Cannot assign to 'MAX_RETRIES' because it is a read-only property
AppConfig.BASE_URL    = "http://localhost"; // ❌ same error

// ✅ readonly statics are constants — by design, they cannot be changed after initialization
// If you need to change them (e.g. in tests), don't use readonly:
class AppConfig {
  static MAX_RETRIES: number = 3;  // mutable static — can be changed
}
AppConfig.MAX_RETRIES = 5; // ✅ works
```

---

## Practice exercises

### Exercise 1 — easy

1. Build a `Logger` class with:
   - Static property `level: "debug" | "info" | "warn" | "error"` (default `"info"`)
   - Static `readonly` property `VERSION = "1.0.0"`
   - Static counter `totalLogged: number = 0` — incremented every time any log method is called
   - Instance property `readonly context: string` — set in constructor
   - Instance methods `debug`, `info`, `warn`, `error` — each checks `Logger.level` to decide whether to print, then increments `Logger.totalLogged` and prints `[LEVEL] [context] message`
   - Static method `setLevel(level)` — changes the log level
   - Static getter `stats(): { totalLogged: number; level: string }`

2. Show these work correctly:
   ```ts
   const apiLogger  = new Logger("API");
   const dbLogger   = new Logger("DB");
   apiLogger.debug("Starting server");  // not printed — level is "info"
   Logger.setLevel("debug");
   apiLogger.debug("Starting server");  // now printed
   dbLogger.info("Connected");
   Logger.totalLogged;  // 2 (only the ones that were actually logged)
   ```

```ts
// Write your code here
```

### Exercise 2 — medium

Implement a `JobQueue` class for a background task system:

- Static `readonly maxConcurrent: number` — set once via `JobQueue.configure({ maxConcurrent })` before first use
- Static `activeJobs: number = 0` — tracks currently running jobs across all queues
- Static `completedJobs: number = 0` — total completed across all queues
- Static `configure(options: { maxConcurrent: number; retryDelayMs?: number }): void` — must be called once before first enqueue; throws if called twice
- Static `get globalStats(): { active: number; completed: number; maxConcurrent: number }`
- Instance property `readonly queueName: string`
- Instance `enqueue<T>(job: () => Promise<T>): Promise<T>` — runs the job if below `maxConcurrent`, otherwise waits; increments/decrements `activeJobs` and `completedJobs`
- Instance `get pendingCount(): number`

Test:
```ts
JobQueue.configure({ maxConcurrent: 3 });
const emailQueue = new JobQueue("email");
const reportQueue = new JobQueue("reports");
await emailQueue.enqueue(() => sendEmail("user@example.com", "Welcome!"));
JobQueue.globalStats;  // { active: 0, completed: 1, maxConcurrent: 3 }
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a fully type-safe **static service registry** that combines generics with statics:

```ts
// Target API:
ServiceRegistry.register("userService",    UserService,    [userRepo, logger]);
ServiceRegistry.register("orderService",   OrderService,   [orderRepo, userRepo, eventBus]);
ServiceRegistry.register("emailService",   EmailService,   [smtpConfig]);

const userSvc  = ServiceRegistry.resolve<UserService>("userService");
const orderSvc = ServiceRegistry.resolve<OrderService>("orderService");

ServiceRegistry.has("emailService");   // true
ServiceRegistry.has("unknownService"); // false
ServiceRegistry.keys();               // ["userService", "orderService", "emailService"]
ServiceRegistry.dispose("emailService");  // removes it
ServiceRegistry.disposeAll();             // clears everything
```

Requirements:
- `register<T>(key: string, Constructor: new (...args: unknown[]) => T, args: unknown[]): void`
  - Stores the constructor + args
  - Throws if key is already registered
- `resolve<T>(key: string): T`
  - Lazily constructs the instance on first call (singleton pattern — same instance returned on subsequent calls)
  - Throws with a clear message if key is not registered
  - The returned type must be `T` — no `unknown` leaking
- `has(key: string): boolean`
- `keys(): string[]`
- `dispose(key: string): void` — removes both the factory and any cached singleton
- `disposeAll(): void` — clears everything
- `_resetForTest(): void` — alias for `disposeAll()` (clear naming intent for test usage)
- Static block that reads `PREREGISTERED_SERVICES` from an environment variable (JSON string) and logs the count of preregistered services if any exist

Demonstrate the full lifecycle: register → resolve (first call constructs) → resolve again (returns same instance) → dispose → resolve throws.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
class Foo {
  // Static property:
  static count: number = 0;
  static readonly VERSION = "1.0.0" as const;
  private static instance: Foo | null = null;

  // Static method:
  static create(): Foo { return new Foo(); }
  static get instanceCount(): number { return Foo.count; }

  // Static block (complex init):
  static { Foo.count = parseInt(process.env.INITIAL ?? "0", 10); }

  // Access from instance method — use class name, not `this`:
  getVersion(): string { return Foo.VERSION; } // ✅
}

// Access statics through the class, never through an instance:
Foo.count;    // ✅
Foo.create(); // ✅
new Foo().count; // ❌ Error — not on instances

// Cannot use class-level generic T in statics:
class Box<T> {
  static empty<U>(): Box<U> { return new Box<U>(); }  // ✅ own type param
  static defaultItem: T;  // ❌ Error
}

// typeof ClassName = the class/constructor type (not the instance type):
function build(C: typeof Foo): Foo { return C.create(); }
```

| Pattern | What it does |
|---|---|
| `static readonly CONSTANT = value` | Named constant tied to the class |
| `private static instance` + `static getInstance()` | Singleton |
| `private constructor()` + `static create()` | Controlled construction / factory |
| `static count` incremented in constructor | Instance counter |
| `static register(key, factory)` | Service / handler registry |
| `static block { }` | Complex one-time initialization |

---

## Connected topics

- **33 — Classes in TypeScript** — full class syntax; static in context of all class features.
- **34 — Access modifiers** — `private static`, `protected static`, `public static` — all valid combinations.
- **35 — Readonly class properties** — `static readonly` for constants.
- **29 — Generic classes** — static members cannot reference the class-level `T`; they must introduce their own.
- **37 — Abstract classes** — `abstract` and `static` can coexist but abstract static methods are not allowed (only static concrete methods on abstract classes).
- **38 — Class implementing interface** — `static` members are NOT checked by `implements` — only instance members are.
- **40 — Type narrowing** — `typeof ClassName` vs `instanceof ClassName` — static vs runtime type checks.
