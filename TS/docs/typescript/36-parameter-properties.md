# 36 — Parameter Properties

## What is this?

A **parameter property** is a TypeScript-only constructor shorthand that **declares and assigns a class property in a single step** — inside the constructor parameter list itself, using an access modifier or `readonly`.

```ts
// Without parameter properties — the verbose way:
class UserService {
  private readonly userRepo: UserRepository;
  private readonly cache:    CacheService;
  public  readonly name:     string;

  constructor(userRepo: UserRepository, cache: CacheService, name: string) {
    this.userRepo = userRepo;
    this.cache    = cache;
    this.name     = name;
  }
}

// With parameter properties — the concise way (identical compiled output):
class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly cache:    CacheService,
    public  readonly name:     string,
  ) {}
}
```

Both are exactly the same at runtime. Parameter properties are purely a TypeScript authoring convenience — they generate the same JavaScript.

## Why does it matter?

In real TypeScript backend code, classes (especially services and controllers) are almost always constructed with injected dependencies. Without parameter properties, every dependency requires:

1. A property declaration line
2. A constructor parameter
3. An assignment in the constructor body

That's three lines of boilerplate per dependency. A service with five dependencies needs fifteen lines just to receive and store them. With parameter properties, it collapses to five lines — the constructor parameters themselves.

This is the dominant pattern in TypeScript codebases using dependency injection (NestJS, InversifyJS, plain DI containers). Understanding it is essential to reading and writing idiomatic TypeScript.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — always the full three-step pattern:
class OrderService {
  #orderRepo;
  #paymentService;
  #emailService;
  #logger;
  #serviceName;

  constructor(orderRepo, paymentService, emailService, logger, serviceName = "OrderService") {
    this.#orderRepo      = orderRepo;
    this.#paymentService = paymentService;
    this.#emailService   = emailService;
    this.#logger         = logger;
    this.#serviceName    = serviceName;
  }
  // 18 lines just for 5 dependencies
}
```

```ts
// TypeScript — parameter properties collapse it to 7 lines:
class OrderService {
  constructor(
    private readonly orderRepo:      OrderRepository,
    private readonly paymentService: PaymentService,
    private readonly emailService:   EmailService,
    private readonly logger:         Logger,
    public  readonly serviceName:    string = "OrderService",
  ) {}
  // Done — all properties declared, all assignments done, type-checked
}
```

---

## Syntax

```ts
class MyService {
  constructor(
    // ── Trigger: any access modifier or readonly ──────────────────────────
    public    publicParam:          string,   // public property + param
    private   privateParam:         number,   // private property + param
    protected protectedParam:       boolean,  // protected property + param
    readonly  readonlyParam:        Date,     // readonly property + param (implicitly public)
    public    readonly pubReadonly:  string,  // public readonly — most common for public IDs
    private   readonly privReadonly: string,  // private readonly — most common for injected deps
    protected readonly protReadonly: string,  // protected readonly — for base class deps

    // ── With default values ───────────────────────────────────────────────
    private timeout: number = 5000,

    // ── Mixed with plain constructor params (no modifier) ─────────────────
    plainParam: string,  // ← just a local variable, NOT a class property
  ) {
    // Only the plain param needs manual handling:
    console.log(plainParam); // ok — it's a local variable in scope here
    // all the others are already on `this` — no manual assignment needed
  }
}
```

**The trigger:** adding any access modifier (`public`, `private`, `protected`) or `readonly` (or both) to a constructor parameter turns it into a parameter property. No modifier = plain parameter, not stored.

---

## How it works — concept by concept

### Concept 1 — What TypeScript compiles parameter properties to

TypeScript compiles parameter properties into the exact boilerplate you would write manually:

```ts
// TypeScript source:
class ApiClient {
  constructor(
    private readonly baseUrl: string,
    private readonly authToken: string,
    public  readonly timeoutMs: number = 10_000,
  ) {}
}
```

Compiled JavaScript output:
```js
class ApiClient {
  baseUrl;      // (in modern output — omitted in older targets)
  authToken;
  timeoutMs;

