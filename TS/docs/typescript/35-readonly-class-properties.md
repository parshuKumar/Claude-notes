# 35 — Readonly Class Properties

## What is this?

The `readonly` modifier on a class property means **the property can only be assigned once** — in the property initializer or in the constructor body — and never changed after that. It is a **mutability constraint**, not a visibility constraint.

This is distinct from:
- `private` — controls *who* can access the property (visibility)
- `readonly` — controls *when* the property can be written (mutability)

They compose freely: `private readonly`, `public readonly`, `protected readonly` are all valid.

## Why does it matter?

Immutability is one of the most effective bug-prevention techniques. Many properties in a backend system should never change after construction:

- `id` — once assigned from the database, it should never be reassigned
- `createdAt` — the creation timestamp is permanent
- `userId` — a request belongs to one user; it shouldn't drift
- Config values — loaded once at startup, never mutated
- Injected dependencies — the service you inject into a class shouldn't be swapped mid-life

Without `readonly`, these can be accidentally overwritten anywhere in the codebase. With `readonly`, TypeScript makes it a compile error to reassign them — catching bugs before they happen.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no protection against reassignment:
class RequestContext {
  constructor(requestId, userId, method, path) {
    this.requestId = requestId;
    this.userId    = userId;
    this.method    = method;
    this.path      = path;
    this.startTime = Date.now();
  }
}

const ctx = new RequestContext("req_abc", 1, "GET", "/api/users");
ctx.userId = 99;     // silently mutates — now wrong user
ctx.startTime = 0;   // silently breaks timing — no error
```

```ts
// TypeScript with readonly — mutations are compile-time errors:
class RequestContext {
  readonly requestId: string;
  readonly userId: number;
  readonly method: string;
  readonly path: string;
  readonly startTime: number;

  constructor(requestId: string, userId: number, method: string, path: string) {
    this.requestId = requestId;
    this.userId    = userId;
    this.method    = method;
    this.path      = path;
    this.startTime = Date.now();
  }
}

const ctx = new RequestContext("req_abc", 1, "GET", "/api/users");
ctx.userId    = 99;  // ❌ Error: Cannot assign to 'userId' — it is a read-only property
ctx.startTime = 0;   // ❌ Error: Cannot assign to 'startTime' — it is a read-only property
```

---

## Syntax

```ts
class Entity {
  // ── Readonly property declaration ─────────────────────────────────────
  readonly id: number;                    // must be assigned in constructor
  readonly createdAt: Date = new Date();  // inline initializer — fine
  readonly label = "entity" as const;     // inferred as literal "entity"

  // ── Readonly + access modifier ────────────────────────────────────────
  public readonly publicId: string;
  protected readonly tableName: string;
  private readonly internalKey: string;

  // ── Readonly parameter property (most common pattern) ─────────────────
  constructor(
    public readonly serviceId: string,   // declared + assigned in one line
    private readonly secret: string,     // private + readonly
    protected readonly config: AppConfig,
  ) {
    this.id          = generateId();
    this.publicId    = `pub_${this.id}`;
    this.tableName   = "entities";
    this.internalKey = computeKey(this.secret);
  }
}

// ── Readonly in an interface (reminder — covered in topic 14) ──────────
interface ImmutablePoint {
  readonly x: number;
  readonly y: number;
}

// ── Readonly in a type alias ───────────────────────────────────────────
type ReadonlyUser = Readonly<User>;  // all properties become readonly (shallow)
```

---

## How it works — rule by rule

### Rule 1 — Assignment is only allowed in the property initializer or constructor

```ts
class Order {
  readonly orderId: string;
  readonly createdAt: Date = new Date();  // ✅ inline initializer
  readonly status = "pending" as const;   // ✅ inline initializer

  constructor(orderId: string) {
    this.orderId = orderId;  // ✅ allowed in constructor
  }

  confirm(): void {
    this.orderId   = "new_id";    // ❌ Error: read-only property — outside constructor
    this.createdAt = new Date();  // ❌ Error: read-only property
    this.status    = "confirmed"; // ❌ Error: read-only property
  }
}
```

### Rule 2 — `readonly` is about *reassignment*, not deep immutability

`readonly` prevents reassigning the *reference*. If the value is an object or array, its *contents* can still be mutated:

```ts
class UserService {
  readonly config: AppConfig;
  readonly tags: string[] = [];

  constructor(config: AppConfig) {
    this.config = config;
  }

