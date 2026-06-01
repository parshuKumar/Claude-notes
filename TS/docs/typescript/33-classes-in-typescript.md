# 33 — Classes in TypeScript

## What is this?

TypeScript classes are JavaScript classes with **type annotations layered on top**. Everything you know about JS classes — `constructor`, `extends`, `super`, getters/setters, static members, private fields (`#`) — works exactly the same. TypeScript adds:

- **Typed properties** — declare property types before the constructor
- **Access modifiers** — `public`, `protected`, `private` (compile-time enforcement)
- **`readonly`** — prevents reassignment after construction
- **Parameter properties** — shorthand to declare and assign constructor parameters in one line
- **`override`** keyword — explicitly marks a method as overriding a parent method
- **`abstract`** classes and methods — covered in topic 34

TypeScript classes compile to plain JavaScript classes — no runtime overhead.

## Why does it matter?

You already use JS classes. TypeScript makes them safer in three ways:

1. **Property declarations** — TypeScript requires you to declare every property your class has. No more accidental `this.typo = value` silently creating new properties.
2. **Access modifiers** — `private` and `protected` enforce encapsulation at compile time (plus JS `#` for true runtime privacy).
3. **Constructor typing** — parameters and return types are fully checked, constructor overloads are possible.

Since your background is OOP and Node.js with classes, this topic will feel very natural — it's your existing knowledge with type safety added.

## The JavaScript way vs the TypeScript way

```js
// JavaScript class — no type enforcement:
class UserService {
  constructor(userRepo, logger) {
    this.userRepo = userRepo;  // could be anything
    this.logger = logger;      // could be anything
    this.cache = {};           // silent property creation
  }

  async getUser(id) {
    return this.userRepo.findById(id);  // no guarantee this exists
  }
}
```

```ts
// TypeScript class — every property, parameter, and return type is enforced:
class UserService {
  private readonly cache = new Map<number, User>();

  constructor(
    private readonly userRepo: UserRepository,  // parameter property shorthand
    private readonly logger: Logger,
  ) {}

  async getUser(id: number): Promise<User | null> {
    if (this.cache.has(id)) return this.cache.get(id)!;
    const user = await this.userRepo.findById(id);  // ✅ TypeScript knows what findById returns
    if (user) this.cache.set(id, user);
    return user;
  }
}
```

---

## Syntax

```ts
// ── Full class anatomy ────────────────────────────────────────────────────
class OrderService {
  // Property declarations (required in TypeScript):
  private orders: Order[] = [];
  protected logger: Logger;
  public readonly serviceId: string;
  static instanceCount = 0;

  // Constructor with parameter properties (declare + assign in one step):
  constructor(
    private readonly orderRepo: OrderRepository,  // private + readonly in one
    protected readonly config: AppConfig,          // protected + readonly
    public readonly serviceName: string,           // public + readonly
  ) {
    this.logger = new ConsoleLogger(serviceName);
    this.serviceId = `svc_${Date.now()}`;
    OrderService.instanceCount++;
  }

  // Public method:
  async getOrder(id: number): Promise<Order | null> {
    return this.orderRepo.findById(id);
  }

  // Protected method — accessible in subclasses:
  protected logAction(action: string, orderId: number): void {
    this.logger.info(`[${this.serviceName}] ${action} orderId=${orderId}`);
  }

  // Private method — only accessible in this class:
  private validateOrder(order: Order): boolean {
    return order.items.length > 0 && order.totalCents > 0;
  }

  // Getter:
  get orderCount(): number {
    return this.orders.length;
  }

  // Setter with validation:
  set maxRetries(value: number) {
    if (value < 1 || value > 10) throw new RangeError("maxRetries must be 1-10");
    this._maxRetries = value;
  }
  private _maxRetries = 3;

  // Static method:
  static resetCount(): void {
    OrderService.instanceCount = 0;
  }

  // Override keyword (TypeScript 4.3+):
  override toString(): string {
    return `${this.serviceName}(orders=${this.orders.length})`;
  }
}
```

---