  constructor(baseUrl, authToken, timeoutMs = 10_000) {
    this.baseUrl   = baseUrl;
    this.authToken = authToken;
    this.timeoutMs = timeoutMs;
  }
}
```

There is zero runtime difference. It is 100% a TypeScript syntax sugar — the language server expands it during compilation.

### Concept 2 — Which modifiers are valid triggers

```ts
class Foo {
  constructor(
    public    a: string,   // ✅ public property
    private   b: string,   // ✅ private property
    protected c: string,   // ✅ protected property
    readonly  d: string,   // ✅ readonly public property (no explicit 'public', defaults to public)
    public readonly  e: string, // ✅ public readonly
    private readonly f: string, // ✅ private readonly
    protected readonly g: string, // ✅ protected readonly

    // abstract modifier is NOT valid on parameter properties
    h: string, // ✅ plain parameter — local variable, NOT stored on `this`
  ) {}
}
```

`abstract` is the only modifier that cannot be used in a parameter property.

### Concept 3 — Mixing parameter properties with regular constructor body logic

Parameter properties don't prevent you from having constructor body logic. The two coexist:

```ts
class DatabasePool {
  private readonly connectionString: string; // can still use the traditional style if needed
  private activeConnections = 0;             // initialized inline — no param needed

  constructor(
    private readonly host:           string,
    private readonly port:           number,
    private readonly database:       string,
    private readonly user:           string,
    private readonly password:       string,  // used below but not stored as-is
    public  readonly maxConnections: number = 10,
    public  readonly poolName:       string = "default",
  ) {
    // Constructor body runs AFTER all parameter properties are assigned.
    // this.host, this.port, etc. are all available here:
    this.connectionString = `postgres://${user}:${password}@${host}:${port}/${database}`;
    // Note: 'password' is still accessible here as a local variable — AND as this.password
    // because it's a parameter property.
    // If you don't want to store it at all, use a plain param (no modifier):
  }
}
```

### Concept 4 — When NOT to store a parameter (no modifier = not stored)

If a constructor argument is only used to compute something during construction but should not be kept on the instance, omit the modifier:

```ts
class HashedCredential {
  readonly hash: string;
  readonly algorithm: string;

  constructor(
    plainPassword: string,  // ← no modifier — NOT stored on instance
    private readonly salt: string,
    algorithm: "sha256" | "bcrypt" = "bcrypt", // ← no modifier — used once, not stored
  ) {
    // plainPassword is a local variable here — not stored as this.plainPassword
    this.hash      = computeHash(plainPassword, salt, algorithm);
    this.algorithm = algorithm;
    // plainPassword leaves scope after constructor — not retained in memory as a property
  }
}

const cred = new HashedCredential("secret123", "some-salt");
cred.hash;          // ✅ available
cred.salt;          // ❌ private — not accessible outside
(cred as any).plainPassword; // undefined — it was never stored
```

This is a security best practice for sensitive data: don't keep plaintext passwords on the instance longer than necessary.

### Concept 5 — Default values in parameter properties

Parameter properties support default values the same way regular parameters do:

```ts
class HttpClient {
  constructor(
    private readonly baseUrl:    string,
    public  readonly timeoutMs:  number  = 30_000,
    public  readonly retries:    number  = 3,
    public  readonly userAgent:  string  = "MyApp/1.0",
    private readonly headers:    Record<string, string> = {},
  ) {}
}

const client = new HttpClient("https://api.example.com");
client.timeoutMs;  // 30000
client.retries;    // 3
client.userAgent;  // "MyApp/1.0"
```

### Concept 6 — Parameter properties in subclasses and `super()`

When a subclass uses parameter properties, `super()` still must be called before any `this` usage, as with all TypeScript/JavaScript classes. TypeScript initializes the parameter properties after `super()` returns:

```ts
abstract class BaseRepository<T> {
  constructor(
    protected readonly tableName: string,
    protected readonly logger:    Logger,
  ) {}
}

class UserRepository extends BaseRepository<User> {
  constructor(
    private readonly db:     DatabasePool,
    logger:                  Logger,       // plain param — passed up, not stored here
    private readonly cache:  CacheService,
  ) {
    super("users", logger); // logger passed to base — stored as protected readonly there
    // After super(), this.db and this.cache are available
  }

  async findById(userId: number): Promise<User | null> {
    this.logger.debug(`${this.tableName}.findById(${userId})`);  // from base
    const cached = await this.cache.get(`user:${userId}`);       // from this class
    if (cached) return cached;
    return this.db.query(`SELECT * FROM ${this.tableName} WHERE id = $1`, [userId]);
  }
}
```

### Concept 7 — Parameter properties and interfaces

Parameter properties satisfy interface implementations automatically — you're still declaring properties:

```ts
interface IUserService {
  readonly serviceName: string;
  findById(id: number): Promise<User | null>;
}