  updateConfig(): void {
    this.config = { ...this.config, port: 8080 };  // ❌ Error: reassigning readonly

    // But mutating the object's contents is fine — readonly is SHALLOW:
    this.config.port = 8080;  // ✅ No error — mutating the object, not reassigning the reference
    this.tags.push("new-tag"); // ✅ No error — mutating the array, not reassigning the reference
  }
}

// For deep immutability: use `as const`, `Object.freeze()`, or `Readonly<T>` recursively
```

### Rule 3 — `readonly` vs `private` — they solve different problems

```ts
class DatabaseConfig {
  //           WHO can access?    WHEN can write?
  private readonly host: string;  // only this class,   only in constructor
  protected port: number;         // this class + subclasses, anytime
  public readonly name: string;   // everyone,          only in constructor
  private url: string;            // only this class,   anytime internally

  constructor(host: string, port: number, name: string) {
    this.host = host;
    this.port = port;
    this.name = name;
    this.url  = `postgres://${host}:${port}/${name}`;
  }

  reconnect(newPort: number): void {
    this.port = newPort;    // ✅ private, but not readonly — can change internally
    this.url  = `postgres://${this.host}:${this.port}/${this.name}`;
    this.host = "other";    // ❌ Error: private readonly — can't change even internally after constructor
  }
}

const config = new DatabaseConfig("localhost", 5432, "appdb");
config.name;   // ✅ public readonly — readable everywhere
config.host;   // ❌ private — not accessible outside
config.port;   // ❌ protected — not accessible outside class hierarchy
config.url;    // ❌ private
```

### Rule 4 — Readonly parameter properties are the dominant pattern

The most common usage in real TypeScript code is `readonly` combined with access modifier in a parameter property:

```ts
// This:
class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly cache: CacheService,
    public readonly serviceName: string,
  ) {}
}

// Is exactly equivalent to this (but far more concise):
class UserService {
  private readonly userRepo: UserRepository;
  private readonly cache: CacheService;
  public readonly serviceName: string;

  constructor(
    userRepo: UserRepository,
    cache: CacheService,
    serviceName: string,
  ) {
    this.userRepo    = userRepo;
    this.cache       = cache;
    this.serviceName = serviceName;
  }
}
```

### Rule 5 — `readonly` propagates to the inferred type

When TypeScript infers a `readonly` property, the narrowed type is preserved:

```ts
class ApiClient {
  // Without 'as const' — inferred as string:
  readonly method = "GET";
  // method: string

  // With 'as const' — inferred as the literal type:
  readonly endpoint = "/api/v2/users" as const;
  // endpoint: "/api/v2/users"

  // Direct literal in readonly — inferred as literal in some cases:
  readonly version: "v1" | "v2" = "v2";
  // version: "v1" | "v2" (union type)
}
```

### Rule 6 — `readonly` in interfaces vs classes

When a class implements an interface with `readonly` properties, the class can either match `readonly` or use a regular (mutable) property — TypeScript allows widening from readonly to mutable in implementations:

```ts
interface Config {
  readonly port: number;
  readonly host: string;
}

// ✅ Class can have readonly (matches exactly):
class ServerConfig implements Config {
  readonly port: number;
  readonly host: string;
  constructor(port: number, host: string) {
    this.port = port;
    this.host = host;
  }
}

// ✅ Class can have mutable (widening — allowed because assignability is checked structurally):
class MutableServerConfig implements Config {
  port: number;   // mutable, but satisfies readonly constraint in interface
  host: string;
  constructor(port: number, host: string) {
    this.port = port;
    this.host = host;
  }
}

// The interface consumer sees 'readonly' regardless of the implementation:
const cfg: Config = new MutableServerConfig(3000, "localhost");
cfg.port = 8080;   // ❌ Error: readonly in the interface type — can't assign through Config
(cfg as MutableServerConfig).port = 8080; // ✅ works — accessing through the mutable type
```

---

## Example 1 — basic

```ts
// Typed value objects — immutable after construction (DDD Value Object pattern)

class Money {
  readonly amountCents: number;
  readonly currency: "USD" | "EUR" | "GBP";

  constructor(amountCents: number, currency: "USD" | "EUR" | "GBP") {
    if (amountCents < 0) throw new RangeError(`Amount cannot be negative: ${amountCents}`);
    if (!Number.isInteger(amountCents)) throw new TypeError(`Amount must be integer cents: ${amountCents}`);
    this.amountCents = amountCents;
    this.currency    = currency;
  }