## How it works — concept by concept

### Property declarations

TypeScript requires you to declare all class properties before using them. This prevents silent dynamic property creation:

```ts
class Session {
  // ✅ Declared — TypeScript knows these exist:
  userId: number;
  token: string;
  expiresAt: Date;
  createdAt: Date;

  constructor(userId: number, token: string, ttlMs: number) {
    this.userId    = userId;
    this.token     = token;
    this.expiresAt = new Date(Date.now() + ttlMs);
    this.createdAt = new Date();
    this.ipAddress = "127.0.0.1";  // ❌ Error: Property 'ipAddress' does not exist on type 'Session'
  }
}
```

**Definite assignment assertion** (`!`) — when you initialize in a method rather than directly:
```ts
class DbConnection {
  private client!: PgClient;  // ! tells TS: "trust me, this will be set before use"

  async connect(): Promise<void> {
    this.client = await createClient();  // initialized here, not in constructor
  }

  async query<T>(sql: string): Promise<T[]> {
    return this.client.query(sql);  // TypeScript won't complain about client being unset
  }
}
```

### Access modifiers

TypeScript has three access modifiers — all compile-time only (erased in JS output):

```ts
class UserAccount {
  public  id: number;       // visible everywhere (default — same as no modifier)
  protected email: string;  // visible in this class and subclasses
  private passwordHash: string;  // visible only in this class

  constructor(id: number, email: string, passwordHash: string) {
    this.id           = id;
    this.email        = email;
    this.passwordHash = passwordHash;
  }

  // Only this class can access passwordHash:
  validatePassword(input: string): boolean {
    return this.passwordHash === hash(input);  // ✅
  }
}

class AdminAccount extends UserAccount {
  getEmailForAdmin(): string {
    return this.email;        // ✅ protected — accessible in subclass
  }
  getPasswordHash(): string {
    return this.passwordHash; // ❌ Error: private — not accessible in subclass
  }
}

const account = new UserAccount(1, "p@dev.io", "hashed");
account.id;           // ✅ public
account.email;        // ❌ Error: protected — not accessible outside class hierarchy
account.passwordHash; // ❌ Error: private
```

### JavaScript `#` private vs TypeScript `private`

TypeScript's `private` is compile-time only — it's erased in the output and accessible at runtime. JavaScript's `#` is true runtime privacy:

```ts
class PaymentProcessor {
  // TypeScript private — erased in output, accessible at runtime via (obj as any).apiKey:
  private apiKey: string;

  // JavaScript native private — truly inaccessible at runtime:
  #secretKey: string;

  constructor(apiKey: string, secretKey: string) {
    this.apiKey    = apiKey;
    this.#secretKey = secretKey;
  }
}

const p = new PaymentProcessor("pub_key", "secret");
(p as any).apiKey;    // ✅ accessible at runtime — TypeScript private is NOT runtime private
(p as any).#secretKey; // ❌ SyntaxError at runtime — true private field
```

Use `#` when you need runtime privacy (security-sensitive data). Use `private` when compile-time checking is sufficient.

### Parameter properties — the most common TypeScript class pattern

Parameter properties let you declare and assign class properties directly in the constructor signature:

```ts
// Without parameter properties — verbose:
class UserService {
  private readonly userRepo: UserRepository;
  private readonly cache: Cache<User>;
  public readonly serviceName: string;

  constructor(userRepo: UserRepository, cache: Cache<User>, serviceName: string) {
    this.userRepo    = userRepo;
    this.cache       = cache;
    this.serviceName = serviceName;
  }
}

// ✅ With parameter properties — concise:
class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly cache: Cache<User>,
    public readonly serviceName: string,
  ) {}  // body is empty — assignments happen automatically
}

// The two are exactly equivalent. Parameter properties require an access modifier.
```

### `readonly` on class properties