class UserService implements IUserService {
  constructor(
    public readonly serviceName: string,   // ✅ satisfies IUserService.serviceName
    private readonly userRepo: UserRepository,
  ) {}

  async findById(id: number): Promise<User | null> {
    return this.userRepo.findById(id);
  }
}
```

---

## Example 1 — basic

```ts
// Side-by-side comparison: verbose vs parameter properties style

// ── Verbose style ──────────────────────────────────────────────────────────
class VerboseEmailService {
  private readonly smtpHost:    string;
  private readonly smtpPort:    number;
  private readonly fromAddress: string;
  private readonly apiKey:      string;
  public  readonly serviceName: string;
  private sentCount:            number = 0;

  constructor(smtpHost: string, smtpPort: number, fromAddress: string, apiKey: string) {
    this.smtpHost    = smtpHost;
    this.smtpPort    = smtpPort;
    this.fromAddress = fromAddress;
    this.apiKey      = apiKey;
    this.serviceName = "EmailService";
  }

  async send(to: string, subject: string, body: string): Promise<void> {
    // ... smtp send logic
    this.sentCount++;
  }

  get totalSent(): number { return this.sentCount; }
}

// ── Parameter properties style — identical runtime behavior ───────────────
class EmailService {
  private sentCount = 0;

  constructor(
    private readonly smtpHost:    string,
    private readonly smtpPort:    number,
    private readonly fromAddress: string,
    private readonly apiKey:      string,
    public  readonly serviceName: string = "EmailService",
  ) {}

  async send(to: string, subject: string, body: string): Promise<void> {
    // ... smtp send logic
    this.sentCount++;
  }

  get totalSent(): number { return this.sentCount; }
}

// Usage — identical:
const emailer = new EmailService("smtp.sendgrid.net", 587, "noreply@api.dev.io", "SG.xxx");
emailer.serviceName;  // "EmailService" — public readonly accessible
emailer.smtpHost;     // ❌ Error: private
emailer.sentCount;    // ❌ Error: private

// ── With a plain (non-stored) parameter ───────────────────────────────────
class TokenGenerator {
  readonly tokenLength: number;
  readonly prefix:      string;

  constructor(
    tokenLength: number,      // plain — used below, not stored
    private readonly secret: string,
    prefix?: string,          // plain — used below, not stored
  ) {
    // Validate once during construction:
    if (tokenLength < 16) throw new RangeError(`Token length too short: ${tokenLength}`);
    this.tokenLength = tokenLength;
    this.prefix      = prefix ?? "tok_";
  }

  generate(): string {
    const rand  = crypto.randomBytes(this.tokenLength).toString("hex");
    const hmac  = computeHmac(rand, this.secret);
    return `${this.prefix}${hmac.slice(0, this.tokenLength)}`;
  }
}
```

---

## Example 2 — real world backend use case

```ts
// NestJS-style service layer using parameter properties throughout

import { randomUUID } from "crypto";

// ── Types ──────────────────────────────────────────────────────────────────

interface User {
  id: number;
  email: string;
  passwordHash: string;
  role: "admin" | "user" | "moderator";
  createdAt: Date;
  updatedAt: Date;
}

interface CreateUserDto {
  email: string;
  password: string;
  role?: "admin" | "user" | "moderator";
}

interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
  requestId: string;
}

// ── Repository ──────────────────────────────────────────────────────────────

abstract class BaseRepository<T, ID> {
  constructor(
    protected readonly tableName: string,
    protected readonly db:        DatabasePool,
    protected readonly logger:    Logger,
  ) {}

  abstract findById(id: ID): Promise<T | null>;
  abstract save(entity: T): Promise<T>;
  abstract delete(id: ID): Promise<void>;

  protected async query<R>(sql: string, params: unknown[]): Promise<R[]> {
    this.logger.debug(`[${this.tableName}] Query: ${sql.slice(0, 80)}`);
    return this.db.query<R>(sql, params);
  }
}

class UserRepository extends BaseRepository<User, number> {
  constructor(
    db:                              DatabasePool,  // plain — passed to super, not stored here
    logger:                          Logger,        // plain — passed to super
    private readonly cache:          CacheService,
    private readonly cachePrefix:    string = "user",
    private readonly cacheTtlSecs:   number = 300,
  ) {
    super("users", db, logger);
  }