  // Operations return new Money instances — never mutate:
  add(other: Money): Money {
    if (other.currency !== this.currency) {
      throw new Error(`Currency mismatch: ${this.currency} vs ${other.currency}`);
    }
    return new Money(this.amountCents + other.amountCents, this.currency);
  }

  subtract(other: Money): Money {
    if (other.currency !== this.currency) {
      throw new Error(`Currency mismatch: ${this.currency} vs ${other.currency}`);
    }
    if (other.amountCents > this.amountCents) {
      throw new RangeError("Subtraction would result in negative amount");
    }
    return new Money(this.amountCents - other.amountCents, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(Math.round(this.amountCents * factor), this.currency);
  }

  get formatted(): string {
    const amount = (this.amountCents / 100).toFixed(2);
    const symbols: Record<typeof this.currency, string> = { USD: "$", EUR: "€", GBP: "£" };
    return `${symbols[this.currency]}${amount}`;
  }

  equals(other: Money): boolean {
    return this.amountCents === other.amountCents && this.currency === other.currency;
  }

  static zero(currency: "USD" | "EUR" | "GBP"): Money {
    return new Money(0, currency);
  }

  static usd(cents: number): Money { return new Money(cents, "USD"); }
  static eur(cents: number): Money { return new Money(cents, "EUR"); }
}

class OrderLineItem {
  readonly productId: number;
  readonly productName: string;
  readonly unitPrice: Money;
  readonly quantity: number;
  readonly addedAt: Date;

  constructor(productId: number, productName: string, unitPriceCents: number, quantity: number) {
    if (quantity < 1) throw new RangeError("Quantity must be at least 1");
    this.productId   = productId;
    this.productName = productName;
    this.unitPrice   = Money.usd(unitPriceCents);
    this.quantity    = quantity;
    this.addedAt     = new Date();
  }

  get subtotal(): Money {
    return this.unitPrice.multiply(this.quantity);
  }

  // Returns a new line item with updated quantity — never mutates:
  withQuantity(newQty: number): OrderLineItem {
    return new OrderLineItem(this.productId, this.productName, this.unitPrice.amountCents, newQty);
  }
}

// Usage:
const item = new OrderLineItem(101, "TypeScript Handbook", 2999, 2);
item.productId = 99;  // ❌ Error: readonly
item.quantity  = 5;   // ❌ Error: readonly

const updated = item.withQuantity(3);  // ✅ new instance — original unchanged
updated.quantity;  // 3
item.quantity;     // still 2

const price   = Money.usd(2999);
const tax     = price.multiply(0.08);
const total   = price.add(tax);
total.formatted;   // "$32.39"
price.formatted;   // "$29.99" — original unchanged
```

---

## Example 2 — real world backend use case

```ts
// Request/response context objects — immutable identity fields prevent drift mid-request

import { randomUUID } from "crypto";

type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

class HttpRequestContext {
  // Identity — set once, never changes:
  readonly requestId: string;
  readonly correlationId: string;
  readonly startedAt: Date;
  readonly method: HttpMethod;
  readonly path: string;
  readonly clientIp: string;

  // Auth — set once after authentication middleware:
  readonly userId: number | null;
  readonly userRole: string | null;

  // Mutable — accumulated during request processing:
  private _responseStatus: number = 200;
  private _errors: string[]       = [];
  private _metadata: Record<string, unknown> = {};

  constructor(
    method: HttpMethod,
    path: string,
    clientIp: string,
    correlationId?: string,
    userId?: number,
    userRole?: string,
  ) {
    this.requestId      = randomUUID();
    this.correlationId  = correlationId ?? randomUUID();
    this.startedAt      = new Date();
    this.method         = method;
    this.path           = path;
    this.clientIp       = clientIp;
    this.userId         = userId ?? null;
    this.userRole       = userRole ?? null;
  }

  // Mutable state — accessed through controlled methods:
  setResponseStatus(status: number): void {
    if (status < 100 || status > 599) throw new RangeError(`Invalid HTTP status: ${status}`);
    this._responseStatus = status;
  }

  addError(message: string): void {
    this._errors.push(message);
  }

  setMeta(key: string, value: unknown): void {
    this._metadata[key] = value;
  }