```ts
class AppConfig {
  readonly port: number;
  readonly databaseUrl: string;
  readonly jwtSecret: string;

  constructor(port: number, databaseUrl: string, jwtSecret: string) {
    this.port        = port;         // ✅ can assign in constructor
    this.databaseUrl = databaseUrl;
    this.jwtSecret   = jwtSecret;
  }
}

const config = new AppConfig(3000, "postgres://...", "s3cr3t");
config.port = 8080;  // ❌ Error: Cannot assign to 'port' because it is a read-only property
```

### `override` keyword — safe method overriding

The `override` keyword makes it an error if the parent class doesn't have the method you're overriding (prevents silent failures when the parent renames a method):

```ts
class BaseService {
  protected log(message: string): void {
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
}

class UserService extends BaseService {
  // ✅ Correctly overrides BaseService.log:
  override protected log(message: string): void {
    console.log(`[UserService] ${message}`);
  }

  // ❌ Error: 'logMessage' does not exist in type 'BaseService' — catches typos:
  override logMessage(message: string): void {  // Error! Parent has 'log', not 'logMessage'
    console.log(message);
  }
}

// Enable noImplicitOverride in tsconfig to force override on ALL overriding methods:
// "noImplicitOverride": true
```

### Getters and setters with types

```ts
class Order {
  private _status: OrderStatus = "pending";
  private _totalCents: number;
  private _items: OrderItem[];

  constructor(items: OrderItem[]) {
    this._items = items;
    this._totalCents = items.reduce((sum, item) => sum + item.priceCents * item.quantity, 0);
  }

  // Getter — no params, returns a type:
  get status(): OrderStatus {
    return this._status;
  }

  // Setter — one param, no return type (void is implicit):
  set status(newStatus: OrderStatus) {
    const validTransitions: Record<OrderStatus, OrderStatus[]> = {
      pending:    ["confirmed", "cancelled"],
      confirmed:  ["processing", "cancelled"],
      processing: ["shipped"],
      shipped:    ["delivered"],
      delivered:  [],
      cancelled:  [],
    };
    if (!validTransitions[this._status].includes(newStatus)) {
      throw new Error(`Invalid transition: ${this._status} → ${newStatus}`);
    }
    this._status = newStatus;
  }

  get totalCents(): number { return this._totalCents; }
  get itemCount(): number  { return this._items.length; }

  // Getter only — no setter = effectively readonly from outside:
  get isShipped(): boolean { return this._status === "shipped" || this._status === "delivered"; }
}

const order = new Order([{ priceCents: 1000, quantity: 2, productId: 1 }]);
order.status;               // ✅ "pending"
order.status = "confirmed"; // ✅ valid transition
order.status = "delivered"; // ❌ throws — invalid transition from confirmed
order.totalCents;           // ✅ 2000
order.isShipped = true;     // ❌ Error: Cannot set 'isShipped' — no setter declared
```

### Static members

```ts
class ApiClient {
  private static readonly instances = new Map<string, ApiClient>();
  static requestCount = 0;

  private constructor(
    private readonly baseUrl: string,
    private readonly authToken: string,
  ) {}

  // Singleton factory — reuse instances per baseUrl:
  static getInstance(baseUrl: string, authToken: string): ApiClient {
    if (!ApiClient.instances.has(baseUrl)) {
      ApiClient.instances.set(baseUrl, new ApiClient(baseUrl, authToken));
    }
    return ApiClient.instances.get(baseUrl)!;
  }

  async get<T>(path: string): Promise<T> {
    ApiClient.requestCount++;
    const response = await fetch(`${this.baseUrl}${path}`, {
      headers: { Authorization: this.authToken },
    });
    return response.json() as Promise<T>;
  }

  static resetCount(): void {
    ApiClient.requestCount = 0;
  }
}

const client = ApiClient.getInstance("https://api.dev.io", "Bearer token");
// new ApiClient(...) // ❌ Error: constructor is private — must use getInstance
```

---

## Example 1 — basic