  async findById(userId: number): Promise<User | null> {
    const key = `${this.cachePrefix}:${userId}`;
    const hit  = await this.cache.get<User>(key);
    if (hit) return hit;

    const [user] = await this.query<User>(
      "SELECT * FROM users WHERE id = $1 LIMIT 1",
      [userId],
    );
    if (user) await this.cache.set(key, user, this.cacheTtlSecs);
    return user ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const [user] = await this.query<User>(
      "SELECT * FROM users WHERE email = $1 LIMIT 1",
      [email.toLowerCase()],
    );
    return user ?? null;
  }

  async save(user: User): Promise<User> {
    const [saved] = await this.query<User>(
      `INSERT INTO users (id, email, password_hash, role, created_at, updated_at)
       VALUES ($1, $2, $3, $4, $5, $6)
       ON CONFLICT (id) DO UPDATE
         SET email = EXCLUDED.email,
             role  = EXCLUDED.role,
             updated_at = now()
       RETURNING *`,
      [user.id, user.email, user.passwordHash, user.role, user.createdAt, user.updatedAt],
    );
    await this.cache.del(`${this.cachePrefix}:${user.id}`);
    return saved;
  }

  async delete(userId: number): Promise<void> {
    await this.query("DELETE FROM users WHERE id = $1", [userId]);
    await this.cache.del(`${this.cachePrefix}:${userId}`);
  }
}

// ── Service ────────────────────────────────────────────────────────────────

class UserService {
  constructor(
    private readonly userRepo:    UserRepository,
    private readonly hashService: PasswordHashService,
    private readonly eventBus:    EventBus,
    private readonly logger:      Logger,
    public  readonly serviceName: string = "UserService",
  ) {}

  async createUser(dto: CreateUserDto, requestId: string): Promise<ApiResponse<User>> {
    this.logger.info(`[${this.serviceName}] createUser`, { email: dto.email, requestId });

    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) {
      return { data: existing, status: 409, message: "Email already registered", requestId };
    }

    const passwordHash = await this.hashService.hash(dto.password);
    const now          = new Date();
    const user: User   = {
      id:           generateSequentialId(),
      email:        dto.email.toLowerCase(),
      passwordHash,
      role:         dto.role ?? "user",
      createdAt:    now,
      updatedAt:    now,
    };

    const saved = await this.userRepo.save(user);
    await this.eventBus.emit("user.created", { userId: saved.id, email: saved.email });
    this.logger.info(`[${this.serviceName}] User created`, { userId: saved.id, requestId });
    return { data: saved, status: 201, message: "User created", requestId };
  }

  async getUser(userId: number, requestId: string): Promise<ApiResponse<User | null>> {
    const user = await this.userRepo.findById(userId);
    return {
      data:      user,
      status:    user ? 200 : 404,
      message:   user ? "ok" : "User not found",
      requestId,
    };
  }
}

// ── Controller ─────────────────────────────────────────────────────────────

class UserController {
  constructor(
    private readonly userService: UserService,
    private readonly logger:      Logger,
    public  readonly routePrefix: string = "/api/users",
  ) {}

  async create(req: Request, res: Response): Promise<void> {
    const requestId = req.headers["x-request-id"] as string ?? randomUUID();
    this.logger.info(`POST ${this.routePrefix}`, { requestId });
    const result = await this.userService.createUser(req.body, requestId);
    res.status(result.status).json(result);
  }

  async getById(req: Request, res: Response): Promise<void> {
    const requestId = req.headers["x-request-id"] as string ?? randomUUID();
    const userId    = parseInt(req.params.id, 10);
    this.logger.info(`GET ${this.routePrefix}/${userId}`, { requestId });
    const result = await this.userService.getUser(userId, requestId);
    res.status(result.status).json(result);
  }
}

// ── Wiring ─────────────────────────────────────────────────────────────────

// Three layers, all using parameter properties. Zero boilerplate property assignments:
const db         = new DatabasePool({ host: "localhost", port: 5432, database: "appdb", user: "app", password: process.env.DB_PASSWORD! });
const cache      = new RedisCache({ host: "localhost", port: 6379 });
const logger     = new ConsoleLogger("info");
const eventBus   = new InMemoryEventBus();
const hashSvc    = new BcryptHashService(12);