  get responseStatus(): number           { return this._responseStatus; }
  get errors(): readonly string[]        { return Object.freeze([...this._errors]); }
  get hasErrors(): boolean               { return this._errors.length > 0; }
  get elapsedMs(): number                { return Date.now() - this.startedAt.getTime(); }
  get meta(): Readonly<Record<string, unknown>> { return Object.freeze({ ...this._metadata }); }

  // Build a log entry — uses readonly fields as immutable record:
  toLogEntry(): Record<string, unknown> {
    return {
      requestId:     this.requestId,
      correlationId: this.correlationId,
      method:        this.method,
      path:          this.path,
      clientIp:      this.clientIp,
      userId:        this.userId,
      userRole:      this.userRole,
      status:        this._responseStatus,
      elapsedMs:     this.elapsedMs,
      errors:        this._errors,
      startedAt:     this.startedAt.toISOString(),
    };
  }
}

// ── Service using readonly context ────────────────────────────────────────

class UserController {
  constructor(
    private readonly userService: UserManagementService,
    private readonly logger: Logger,
  ) {}

  async handleGetUser(ctx: HttpRequestContext, userId: number): Promise<ApiResponse<User | null>> {
    this.logger.info("Handling GET /users/:id", {
      requestId: ctx.requestId,  // ✅ readonly — guaranteed to be the right ID
      userId:    ctx.userId,     // ✅ readonly — guaranteed to be the authenticated user
    });

    // Cannot accidentally mutate identity:
    // ctx.requestId = "other";  // ❌ Error: readonly
    // ctx.userId    = 0;        // ❌ Error: readonly

    try {
      const user = await this.userService.findById(userId);
      ctx.setResponseStatus(user ? 200 : 404);
      return { data: user, status: ctx.responseStatus, message: user ? "ok" : "Not found", timestamp: new Date().toISOString() };
    } catch (err) {
      ctx.setResponseStatus(500);
      ctx.addError(String(err));
      this.logger.error("Error in handleGetUser", { requestId: ctx.requestId, error: err });
      throw err;
    } finally {
      this.logger.info("Request complete", ctx.toLogEntry());
    }
  }
}

// ── Readonly in service config ─────────────────────────────────────────────

class PaymentService {
  // Once injected, these never change — readonly prevents accidental replacement:
  constructor(
    private readonly stripeClient: StripeClient,
    private readonly orderRepo: OrderRepository,
    private readonly webhookSecret: string,  // readonly via parameter property (private readonly)
    public readonly serviceName: string = "PaymentService",
  ) {}

  async chargeOrder(orderId: number, amountCents: number): Promise<string> {
    // this.stripeClient = anotherClient;  // ❌ Error — readonly, can't swap mid-execution
    return this.stripeClient.charge(amountCents);
  }
}
```

---

## Common mistakes

### Mistake 1 — Thinking `readonly` provides deep immutability

```ts
class AppConfig {
  readonly server: { port: number; host: string };
  readonly allowedOrigins: string[];

  constructor() {
    this.server         = { port: 3000, host: "0.0.0.0" };
    this.allowedOrigins = ["https://app.dev.io"];
  }
}

const config = new AppConfig();
config.server = { port: 8080, host: "localhost" };  // ❌ Error: readonly (reassignment blocked)
config.server.port = 8080;                           // ✅ NO error — readonly is SHALLOW
config.allowedOrigins.push("http://evil.com");       // ✅ NO error — contents mutable!

// ✅ For deep immutability, use explicit readonly types:
class AppConfig {
  readonly server: Readonly<{ port: number; host: string }>;
  readonly allowedOrigins: readonly string[];

  constructor() {
    this.server         = { port: 3000, host: "0.0.0.0" };
    this.allowedOrigins = ["https://app.dev.io"];
  }
}

const config2 = new AppConfig();
config2.server.port = 8080;           // ❌ Error: Readonly — inner property protected
config2.allowedOrigins.push("evil");  // ❌ Error: readonly array — push not allowed
```

### Mistake 2 — Trying to assign a `readonly` property outside the constructor

```ts
class Session {
  readonly sessionId: string;

  constructor() {
    this.sessionId = generateId();  // ✅ constructor — fine
  }