```ts
// Fully typed User model class with validation, access control, and computed properties

type UserRole   = "admin" | "editor" | "viewer";
type UserStatus = "active" | "suspended" | "deleted";

interface UserData {
  id: number;
  name: string;
  email: string;
  role: UserRole;
  status: UserStatus;
  loginCount: number;
  lastLoginAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

class User implements UserData {
  readonly id: number;
  readonly createdAt: Date;
  updatedAt: Date;

  private _name: string;
  private _email: string;
  private _role: UserRole;
  private _status: UserStatus;
  private _loginCount: number;
  private _lastLoginAt: Date | null;

  constructor(data: UserData) {
    this.id           = data.id;
    this._name        = data.name;
    this._email       = data.email;
    this._role        = data.role;
    this._status      = data.status;
    this._loginCount  = data.loginCount;
    this._lastLoginAt = data.lastLoginAt;
    this.createdAt    = data.createdAt;
    this.updatedAt    = data.updatedAt;
  }

  // Getters — read-only access to private state:
  get name(): string             { return this._name; }
  get email(): string            { return this._email; }
  get role(): UserRole           { return this._role; }
  get status(): UserStatus       { return this._status; }
  get loginCount(): number       { return this._loginCount; }
  get lastLoginAt(): Date | null { return this._lastLoginAt; }

  // Computed properties:
  get isActive(): boolean        { return this._status === "active"; }
  get isAdmin(): boolean         { return this._role === "admin"; }
  get displayName(): string      { return `${this._name} <${this._email}>`; }
  get daysSinceLogin(): number | null {
    if (!this._lastLoginAt) return null;
    return Math.floor((Date.now() - this._lastLoginAt.getTime()) / 86_400_000);
  }

  // State mutations with validation:
  updateEmail(newEmail: string): void {
    if (!newEmail.includes("@")) throw new Error(`Invalid email: ${newEmail}`);
    this._email  = newEmail;
    this.updatedAt = new Date();
  }

  promoteToAdmin(): void {
    if (this._status !== "active") throw new Error("Cannot promote non-active user");
    this._role   = "admin";
    this.updatedAt = new Date();
  }

  recordLogin(ipAddress: string): void {
    this._loginCount++;
    this._lastLoginAt = new Date();
    this.updatedAt    = new Date();
  }

  suspend(reason: string): void {
    if (this._status === "deleted") throw new Error("Cannot suspend deleted user");
    this._status  = "suspended";
    this.updatedAt = new Date();
    console.log(`User ${this.id} suspended: ${reason}`);
  }

  // Serialize to plain object for JSON response:
  toPublicJSON(): Omit<UserData, "loginCount"> {
    return {
      id:          this.id,
      name:        this._name,
      email:       this._email,
      role:        this._role,
      status:      this._status,
      lastLoginAt: this._lastLoginAt,
      createdAt:   this.createdAt,
      updatedAt:   this.updatedAt,
    };
  }

  // Static factory from raw DB row:
  static fromDbRow(row: UserData): User {
    return new User(row);
  }
}

// Usage:
const user = User.fromDbRow({
  id: 1, name: "Parsh", email: "p@dev.io", role: "editor",
  status: "active", loginCount: 12, lastLoginAt: new Date("2026-05-01"),
  createdAt: new Date("2026-01-01"), updatedAt: new Date("2026-05-01"),
});

user.isActive;        // ✅ boolean
user.isAdmin;         // ✅ boolean
user.displayName;     // ✅ "Parsh <p@dev.io>"
user.daysSinceLogin;  // ✅ number | null
user.promoteToAdmin();
user.role;            // ✅ "admin"

user._role = "viewer"; // ❌ Error: private property
user.id    = 99;       // ❌ Error: readonly property
```

---

## Example 2 — real world backend use case