const userRepo   = new UserRepository(db, logger, cache);
const userSvc    = new UserService(userRepo, hashSvc, eventBus, logger);
const userCtrl   = new UserController(userSvc, logger);

userCtrl.routePrefix;   // "/api/users" — public readonly accessible
userCtrl.userService;   // ❌ Error: private
```

---

## Common mistakes

### Mistake 1 — Forgetting that a plain parameter is NOT a property

```ts
class SessionManager {
  constructor(
    private readonly sessionStore: SessionStore,
    expirySeconds: number,  // ← no modifier — plain local variable
  ) {
    // During construction, expirySeconds is accessible:
    console.log(expirySeconds); // ✅ ok here

    // But it's NOT stored — you must save it yourself if needed later:
  }

  get(sessionId: string): SessionData | null {
    return this.sessionStore.get(sessionId);
    // ❌ Cannot use expirySeconds here — it was never stored
  }
}

// ✅ Fix — add a modifier to store it:
class SessionManager {
  constructor(
    private readonly sessionStore:    SessionStore,
    private readonly expirySeconds:   number,  // ← now stored as a property
  ) {}

  get(sessionId: string): SessionData | null {
    const session = this.sessionStore.get(sessionId);
    if (!session) return null;
    if (Date.now() - session.createdAt > this.expirySeconds * 1000) {
      this.sessionStore.delete(sessionId);
      return null;
    }
    return session;
  }
}
```

### Mistake 2 — Mixing super() call with parameter property initialization order

```ts
class BaseService {
  constructor(protected readonly logger: Logger) {}
}

class AuthService extends BaseService {
  constructor(
    private readonly tokenRepo: TokenRepository,
    logger: Logger,  // passed to super — the order matters
  ) {
    // ❌ Can't use this.tokenRepo before super():
    // this.tokenRepo.ping();  // Error: 'super' must be called before accessing 'this'
    super(logger);             // ✅ call super first
    // ✅ Now all parameter properties including this.tokenRepo are initialized:
    this.logger.info("AuthService ready");  // this.logger comes from base — available now
  }
}

// ✅ Correct pattern — super() first, then use properties:
class AuthService extends BaseService {
  constructor(
    private readonly tokenRepo:   TokenRepository,
    private readonly jwtSecret:   string,
    logger:                       Logger,
  ) {
    super(logger);
    // this.tokenRepo, this.jwtSecret, and this.logger are all available here
    this.logger.info("AuthService initialized");
  }
}
```

### Mistake 3 — Trying to use parameter property syntax outside the constructor

```ts
class Foo {
  // ❌ Can't use parameter property syntax in a regular method:
  someMethod(private name: string): void {  // Error: Parameter properties are only allowed in TS constructors
    console.log(name);
  }

  // ❌ Can't use it in static factory methods:
  static create(public readonly id: number): Foo {  // Error: same
    return new Foo();
  }
}

// ✅ Parameter properties are ONLY valid in constructors:
class Foo {
  constructor(private readonly id: number) {}  // ✅