  regenerate(): void {
    this.sessionId = generateId();  // ❌ Error: read-only property
  }
}

// ✅ If you need re-generation, don't use readonly — or create a new instance:
class Session {
  sessionId: string;  // mutable
  constructor() { this.sessionId = generateId(); }
  regenerate(): void { this.sessionId = generateId(); }  // ✅ now works
}

// ✅ OR: create a new instance (immutable style):
class Session {
  readonly sessionId: string;
  constructor(sessionId?: string) { this.sessionId = sessionId ?? generateId(); }
  regenerate(): Session { return new Session(); }  // returns a new instance
}
```

### Mistake 3 — Confusing `readonly` (property) with `Readonly<T>` (utility type)

```ts
// readonly on a property:
class Foo {
  readonly id: number = 1;  // that specific property cannot be reassigned
}

// Readonly<T> utility type — makes ALL properties of T readonly:
type ReadonlyFoo = Readonly<Foo>;
// { readonly id: number }  — same in this case, but applied to the type

// They are different:
class Bar {
  name: string = "bar";  // mutable
  readonly id: number = 1; // readonly
}

type ReadonlyBar = Readonly<Bar>;
// { readonly name: string; readonly id: number }
// — Readonly<T> adds readonly to name too (Bar.name was mutable, ReadonlyBar.name is not)

// Common confusion:
const bar = new Bar();
bar.name = "new";  // ✅ mutable on the class instance

const readonlyBar: ReadonlyBar = bar;
readonlyBar.name = "new";  // ❌ Error — through ReadonlyBar type, name is readonly
// But: bar.name = "new" still works — the underlying object is still mutable
// ReadonlyBar is just a lens that hides mutability
```

---

## Practice exercises

### Exercise 1 — easy

1. Build a `ServerConfig` class where ALL constructor-supplied values are `readonly`:

```ts
class ServerConfig {
  // All of these should be readonly:
  // host: string, port: number, protocol: "http" | "https",
  // maxConnections: number, timeoutMs: number, corsOrigins: readonly string[]
  
  // Computed (derived from the above, also readonly):
  // baseUrl: string  (e.g. "https://api.dev.io:3000")
  // isProduction: boolean  (true if protocol === "https" && port === 443)
}
```

Show that after construction, nothing can be reassigned. Show that `corsOrigins` typed as `readonly string[]` prevents `push()` on it.

2. Refactor this broken class to use `readonly` correctly:

```ts
class RequestId {
  id: string;
  createdAt: Date;
  
  constructor() {
    this.id        = Math.random().toString(36).slice(2);
    this.createdAt = new Date();
  }
  
  reset(): void {         // this should NOT be allowed
    this.id        = Math.random().toString(36).slice(2);
    this.createdAt = new Date();
  }
}
```

```ts
// Write your code here
```

### Exercise 2 — medium

Build a `DatabaseConnectionPool` class that demonstrates a real-world use of `readonly` for injected dependencies and configuration, plus mutable internal state:

```ts
class DatabaseConnectionPool {
  // Readonly — set at construction, never change:
  // connectionString: string (private readonly — built from host/port/db/password)
  // maxConnections: number (public readonly)
  // minConnections: number (public readonly)
  // acquireTimeoutMs: number (public readonly)
  // poolName: string (public readonly)
  // createdAt: Date (public readonly)

  // Mutable — internal pool state:
  // private activeConnections: Connection[]
  // private idleConnections: Connection[]
  // private waitingRequests: Array<(conn: Connection) => void>
  // private _totalCreated: number
  // private _totalReleased: number

  constructor(config: {
    host: string;
    port: number;
    database: string;
    user: string;
    password: string;  // used to build connectionString but not stored as-is
    maxConnections?: number;
    minConnections?: number;
    acquireTimeoutMs?: number;
    poolName?: string;
  })

  // Public methods:
  async acquire(): Promise<Connection>  // returns a connection from the pool
  release(conn: Connection): void       // returns a connection to the pool
  async drain(): Promise<void>          // waits for all active connections to be released then closes all
  
  // Getters (computed from mutable state):
  get activeCount(): number
  get idleCount(): number
  get totalCount(): number
  get stats(): { active: number; idle: number; waiting: number; totalCreated: number }
}
```

Implement it fully. Verify that attempting to reassign `maxConnections`, `poolName`, or `createdAt` from outside the class causes a TypeScript error.

```ts
// Write your code here
```

### Exercise 3 — hard

Implement a full **immutable linked list** using `readonly` class properties — a `LinkedList<T>` where every operation returns a new list instead of mutating the existing one:

```ts
// Each node is immutable:
class ListNode<T> {
  readonly value: T;
  readonly next: ListNode<T> | null;
  readonly index: number;