```ts
// Full service class with dependency injection pattern

interface Logger {
  info(message: string, meta?: Record<string, unknown>): void;
  error(message: string, error?: unknown): void;
  warn(message: string, meta?: Record<string, unknown>): void;
}

interface UserRepository {
  findById(id: number): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(input: Omit<UserData, "id" | "createdAt" | "updatedAt">): Promise<User>;
  update(id: number, input: Partial<Omit<UserData, "id" | "createdAt">>): Promise<User | null>;
  delete(id: number): Promise<boolean>;
}

interface EmailService {
  sendWelcomeEmail(email: string, name: string): Promise<void>;
  sendPasswordResetEmail(email: string, resetToken: string): Promise<void>;
}

interface CacheService {
  get<T>(key: string): T | undefined;
  set<T>(key: string, value: T, ttlMs: number): void;
  delete(key: string): void;
}

class UserManagementService {
  private static readonly USER_CACHE_TTL = 5 * 60 * 1000; // 5 min

  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailService: EmailService,
    private readonly cache: CacheService,
    private readonly logger: Logger,
  ) {}

  // ── Read ────────────────────────────────────────────────────────────────

  async findById(id: number): Promise<User | null> {
    const cacheKey = `user:${id}`;

    const cached = this.cache.get<User>(cacheKey);
    if (cached) {
      this.logger.info("Cache hit", { userId: id });
      return cached;
    }

    const user = await this.userRepo.findById(id);

    if (user) {
      this.cache.set(cacheKey, user, UserManagementService.USER_CACHE_TTL);
    }

    return user;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.userRepo.findByEmail(email.toLowerCase().trim());
  }

  // ── Write ───────────────────────────────────────────────────────────────

  async createUser(input: {
    name: string;
    email: string;
    role: UserRole;
    password: string;
  }): Promise<User> {
    const existing = await this.findByEmail(input.email);
    if (existing) throw new Error(`Email already registered: ${input.email}`);

    const passwordHash = await this.hashPassword(input.password);

    const user = await this.userRepo.create({
      name:         input.name,
      email:        input.email.toLowerCase().trim(),
      role:         input.role,
      status:       "active",
      passwordHash,
      loginCount:   0,
      lastLoginAt:  null,
    });

    await this.emailService.sendWelcomeEmail(user.email, user.name).catch(err => {
      this.logger.warn("Welcome email failed — non-fatal", { userId: user.id, error: err });
    });

    this.logger.info("User created", { userId: user.id, email: user.email });
    return user;
  }

  async updateUser(
    id: number,
    input: Partial<Pick<UserData, "name" | "email" | "role">>,
  ): Promise<User | null> {
    const updated = await this.userRepo.update(id, input);

    if (updated) {
      this.cache.delete(`user:${id}`);  // invalidate cache
      this.logger.info("User updated", { userId: id, fields: Object.keys(input) });
    }

    return updated;
  }

  async deleteUser(id: number): Promise<boolean> {
    const deleted = await this.userRepo.delete(id);

    if (deleted) {
      this.cache.delete(`user:${id}`);
      this.logger.info("User deleted", { userId: id });
    }

    return deleted;
  }

  // ── Auth helpers ─────────────────────────────────────────────────────────

  async recordLogin(userId: number): Promise<void> {
    await this.userRepo.update(userId, {
      lastLoginAt: new Date(),
      loginCount:  (await this.findById(userId))?.loginCount ?? 0 + 1,
    });
    this.cache.delete(`user:${userId}`);
  }

  // ── Private helpers ──────────────────────────────────────────────────────

  private async hashPassword(password: string): Promise<string> {
    // In production: use bcrypt or argon2
    return `hashed_${password}_${Date.now()}`;
  }

  private buildCacheKey(userId: number): string {
    return `user:${userId}`;
  }

  // ── Static utilities ─────────────────────────────────────────────────────

  static validateEmailFormat(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  static sanitizeInput(input: string): string {
    return input.trim().toLowerCase();
  }
}

// Dependency injection — swap implementations freely:
const service = new UserManagementService(
  new PostgresUserRepository(),  // implements UserRepository
  new SendGridEmailService(),    // implements EmailService
  new RedisCache(),              // implements CacheService
  new WinstonLogger(),           // implements Logger
);

// Or for tests:
const testService = new UserManagementService(
  new InMemoryUserRepository(),
  new MockEmailService(),
  new InMemoryCache(),
  new NoopLogger(),
);
```

---

## Common mistakes

### Mistake 1 — Forgetting to declare properties before using them in the constructor