  someMethod(name: string): void {  // ← plain typed parameter
    console.log(name);
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

Refactor both of these verbose classes to use parameter properties. The behavior must be identical:

```ts
// Class A:
class RouteConfig {
  public basePath: string;
  public version: string;
  public prefix: string;

  constructor(basePath: string, version: string) {
    this.basePath = basePath;
    this.version  = version;
    this.prefix   = `${basePath}/${version}`;
  }

  resolve(endpoint: string): string {
    return `${this.prefix}${endpoint}`;
  }
}

// Class B:
class JwtConfig {
  private secret: string;
  private issuer: string;
  public expiresInSeconds: number;
  public algorithm: string;

  constructor(secret: string, issuer: string, expiresInSeconds: number = 3600) {
    this.secret           = secret;
    this.issuer           = issuer;
    this.expiresInSeconds = expiresInSeconds;
    this.algorithm        = "HS256";
  }

  get expiresIn(): string {
    return `${this.expiresInSeconds}s`;
  }
}
```

Also show: what happens if you try to access `jwtConfig.secret` from outside the class — write the line and the expected error.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a `CacheManager` class using parameter properties. It must:

- Depend on `StorageAdapter` (private readonly), `Logger` (private readonly), `Serializer` (private readonly)
- Have config values: `namespace` (public readonly, default `"cache"`), `defaultTtlMs` (private readonly, default `300_000`), `maxEntries` (public readonly, default `1000`)
- Have a plain constructor param `initialWarmupKeys: string[]` — used only during construction to pre-warm the cache, NOT stored as a property
- Have mutable internal state: `private hitCount = 0` and `private missCount = 0` (inline initialized, no constructor param)
- Methods: `get<T>(key: string): Promise<T | null>`, `set<T>(key: string, value: T, ttlMs?: number): Promise<void>`, `delete(key: string): Promise<void>`
- A getter: `get stats(): { hits: number; misses: number; ratio: number }`

Show in a comment at the bottom which fields are stored properties and which are local-only.

```ts
// Write your code here
```

### Exercise 3 — hard

Design a full **three-layer architecture** — Repository, Service, Controller — for a `Product` resource, where every class uses parameter properties exclusively for its dependencies. Requirements:

Types to define:
```ts
interface Product { id: number; name: string; priceCents: number; stock: number; categoryId: number; createdAt: Date; }
interface CreateProductDto { name: string; priceCents: number; stock: number; categoryId: number; }
interface UpdateProductDto { name?: string; priceCents?: number; stock?: number; }
```

`ProductRepository`:
- Depends on: `DatabasePool` (private readonly), `Logger` (private readonly)
- Plain (non-stored) constructor param: `schemaName: string` — used only to build the fully-qualified table name `"${schemaName}.products"` stored as `private readonly tableName`
- Methods: `findById`, `findByCategoryId`, `save`, `update`, `delete`

`ProductService`:
- Depends on: `ProductRepository` (private readonly), `EventBus` (private readonly), `Logger` (private readonly)
- Config: `public readonly serviceName: string = "ProductService"`, `private readonly lowStockThreshold: number = 5`
- Methods: `createProduct(dto, requestId)`, `updateProduct(id, dto, requestId)`, `decrementStock(productId, quantity, requestId)`

`ProductController`:
- Depends on: `ProductService` (private readonly), `Logger` (private readonly)
- Config: `public readonly routePrefix: string = "/api/products"`
- Methods: `handleCreate(req, res)`, `handleGetById(req, res)`, `handleUpdateStock(req, res)`

Every class body must have **zero** manual `this.x = x` assignments — all must come from parameter properties. Wire them all together at the bottom.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// The rule: add any modifier to a constructor param → it becomes a property
class Service {
  constructor(
    public  repo:              Repository,   // declares + assigns this.repo
    private secret:            string,       // declares + assigns this.secret
    protected config:          Config,       // declares + assigns this.config
    readonly id:               number,       // declares + assigns this.id (public readonly)
    public readonly name:      string,       // public readonly
    private readonly apiKey:   string,       // private readonly (most common for deps)

    plainParam:  string,  // NOT stored — just a local variable in constructor scope
  ) {
    // All properties are ready to use here — even before body runs implicitly
    // (they're assigned as part of the parameter binding process)
    console.log(this.repo);    // ✅
    console.log(plainParam);   // ✅ local variable — accessible only here
  }
}
```

| Style | Lines for 4 deps | Behaviour |
|---|---|---|
| Manual (JS style) | 4 + 4 + 4 = 12 | Identical |
| Parameter properties | 4 (in constructor) | Identical |
| Mix (some manual, some param) | Varies | Fine — valid |

**Rules to remember:**
- Only valid in constructors (not methods, not static factories)
- Works with: `public`, `private`, `protected`, `readonly`, combinations
- Default values are supported: `private timeout: number = 5000`
- `super()` must still come before any `this` usage — same rule as always
- No modifier = plain parameter = NOT stored as property

---

## Connected topics

- **33 — Classes in TypeScript** — full class syntax; parameter properties in context of the whole class.
- **34 — Access modifiers** — `public`, `private`, `protected` — these are the triggers for parameter properties.
- **35 — Readonly class properties** — `readonly` is the other trigger; review the difference between `readonly` and `private`.
- **37 — Abstract classes** — `abstract` is the one modifier that does NOT work as a parameter property trigger.
- **19 — Implementing interfaces in classes** — parameter properties satisfy interface property requirements.
- **29 — Generic classes** — parameter properties work with generic type parameters: `constructor(private readonly repo: Repository<T>)`.