  constructor(value: T, next: ListNode<T> | null = null, index = 0) { ... }
}

// The list is a wrapper around the head node:
class ImmutableLinkedList<T> {
  readonly head: ListNode<T> | null;
  readonly size: number;

  // Static factory:
  static empty<T>(): ImmutableLinkedList<T>
  static of<T>(...values: T[]): ImmutableLinkedList<T>
  static from<T>(iterable: Iterable<T>): ImmutableLinkedList<T>

  // All operations return new lists — never mutate:
  prepend(value: T): ImmutableLinkedList<T>    // O(1) — adds to front
  append(value: T): ImmutableLinkedList<T>     // O(n) — adds to back
  tail(): ImmutableLinkedList<T>               // O(1) — all except first element
  drop(n: number): ImmutableLinkedList<T>      // O(n) — remove first n elements
  take(n: number): ImmutableLinkedList<T>      // O(n) — keep only first n elements
  map<U>(fn: (value: T, index: number) => U): ImmutableLinkedList<U>
  filter(predicate: (value: T, index: number) => boolean): ImmutableLinkedList<T>
  find(predicate: (value: T) => boolean): T | undefined
  toArray(): T[]
  [Symbol.iterator](): Iterator<T>
}
```

Requirements:
- Every `ListNode` property is `readonly` — nodes can never change
- Every `ImmutableLinkedList` property is `readonly` — lists can never change
- All methods that "modify" the list return a new `ImmutableLinkedList` — original is untouched
- `map` and `filter` preserve the linked structure (not converting to array and back for the internal representation)

Use it:
```ts
const list1 = ImmutableLinkedList.of(1, 2, 3, 4, 5);
const list2 = list1.prepend(0);   // [0, 1, 2, 3, 4, 5]
const list3 = list1.append(6);    // [1, 2, 3, 4, 5, 6]
const list4 = list1.map(n => n * 2); // [2, 4, 6, 8, 10]

// list1 is unchanged throughout:
list1.toArray();  // [1, 2, 3, 4, 5]

// All of these should be compile-time errors:
// list1.head = someNode;          // ❌ readonly
// list1.head!.value = 99;         // ❌ readonly
// list1.head!.next = null;        // ❌ readonly
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Property declaration:
class Foo {
  readonly id: number;              // must be set in constructor
  readonly tag = "foo";             // inline initializer — fine
  readonly items: readonly number[] = []; // readonly + readonly array (deep)
}

// Parameter property (most common):
constructor(
  public readonly userId: number,
  private readonly secret: string,
  protected readonly config: AppConfig,
) {}

// Assignment rules:
class Bar {
  readonly x: number;
  constructor(x: number) {
    this.x = x;       // ✅ constructor — OK
  }
  update(): void {
    this.x = 1;       // ❌ Error: outside constructor
  }
}

// readonly is SHALLOW — inner object can still mutate:
class Config {
  readonly settings: { timeout: number };  // settings ref is readonly
  constructor() { this.settings = { timeout: 5000 }; }
  mutate(): void {
    this.settings.timeout = 9999;    // ✅ shallow mutation — OK (be careful!)
    this.settings = { timeout: 1 };  // ❌ reassignment blocked
  }
}

// For deep immutability:
readonly settings: Readonly<{ timeout: number }>;  // inner props also readonly
readonly tags: readonly string[];                   // array contents readonly
```

| | `readonly` | `private` |
|---|---|---|
| Controls | *When* written | *Who* can access |
| Visible outside class? | Yes (unless also `private`) | No |
| Can combine? | `private readonly` ✅ | `private readonly` ✅ |
| Mutable internally? | Only in constructor | Yes, anytime in the class |
| Deep immutability? | No — shallow only | n/a |
| Runtime enforcement? | No — compile-time only | No — compile-time only |

---

## Connected topics

- **33 — Classes in TypeScript** — full class syntax; `readonly` in context of all other class features.
- **34 — Access modifiers** — `private`, `protected`, `public` — control *who* can access (orthogonal to `readonly`).
- **14 — Interfaces** — `readonly` in interface property declarations.
- **32 — Utility types** — `Readonly<T>` makes all properties of T readonly (shallow); `as const` for deep readonly.
- **36 — Static members** — `static readonly` constants; `static #` for private static.
- **35 → 37 — Abstract classes** — `readonly` is commonly combined with `protected` in abstract base classes.