```ts
// ❌ Error: TypeScript requires property declarations:
class ProductService {
  constructor() {
    this.products = [];   // Error: Property 'products' does not exist on type 'ProductService'
    this.cache = {};      // Error: Property 'cache' does not exist
  }
}

// ✅ Declare them above the constructor:
class ProductService {
  private products: Product[] = [];
  private cache = new Map<number, Product>();

  constructor() {
    // Now assignments work — or use parameter properties if passing in via constructor
  }
}
```

### Mistake 2 — TypeScript `private` is NOT runtime private

```ts
class PaymentGateway {
  private apiKey: string;
  constructor(apiKey: string) { this.apiKey = apiKey; }
}

const gateway = new PaymentGateway("sk_live_secret");
console.log((gateway as any).apiKey); // "sk_live_secret" — accessible! TypeScript private is compile-time only

// ✅ For true runtime privacy, use JavaScript native private fields (#):
class PaymentGateway {
  #apiKey: string;
  constructor(apiKey: string) { this.#apiKey = apiKey; }
  charge(amountCents: number): void {
    // use this.#apiKey internally
  }
}
console.log((gateway as any).apiKey); // undefined — # field is truly private
```

### Mistake 3 — Forgetting `override` when shadowing parent methods (with `noImplicitOverride` on)

```ts
class BaseController {
  protected handleError(error: unknown): void {
    console.error("Base error handler:", error);
  }
}

// ❌ With noImplicitOverride: true — silently shadows parent method without 'override':
class UserController extends BaseController {
  protected handleError(error: unknown): void {   // Error with noImplicitOverride: true
    console.error("[UserController]", error);
  }
}

// ✅ Explicit override — TypeScript also validates the parent has this method:
class UserController extends BaseController {
  protected override handleError(error: unknown): void {
    console.error("[UserController]", error);
  }
}

// ✅ Also catches renames — if parent renames 'handleError' to 'handleException',
// the override keyword makes TypeScript error on the child instead of silently calling the old parent method
```

---

## Practice exercises

### Exercise 1 — easy

Build a `Session` class that represents an authenticated user session:

```ts
// Required shape:
class Session {
  readonly sessionId: string;    // auto-generated UUID-like string
  readonly userId: number;
  readonly createdAt: Date;
  private expiresAt: Date;
  private _isRevoked: boolean;
  private _lastAccessedAt: Date;

  // Constructor: takes userId and ttlMs (time-to-live in ms)
  // getter: isExpired — true if expiresAt < now OR _isRevoked
  // getter: isRevoked — exposes _isRevoked
  // getter: lastAccessedAt
  // getter: remainingMs — ms until expiry (0 if expired)
  // method: touch() — updates _lastAccessedAt to now; extends expiresAt by original TTL
  // method: revoke() — sets _isRevoked = true
  // method: toJSON() — returns plain object with all public info (no _isRevoked internals, just isActive: boolean)
  // static: fromToken(token: string) — stub that parses a token string and returns a Session
}
```

Write it in full. Demonstrate: create a session, call `touch()`, `revoke()`, check `isExpired`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a `RateLimiter` class for Express-style API route protection:

```ts
class RateLimiter {
  // Each key (e.g., IP address or userId string) gets a window of requests
  // constructor(maxRequests: number, windowMs: number)
  // method: check(key: string): { allowed: boolean; remaining: number; resetAt: Date }
  // method: reset(key: string): void
  // method: resetAll(): void
  // getter: activeKeys(): string[]  — keys that currently have tracked requests
  // static: perMinute(max: number): RateLimiter
  // static: perHour(max: number): RateLimiter
  // static: perDay(max: number): RateLimiter
}
```

Requirements:
- Track request counts per key with expiry windows
- `check()` should both record the request AND return whether it's allowed
- Expired windows should be automatically cleaned up on `check()`
- `remaining` = maxRequests - currentCount (minimum 0)

