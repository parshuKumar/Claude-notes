# 37 — Abstract Classes

## What is this?

An **abstract class** is a class that is **incomplete by design** — it defines structure, shared behaviour, and a contract of methods that subclasses *must* implement, but it cannot be instantiated directly itself.

Two pieces of syntax power abstract classes:

- `abstract class Name` — marks the class as abstract (cannot be `new`'d)
- `abstract methodName(): ReturnType` — marks a method as abstract (must be overridden in every concrete subclass)

```ts
abstract class BaseHandler {
  abstract handle(request: Request): Promise<Response>; // must be implemented
  
  protected log(message: string): void {                // shared — available to all subclasses
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
}

// const h = new BaseHandler(); // ❌ Error: Cannot create an instance of an abstract class

class GetUserHandler extends BaseHandler {
  async handle(request: Request): Promise<Response> { // ✅ concrete implementation
    this.log("Handling GET /users");
    return new Response("ok");
  }
}
```

## Why does it matter?

Abstract classes solve a problem that pure interfaces can't: **sharing real, reusable code** while simultaneously **enforcing a contract** on subclasses.

Consider a backend where you have a `UserRepository`, `OrderRepository`, and `ProductRepository`. All three:
- Share identical logic for pagination, logging, error handling
- But differ in their table name, entity type, and specific query methods

Without abstract classes you duplicate the shared logic across all three. With an abstract class `BaseRepository<T>`, you write the shared logic once and force each subclass to supply the pieces that differ (`tableName`, `mapRow()`).

The key choice: use an abstract class when subclasses need **shared state or shared method implementations**. Use an interface when you only need a **shape contract** with no shared code.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no abstract keyword. Convention only, no enforcement:
class BaseRepository {
  constructor(tableName, db) {
    if (new.target === BaseRepository) {
      throw new Error("BaseRepository cannot be instantiated directly");
    }
    this.tableName = tableName;
    this.db        = db;
  }

  // "Abstract" method — convention, no enforcement:
  mapRow(row) {
    throw new Error(`${this.constructor.name} must implement mapRow()`);
  }

  async findAll() {
    const rows = await this.db.query(`SELECT * FROM ${this.tableName}`);
    return rows.map(r => this.mapRow(r));
  }
}

class UserRepository extends BaseRepository {
  constructor(db) { super("users", db); }
  // If you forget mapRow(), it throws at RUNTIME. No earlier warning.
}

const repo = new BaseRepository("users", db); // Runtime error — but only if you remember the check
```

```ts
// TypeScript — abstract is enforced at compile time, no runtime hacks needed:
abstract class BaseRepository<T> {
  constructor(
    protected readonly tableName: string,
    protected readonly db:        DatabasePool,
  ) {}

  abstract mapRow(row: Record<string, unknown>): T;  // must be implemented

  async findAll(): Promise<T[]> {
    const rows = await this.db.query<Record<string, unknown>>(`SELECT * FROM ${this.tableName}`);
    return rows.map(row => this.mapRow(row));
  }
}

class UserRepository extends BaseRepository<User> {
  constructor(db: DatabasePool) { super("users", db); }
  // ❌ TypeScript error here if mapRow is missing:
  // Non-abstract class 'UserRepository' does not implement inherited abstract member 'mapRow'
  mapRow(row: Record<string, unknown>): User {
    return { id: row.id as number, email: row.email as string };
  }
}

const repo = new BaseRepository<User>(db, "users"); // ❌ Error: cannot instantiate abstract class
```

---

## Syntax

```ts
// ── Declaring an abstract class ────────────────────────────────────────────
abstract class ClassName {

  // Abstract method — no body, must be overridden:
  abstract methodName(param: Type): ReturnType;

  // Abstract getter — no body, must be overridden:
  abstract get propertyName(): Type;

  // Abstract property — declared but not implemented:
  abstract readonly requiredField: string;

  // Concrete method — has a body, inherited as-is:
  concreteMethod(): void {
    console.log("shared logic");
  }

  // Constructor — abstract classes CAN have constructors (called via super()):
  constructor(protected readonly sharedDep: SharedDependency) {}
}

// ── Extending an abstract class ────────────────────────────────────────────
class ConcreteClass extends ClassName {

  // Must implement every abstract member:
  methodName(param: Type): ReturnType {
    return /* implementation */;
  }

  get propertyName(): Type {
    return /* implementation */;
  }

  readonly requiredField = "concrete value";

  constructor(sharedDep: SharedDependency) {
    super(sharedDep);  // abstract class constructor called via super()
  }
}
```

---

## How it works — concept by concept

### Concept 1 — Abstract methods have no body

An abstract method is a declaration only — it has no `{}` body. TypeScript tracks it as a "must-implement" requirement:

```ts
abstract class Validator<T> {
  // ✅ Abstract — declaration only, no body:
  abstract validate(value: T): ValidationResult;
  abstract get ruleName(): string;

  // ✅ Concrete — has a body:
  validateOrThrow(value: T): T {
    const result = this.validate(value);  // calls the subclass's implementation
    if (!result.valid) throw new ValidationError(result.errors);
    return value;
  }
}

// ❌ Abstract method WITH a body is a syntax error:
abstract class Bad {
  abstract validate(v: unknown): boolean { return true; }  // Error: Method cannot have an implementation
}
```

### Concept 2 — Concrete subclasses must implement all abstract members

TypeScript checks at the point of class definition that every abstract member is implemented:

```ts
abstract class NotificationSender {
  abstract send(to: string, message: string): Promise<void>;
  abstract readonly channelName: string;
  abstract get maxMessageLength(): number;
}

// ❌ Missing implementations — compile-time error:
class BrokenSender extends NotificationSender {
  // Error: Non-abstract class 'BrokenSender' does not implement
  // inherited abstract members: 'send', 'channelName', 'maxMessageLength'
}

// ✅ All abstract members implemented:
class SmsNotificationSender extends NotificationSender {
  readonly channelName = "SMS";
  get maxMessageLength(): number { return 160; }

  async send(to: string, message: string): Promise<void> {
    if (message.length > this.maxMessageLength) {
      throw new Error(`SMS too long: ${message.length} > ${this.maxMessageLength}`);
    }
    await smsGateway.send(to, message);
  }
}
```

### Concept 3 — Abstract classes can have constructors, properties, and full implementations

Unlike interfaces, abstract classes can carry real, shared state and real logic:

```ts
abstract class BaseApiClient {
  private requestCount = 0;
  private readonly rateLimiter: RateLimiter;

  constructor(
    protected readonly baseUrl:    string,
    protected readonly authToken:  string,
    rateLimit:                     number = 100,  // requests per minute
  ) {
    this.rateLimiter = new RateLimiter(rateLimit);
  }

  // Abstract — each concrete client defines its own retry strategy:
  abstract shouldRetry(error: ApiError, attempt: number): boolean;

  // Concrete shared logic — all subclasses inherit this:
  protected async request<T>(path: string, init?: RequestInit): Promise<T> {
    await this.rateLimiter.acquire();
    this.requestCount++;

    const url = `${this.baseUrl}${path}`;
    const response = await fetch(url, {
      ...init,
      headers: { Authorization: `Bearer ${this.authToken}`, ...init?.headers },
    });

    if (!response.ok) {
      const error = new ApiError(response.status, await response.text());
      if (this.shouldRetry(error, this.requestCount)) {
        return this.request<T>(path, init);
      }
      throw error;
    }

    return response.json() as Promise<T>;
  }

  get totalRequests(): number { return this.requestCount; }
}
```

### Concept 4 — You can type variables with the abstract class

Variables typed as the abstract class accept any concrete subclass instance. This enables polymorphism:

```ts
abstract class PaymentProvider {
  abstract charge(amountCents: number, currency: string): Promise<ChargeResult>;
  abstract refund(chargeId: string, amountCents: number): Promise<RefundResult>;
  abstract readonly providerName: string;
}

class StripeProvider  extends PaymentProvider { readonly providerName = "Stripe";  /* ... */ }
class PayPalProvider  extends PaymentProvider { readonly providerName = "PayPal";  /* ... */ }
class BraintreeProvider extends PaymentProvider { readonly providerName = "Braintree"; /* ... */ }

// Type is the abstract class — accepts any concrete subclass:
let provider: PaymentProvider;
provider = new StripeProvider(process.env.STRIPE_KEY!);
provider = new PayPalProvider(process.env.PAYPAL_KEY!);  // ✅ both work

// The abstract class acts like a contract — you call the interface, not the implementation:
async function processPayment(provider: PaymentProvider, amountCents: number): Promise<void> {
  const result = await provider.charge(amountCents, "USD");  // works for any provider
  console.log(`Charged via ${provider.providerName}: ${result.chargeId}`);
}
```

### Concept 5 — Abstract class vs Interface — the decision

| | Abstract class | Interface |
|---|---|---|
| Can have concrete method bodies? | ✅ Yes | ❌ No |
| Can have instance state (properties)? | ✅ Yes | ❌ No (declaration only) |
| Can have a constructor? | ✅ Yes | ❌ No |
| Can be extended by only one parent? | ✅ Yes (single inheritance) | ✅ No — a class can implement many |
| Creates a JS value at runtime? | ✅ Yes (a class) | ❌ No (erased) |
| Suitable for unrelated shapes? | ❌ Use interface | ✅ Use interface |
| Suitable for shared base behaviour? | ✅ Use abstract class | ❌ Use abstract class |

**Rule of thumb:** If you find yourself writing the same private method or the same state initialization in multiple related classes, that's the signal to extract an abstract base class.

### Concept 6 — Abstract classes can extend other abstract classes

Abstract classes can extend each other, and the chain is checked by TypeScript:

```ts
abstract class BaseEntity {
  abstract get tableName(): string;
  readonly createdAt: Date = new Date();
  readonly updatedAt: Date = new Date();
}

abstract class AuditedEntity extends BaseEntity {
  abstract get auditLogTable(): string;
  // tableName still abstract — not implemented here, passes obligation to concrete subclass
  
  async logChange(action: string, userId: number): Promise<void> {
    console.log(`[${this.auditLogTable}] ${action} by user ${userId}`);
  }
}

class UserEntity extends AuditedEntity {
  get tableName():     string { return "users"; }       // ✅ implements BaseEntity.tableName
  get auditLogTable(): string { return "user_audit"; }  // ✅ implements AuditedEntity.auditLogTable
}
```

---

## Example 1 — basic

```ts
// Typed validation framework with shared logic in the abstract base

interface ValidationResult {
  valid: boolean;
  errors: string[];
}

abstract class FieldValidator<T> {
  abstract validate(value: T): ValidationResult;
  abstract readonly fieldName: string;

  protected pass(): ValidationResult {
    return { valid: true, errors: [] };
  }

  protected fail(...errors: string[]): ValidationResult {
    return { valid: false, errors };
  }

  // Shared utility — subclasses inherit without reimplementing:
  validateOrThrow(value: T): T {
    const result = this.validate(value);
    if (!result.valid) {
      throw new Error(`Validation failed for '${this.fieldName}': ${result.errors.join(", ")}`);
    }
    return value;
  }

  isValid(value: T): boolean {
    return this.validate(value).valid;
  }
}

class EmailValidator extends FieldValidator<string> {
  readonly fieldName = "email";

  validate(value: string): ValidationResult {
    if (!value || value.trim().length === 0) return this.fail("Email is required");
    if (!value.includes("@"))               return this.fail("Email must contain @");
    if (value.length > 254)                 return this.fail("Email too long (max 254 chars)");
    const [local, domain] = value.split("@");
    if (!domain || !domain.includes("."))   return this.fail("Email domain is invalid");
    if (local.length > 64)                  return this.fail("Email local part too long");
    return this.pass();
  }
}

class PasswordValidator extends FieldValidator<string> {
  readonly fieldName = "password";

  constructor(
    private readonly minLength: number = 8,
    private readonly requireSpecial: boolean = true,
  ) { super(); }

  validate(value: string): ValidationResult {
    const errors: string[] = [];
    if (value.length < this.minLength)        errors.push(`Min length is ${this.minLength}`);
    if (!/[A-Z]/.test(value))                 errors.push("Must contain uppercase letter");
    if (!/[0-9]/.test(value))                 errors.push("Must contain a digit");
    if (this.requireSpecial && !/[^A-Za-z0-9]/.test(value)) errors.push("Must contain a special character");
    return errors.length ? this.fail(...errors) : this.pass();
  }
}

class PositiveIntegerValidator extends FieldValidator<number> {
  constructor(
    readonly fieldName: string,
    private readonly max?: number,
  ) { super(); }

  validate(value: number): ValidationResult {
    if (!Number.isInteger(value))                      return this.fail("Must be an integer");
    if (value <= 0)                                    return this.fail("Must be positive");
    if (this.max !== undefined && value > this.max)    return this.fail(`Max value is ${this.max}`);
    return this.pass();
  }
}

// All work through the shared abstract base:
const emailVal   = new EmailValidator();
const passwordVal = new PasswordValidator(12, true);
const userIdVal  = new PositiveIntegerValidator("userId", 2_147_483_647);

emailVal.isValid("user@example.com");   // true
emailVal.isValid("not-an-email");        // false

passwordVal.validateOrThrow("Abcdef1!");          // returns the password
passwordVal.validateOrThrow("weak");              // throws Error: ...
userIdVal.validate(0);                            // { valid: false, errors: ["Must be positive"] }
```

---

## Example 2 — real world backend use case

```ts
// Repository base class — shared pagination, error handling, logging; per-entity query methods

interface PaginationOptions { page: number; perPage: number; }
interface PaginatedResult<T> { data: T[]; total: number; page: number; perPage: number; totalPages: number; }

abstract class BaseRepository<TEntity, TId extends number | string> {

  constructor(
    protected readonly db:     DatabasePool,
    protected readonly logger: Logger,
  ) {}

  // ── Abstract contract — subclasses define entity-specific behaviour ──────
  abstract get tableName(): string;
  abstract mapRow(row: Record<string, unknown>): TEntity;
  abstract get idColumn(): string;

  // ── Concrete shared behaviour — never rewritten in subclasses ─────────────

  async findById(id: TId): Promise<TEntity | null> {
    this.logger.debug(`[${this.tableName}] findById(${id})`);
    const rows = await this.db.query<Record<string, unknown>>(
      `SELECT * FROM ${this.tableName} WHERE ${this.idColumn} = $1 LIMIT 1`,
      [id],
    );
    return rows[0] ? this.mapRow(rows[0]) : null;
  }

  async findAll(options?: PaginationOptions): Promise<PaginatedResult<TEntity>> {
    const page    = options?.page    ?? 1;
    const perPage = options?.perPage ?? 20;
    const offset  = (page - 1) * perPage;

    this.logger.debug(`[${this.tableName}] findAll(page=${page}, perPage=${perPage})`);

    const [countRow] = await this.db.query<{ count: string }>(
      `SELECT COUNT(*) AS count FROM ${this.tableName}`,
      [],
    );
    const total      = parseInt(countRow.count, 10);
    const rows       = await this.db.query<Record<string, unknown>>(
      `SELECT * FROM ${this.tableName} ORDER BY ${this.idColumn} LIMIT $1 OFFSET $2`,
      [perPage, offset],
    );

    return {
      data:       rows.map(r => this.mapRow(r)),
      total,
      page,
      perPage,
      totalPages: Math.ceil(total / perPage),
    };
  }

  async exists(id: TId): Promise<boolean> {
    const [row] = await this.db.query<{ exists: boolean }>(
      `SELECT EXISTS(SELECT 1 FROM ${this.tableName} WHERE ${this.idColumn} = $1) AS exists`,
      [id],
    );
    return row.exists;
  }

  async delete(id: TId): Promise<void> {
    this.logger.debug(`[${this.tableName}] delete(${id})`);
    await this.db.query(`DELETE FROM ${this.tableName} WHERE ${this.idColumn} = $1`, [id]);
  }
}

// ── Concrete repositories — only entity-specific code ─────────────────────

interface User  { id: number; email: string; role: string; createdAt: Date; }
interface Order { id: number; userId: number; totalCents: number; status: string; createdAt: Date; }

class UserRepository extends BaseRepository<User, number> {
  constructor(db: DatabasePool, logger: Logger) { super(db, logger); }

  get tableName(): string  { return "users"; }
  get idColumn():  string  { return "id"; }

  mapRow(row: Record<string, unknown>): User {
    return {
      id:        row.id        as number,
      email:     row.email     as string,
      role:      row.role      as string,
      createdAt: new Date(row.created_at as string),
    };
  }

  async findByEmail(email: string): Promise<User | null> {
    const [row] = await this.db.query<Record<string, unknown>>(
      "SELECT * FROM users WHERE email = $1",
      [email.toLowerCase()],
    );
    return row ? this.mapRow(row) : null;
  }
}

class OrderRepository extends BaseRepository<Order, number> {
  constructor(db: DatabasePool, logger: Logger) { super(db, logger); }

  get tableName(): string { return "orders"; }
  get idColumn():  string { return "id"; }

  mapRow(row: Record<string, unknown>): Order {
    return {
      id:          row.id           as number,
      userId:      row.user_id      as number,
      totalCents:  row.total_cents  as number,
      status:      row.status       as string,
      createdAt:   new Date(row.created_at as string),
    };
  }

  async findByUserId(userId: number, options?: PaginationOptions): Promise<PaginatedResult<Order>> {
    const page    = options?.page    ?? 1;
    const perPage = options?.perPage ?? 20;
    const offset  = (page - 1) * perPage;

    const [countRow] = await this.db.query<{ count: string }>(
      "SELECT COUNT(*) AS count FROM orders WHERE user_id = $1",
      [userId],
    );
    const total = parseInt(countRow.count, 10);
    const rows  = await this.db.query<Record<string, unknown>>(
      "SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3",
      [userId, perPage, offset],
    );
    return {
      data: rows.map(r => this.mapRow(r)),
      total, page, perPage,
      totalPages: Math.ceil(total / perPage),
    };
  }
}

// Usage — polymorphic: both work through the abstract type:
async function countRecords(repo: BaseRepository<unknown, number | string>): Promise<number> {
  const result = await repo.findAll({ page: 1, perPage: 1 });
  return result.total;
}
```

---

## Common mistakes

### Mistake 1 — Trying to instantiate an abstract class directly

```ts
abstract class BaseService {
  abstract process(requestBody: unknown): Promise<void>;
}

// ❌ Error: Cannot create an instance of an abstract class:
const service = new BaseService();

// ✅ Instantiate a concrete subclass:
class ConcreteService extends BaseService {
  async process(requestBody: unknown): Promise<void> { /* ... */ }
}
const service = new ConcreteService();

// ✅ Type the variable as the abstract class — accept any subclass:
let svc: BaseService = new ConcreteService();
```

### Mistake 2 — Forgetting to implement all abstract members in a concrete subclass

```ts
abstract class BaseNotifier {
  abstract send(userId: number, message: string): Promise<void>;
  abstract readonly channelName: string;
  abstract get maxRetries(): number;
}

// ❌ This compiles only if all three abstract members are present:
class EmailNotifier extends BaseNotifier {
  // Error: Non-abstract class 'EmailNotifier' does not implement
  // inherited abstract members: 'channelName', 'maxRetries'
  async send(userId: number, message: string): Promise<void> { /* ... */ }
}

// ✅ All three implemented:
class EmailNotifier extends BaseNotifier {
  readonly channelName = "email";
  get maxRetries(): number { return 3; }
  async send(userId: number, message: string): Promise<void> { /* ... */ }
}
```

### Mistake 3 — Giving an abstract method a body (concrete body in abstract class for abstract method)

```ts
abstract class Parser {
  // ❌ Error: Method 'parse' cannot have an implementation because it is marked abstract:
  abstract parse(input: string): unknown {
    return JSON.parse(input);
  }
}

// ✅ If you want a default implementation, make it concrete (no 'abstract' keyword):
abstract class Parser {
  // Concrete with default — subclasses can override but don't have to:
  parse(input: string): unknown {
    return JSON.parse(input);  // default implementation
  }
}

// ✅ Or keep it abstract with no body — force subclasses to implement:
abstract class Parser {
  abstract parse(input: string): unknown;  // no body — subclass must provide one
}
```

---

## Practice exercises

### Exercise 1 — easy

Build an abstract `BaseFormatter<T>` class for formatting data into different output formats. It must have:

- Abstract method `format(data: T): string` — converts the data to the output format
- Abstract readonly property `mimeType: string` — the MIME type of the output (`"application/json"`, `"text/csv"`, etc.)
- Concrete method `formatAndWrap(data: T, label: string): string` — calls `this.format(data)` and wraps it with `=== ${label} ===\n${formatted}\n=== END ===`
- Concrete method `isValid(data: T): boolean` — tries `this.format(data)` and returns `true` if it doesn't throw, `false` otherwise

Then implement these two concrete classes:

1. `JsonFormatter<T>` — formats using `JSON.stringify(data, null, 2)`, mimeType `"application/json"`
2. `CsvFormatter` (non-generic, for `Record<string, string | number>[]`) — formats as CSV (first row headers from object keys, remaining rows values), mimeType `"text/csv"`

Show that both work through the abstract type `BaseFormatter<unknown>`.

```ts
// Write your code here
```

### Exercise 2 — medium

Design an abstract `BaseJob` class for a background job system. Requirements:

- Abstract method `execute(jobId: string): Promise<void>` — the job logic
- Abstract method `shouldRetry(error: Error, attempt: number): boolean` — retry decision
- Abstract readonly `jobName: string` — human-readable name
- Abstract readonly `timeoutMs: number` — max execution time
- Concrete method `run(jobId: string): Promise<JobResult>` — orchestrates execution with retry logic:
  - Calls `execute(jobId)` up to `maxAttempts` times
  - On failure calls `shouldRetry(error, attempt)` to decide
  - Catches timeout using `timeoutMs`
  - Returns `JobResult: { jobId, jobName, success, attempts, durationMs, error?: string }`
- Concrete method `schedule(cronExpression: string): ScheduledJob` — wraps this job in a scheduler
- Constructor takes `private readonly maxAttempts: number = 3`

Then implement:
1. `SendWelcomeEmailJob` — sends a welcome email; retries up to 3 times on network errors only
2. `GenerateMonthlyReportJob` — generates a report; no retries (shouldRetry always returns false); timeoutMs is 5 minutes

```ts
// Write your code here
```

### Exercise 3 — hard

Build a full abstract **event sourcing** base using abstract classes:

```ts
// An event is an immutable record of something that happened:
interface DomainEvent {
  readonly eventId:        string;
  readonly eventType:      string;
  readonly aggregateId:    string;
  readonly aggregateType:  string;
  readonly occurredAt:     Date;
  readonly payload:        Record<string, unknown>;
  readonly version:        number;
}

// Abstract aggregate root — the base for all event-sourced entities:
abstract class AggregateRoot<TState> {
  // Abstract — each aggregate defines its own initial state:
  protected abstract get initialState(): TState;

  // Abstract — each aggregate knows how to apply its own events to state:
  protected abstract apply(state: TState, event: DomainEvent): TState;

  // Abstract — the aggregate type name (e.g. "User", "Order"):
  abstract get aggregateType(): string;

  // Concrete shared behaviour:
  // - reconstructState(events): rebuilds state by folding events through apply()
  // - recordEvent(type, payload): creates a DomainEvent and stores it in uncommittedEvents
  // - get currentState(): the reconstructed state from all committed + uncommitted events
  // - get uncommittedEvents(): events recorded since last save
  // - markCommitted(): clears uncommittedEvents (called after persistence)
  // - get version(): current event count
}

// Concrete aggregate:
interface UserState {
  id:           string;
  email:        string;
  role:         "admin" | "user";
  isDeleted:    boolean;
  loginCount:   number;
}

class UserAggregate extends AggregateRoot<UserState> {
  constructor(private readonly userId: string) { super(); }
  get aggregateType(): string { return "User"; }
  // initialState, apply(), plus domain methods:
  // register(email), promoteToAdmin(), recordLogin(), softDelete()
  // Each domain method calls this.recordEvent() internally
}
```

Requirements:
- `AggregateRoot` must be fully abstract — zero possibility of direct instantiation
- `apply()` must be pure — given the same state and event, always returns the same new state
- `currentState` must always be consistent with the event history
- `uncommittedEvents` must be cleared after `markCommitted()`
- `UserAggregate` must reconstruct correctly from a list of events (simulate loading from event store)

Demonstrate:
```ts
const user = new UserAggregate("user_abc");
user.register("dev@api.dev.io");
user.recordLogin();
user.recordLogin();
user.promoteToAdmin();
user.currentState;  // { id: "user_abc", email: "dev@api.dev.io", role: "admin", isDeleted: false, loginCount: 2 }
user.uncommittedEvents.length;  // 4 (register, login, login, promote)
user.markCommitted();
user.uncommittedEvents.length;  // 0
user.version;  // 4
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Declaration:
abstract class BaseName {
  abstract requiredMethod(param: Type): ReturnType;  // no body
  abstract readonly requiredProp: string;            // no value
  abstract get requiredGetter(): number;             // no body

  // These are fine — concrete members in an abstract class:
  concreteMethod(): void { /* shared logic */ }
  readonly sharedProp: string = "shared";
  constructor(protected dep: Dependency) {}
}

// Extension:
class ConcreteName extends BaseName {
  readonly requiredProp = "implemented";             // ✅
  get requiredGetter(): number { return 42; }        // ✅
  requiredMethod(param: Type): ReturnType { ... }    // ✅
  constructor(dep: Dependency) { super(dep); }
}

// Cannot instantiate:
new BaseName();     // ❌ Error
new ConcreteName(); // ✅

// Type variable with abstract class:
let x: BaseName = new ConcreteName(); // ✅ polymorphism
```

| Feature | Abstract class | Interface |
|---|---|---|
| Concrete methods | ✅ | ❌ |
| Constructor | ✅ | ❌ |
| Instance state | ✅ | ❌ (declaration only) |
| Multiple "parents" | ❌ (single `extends`) | ✅ (many `implements`) |
| Runtime JS value | ✅ (class exists) | ❌ (erased) |
| Best for | Shared base behaviour | Shape contracts |

---

## Connected topics

- **33 — Classes in TypeScript** — full class syntax, inheritance with `extends`.
- **34 — Access modifiers** — `protected` is the typical modifier for abstract class members shared with subclasses.
- **36 — Parameter properties** — abstract class constructors use parameter properties for shared dependencies.
- **38 — Class implementing interface** — a concrete subclass can also implement interfaces at the same time.
- **19 — Implementing interfaces in classes** — abstract classes vs interfaces: structural vs nominal, shared code vs pure contract.
- **29 — Generic classes** — abstract classes are commonly generic: `abstract class BaseRepository<T>`.
