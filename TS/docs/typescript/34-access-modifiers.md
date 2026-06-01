# 34 — Access Modifiers

## What is this?

Access modifiers control **where a class member can be read or written**. TypeScript has three:

| Modifier | Accessible from |
|----------|----------------|
| `public` | Everywhere (default — no modifier needed) |
| `protected` | This class + any subclass |
| `private` | This class only |

Plus JavaScript's native private fields:

| Syntax | Type | Accessible from |
|--------|------|----------------|
| `#field` | JS runtime private | This class only — enforced at **runtime** |

TypeScript's `public`, `protected`, and `private` are **compile-time only** — they are erased in the JavaScript output and have zero runtime effect. `#fields` are true ECMAScript private fields that remain private even at runtime.

## Why does it matter?

Access modifiers enforce **encapsulation** — the OOP principle of hiding internal state and exposing only a controlled public interface. Without them:

- Internal implementation details leak out
- Callers couple to internals that might change
- You can't trust that internal state is only changed through your own methods
- Testing becomes harder because anything can change anything

With them:
- The class owns its own state — only its own methods change it
- Subclasses get controlled access via `protected`
- `private` creates a hard boundary that TypeScript enforces at every call site
- `#` goes one step further — enforced at runtime too

## The JavaScript way vs the TypeScript way

```js
// JavaScript — convention-only privacy (underscore prefix is just a hint):
class OrderProcessor {
  constructor(paymentGateway) {
    this._paymentGateway = paymentGateway;  // underscore = "please don't touch"
    this._orderHistory   = [];              // but nothing stops you
  }

  processOrder(order) {
    this._chargeCard(order);  // "private" by convention
  }

  _chargeCard(order) {
    return this._paymentGateway.charge(order.total);  // callable from anywhere
  }
}

const processor = new OrderProcessor(gateway);
processor._orderHistory.push(fakeOrder);  // works — no enforcement
processor._chargeCard({ total: 999999 }); // works — dangerous
```

```ts
// TypeScript — enforced at compile time:
class OrderProcessor {
  private readonly orderHistory: Order[] = [];

  constructor(
    private readonly paymentGateway: PaymentGateway,
  ) {}

  async processOrder(order: Order): Promise<Receipt> {
    const receipt = await this.chargeCard(order);  // ✅ private method, called internally
    this.orderHistory.push(order);
    return receipt;
  }

  private async chargeCard(order: Order): Promise<Receipt> {
    return this.paymentGateway.charge(order.totalCents);
  }
}

const processor = new OrderProcessor(gateway);
processor.orderHistory;             // ❌ Error: 'orderHistory' is private
processor.chargeCard({ ... });      // ❌ Error: 'chargeCard' is private
processor.processOrder(order);      // ✅ only public method
```

---

## Syntax

```ts
class BankAccount {
  // ── public (default) — accessible everywhere ─────────────────────────
  public accountId: string;
  public readonly createdAt: Date;

  // ── protected — accessible in this class and subclasses ───────────────
  protected balanceCents: number;
  protected ownerId: number;

  // ── private — accessible ONLY in this class ───────────────────────────
  private transactionLog: Transaction[] = [];
  private _pin: string;

  // ── # — ECMAScript runtime private ────────────────────────────────────
  #encryptionKey: string;

  constructor(
    accountId: string,
    ownerId: number,
    initialBalanceCents: number,
    pin: string,
    encryptionKey: string,
  ) {
    this.accountId      = accountId;
    this.createdAt      = new Date();
    this.balanceCents   = initialBalanceCents;
    this.ownerId        = ownerId;
    this._pin           = pin;
    this.#encryptionKey = encryptionKey;
  }

  // ── public method ─────────────────────────────────────────────────────
  public getBalance(): number {
    return this.balanceCents / 100;
  }

  // ── protected method — subclasses can call/override ───────────────────
  protected recordTransaction(tx: Transaction): void {
    this.transactionLog.push(tx);
  }

  // ── private method — internal implementation detail ───────────────────
  private validatePin(input: string): boolean {
    return this._pin === input;
  }
}
```

---

## How it works — modifier by modifier

### `public` — the default, accessible everywhere

If you don't write any modifier, the member is `public`. There is no practical difference between writing `public name: string` and just `name: string` — the explicit keyword is only useful for clarity or when using parameter properties where a modifier is required:

```ts
class UserProfile {
  name: string;          // implicitly public
  public email: string;  // explicitly public — identical behavior

  // Parameter properties REQUIRE a modifier (any of the three):
  constructor(
    public readonly userId: number,  // must write 'public' for parameter property syntax
    name: string,                    // no modifier — NOT a parameter property, just a param
  ) {
    this.name = name;  // must assign manually
  }
}

const profile = new UserProfile(1, "Parsh");
profile.name;   // ✅ public
profile.email;  // ✅ public
profile.userId; // ✅ public readonly
```

### `protected` — the subclass contract

`protected` is the mechanism for controlled extension. It says "this is internal, but subclasses are allowed to use it because they are an extension of this class":

```ts
class BaseRepository {
  protected readonly tableName: string;
  protected db: DatabaseConnection;

  constructor(tableName: string, db: DatabaseConnection) {
    this.tableName = tableName;
    this.db        = db;
  }

  // Protected utility — subclasses use it, external code doesn't:
  protected async executeQuery<T>(sql: string, params: unknown[]): Promise<T[]> {
    return this.db.query<T>(sql, params);
  }

  // Public interface:
  async count(): Promise<number> {
    const rows = await this.executeQuery<{ count: number }>(
      `SELECT COUNT(*) AS count FROM ${this.tableName}`, [],
    );
    return rows[0].count;
  }
}

class UserRepository extends BaseRepository {
  constructor(db: DatabaseConnection) {
    super("users", db);  // tableName set via super
  }

  async findByEmail(email: string): Promise<User | null> {
    // ✅ Can call protected method:
    const rows = await this.executeQuery<User>(
      `SELECT * FROM ${this.tableName} WHERE email = $1`, [email],
    );
    return rows[0] ?? null;
  }
}

const repo = new UserRepository(db);
repo.executeQuery("SELECT 1", []);  // ❌ Error: 'executeQuery' is protected
repo.tableName;                      // ❌ Error: 'tableName' is protected
repo.count();                        // ✅ public method
```

### `private` — compile-time boundary, runtime transparent

TypeScript `private` is erased completely in the compiled JavaScript output. It provides compile-time encapsulation — every access through `obj.privateField` is caught as a type error:

```ts
class TokenService {
  private readonly jwtSecret: string;
  private readonly algorithm = "HS256" as const;

  constructor(secret: string) {
    this.jwtSecret = secret;
  }

  sign(payload: Record<string, unknown>, expiresIn: number): string {
    return this.encode(payload, this.jwtSecret, expiresIn);  // ✅ internal use
  }

  verify(token: string): Record<string, unknown> | null {
    return this.decode(token, this.jwtSecret);               // ✅ internal use
  }

  private encode(payload: Record<string, unknown>, secret: string, expiresIn: number): string {
    // stub
    return Buffer.from(JSON.stringify({ ...payload, exp: Date.now() + expiresIn })).toString("base64");
  }

  private decode(token: string, secret: string): Record<string, unknown> | null {
    try {
      return JSON.parse(Buffer.from(token, "base64").toString());
    } catch {
      return null;
    }
  }
}

const svc = new TokenService("s3cr3t");
svc.sign({ userId: 1 }, 3600);  // ✅ public
svc.jwtSecret;                   // ❌ Error: private
svc.encode({ }, "", 0);          // ❌ Error: private

// ⚠️ But at runtime — TypeScript private is NOT enforced:
console.log((svc as any).jwtSecret);  // "s3cr3t" — still accessible at runtime!
```

### `#` — true ECMAScript private (runtime enforcement)

JavaScript native private fields (`#`) are truly inaccessible outside the class — not just at compile time but at runtime too:

```ts
class PaymentProcessor {
  // TypeScript private — erased to a normal property in JS output:
  private readonly apiKey: string;

  // JavaScript # private — stays private in JS output, inaccessible everywhere:
  readonly #webhookSecret: string;
  #transactionCount = 0;

  constructor(apiKey: string, webhookSecret: string) {
    this.apiKey        = apiKey;
    this.#webhookSecret = webhookSecret;
  }

  async charge(amountCents: number): Promise<string> {
    this.#transactionCount++;
    return `txn_${this.#transactionCount}_${amountCents}`;
  }

  verifyWebhook(signature: string, payload: string): boolean {
    return this.computeHmac(payload, this.#webhookSecret) === signature;
  }

  get transactionCount(): number { return this.#transactionCount; }

  private computeHmac(data: string, secret: string): string {
    return `hmac_${data}_${secret}`;  // stub
  }
}

const processor = new PaymentProcessor("pk_live", "whsec_xyz");

// TypeScript errors for both:
processor.apiKey;         // ❌ TS Error: private
processor.#webhookSecret; // ❌ TS Error: private field

// Runtime behavior differs:
(processor as any).apiKey;         // "pk_live" — accessible at runtime (TS private is just compile-time)
(processor as any).webhookSecret;  // undefined — # field is truly hidden in JS runtime
// You literally cannot access #webhookSecret by any means at runtime
```

### `private` vs `#` — when to use which

```
TypeScript `private`     Use when:
  - Simple encapsulation is enough
  - You don't need runtime enforcement
  - You want slightly simpler class syntax
  - The class doesn't handle security-sensitive secrets

JavaScript `#`          Use when:
  - True runtime privacy matters (security-sensitive data)
  - Subclasses must not accidentally access the field
  - You're writing a library where external code might bypass TypeScript
  - API keys, secrets, cryptographic material, private keys
```

### Access modifiers in inheritance

```ts
class Vehicle {
  public make: string;
  protected engineType: "gas" | "electric" | "hybrid";
  private vin: string;
  #licenseKey: string;

  constructor(make: string, engineType: "gas" | "electric" | "hybrid", vin: string, licenseKey: string) {
    this.make         = make;
    this.engineType   = engineType;
    this.vin          = vin;
    this.#licenseKey  = licenseKey;
  }

  protected getEngineInfo(): string {
    return `${this.make} (${this.engineType})`;
  }
}

class ElectricCar extends Vehicle {
  private batteryCapacityKwh: number;

  constructor(make: string, vin: string, batteryCapacityKwh: number) {
    super(make, "electric", vin, "ek_placeholder");
    this.batteryCapacityKwh = batteryCapacityKwh;
  }

  getRange(): number {
    // ✅ protected — accessible in subclass:
    const baseRange = this.engineType === "electric" ? 4 : 2;
    return this.batteryCapacityKwh * baseRange;
  }

  describe(): string {
    return this.getEngineInfo();    // ✅ protected method — accessible
  }

  getVin(): string {
    return this.vin;               // ❌ Error: 'vin' is private to Vehicle
  }

  getLicenseKey(): string {
    return this.#licenseKey;       // ❌ SyntaxError: # fields are not inherited
  }
}

const car = new ElectricCar("Tesla", "5YJSA1E26MF123456", 100);
car.make;              // ✅ public
car.engineType;        // ❌ protected — outside class hierarchy
car.getRange();        // ✅ public
car.getEngineInfo();   // ❌ protected
```

### Access modifiers with interfaces

TypeScript `private` and `protected` members are **not part of an interface contract**. An interface can only describe `public` members:

```ts
interface UserStore {
  findById(id: number): Promise<User | null>;   // must be public to be in interface
  save(user: User): Promise<User>;
}

class PostgresUserStore implements UserStore {
  private db: DatabaseConnection;
  protected logger: Logger;

  constructor(db: DatabaseConnection, logger: Logger) {
    this.db     = db;
    this.logger = logger;
  }

  // ✅ Public — satisfies the interface contract:
  async findById(id: number): Promise<User | null> {
    return this.db.query(`SELECT * FROM users WHERE id = $1`, [id])
      .then(rows => rows[0] ?? null);
  }

  async save(user: User): Promise<User> {
    return user;
  }
}
```

---

## Example 1 — basic

```ts
// Typed bank account with strict access control

type TransactionType = "credit" | "debit" | "transfer_in" | "transfer_out" | "fee";

interface Transaction {
  id: string;
  type: TransactionType;
  amountCents: number;
  description: string;
  balanceAfterCents: number;
  timestamp: Date;
}

class BankAccount {
  public readonly accountId: string;
  public readonly ownerId: number;
  public readonly currency: string;
  public readonly createdAt: Date;

  protected balanceCents: number;

  private transactions: Transaction[] = [];
  private _frozen: boolean = false;
  #encryptionSalt: string;

  constructor(
    accountId: string,
    ownerId: number,
    initialBalanceCents: number,
    currency: string,
    encryptionSalt: string,
  ) {
    this.accountId      = accountId;
    this.ownerId        = ownerId;
    this.balanceCents   = initialBalanceCents;
    this.currency       = currency;
    this.createdAt      = new Date();
    this.#encryptionSalt = encryptionSalt;
  }

  // ── Public interface ───────────────────────────────────────────────────

  get balance(): number { return this.balanceCents / 100; }
  get isFrozen(): boolean { return this._frozen; }
  get transactionCount(): number { return this.transactions.length; }

  deposit(amountCents: number, description = "Deposit"): Transaction {
    this.assertNotFrozen();
    this.assertPositiveAmount(amountCents);
    this.balanceCents += amountCents;
    return this.recordTransaction("credit", amountCents, description);
  }

  withdraw(amountCents: number, description = "Withdrawal"): Transaction {
    this.assertNotFrozen();
    this.assertPositiveAmount(amountCents);
    this.assertSufficientFunds(amountCents);
    this.balanceCents -= amountCents;
    return this.recordTransaction("debit", amountCents, description);
  }

  getStatement(): Readonly<Transaction[]> {
    return Object.freeze([...this.transactions]);
  }

  // ── Protected — subclasses like SavingsAccount can use these ──────────

  protected assertSufficientFunds(amountCents: number): void {
    if (this.balanceCents < amountCents) {
      throw new Error(`Insufficient funds: have ${this.balanceCents / 100}, need ${amountCents / 100}`);
    }
  }

  protected recordTransaction(
    type: TransactionType,
    amountCents: number,
    description: string,
  ): Transaction {
    const tx: Transaction = {
      id:                 `tx_${Date.now()}_${Math.random().toString(36).slice(2, 6)}`,
      type,
      amountCents,
      description,
      balanceAfterCents:  this.balanceCents,
      timestamp:          new Date(),
    };
    this.transactions.push(tx);
    return tx;
  }

  // ── Private — only this class ─────────────────────────────────────────

  private assertNotFrozen(): void {
    if (this._frozen) throw new Error(`Account ${this.accountId} is frozen`);
  }

  private assertPositiveAmount(amountCents: number): void {
    if (amountCents <= 0) throw new Error(`Amount must be positive, got ${amountCents}`);
  }
}

class SavingsAccount extends BankAccount {
  private readonly interestRatePercent: number;
  private readonly overdraftProtection: boolean;

  constructor(
    accountId: string,
    ownerId: number,
    initialBalanceCents: number,
    interestRatePercent: number,
    overdraftProtection = false,
  ) {
    super(accountId, ownerId, initialBalanceCents, "USD", "savings_salt");
    this.interestRatePercent = interestRatePercent;
    this.overdraftProtection = overdraftProtection;
  }

  applyMonthlyInterest(): Transaction {
    const interestCents = Math.floor(this.balanceCents * (this.interestRatePercent / 100 / 12));
    this.balanceCents += interestCents;
    return this.recordTransaction("credit", interestCents, "Monthly interest");  // ✅ protected
  }

  override withdraw(amountCents: number, description?: string): Transaction {
    if (this.overdraftProtection && amountCents > this.balanceCents) {
      // Override assertSufficientFunds behavior — allow overdraft:
      this.balanceCents -= amountCents;
      const tx = this.recordTransaction("debit", amountCents, description ?? "Withdrawal");
      this.recordTransaction("fee", 3500, "Overdraft fee");  // ✅ protected
      return tx;
    }
    return super.withdraw(amountCents, description);
  }
}

const checking = new BankAccount("chk_001", 1, 100000, "USD", "salt_abc");
checking.deposit(50000, "Paycheck");
checking.withdraw(25000, "Rent");
checking.balance;          // ✅ 1250.00
checking.transactions;     // ❌ private
checking.balanceCents;     // ❌ protected

const savings = new SavingsAccount("sav_001", 1, 500000, 4.5);
savings.applyMonthlyInterest();
savings.balanceCents;      // ❌ protected — not accessible outside class hierarchy
```

---

## Example 2 — real world backend use case

```ts
// JWT authentication service with true runtime privacy for secrets

class JwtAuthService {
  // True runtime private — the secret key can NEVER be extracted by external code:
  readonly #secretKey: string;
  readonly #algorithm = "HS256";
  readonly #issuer: string;

  // Compile-time private — internal state:
  private readonly tokenBlacklist = new Set<string>();
  private refreshTokenStore = new Map<number, string>();

  // Protected — subclasses like AdminAuthService may need these:
  protected readonly accessTokenTtlMs: number;
  protected readonly refreshTokenTtlMs: number;

  // Public config:
  public readonly serviceName: string;

  constructor(config: {
    secretKey: string;
    issuer: string;
    serviceName: string;
    accessTokenTtlMs?: number;
    refreshTokenTtlMs?: number;
  }) {
    this.#secretKey          = config.secretKey;
    this.#issuer             = config.issuer;
    this.serviceName         = config.serviceName;
    this.accessTokenTtlMs    = config.accessTokenTtlMs  ?? 15 * 60 * 1000;   // 15 min
    this.refreshTokenTtlMs   = config.refreshTokenTtlMs ?? 7 * 24 * 60 * 60 * 1000; // 7 days
  }

  // ── Public API ─────────────────────────────────────────────────────────

  generateAccessToken(userId: number, role: string): string {
    const payload = { userId, role, type: "access" };
    return this.sign(payload, this.accessTokenTtlMs);
  }

  generateRefreshToken(userId: number): string {
    const token = this.sign({ userId, type: "refresh" }, this.refreshTokenTtlMs);
    this.refreshTokenStore.set(userId, token);
    return token;
  }

  verifyAccessToken(token: string): { userId: number; role: string } | null {
    if (this.tokenBlacklist.has(token)) return null;
    const payload = this.verify(token);
    if (!payload || payload.type !== "access") return null;
    return { userId: payload.userId as number, role: payload.role as string };
  }

  revokeToken(token: string): void {
    this.tokenBlacklist.add(token);
  }

  revokeAllForUser(userId: number): void {
    this.refreshTokenStore.delete(userId);
    // In production: also invalidate access tokens (requires token versioning or short TTL)
  }

  // ── Protected — available to subclasses ─────────────────────────────────

  protected sign(payload: Record<string, unknown>, ttlMs: number): string {
    // Stub — uses this.#secretKey internally (but can't expose it)
    const header  = Buffer.from(JSON.stringify({ alg: this.#algorithm, typ: "JWT" })).toString("base64url");
    const body    = Buffer.from(JSON.stringify({
      ...payload,
      iss: this.#issuer,
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor((Date.now() + ttlMs) / 1000),
    })).toString("base64url");
    const sig     = this.hmac(`${header}.${body}`, this.#secretKey);
    return `${header}.${body}.${sig}`;
  }

  protected verify(token: string): Record<string, unknown> | null {
    try {
      const [header, body, sig] = token.split(".");
      const expectedSig = this.hmac(`${header}.${body}`, this.#secretKey);
      if (sig !== expectedSig) return null;
      const payload = JSON.parse(Buffer.from(body, "base64url").toString());
      if (payload.exp < Math.floor(Date.now() / 1000)) return null;
      return payload;
    } catch {
      return null;
    }
  }

  // ── Private — only this class ─────────────────────────────────────────

  private hmac(data: string, secret: string): string {
    // Stub — use crypto.createHmac in production
    return Buffer.from(`${data}:${secret}`).toString("base64url").slice(0, 20);
  }
}

// Subclass adds admin-only token generation:
class AdminAuthService extends JwtAuthService {
  generateAdminToken(adminId: number, permissions: string[]): string {
    // ✅ Can call protected sign():
    return this.sign({ adminId, permissions, type: "admin" }, this.accessTokenTtlMs / 2);
  }

  verifyAdminToken(token: string): { adminId: number; permissions: string[] } | null {
    // ✅ Can call protected verify():
    const payload = this.verify(token);
    if (!payload || payload.type !== "admin") return null;
    return { adminId: payload.adminId as number, permissions: payload.permissions as string[] };
  }
}

const authService = new JwtAuthService({
  secretKey: process.env.JWT_SECRET ?? "dev_secret",
  issuer:    "api.dev.io",
  serviceName: "AuthService",
});

const token = authService.generateAccessToken(1, "admin");
const payload = authService.verifyAccessToken(token);

// Cannot extract the secret:
(authService as any).secretKey;   // undefined — # field, truly inaccessible
authService.serviceName;           // ✅ "AuthService" — public
```

---

## Common mistakes

### Mistake 1 — Assuming TypeScript `private` protects at runtime

```ts
class Config {
  private readonly dbPassword: string;
  constructor(password: string) { this.dbPassword = password; }
}

const config = new Config("supersecret");

// ❌ WRONG assumption: TypeScript 'private' protects at runtime
// This works perfectly at runtime — TypeScript erases 'private':
const leaked = (config as any).dbPassword;   // "supersecret"
JSON.stringify(config);                       // '{"dbPassword":"supersecret"}' — leaked!

// ✅ For true runtime privacy of secrets, use # private fields:
class Config {
  readonly #dbPassword: string;
  constructor(password: string) { this.#dbPassword = password; }
  getConnectionString(host: string, db: string): string {
    return `postgres://${this.#dbPassword}@${host}/${db}`;  // used internally, never exposed
  }
}

const config2 = new Config("supersecret");
(config2 as any).dbPassword;  // undefined — # is truly inaccessible
JSON.stringify(config2);       // '{}'  — # fields are excluded from serialization
```

### Mistake 2 — Using `protected` when you mean `private`

```ts
// ❌ Marking internal helpers as protected "just in case":
class UserService {
  protected validateEmail(email: string): boolean {  // no subclass needs this
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
  protected hashPassword(pass: string): string {     // definitely shouldn't be extended
    return `hashed_${pass}`;
  }
}

// Now ANY subclass can accidentally call or override these:
class SpecialUserService extends UserService {
  override validateEmail(email: string): boolean {
    return true;  // overrides validation — silently breaks security!
  }
}

// ✅ Private unless you explicitly intend subclasses to use/override it:
class UserService {
  private validateEmail(email: string): boolean { ... }
  private hashPassword(pass: string): string { ... }
}
```

### Mistake 3 — Forgetting that `#` fields are NOT inherited in subclasses

```ts
class Base {
  #value: number = 42;

  protected getValue(): number {
    return this.#value;  // ✅ accessible here
  }
}

class Child extends Base {
  showValue(): void {
    console.log(this.#value);  // ❌ SyntaxError: #value is not accessible in Child
    // # fields belong EXCLUSIVELY to the class that declares them — not inherited
  }

  showValueCorrectly(): void {
    console.log(this.getValue());  // ✅ use the protected accessor instead
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

Build a `PasswordManager` class that demonstrates the difference between `private` and `#`:

1. Create `class PasswordVault` with:
   - `#masterKey: string` (# private — true runtime privacy)
   - `private encryptedPasswords: Map<string, string>` (TS private)
   - `public readonly vaultId: string`
   - `store(service: string, password: string): void` — encrypts with `#masterKey` and stores
   - `retrieve(service: string, pin: string): string | null` — decrypts if pin matches `#masterKey`
   - `list(): string[]` — returns service names (not passwords)

2. After creating an instance, use `(vault as any)` to show:
   - `encryptedPasswords` IS accessible at runtime (TS private is compile-time only)
   - `masterKey` property is `undefined` (# field is not visible in the object at all)

3. Show that `JSON.stringify(vault)` leaks `encryptedPasswords` but NOT `#masterKey`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed HTTP request pipeline using `protected` for extensible middleware:

```ts
class BaseHttpClient {
  protected baseUrl: string;
  protected defaultHeaders: Record<string, string>;
  private requestCount = 0;
  private readonly maxRequestsPerMinute: number;
  private requestTimestamps: number[] = [];

  constructor(baseUrl: string, maxRequestsPerMinute = 60) { ... }

  // Public:
  async get<T>(path: string, headers?: Record<string, string>): Promise<T>
  async post<T>(path: string, body: unknown, headers?: Record<string, string>): Promise<T>

  // Protected — subclasses can override these hooks:
  protected buildHeaders(extra?: Record<string, string>): Record<string, string>
  protected handleResponse<T>(response: Response): Promise<T>
  protected onRequestStart(method: string, url: string): void
  protected onRequestEnd(method: string, url: string, durationMs: number): void

  // Private — internal rate limiting:
  private checkRateLimit(): void
  private recordRequest(): void
}
```

Then create:
1. `AuthenticatedHttpClient extends BaseHttpClient` — adds `#authToken: string`, overrides `buildHeaders` to include `Authorization` header, adds `refreshToken(newToken: string): void`
2. `LoggingHttpClient extends AuthenticatedHttpClient` — overrides `onRequestStart` and `onRequestEnd` to log with timestamps

Show that external code cannot access `#authToken`, `requestTimestamps`, or `defaultHeaders`.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a `SecureConfigStore` that demonstrates layered access control for a real backend configuration system:

```ts
// Requirements:
// 1. All secrets (API keys, DB passwords, JWT secrets) stored in # fields — true runtime private
// 2. Non-secret config (timeouts, URLs, feature flags) in protected fields — subclasses can read
// 3. Public read-only accessors for safe config values only (never raw secrets)
// 4. A 'seal()' method that freezes the config after startup — no changes after sealing
// 5. An audit log (private) of every time a secret is accessed internally

class SecureConfigStore {
  // # private: all secret values
  #databasePassword: string;
  #jwtSecret: string;
  #stripeSecretKey: string;
  #sendgridApiKey: string;

  // protected: non-sensitive config
  protected databaseUrl: string;  // host+db but NOT password
  protected redisUrl: string;
  protected port: number;
  protected nodeEnv: "development" | "staging" | "production";

  // private: internal state
  private sealed = false;
  private secretAccessLog: Array<{ secret: string; accessedAt: Date; caller: string }> = [];

  // public readonly: safe info
  public readonly appName: string;
  public readonly version: string;
}
```

The class must:
- Provide `getDatabaseConnectionString(): string` — builds the full connection string using `#databasePassword` internally, but never exposes the password separately
- Provide `getJwtSecret(): string` — logs the access to `secretAccessLog` before returning (internal use for signing only)
- Provide `seal(): void` — marks sealed; after which any setter throws
- Provide `getAuditReport(): Array<{secret: string; accessedAt: Date}>` — returns the access log (without the `caller` field — that's internal)
- Show that after `JSON.stringify(config)` none of the # fields appear
- A subclass `DevelopmentConfigStore` that adds `printSafeConfig(): void` using the `protected` fields but cannot access any `#` secrets

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Declare with modifiers:
class Foo {
  public name: string;      // accessible everywhere (default)
  protected id: number;     // this class + subclasses
  private secret: string;   // this class only (compile-time)
  #key: string;             // this class only (runtime — true JS private)
  readonly tag: string;     // cannot be reassigned after construction
}

// Parameter properties (modifier required):
constructor(
  public name: string,
  protected id: number,
  private readonly secret: string,
) {}

// Override with modifier (keeps same or wider accessibility):
class Child extends Parent {
  public override name: string = "child";  // ✅ widening: protected → public is allowed
  // private override name: string = "";   // ❌ narrowing: public → private is NOT allowed
}
```

| | `public` | `protected` | `private` | `#field` |
|---|---|---|---|---|
| Same class | ✅ | ✅ | ✅ | ✅ |
| Subclass | ✅ | ✅ | ❌ | ❌ |
| Outside class | ✅ | ❌ | ❌ | ❌ |
| Runtime enforced | — | — | ❌ No | ✅ Yes |
| In interface contract | ✅ | ❌ | ❌ | ❌ |
| JSON.stringify visible | ✅ | ✅ | ✅ | ❌ |
| Inherited | ✅ | ✅ | ❌ | ❌ |
| Can bypass with `as any` | ✅ | ✅ | ✅ | ❌ |

**Decision guide:**
- Default to `private` for internal state
- Use `protected` only when you explicitly design for subclassing
- Use `#` when the field contains security-sensitive data (secrets, keys, tokens)
- Use `public readonly` for identity fields (`id`, `createdAt`) that should be readable everywhere

---

## Connected topics

- **33 — Classes in TypeScript** — the full class syntax; property declarations, parameter properties, getters/setters.
- **35 — Abstract classes** — `abstract` modifier with `protected` methods for template method pattern.
- **36 — Static members** — `static private`, `static readonly`, `static #` — the same modifiers apply to statics.
- **19 — Implementing interfaces** — interfaces only see `public` members; modifiers affect what the interface contract covers.
- **29 — Generic classes** — access modifiers and generics combine in repository/service patterns.