Then build an `AuthRateLimiter extends RateLimiter` that adds:
- `recordFailedAttempt(key: string): void` — counts failed login attempts separately
- `isBlocked(key: string): boolean` — true if > 5 failed attempts in the window
- `clearFailures(key: string): void`

```ts
// Write your code here
```

### Exercise 3 — hard

Build a `ServiceContainer` class — a lightweight dependency injection container:

```ts
class ServiceContainer {
  // Register a singleton (created once on first resolve):
  registerSingleton<T>(
    token: string,
    factory: (container: ServiceContainer) => T,
  ): this

  // Register a transient (new instance each resolve):
  registerTransient<T>(
    token: string,
    factory: (container: ServiceContainer) => T,
  ): this

  // Register a pre-created instance:
  registerInstance<T>(token: string, instance: T): this

  // Resolve a registered dependency:
  resolve<T>(token: string): T  // throws if not registered

  // Check if registered:
  has(token: string): boolean

  // Create a child container that inherits registrations from parent:
  createChild(): ServiceContainer

  // List all registered tokens:
  get registeredTokens(): string[]
}
```

Requirements:
- Singletons are created lazily (on first `resolve`) and cached
- Transients are created fresh every `resolve`
- Child container: resolves from own registrations first, falls back to parent
- Circular dependency detection: if resolving A requires A, throw with a clear message including the cycle path

Use it to wire up a full backend service graph:

```ts
const container = new ServiceContainer()
  .registerInstance("config", { dbUrl: "postgres://...", jwtSecret: "s3cr3t", port: 3000 })
  .registerSingleton("db", c => new DatabaseConnection(c.resolve<AppConfig>("config").dbUrl))
  .registerSingleton("cache", c => new InMemoryCache())
  .registerSingleton("logger", c => new ConsoleLogger())
  .registerSingleton("userRepo", c => new UserRepository(c.resolve("db")))
  .registerSingleton("userService", c => new UserManagementService(
    c.resolve("userRepo"),
    c.resolve("cache"),
    c.resolve("logger"),
  ))
  .registerTransient("requestLogger", c => new RequestLogger(c.resolve("logger")));

const userService = container.resolve<UserManagementService>("userService");
const requestLogger1 = container.resolve<RequestLogger>("requestLogger");
const requestLogger2 = container.resolve<RequestLogger>("requestLogger");
// requestLogger1 !== requestLogger2 — transient, new instance each time
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Property declaration:
class Foo {
  name: string;          // public (default)
  protected id: number;  // subclasses only
  private secret: string; // this class only
  #key: string;          // true runtime private (JS native)
  readonly tag: string;  // cannot reassign after construction
  static count = 0;      // shared across all instances
  prop!: string;         // definite assignment — initialized elsewhere
}

// Parameter properties (declare + assign in constructor):
class Bar {
  constructor(
    public name: string,
    protected id: number,
    private readonly secret: string,
  ) {}
}

// Getter / setter:
get value(): number { return this._value; }
set value(n: number) { this._value = n; }

// Override:
override toString(): string { return "..."; }

// Static factory pattern:
static create(data: InputData): MyClass { return new MyClass(data); }
```

| Modifier | Accessible from |
|----------|----------------|
| `public` (default) | Everywhere |
| `protected` | This class + subclasses |
| `private` | This class only (compile-time) |
| `#field` | This class only (runtime) |
| `readonly` | Read anywhere, write only in constructor |
| `static` | On the class itself, not instances |
| `override` | Marks a method as intentionally overriding parent |

---

## Connected topics

- **19 — Implementing interfaces** — `implements` keyword; class fulfilling an interface contract.
- **34 — Abstract classes** — `abstract` modifier; forcing subclasses to implement methods.
- **35 — Access modifiers deep dive** — `public`, `protected`, `private`, `#` in more depth.
- **36 — Static members** — static properties, static methods, static blocks.
- **28 — Generic interfaces** — implementing generic interfaces in classes.
- **29 — Generic classes** — `class Repo<T>` — generic type params on the class declaration.
- **29 → 32 — Generics phase** — parameter properties + generics combine for repository/service patterns.
