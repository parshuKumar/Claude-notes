# 68 — Dependency injection basics in TS

## What is this?

**Dependency injection (DI)** is one sentence: *a thing does not build the things it needs — it is handed them.*

That's it. There is no framework in that sentence, no container, no decorator, no `reflect-metadata`. DI is a **shape you give your code**, and TypeScript's job is to make that shape checkable.

```ts
// ❌ Not injected — the service reaches out and constructs its own world.
class UserService {
  private readonly users  = new MongoUserRepository(UserModel);  // hard-wired to Mongo
  private readonly mailer = new SendGridMailer(process.env.SENDGRID_KEY!); // hard-wired to SendGrid
  private now() { return new Date(); }                            // hard-wired to the wall clock
}

// ✅ Injected — the service declares what it needs, as interfaces, and receives them.
class UserService {
  constructor(
    private readonly users:  UserRepository,   // some UserRepository. Any UserRepository.
    private readonly mailer: Mailer,
    private readonly clock:  Clock,
  ) {}
}
```

The second version cannot be constructed without someone deciding *which* repository, *which* mailer, *which* clock. That someone is the **composition root** — one file, run once at startup, that knows every concrete class in your application. Everything else knows only interfaces.

In TypeScript specifically, three things make DI feel different from the JavaScript version you may already be doing informally:

1. **Parameter properties** (`constructor(private readonly users: UserRepository) {}`) collapse declare-assign-annotate into one line.
2. **Interfaces are compile-time only** — they are erased. This is what makes DI cheap *and* what makes automatic container resolution awkward (hence "injection tokens").
3. **Structural typing** means a test fake does not have to `implements` anything or inherit from anything. If its shape matches, it is accepted.

---

## Why does it matter?

Take a realistic backend method and watch what non-injected dependencies cost you.

```ts
// The version with hidden dependencies.
class SignupService {
  async signup(requestBody: { email: string; password: string }): Promise<ApiResponse<User>> {
    const existing = await UserModel.findOne({ email: requestBody.email });   // hidden: Mongo
    if (existing) return { ok: false, error: "EMAIL_TAKEN" };

    const passwordHash = await bcrypt.hash(requestBody.password, 12);         // hidden: bcrypt, 12 rounds
    const user = await UserModel.create({
      email: requestBody.email,
      passwordHash,
      createdAt: new Date(),                                                  // hidden: wall clock
    });

    await sendgrid.send({                                                     // hidden: network I/O
      to: user.email,
      templateId: "d-welcome",
    });

    logger.info(`signup ${user._id}`);                                        // hidden: global logger
    return { ok: true, data: toDomain(user) };
  }
}
```

To test `signup` you now need: a live MongoDB (or a Mongoose mock), a real bcrypt run at 12 rounds (~250ms per test), a network stub for SendGrid, a way to freeze `Date`, and a logger that doesn't spam your test output. Most teams respond by not testing it.

Every one of those is a **hard-coded edge to the outside world**, and none of them appears in the type signature. The method claims to take a `requestBody` and return an `ApiResponse<User>`. It lies — it also takes a database, a clock, an email provider, and a log sink.

With DI, the lie becomes a signature:

```ts
class SignupService {
  constructor(
    private readonly users:  UserRepository,
    private readonly hasher: PasswordHasher,
    private readonly mailer: Mailer,
    private readonly clock:  Clock,
    private readonly logger: Logger,
  ) {}
  // signup() body is identical, but every external effect goes through a field.
}
```

What you actually buy:

- **Tests become trivial and fast.** `new SignupService(new InMemoryUserRepository(), fakeHasher, fakeMailer, fixedClock, silentLogger)` — no Docker, no network, no `jest.mock` path-string voodoo, deterministic timestamps, and you can *assert on the fake mailer* ("exactly one welcome email was sent to that address").
- **Swapping an implementation is a one-line change in one file.** SendGrid → SES is a change to the composition root. Zero service files touched.
- **The dependency graph becomes visible and reviewable.** Reading a constructor tells you everything that class touches. `grep "new SignupService"` finds every wiring site — there is exactly one.
- **Cyclic and god-object designs become obvious.** A class with 11 constructor parameters is screaming at you. With `import`-and-call, that same class looks fine.
- **Per-request state (tenant, auth context, request id) has a home** that isn't a global or an AsyncLocalStorage hack you regret.

And the honest cost: one more file (the composition root), a little constructor boilerplate, and an extra hop when you're reading code and want to know "which implementation is actually running here?".

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: the module system IS the container ───────────────────────────
// services/user.service.js
const userRepo = require("../repositories/user.repository");   // concrete, singleton, eager
const mailer   = require("../lib/mailer");
const logger   = require("../lib/logger");

async function createUser(requestBody) {
  const existing = await userRepo.findByEmail(requestBody.email);
  if (existing) throw new Error("EMAIL_TAKEN");

  const user = await userRepo.create({
    email:     requestBody.email,
    createdAt: new Date(),            // untestable without hijacking global Date
  });

  await mailer.sendWelcome(user.email);
  logger.info("created user", user.id);
  return user;
}

module.exports = { createUser };

// Test file — the only lever you have is monkey-patching the module registry:
jest.mock("../repositories/user.repository");   // string path, no type checking
jest.mock("../lib/mailer");
// Rename the file → the mock silently stops applying and the test hits the real DB.
// Import order matters. Module state leaks between test files. Nothing is checked.
```

The JS approach works, and plenty of production systems run on it. Its failure modes are all *silent*: a mistyped mock path, a stale module cache, a dependency that got added to a service six months ago and now every test quietly opens a Redis connection.

```ts
// ── TypeScript: dependencies are parameters, and they are typed ──────────────

// ── 1. Contracts. No implementation, no imports of infrastructure. ───────────
interface UserRepository {
  findByEmail(email: string): Promise<User | null>;
  create(input: CreateUserInput): Promise<User>;
}
interface Mailer { sendWelcome(email: string, displayName: string): Promise<void>; }
interface Clock  { now(): Date; }
interface Logger { info(message: string, meta?: Record<string, unknown>): void; }

// ── 2. The service depends on the contracts only. ────────────────────────────
class UserService {
  constructor(
    private readonly users:  UserRepository,
    private readonly mailer: Mailer,
    private readonly clock:  Clock,
    private readonly logger: Logger,
  ) {}

  async createUser(requestBody: CreateUserInput): Promise<User> {
    const existing = await this.users.findByEmail(requestBody.email);
    if (existing) throw new Error("EMAIL_TAKEN");

    const user = await this.users.create({ ...requestBody, createdAt: this.clock.now() });
    await this.mailer.sendWelcome(user.email, user.displayName);
    this.logger.info("user.created", { userId: user.userId });
    return user;
  }
}

// ── 3. Production wiring — the ONLY file that names concrete classes. ────────
const userService = new UserService(
  new MongoUserRepository(UserModel),
  new SendGridMailer(config.sendgridKey),
  { now: () => new Date() },
  pinoLogger,
);

// ── 4. Test wiring — same constructor, different arguments. No mocking library.
const sentEmails: string[] = [];
const testService = new UserService(
  new InMemoryUserRepository(),
  { async sendWelcome(email) { sentEmails.push(email); } },   // ✅ structurally a Mailer
  { now: () => new Date("2026-07-22T00:00:00Z") },            // frozen time
  { info() {} },                                              // silent
);
```

The revelation: **the test double is checked by the compiler**. If someone adds `sendPasswordReset` to `Mailer`, the test fake fails to compile at the wiring site — you are told immediately, at the exact line, instead of discovering it when a test hits `undefined is not a function` six months later.

---

## Syntax

```ts
// ── Parameter properties: declare + annotate + assign, in one place ──────────
class OrderService {
  constructor(
    private readonly orders: OrderRepository,   // → this.orders,  private, readonly
    private readonly clock:  Clock,             // → this.clock
    protected readonly log:  Logger,            // → this.log, visible to subclasses
    public readonly region:  string,            // → this.region, part of the public API
    private retries = 3,                        // default value, type inferred as number
  ) {}
  // No field declarations. No `this.orders = orders`. TS emits them for you.
}

// Equivalent longhand:
class OrderServiceLonghand {
  private readonly orders: OrderRepository;
  private readonly clock:  Clock;
  constructor(orders: OrderRepository, clock: Clock) {
    this.orders = orders;
    this.clock  = clock;
  }
}

// ── Injecting via an options object — better past ~4 dependencies ────────────
interface OrderServiceDeps {
  readonly orders:   OrderRepository;
  readonly payments: PaymentGateway;
  readonly mailer:   Mailer;
  readonly clock:    Clock;
  readonly logger:   Logger;
}
class OrderServiceB {
  constructor(private readonly deps: OrderServiceDeps) {}
  // call sites are named, order-independent, and a missing key is a compile error
}

// ── Factory function injection — no classes at all ───────────────────────────
function makeOrderService(deps: OrderServiceDeps) {
  return {
    async placeOrder(userId: UserId, lines: readonly OrderLine[]): Promise<Order> {
      return deps.orders.create({ userId, lines, status: "pending", currency: "USD" });
    },
  };
}
// The public type of the module is inferred:
type OrderServiceApi = ReturnType<typeof makeOrderService>;

// ── Injection tokens — needed because interfaces do not exist at runtime ─────
const TOKENS = {
  UserRepository: Symbol("UserRepository"),
  Mailer:         Symbol("Mailer"),
  Clock:          Symbol("Clock"),
} as const;

// ── A typed manual container ─────────────────────────────────────────────────
interface AppServices {
  readonly users:  UserRepository;
  readonly mailer: Mailer;
  readonly clock:  Clock;
}
function buildContainer(config: AppConfig): AppServices { /* ... */ }
```

---

## How it works — concept by concept

### Concept 1 — Parameter properties: the syntax that makes DI painless

TypeScript lets you put an access modifier (`public`, `private`, `protected`) and/or `readonly` on a constructor parameter. That single change turns the parameter into an instance field, assigned before the constructor body runs.

```ts
class PaymentService {
  constructor(
    private readonly gateway: PaymentGateway,
    private readonly orders:  OrderRepository,
  ) {}

  async charge(orderId: OrderId): Promise<PaymentResult> {
    const order = await this.orders.findById(orderId);   // ✅ this.orders exists
    if (!order) throw new Error(`Order not found: ${orderId}`);
    return this.gateway.charge(order.totalCents, order.currency);
  }
}
```

Rules and gotchas worth knowing:

```ts
// 1) The modifier is REQUIRED for the field to be created. A bare parameter is just a parameter.
class A {
  constructor(users: UserRepository) {}
  method() { /* this.users → ❌ Property 'users' does not exist on type 'A' */ }
}

// 2) `readonly` alone is enough — you do not need `private` too.
class B { constructor(readonly clock: Clock) {} }   // public + readonly

// 3) Parameter properties are assigned BEFORE the constructor body.
class C {
  constructor(private readonly users: UserRepository) {
    void this.users;             // ✅ already assigned here
  }
}

// 4) In a subclass, super() must run first — parameter properties on the SUBCLASS
//    are assigned after super() returns.
class BaseService { constructor(protected readonly logger: Logger) {} }
class AuditService extends BaseService {
  constructor(logger: Logger, private readonly audits: AuditRepository) {
    super(logger);               // ✅ must come before touching this.audits
    this.logger.info("audit service ready");
  }
}

// 5) They do NOT work on overload signatures — only on the implementation signature.
class D {
  constructor(users: UserRepository);
  constructor(users: UserRepository, clock: Clock);
  constructor(private readonly users: UserRepository, private readonly clock?: Clock) {}
  // ✅ modifiers only on the implementation
}

// 6) `useDefineForClassFields` / ES2022 class fields interaction:
//    parameter properties are emitted as plain assignments, so they are unaffected by
//    the [[Define]] vs [[Set]] semantics that bite ordinary declared fields.
```

`private` here is a **compile-time** modifier — at runtime the field is a normal enumerable property, visible in `console.log(service)` and reachable from JS. If you want genuine runtime privacy, use `#users` — but then it cannot be a parameter property, and you must assign it in the body:

```ts
class StrictService {
  #users: UserRepository;                     // truly private at runtime
  constructor(users: UserRepository) { this.#users = users; }
}
```

For DI purposes `private readonly` is almost always the right choice: `readonly` stops a method from reassigning a dependency mid-flight, and `private` stops callers reaching through the service into its repository.

### Concept 2 — Programming to interfaces (and why structural typing changes the game)

The whole value of DI comes from the *type* of the constructor parameter being an interface, not a class.

```ts
// ❌ Injected, but pointless — the type is the concrete class.
class ReportService {
  constructor(private readonly orders: MongoOrderRepository) {}
  // You can inject a different instance, but never a different implementation.
  // Every consumer transitively depends on mongoose. Tests still need Mongo.
}

// ✅ Injected against a contract.
class ReportService {
  constructor(private readonly orders: OrderRepository) {}
}
```

In a nominal language (Java, C#) a class must *declare* that it implements the interface. TypeScript is **structural**: anything with the right shape is assignable, declaration or not. That has three big consequences for DI.

```ts
interface Clock { now(): Date; }

// (a) An object literal is a valid dependency. No class, no `implements`, no library.
const fixedClock: Clock = { now: () => new Date("2026-07-22T09:00:00Z") };

// (b) A built-in or third-party object can satisfy your interface accidentally-on-purpose.
interface Cache {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
}
// A thin adapter over ioredis, written inline — no wrapper class needed:
const redisCache: Cache = {
  get: (key) => redis.get(key),
  set: async (key, value) => { await redis.set(key, value); },
};

// (c) A partial fake is a compile error, which is exactly what you want.
// const brokenClock: Clock = {};   // ❌ Property 'now' is missing
```

Still, `implements` is worth writing on production classes — not because it's required, but because it moves the error to the *implementation* file instead of the wiring file:

```ts
// Without `implements`, a typo surfaces 300 lines away at `new UserService(repo, ...)`.
class MongoUserRepository implements UserRepository {
  //                       ^ error appears HERE if a method is missing or mistyped
  async findByEmail(email: string): Promise<User | null> { /* ... */ return null; }
  async create(input: CreateUserInput): Promise<User> { /* ... */ throw new Error(); }
}
```

One subtlety carried over from doc 67: `implements` **checks**, it does not **contextually type**. Parameters still need annotations under `noImplicitAny`.

Interface design rules that matter specifically for injectability:

```ts
// ❌ Too wide. The service only ever calls findById, but now every fake must
//    implement 9 methods, and any change to the repo breaks every test.
class InvoiceService { constructor(private readonly users: UserRepository) {} }

// ✅ Depend on the narrowest contract that does the job (Interface Segregation).
interface UserLookup { findById(userId: UserId): Promise<User | null>; }
class InvoiceService { constructor(private readonly users: UserLookup) {} }
// MongoUserRepository satisfies UserLookup structurally — no extra declaration needed.
// The test fake is now: { async findById() { return alice; } }
```

That last trick is uniquely pleasant in TypeScript: because typing is structural, you can define a tiny consumer-side interface and the existing production class satisfies it for free.

### Concept 3 — The composition root

A **composition root** is the single place in your application where concrete implementations are named and wired. It runs once, at startup, as close to the entry point as possible. Everywhere else, `new SomeConcreteThing()` is forbidden.

```ts
// ── src/config.ts — parse and validate the environment once ──────────────────
export interface AppConfig {
  readonly nodeEnv:      "development" | "test" | "production";
  readonly port:         number;
  readonly mongoUri:     string;
  readonly redisUrl:     string;
  readonly sendgridKey:  string;
  readonly jwtSecret:    string;
  readonly bcryptRounds: number;
}

export function loadConfig(env: NodeJS.ProcessEnv): AppConfig {
  const required = (key: string): string => {
    const value = env[key];
    if (!value) throw new Error(`Missing required env var: ${key}`);
    return value;
  };
  return {
    nodeEnv:      (env.NODE_ENV ?? "development") as AppConfig["nodeEnv"],
    port:         Number(env.PORT ?? 3000),
    mongoUri:     required("MONGO_URI"),
    redisUrl:     required("REDIS_URL"),
    sendgridKey:  required("SENDGRID_API_KEY"),
    jwtSecret:    required("JWT_SECRET"),
    bcryptRounds: Number(env.BCRYPT_ROUNDS ?? 12),
  };
}

// ── src/container.ts — the composition root ─────────────────────────────────
// The shape of everything the app can hand out. This is your "container type".
export interface AppContainer {
  readonly config:       AppConfig;
  readonly clock:        Clock;
  readonly logger:       Logger;
  readonly users:        UserRepository;
  readonly orders:       OrderRepository;
  readonly mailer:       Mailer;
  readonly hasher:       PasswordHasher;
  readonly tokens:       TokenIssuer;
  readonly userService:  UserService;
  readonly authService:  AuthService;
  readonly orderService: OrderService;
}

export function buildContainer(config: AppConfig): AppContainer {
  // Level 0 — leaf dependencies with no dependencies of their own.
  const clock:  Clock  = { now: () => new Date() };
  const logger: Logger = createPinoLogger({ level: config.nodeEnv === "production" ? "info" : "debug" });

  // Level 1 — infrastructure adapters.
  const users:  UserRepository  = new MongoUserRepository(UserModel);
  const orders: OrderRepository = new MongoOrderRepository(OrderModel);
  const mailer: Mailer          = new SendGridMailer(config.sendgridKey, logger);
  const hasher: PasswordHasher  = new BcryptHasher(config.bcryptRounds);
  const tokens: TokenIssuer     = new JwtTokenIssuer(config.jwtSecret, clock);

  // Level 2 — application services, built from the above.
  const userService  = new UserService(users, mailer, clock, logger);
  const authService  = new AuthService(users, hasher, tokens, clock, logger);
  const orderService = new OrderService(orders, users, clock, logger);

  return {
    config, clock, logger, users, orders, mailer, hasher, tokens,
    userService, authService, orderService,
  };
}

// ── src/main.ts — the entry point ───────────────────────────────────────────
async function main(): Promise<void> {
  const config    = loadConfig(process.env);
  await mongoose.connect(config.mongoUri);
  const container = buildContainer(config);
  const app       = createApp(container);            // routes receive the container
  app.listen(config.port, () => container.logger.info("listening", { port: config.port }));
}
void main();
```

Why this beats a "clever" container:

- It is **plain TypeScript**. Ordering, conditionals, and env-specific branches are just code.
- Every dependency is **checked at compile time**. A missing or mistyped one fails `tsc`, not a request at 3am.
- **Go-to-definition works.** Click `users` and land on `MongoUserRepository`. No magic string lookup.
- Startup order is explicit — no lazy-resolution surprises where a connection isn't open yet.

Layer the container by environment when you need to:

```ts
export function buildContainer(config: AppConfig): AppContainer {
  const clock: Clock = config.nodeEnv === "test"
    ? { now: () => new Date("2026-07-22T00:00:00Z") }
    : { now: () => new Date() };

  const mailer: Mailer = config.nodeEnv === "production"
    ? new SendGridMailer(config.sendgridKey, logger)
    : new ConsoleMailer(logger);        // dev/test: print, don't send

  /* ...rest identical... */
}
```

### Concept 4 — Factory functions: DI without classes

Nothing about DI requires classes. Closures capture dependencies just as well, and in a Node codebase that already leans functional this reads better.

```ts
// ── Dependencies as an explicit, named object ───────────────────────────────
interface AuthServiceDeps {
  readonly users:  UserRepository;
  readonly hasher: PasswordHasher;
  readonly tokens: TokenIssuer;
  readonly clock:  Clock;
}

// ── The factory returns an object of functions closing over deps ────────────
export function createAuthService({ users, hasher, tokens, clock }: AuthServiceDeps) {
  // Private helpers live here — genuinely unreachable from outside, unlike `private`.
  const isLockedOut = (user: User): boolean =>
    user.lockedUntil !== null && user.lockedUntil > clock.now();

  return {
    async login(email: string, password: string): Promise<ApiResponse<{ authToken: string }>> {
      const user = await users.findByEmail(email);
      if (!user) return { ok: false, error: "INVALID_CREDENTIALS" };
      if (isLockedOut(user)) return { ok: false, error: "ACCOUNT_LOCKED" };

      const valid = await hasher.verify(password, user.passwordHash);
      if (!valid) return { ok: false, error: "INVALID_CREDENTIALS" };

      const authToken = await tokens.issue({ userId: user.userId, role: user.role });
      return { ok: true, data: { authToken } };
    },

    async logout(authToken: string): Promise<void> {
      await tokens.revoke(authToken);
    },
  };
}

// ── The public type is inferred, not hand-written ───────────────────────────
export type AuthService = ReturnType<typeof createAuthService>;
```

Two ways to state the contract, with different trade-offs:

```ts
// (a) Infer the type from the factory — zero duplication, but the factory IS the contract,
//     so you cannot write a second implementation against a pre-existing interface.
export type AuthService = ReturnType<typeof createAuthService>;

// (b) Declare the interface first, annotate the factory's return type — the contract is
//     independent, multiple implementations are possible, and errors point at the factory.
export interface AuthService {
  login(email: string, password: string): Promise<ApiResponse<{ authToken: string }>>;
  logout(authToken: string): Promise<void>;
}
export function createAuthService(deps: AuthServiceDeps): AuthService { /* ... */ }
```

Prefer (b) at boundaries you fake in tests or implement twice; (a) is fine for internal, single-implementation modules.

A partial-application variant is useful for route handlers:

```ts
// Each handler is a function of deps returning an Express handler.
export const makeGetUserHandler =
  (deps: { users: UserRepository }) =>
  async (req: Request, res: Response): Promise<void> => {
    const user = await deps.users.findById(parseUserId(req.params.userId));
    if (!user) { res.status(404).json({ ok: false, error: "NOT_FOUND" }); return; }
    res.json({ ok: true, data: user });
  };

// Wiring: router.get("/users/:userId", makeGetUserHandler(container));
```

Note `makeGetUserHandler` asks for `{ users: UserRepository }` — not the whole container. The narrower the ask, the easier the test.

### Concept 5 — Why DI makes tests trivial

This is the payoff, so it deserves concrete code rather than assertion. Three classic fakes: a repository, a clock, and a mailer.

```ts
// ── Fake 1: the in-memory repository (from doc 67) ──────────────────────────
class InMemoryUserRepository implements UserRepository {
  private readonly rows = new Map<UserId, User>();
  private seq = 0;

  constructor(seed: readonly User[] = []) {
    for (const user of seed) this.rows.set(user.userId, user);
  }
  async findById(userId: UserId): Promise<User | null> { return this.rows.get(userId) ?? null; }
  async findByEmail(email: string): Promise<User | null> {
    return [...this.rows.values()].find((u) => u.email === email.toLowerCase()) ?? null;
  }
  async create(input: CreateUserInput): Promise<User> {
    const user: User = { ...input, userId: parseUserId(`usr_${++this.seq}`) };
    this.rows.set(user.userId, user);
    return user;
  }
}

// ── Fake 2: a controllable clock ────────────────────────────────────────────
class FakeClock implements Clock {
  constructor(private current: Date = new Date("2026-07-22T00:00:00Z")) {}
  now(): Date { return new Date(this.current); }          // copy — callers can't mutate ours
  advance(ms: number): void { this.current = new Date(this.current.getTime() + ms); }
  setTo(iso: string): void { this.current = new Date(iso); }
}

// ── Fake 3: a recording mailer (a "spy") ────────────────────────────────────
interface SentEmail { readonly to: string; readonly template: string; readonly vars: Record<string, string>; }

class RecordingMailer implements Mailer {
  readonly sent: SentEmail[] = [];
  async sendWelcome(to: string, displayName: string): Promise<void> {
    this.sent.push({ to, template: "welcome", vars: { displayName } });
  }
  async sendPasswordReset(to: string, resetToken: string): Promise<void> {
    this.sent.push({ to, template: "password-reset", vars: { resetToken } });
  }
  lastTo(email: string): SentEmail | undefined {
    return [...this.sent].reverse().find((sent) => sent.to === email);
  }
}

// ── Fake 4: one that fails on demand — testing the unhappy path ─────────────
class FlakyMailer implements Mailer {
  constructor(private readonly failures: number) {}
  private attempts = 0;
  async sendWelcome(): Promise<void> {
    if (++this.attempts <= this.failures) throw new Error("SMTP 421 service unavailable");
  }
  async sendPasswordReset(): Promise<void> {}
}

// ── The test itself: three lines of setup, no framework magic ───────────────
import assert from "node:assert/strict";
import { test } from "node:test";

test("signup creates a user and sends exactly one welcome email", async () => {
  const users  = new InMemoryUserRepository();
  const mailer = new RecordingMailer();
  const clock  = new FakeClock("2026-07-22T09:30:00Z" as unknown as Date);
  const service = new UserService(users, mailer, clock, { info() {}, error() {} });

  const user = await service.createUser({
    email:       "ada@example.com",
    displayName: "Ada Lovelace",
    passwordHash: "hashed",
  });

  assert.equal(mailer.sent.length, 1);
  assert.equal(mailer.lastTo("ada@example.com")?.template, "welcome");
  assert.equal((await users.findById(user.userId))?.displayName, "Ada Lovelace");
});

test("a session expires exactly 30 minutes after issue", async () => {
  const clock   = new FakeClock();
  const service = new SessionService(new InMemorySessionRepository(), clock);

  const session = await service.issue(parseUserId("usr_1"));
  clock.advance(29 * 60_000);
  assert.equal(await service.isValid(session.authToken), true);
  clock.advance(2 * 60_000);              // now 31 minutes
  assert.equal(await service.isValid(session.authToken), false);
});
```

Compare with the non-DI equivalent: `jest.useFakeTimers()` (global, leaks across tests, breaks anything using real timers), `jest.mock("../lib/mailer")` (a string path, unchecked, silently stops working on rename), and an actual MongoDB in CI.

The deeper win is **compile-time coverage of your test doubles**. Add `sendInvoice` to `Mailer` and `RecordingMailer` fails to compile with a precise message. A `jest.mock` auto-mock happily returns `undefined` and your test passes while production breaks.

A useful helper for the "I only care about two of the eight dependencies" case:

```ts
// Build a container of no-op fakes, then override just what the test cares about.
function testContainer(overrides: Partial<AppContainer> = {}): AppContainer {
  const clock  = new FakeClock();
  const users  = new InMemoryUserRepository();
  const logger: Logger = { info() {}, error() {}, warn() {} };
  const base: AppContainer = {
    config: testConfig, clock, logger, users,
    orders: new InMemoryOrderRepository(),
    mailer: new RecordingMailer(),
    hasher: { async hash(p) { return `hashed:${p}`; }, async verify(p, h) { return h === `hashed:${p}`; } },
    tokens: new FakeTokenIssuer(clock),
    userService:  {} as UserService,     // filled in below
    authService:  {} as AuthService,
    orderService: {} as OrderService,
  };
  return { ...base, ...overrides };
}

// const container = testContainer({ mailer: new FlakyMailer(2) });
```

`Partial<AppContainer>` gives you the override ergonomics of a container without a container. Note the `{} as UserService` casts — a sign you should split `AppContainer` into `Infrastructure` and `Services` so that only the leaves need faking. Do that split early.

### Concept 6 — Interfaces vanish at runtime, so automatic DI needs tokens

This is *the* TypeScript-specific wrinkle in DI, and the reason DI containers in TS look stranger than in Java or C#.

```ts
interface Mailer { sendWelcome(to: string, displayName: string): Promise<void>; }
```

compiles to… nothing. Zero JavaScript. `Mailer` exists only in the type checker.

```ts
// ❌ All of these are impossible:
// container.resolve(Mailer);          // Mailer is not a value
// container.register(Mailer, sendgridMailer);
// if (x instanceof Mailer) { }        // no runtime representation
// typeof Mailer                       // ❌ 'Mailer' only refers to a type
```

So any container that resolves dependencies at runtime needs a **runtime key** — an *injection token* — plus a way to associate a static type with it.

**Approach A — a `symbol` token map plus a typed registry:**

```ts
// Tokens are runtime values; the type association is carried by a mapping interface.
const TOKENS = {
  UserRepository: Symbol("UserRepository"),
  Mailer:         Symbol("Mailer"),
  Clock:          Symbol("Clock"),
} as const;

// The map from token → the type it provides.
interface TokenTypeMap {
  [TOKENS.UserRepository]: UserRepository;
  [TOKENS.Mailer]:         Mailer;
  [TOKENS.Clock]:          Clock;
}
// NOTE: this requires `unique symbol` types, which `as const` on a Symbol() call does
// NOT produce. In practice you declare them individually:
const USER_REPOSITORY: unique symbol = Symbol("UserRepository");
const MAILER:          unique symbol = Symbol("Mailer");
```

That works but is fiddly. **Approach B — a branded token class** is far more common and much nicer:

```ts
// A token that carries its type in a phantom field. `_type` never exists at runtime.
class Token<T> {
  declare readonly _type: T;            // `declare` → type-only, emits no JS
  constructor(readonly name: string) {}
  toString(): string { return `Token(${this.name})`; }
}

const UserRepositoryToken = new Token<UserRepository>("UserRepository");
const MailerToken         = new Token<Mailer>("Mailer");
const ClockToken          = new Token<Clock>("Clock");

// A minimal, fully type-safe container.
class Container {
  private readonly factories = new Map<Token<unknown>, (c: Container) => unknown>();
  private readonly instances = new Map<Token<unknown>, unknown>();

  register<T>(token: Token<T>, factory: (c: Container) => T): this {
    this.factories.set(token as Token<unknown>, factory as (c: Container) => unknown);
    return this;
  }

  resolve<T>(token: Token<T>): T {
    const cached = this.instances.get(token as Token<unknown>);
    if (cached !== undefined) return cached as T;

    const factory = this.factories.get(token as Token<unknown>);
    if (!factory) throw new Error(`Nothing registered for ${token.name}`);

    const instance = factory(this) as T;
    this.instances.set(token as Token<unknown>, instance);   // singleton by default
    return instance;
  }
}

// Registration — factories can resolve their own dependencies.
const container = new Container()
  .register(ClockToken,          () => ({ now: () => new Date() }))
  .register(UserRepositoryToken, () => new MongoUserRepository(UserModel))
  .register(MailerToken,         (c) => new SendGridMailer(config.sendgridKey));

const mailer = container.resolve(MailerToken);   // ✅ inferred as Mailer, not unknown
// container.resolve(MailerToken).findById(...); // ❌ Property 'findById' does not exist
```

`Token<T>` gets you full inference on `resolve`. But notice what it still cannot do: it cannot know that `SendGridMailer`'s constructor wants a `Logger`. You wrote that by hand in the factory. **Automatic constructor injection requires runtime type information**, which TypeScript deletes — which is exactly what `reflect-metadata` and `emitDecoratorMetadata` exist to partially restore (Concept 9).

**Approach C — skip tokens entirely.** A plain object *is* a typed container:

```ts
interface AppContainer { readonly users: UserRepository; readonly mailer: Mailer; readonly clock: Clock; }
const container: AppContainer = { users, mailer, clock };
container.mailer;      // ✅ Mailer. No token, no Map, no cast, no `unknown`.
```

For 95% of Node backends this is the correct answer. Reach for `Token<T>` only when you genuinely need *runtime* resolution — plugin systems, dynamically loaded modules, per-request scopes with many participants.

### Concept 7 — Lifetimes: singleton vs per-request (vs transient)

Every dependency has a **lifetime**: how long one instance lives and who shares it.

| Lifetime | Created | Shared by | Typical examples |
|---|---|---|---|
| **Singleton** | once at startup | the whole process | DB pool, Redis client, logger, config, repositories, stateless services |
| **Scoped (per-request)** | once per HTTP request / job | everything in that request | request id, auth context, tenant scope, DataLoader, DB transaction |
| **Transient** | every time it is asked for | nobody | short-lived builders, things holding mutable per-operation state |

Getting this wrong is one of the few DI bugs that is genuinely dangerous.

```ts
// ❌ CATASTROPHIC: a singleton holding per-request state.
class RequestContextService {
  private currentUserId: UserId | null = null;             // shared across ALL requests
  setUser(userId: UserId): void { this.currentUserId = userId; }
  getUser(): UserId | null { return this.currentUserId; }
}
const contextService = new RequestContextService();        // one instance, whole process
// Two concurrent requests: A sets alice, B sets bob, A reads → bob.
// You just served Alice Bob's data. This is a real, common, production incident.
```

The fix is to make per-request state a **parameter** or a **per-request container**, never singleton field state.

```ts
// ── The request-scoped context type ─────────────────────────────────────────
export interface RequestContext {
  readonly requestId: string;
  readonly userId:    UserId | null;
  readonly orgId:     OrgId | null;
  readonly authToken: string | null;
  readonly startedAt: Date;
}

// ── Option 1: pass the context explicitly (simplest, most honest) ───────────
class ProjectService {
  constructor(private readonly projects: ProjectRepository, private readonly logger: Logger) {}

  async listProjects(ctx: RequestContext): Promise<Project[]> {
    if (!ctx.orgId) throw new Error("UNAUTHENTICATED");
    this.logger.info("projects.list", { requestId: ctx.requestId, orgId: ctx.orgId });
    return this.projects.findMany({ orgId: ctx.orgId });
  }
}
// Verbose, but the compiler guarantees no handler forgets the tenant scope.

// ── Option 2: a per-request container built from the singleton one ──────────
export interface RequestContainer {
  readonly ctx:      RequestContext;
  readonly logger:   Logger;                   // child logger bound to requestId
  readonly projects: ProjectRepository;        // tenant-scoped decorator
  readonly projectService: ProjectService;
}

export function buildRequestContainer(app: AppContainer, ctx: RequestContext): RequestContainer {
  // Singletons are reused; only the request-shaped things are rebuilt.
  const logger   = app.logger.child({ requestId: ctx.requestId, userId: ctx.userId });
  const projects = ctx.orgId
    ? new TenantScopedProjectRepository(app.projects, ctx.orgId)   // wraps the singleton
    : app.projects;
  return { ctx, logger, projects, projectService: new ProjectService(projects, logger) };
}

// Express middleware turns every request into a container:
app.use((req, res, next) => {
  const ctx: RequestContext = {
    requestId: randomUUID(),
    userId:    req.auth?.userId ?? null,
    orgId:     req.auth?.orgId ?? null,
    authToken: req.headers.authorization?.replace(/^Bearer /, "") ?? null,
    startedAt: new Date(),
  };
  req.container = buildRequestContainer(appContainer, ctx);   // see doc 53 for the typing
  next();
});
```

The cost of Option 2 is constructing a handful of small objects per request — measured in microseconds, irrelevant next to a single database round trip. The benefit is that tenant scoping is structural: a handler that uses `req.container.projects` **cannot** see another org's rows.

Which things must be per-request:

```ts
// ✅ singleton — stateless, or state is intentionally global
const clock: Clock = { now: () => new Date() };
const pool = new Pg.Pool({ connectionString: config.databaseUrl });   // the POOL is a singleton
const users = new MongoUserRepository(UserModel);

// ✅ per-request — carries request identity or mutable state
const loader   = new DataLoader<UserId, User>(batchLoadUsers);        // cache MUST NOT be shared
const tx       = await pool.connect();                                // a CLIENT is per-operation
const scoped   = new TenantScopedProjectRepository(projects, ctx.orgId);
const reqLogger = logger.child({ requestId: ctx.requestId });
```

A DataLoader as a singleton is the subtle version of the same bug: its cache would serve stale, cross-tenant rows forever.

### Concept 8 — Circular dependencies

Two services that need each other is DI's classic trap, and it shows up in two distinct forms.

**Form 1 — module-level import cycle.** `a.ts` imports `b.ts`, `b.ts` imports `a.ts`. TypeScript compiles it; Node's module loader gives one of them a partially-initialised module object, and you get `undefined` at runtime.

```ts
// ── order.service.ts
import { InvoiceService } from "./invoice.service";      // ← cycle
export class OrderService {
  constructor(private readonly invoices: InvoiceService) {}
}

// ── invoice.service.ts
import { OrderService } from "./order.service";          // ← cycle
export class InvoiceService {
  constructor(private readonly orders: OrderService) {}
}

// buildContainer() cannot construct either — each needs the other first:
// const orders   = new OrderService(invoices);   // invoices doesn't exist yet
// const invoices = new InvoiceService(orders);   // orders doesn't exist yet
```

Note that a cycle of *types only* is harmless — `import type` is erased:

```ts
import type { Invoice } from "./invoice.types";   // ✅ erased, no runtime cycle
```

Under `"importsNotUsedAsValues": "error"` or `verbatimModuleSyntax`, using `import type` consistently eliminates a surprising share of "cycles".

**Fix 1 — extract the shared contract (best).** Usually a cycle means one third concept is hiding.

```ts
// ── contracts.ts — no implementations, no cycle ─────────────────────────────
export interface OrderLookup  { findById(orderId: OrderId): Promise<Order | null>; }
export interface InvoiceIssuer { issueFor(order: Order): Promise<Invoice>; }

// ── order.service.ts — depends on the narrow contract, not the class ────────
import type { InvoiceIssuer } from "./contracts";
export class OrderService {
  constructor(private readonly orders: OrderRepository, private readonly invoices: InvoiceIssuer) {}
}

// ── invoice.service.ts ──────────────────────────────────────────────────────
import type { OrderLookup } from "./contracts";
export class InvoiceService implements InvoiceIssuer {
  constructor(private readonly orders: OrderLookup) {}
  async issueFor(order: Order): Promise<Invoice> { /* ... */ throw new Error(); }
}

// ── container.ts — still a construction cycle? No: InvoiceService needs only the
//    REPOSITORY, not OrderService. The cycle dissolved when the contract narrowed.
const invoiceService = new InvoiceService(orderRepository);
const orderService   = new OrderService(orderRepository, invoiceService);
```

**Fix 2 — invert with events.** If A must tell B something but B must also tell A something, that is usually a message, not a call.

```ts
interface DomainEvents {
  "order.paid":     { orderId: OrderId; userId: UserId; totalCents: number };
  "invoice.issued": { invoiceId: InvoiceId; orderId: OrderId };
}
interface EventBus {
  publish<K extends keyof DomainEvents>(event: K, payload: DomainEvents[K]): Promise<void>;
  subscribe<K extends keyof DomainEvents>(event: K, handler: (payload: DomainEvents[K]) => Promise<void>): void;
}

class OrderService {
  constructor(private readonly orders: OrderRepository, private readonly bus: EventBus) {}
  async markPaid(orderId: OrderId): Promise<void> {
    const order = await this.orders.update(orderId, { status: "paid" });
    if (!order) throw new Error("Order vanished");
    await this.bus.publish("order.paid", { orderId, userId: order.userId, totalCents: order.totalCents });
  }
}
// InvoiceService subscribes to "order.paid". Neither service imports the other.
```

**Fix 3 — lazy injection (the escape hatch).** Inject a getter instead of the instance.

```ts
class OrderService {
  // A thunk: resolved on first use, after both objects exist.
  constructor(private readonly getInvoices: () => InvoiceService) {}
  async refund(orderId: OrderId): Promise<void> {
    await this.getInvoices().credit(orderId);      // resolved lazily
  }
}

const orderService   = new OrderService(() => invoiceService);   // captures the binding
const invoiceService = new InvoiceService(orderService);         // `let`/hoisting caveats apply
```

This works, and it hides a design smell. Use it to unblock a refactor, not as the destination.

**Fix 4 — setter injection.** Construct both, then wire.

```ts
class OrderService {
  private invoices?: InvoiceService;
  setInvoices(invoices: InvoiceService): void { this.invoices = invoices; }
  private requireInvoices(): InvoiceService {
    if (!this.invoices) throw new Error("OrderService.setInvoices was never called");
    return this.invoices;
  }
}
```

The type is now `InvoiceService | undefined` forever, and every use needs a guard. Constructor injection's whole appeal is that a constructed object is fully valid; setter injection gives that up. Last resort.

### Concept 9 — Decorator-based DI, honestly

`tsyringe` and `InversifyJS` let you write this:

```ts
import "reflect-metadata";
import { injectable, inject, singleton, container } from "tsyringe";

@injectable()
class SendGridMailer implements Mailer {
  constructor(
    @inject("SENDGRID_KEY") private readonly apiKey: string,
    private readonly logger: PinoLogger,        // resolved automatically — it's a CLASS
  ) {}
  async sendWelcome(to: string, displayName: string): Promise<void> { /* ... */ }
}

@singleton()
class UserService {
  constructor(
    @inject("UserRepository") private readonly users: UserRepository,   // interface → token needed
    private readonly mailer: SendGridMailer,                            // class → automatic
  ) {}
}

container.register("UserRepository", { useClass: MongoUserRepository });
container.registerInstance("SENDGRID_KEY", process.env.SENDGRID_API_KEY!);
const userService = container.resolve(UserService);   // graph built automatically
```

**How it actually works.** With `experimentalDecorators: true` and `emitDecoratorMetadata: true`, TypeScript emits, for every decorated class, a `design:paramtypes` metadata array containing the *runtime constructor functions* of the parameter types. `reflect-metadata` provides the `Reflect.getMetadata` API to read it back. The container reads that array and recursively resolves each entry.

That mechanism has a hard limit, and it is the crux of this whole doc:

```ts
// `design:paramtypes` can only record RUNTIME values.
@injectable()
class UserService {
  constructor(
    private readonly users:  UserRepository,   // interface → emitted as `Object`. Useless.
    private readonly mailer: SendGridMailer,   // class     → emitted as SendGridMailer. Works.
    private readonly rounds: number,           // primitive → emitted as `Number`. Useless.
  ) {}
}
// Which is why every interface dependency needs an explicit @inject("Token").
```

So decorator DI gives you automatic wiring **only for concrete classes** — precisely the dependencies you least want to depend on. For every interface (the ones that matter) you are back to hand-written string or symbol tokens, plus a registration line. The magic evaporates exactly where you wanted it.

The rest of the honest ledger:

**Costs**
- `import "reflect-metadata"` must execute **once, before any decorated class is loaded**, in the true entry point. Get the order wrong and you get `Cannot read properties of undefined` from inside the container, with a stack trace pointing nowhere useful.
- `experimentalDecorators` is the **legacy** decorator proposal. TypeScript 5.0 shipped standard (Stage 3) decorators, which do **not** support `emitDecoratorMetadata` and do **not** support parameter decorators at all. tsyringe/Inversify still require the legacy flag. You are pinned to a deprecated mode.
- Bundlers, `esbuild`, `swc`, and Vite need explicit configuration for legacy decorators + metadata emit; several combinations silently drop metadata, producing a container that fails at runtime while `tsc` is happy.
- Errors move from **compile time** to **runtime**. A forgotten registration is `Error: Cannot inject the dependency at position 0` on first request, not a red squiggle.
- Your **domain classes now import a DI library**. `@injectable()` in a service file couples business logic to infrastructure — the exact coupling DI exists to remove.
- Go-to-definition on a dependency lands on the interface, not the implementation. Finding "what actually runs" means grepping registration strings.
- Test setup becomes container setup: `container.createChildContainer()`, `registerInstance`, `container.clearInstances()` in `afterEach`. That is more machinery than `new UserService(fake1, fake2)`.

**Genuine benefits**
- At 60+ services with deep graphs, manual wiring is a long file that people fight over in merge conflicts. A container absorbs that.
- Request scoping, disposal hooks, and lazy resolution are built in and tested. Rolling your own `@scoped(Lifecycle.ContainerScoped)` correctly is real work.
- If you use **NestJS**, this is not a choice — the whole framework is built on it, module-scoped providers work well, and fighting it is worse than adopting it.
- Plugin architectures where modules register themselves at runtime need runtime resolution by definition.

**The recommendation.** Start with a hand-written composition root. It is ~100 lines for a substantial service, fully type-checked, zero dependencies, zero config flags, and trivially debuggable. Move to a container when the wiring file genuinely hurts — and notice that "it hurts" often means "this app should be two apps".

```ts
// The manual equivalent of the tsyringe example above — no decorators, no metadata,
// no reflect-metadata import order hazard, and every error is a compile error.
const logger: Logger = createPinoLogger({ level: "info" });
const mailer: Mailer = new SendGridMailer(config.sendgridKey, logger);
const users:  UserRepository = new MongoUserRepository(UserModel);
const userService = new UserService(users, mailer, clock, logger);
```

Nine lines of decorators and two config flags replaced by four lines of assignment.

---

## Example 1 — basic

```ts
// A complete, minimal, compile-valid DI setup: contracts, one service,
// a production wiring, and a test wiring — no framework of any kind.

// ════════════════════════════════════════════════════════════════════════════
// 1. Domain types
// ════════════════════════════════════════════════════════════════════════════

declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type UserId = Brand<string, "UserId">;
export const parseUserId = (raw: string): UserId => {
  if (raw.length === 0) throw new Error("userId must not be empty");
  return raw as UserId;
};

export interface User {
  readonly userId:      UserId;
  readonly email:       string;
  readonly displayName: string;
  readonly status:      "active" | "suspended";
  readonly createdAt:   Date;
}

export type CreateUserInput = Omit<User, "userId" | "createdAt">;

// A discriminated result type used by every service method (see doc 58).
export type ApiResponse<T> =
  | { readonly ok: true;  readonly data: T }
  | { readonly ok: false; readonly error: string };

// ════════════════════════════════════════════════════════════════════════════
// 2. The contracts — everything the service needs from the outside world
// ════════════════════════════════════════════════════════════════════════════

export interface UserRepository {
  findById(userId: UserId): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(input: CreateUserInput): Promise<User>;
}

export interface Mailer {
  sendWelcome(to: string, displayName: string): Promise<void>;
}

export interface Clock {
  now(): Date;
}

export interface Logger {
  info(message: string, meta?: Record<string, unknown>): void;
  error(message: string, meta?: Record<string, unknown>): void;
}

// ════════════════════════════════════════════════════════════════════════════
// 3. The service — constructor injection via parameter properties
// ════════════════════════════════════════════════════════════════════════════

export class UserService {
  constructor(
    private readonly users:  UserRepository,   // ← parameter properties: declared,
    private readonly mailer: Mailer,           //   annotated and assigned in one line
    private readonly clock:  Clock,
    private readonly logger: Logger,
  ) {}

  async register(requestBody: CreateUserInput): Promise<ApiResponse<User>> {
    const existing = await this.users.findByEmail(requestBody.email);
    if (existing) {
      this.logger.info("register.duplicate", { email: requestBody.email });
      return { ok: false, error: "EMAIL_ALREADY_REGISTERED" };
    }

    const user = await this.users.create({
      ...requestBody,
      email: requestBody.email.toLowerCase(),
    });

    // Email failure must not fail the registration — log it and move on.
    try {
      await this.mailer.sendWelcome(user.email, user.displayName);
    } catch (cause) {
      this.logger.error("register.welcome_email_failed", {
        userId: user.userId,
        cause:  cause instanceof Error ? cause.message : String(cause),
      });
    }

    this.logger.info("register.ok", { userId: user.userId, at: this.clock.now().toISOString() });
    return { ok: true, data: user };
  }

  async suspend(userId: UserId): Promise<ApiResponse<User>> {
    const user = await this.users.findById(userId);
    if (!user) return { ok: false, error: "USER_NOT_FOUND" };
    if (user.status === "suspended") return { ok: false, error: "ALREADY_SUSPENDED" };
    return { ok: true, data: { ...user, status: "suspended" } };
  }
}

// ════════════════════════════════════════════════════════════════════════════
// 4. Implementations — infrastructure lives only in these classes
// ════════════════════════════════════════════════════════════════════════════

export class InMemoryUserRepository implements UserRepository {
  private readonly rows = new Map<UserId, User>();
  private seq = 0;

  constructor(seed: readonly User[] = []) {
    for (const user of seed) this.rows.set(user.userId, user);
  }

  async findById(userId: UserId): Promise<User | null> {
    return this.rows.get(userId) ?? null;                    // normalise undefined → null
  }

  async findByEmail(email: string): Promise<User | null> {
    const needle = email.toLowerCase();
    return [...this.rows.values()].find((u) => u.email === needle) ?? null;
  }

  async create(input: CreateUserInput): Promise<User> {
    const user: User = {
      ...input,
      userId:    parseUserId(`usr_${++this.seq}`),
      createdAt: new Date(),
    };
    this.rows.set(user.userId, user);
    return user;
  }
}

export class ConsoleMailer implements Mailer {
  constructor(private readonly logger: Logger) {}           // implementations get deps too
  async sendWelcome(to: string, displayName: string): Promise<void> {
    this.logger.info("mail.welcome", { to, displayName });  // dev mode: print, don't send
  }
}

export class RecordingMailer implements Mailer {
  readonly sent: { to: string; displayName: string }[] = [];
  async sendWelcome(to: string, displayName: string): Promise<void> {
    this.sent.push({ to, displayName });
  }
}

export class FakeClock implements Clock {
  constructor(private current: Date = new Date("2026-07-22T00:00:00Z")) {}
  now(): Date { return new Date(this.current); }
  advance(ms: number): void { this.current = new Date(this.current.getTime() + ms); }
}

// ════════════════════════════════════════════════════════════════════════════
// 5. Wiring
// ════════════════════════════════════════════════════════════════════════════

// ── Production ──────────────────────────────────────────────────────────────
const productionLogger: Logger = {
  info:  (message, meta) => console.log(JSON.stringify({ level: "info", message, ...meta })),
  error: (message, meta) => console.error(JSON.stringify({ level: "error", message, ...meta })),
};

export const userService = new UserService(
  new InMemoryUserRepository(),                  // swap for MongoUserRepository in real life
  new ConsoleMailer(productionLogger),           // swap for SendGridMailer
  { now: () => new Date() },                     // object literal is a valid Clock
  productionLogger,
);

// ── Test ────────────────────────────────────────────────────────────────────
const silentLogger: Logger = { info() {}, error() {} };
const recordingMailer = new RecordingMailer();

export const testUserService = new UserService(
  new InMemoryUserRepository(),
  recordingMailer,
  new FakeClock(),
  silentLogger,
);

// const result = await testUserService.register({
//   email: "ADA@Example.com", displayName: "Ada Lovelace", status: "active",
// });
// result.ok === true
// recordingMailer.sent.length === 1
// recordingMailer.sent[0].to === "ada@example.com"

// ── What no longer compiles ─────────────────────────────────────────────────
// new UserService(new InMemoryUserRepository(), recordingMailer, new FakeClock());
//   ❌ Expected 4 arguments, but got 3.
// new UserService(recordingMailer, new InMemoryUserRepository(), clock, logger);
//   ❌ Argument of type 'RecordingMailer' is not assignable to 'UserRepository'.
// const half: Mailer = { };
//   ❌ Property 'sendWelcome' is missing in type '{}'.
```

---

## Example 2 — real world backend use case

```ts
// A full Express application wired by hand: config, contracts, adapters, services,
// a singleton composition root, a per-request container, typed routes, and tests.

import express, { type Request, type Response, type NextFunction } from "express";
import { randomUUID } from "node:crypto";

// ════════════════════════════════════════════════════════════════════════════
// 1. Domain
// ════════════════════════════════════════════════════════════════════════════

declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type UserId    = Brand<string, "UserId">;
export type OrgId     = Brand<string, "OrgId">;
export type ProjectId = Brand<string, "ProjectId">;

export const parseUserId    = (raw: string): UserId    => raw as UserId;
export const parseOrgId     = (raw: string): OrgId     => raw as OrgId;
export const parseProjectId = (raw: string): ProjectId => raw as ProjectId;

export interface User {
  readonly userId:       UserId;
  readonly orgId:        OrgId;
  readonly email:        string;
  readonly displayName:  string;
  readonly passwordHash: string;
  readonly role:         "member" | "admin" | "owner";
  readonly createdAt:    Date;
}

export interface Project {
  readonly projectId:  ProjectId;
  readonly orgId:      OrgId;
  readonly name:       string;
  readonly ownerId:    UserId;
  readonly archivedAt: Date | null;
  readonly createdAt:  Date;
}

export type ApiResponse<T> =
  | { readonly ok: true;  readonly data: T }
  | { readonly ok: false; readonly error: string; readonly detail?: string };

// ════════════════════════════════════════════════════════════════════════════
// 2. Contracts — the complete list of things this app needs from the outside
// ════════════════════════════════════════════════════════════════════════════

export interface Clock { now(): Date; }

export interface Logger {
  info(message: string, meta?: Record<string, unknown>): void;
  warn(message: string, meta?: Record<string, unknown>): void;
  error(message: string, meta?: Record<string, unknown>): void;
  child(bindings: Record<string, unknown>): Logger;      // per-request child loggers
}

export interface UserRepository {
  findById(userId: UserId): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(input: Omit<User, "userId" | "createdAt">): Promise<User>;
}

export interface ProjectRepository {
  findById(projectId: ProjectId): Promise<Project | null>;
  findMany(filter: { orgId: OrgId; archived?: boolean }): Promise<Project[]>;
  create(input: Omit<Project, "projectId" | "createdAt" | "archivedAt">): Promise<Project>;
  archive(projectId: ProjectId, at: Date): Promise<Project | null>;
}

export interface PasswordHasher {
  hash(plain: string): Promise<string>;
  verify(plain: string, hashed: string): Promise<boolean>;
}

export interface TokenIssuer {
  issue(claims: { userId: UserId; orgId: OrgId; role: User["role"] }): Promise<string>;
  verify(authToken: string): Promise<{ userId: UserId; orgId: OrgId; role: User["role"] } | null>;
}

export interface Mailer {
  sendWelcome(to: string, displayName: string): Promise<void>;
  sendProjectInvite(to: string, projectName: string, inviteUrl: string): Promise<void>;
}

export interface Metrics {
  increment(name: string, tags?: Record<string, string>): void;
  timing(name: string, ms: number, tags?: Record<string, string>): void;
}

// ════════════════════════════════════════════════════════════════════════════
// 3. Config — parsed once, injected everywhere it is needed
// ════════════════════════════════════════════════════════════════════════════

export interface AppConfig {
  readonly nodeEnv:      "development" | "test" | "production";
  readonly port:         number;
  readonly jwtSecret:    string;
  readonly bcryptRounds: number;
  readonly sendgridKey:  string;
  readonly appBaseUrl:   string;
}

export function loadConfig(env: NodeJS.ProcessEnv): AppConfig {
  const required = (key: string): string => {
    const value = env[key];
    if (value === undefined || value === "") throw new Error(`Missing env var: ${key}`);
    return value;
  };
  const nodeEnv = (env.NODE_ENV ?? "development") as AppConfig["nodeEnv"];
  return {
    nodeEnv,
    port:         Number(env.PORT ?? 3000),
    jwtSecret:    required("JWT_SECRET"),
    bcryptRounds: Number(env.BCRYPT_ROUNDS ?? 12),
    sendgridKey:  nodeEnv === "production" ? required("SENDGRID_API_KEY") : "dev-key",
    appBaseUrl:   env.APP_BASE_URL ?? "http://localhost:3000",
  };
}

// ════════════════════════════════════════════════════════════════════════════
// 4. Services — pure application logic, zero infrastructure imports
// ════════════════════════════════════════════════════════════════════════════

export class AuthService {
  constructor(
    private readonly users:  UserRepository,
    private readonly hasher: PasswordHasher,
    private readonly tokens: TokenIssuer,
    private readonly mailer: Mailer,
    private readonly clock:  Clock,
    private readonly logger: Logger,
  ) {}

  async register(requestBody: {
    email: string; password: string; displayName: string; orgId: OrgId;
  }): Promise<ApiResponse<{ userId: UserId; authToken: string }>> {
    const email = requestBody.email.trim().toLowerCase();

    if (await this.users.findByEmail(email)) {
      return { ok: false, error: "EMAIL_ALREADY_REGISTERED" };
    }
    if (requestBody.password.length < 12) {
      return { ok: false, error: "WEAK_PASSWORD", detail: "minimum 12 characters" };
    }

    const passwordHash = await this.hasher.hash(requestBody.password);
    const user = await this.users.create({
      orgId:       requestBody.orgId,
      email,
      displayName: requestBody.displayName,
      passwordHash,
      role:        "member",
    });

    const authToken = await this.tokens.issue({
      userId: user.userId, orgId: user.orgId, role: user.role,
    });

    // Best-effort side effect — never fail a registration because SMTP is down.
    void this.mailer.sendWelcome(user.email, user.displayName)
      .catch((cause: unknown) => this.logger.warn("welcome_email_failed", {
        userId: user.userId, cause: String(cause),
      }));

    this.logger.info("auth.registered", {
      userId: user.userId, at: this.clock.now().toISOString(),
    });
    return { ok: true, data: { userId: user.userId, authToken } };
  }

  async login(email: string, password: string): Promise<ApiResponse<{ authToken: string }>> {
    const user = await this.users.findByEmail(email.trim().toLowerCase());

    // Always run a verify, even on a miss, so timing doesn't leak account existence.
    const hashToCheck = user?.passwordHash ?? "$2b$12$invalidinvalidinvalidinvalidinvalidinva";
    const valid = await this.hasher.verify(password, hashToCheck);

    if (!user || !valid) {
      this.logger.info("auth.login_failed", { email });
      return { ok: false, error: "INVALID_CREDENTIALS" };
    }

    const authToken = await this.tokens.issue({
      userId: user.userId, orgId: user.orgId, role: user.role,
    });
    return { ok: true, data: { authToken } };
  }
}

export class ProjectService {
  constructor(
    private readonly projects: ProjectRepository,   // already tenant-scoped when per-request
    private readonly users:    UserRepository,
    private readonly mailer:   Mailer,
    private readonly clock:    Clock,
    private readonly logger:   Logger,
    private readonly config:   AppConfig,
  ) {}

  async list(orgId: OrgId): Promise<Project[]> {
    return this.projects.findMany({ orgId, archived: false });
  }

  async create(orgId: OrgId, ownerId: UserId, name: string): Promise<ApiResponse<Project>> {
    const trimmed = name.trim();
    if (trimmed.length < 3) return { ok: false, error: "NAME_TOO_SHORT" };

    const project = await this.projects.create({ orgId, name: trimmed, ownerId });
    this.logger.info("project.created", { projectId: project.projectId, orgId });
    return { ok: true, data: project };
  }

  async invite(projectId: ProjectId, inviteeUserId: UserId): Promise<ApiResponse<null>> {
    const [project, invitee] = await Promise.all([
      this.projects.findById(projectId),
      this.users.findById(inviteeUserId),
    ]);
    if (!project) return { ok: false, error: "PROJECT_NOT_FOUND" };
    if (!invitee) return { ok: false, error: "USER_NOT_FOUND" };

    const inviteUrl = `${this.config.appBaseUrl}/projects/${project.projectId}/join`;
    await this.mailer.sendProjectInvite(invitee.email, project.name, inviteUrl);
    return { ok: true, data: null };
  }

  async archive(projectId: ProjectId): Promise<ApiResponse<Project>> {
    const archived = await this.projects.archive(projectId, this.clock.now());
    if (!archived) return { ok: false, error: "PROJECT_NOT_FOUND" };
    return { ok: true, data: archived };
  }
}

// ════════════════════════════════════════════════════════════════════════════
// 5. A decorator: tenant scoping, implemented as a repository wrapper
// ════════════════════════════════════════════════════════════════════════════

// Injected per request. It is structurally impossible for a caller holding this
// object to read another org's rows — the orgId is baked in, not passed in.
export class TenantScopedProjectRepository implements ProjectRepository {
  constructor(
    private readonly inner: ProjectRepository,
    private readonly orgId: OrgId,
  ) {}

  async findById(projectId: ProjectId): Promise<Project | null> {
    const project = await this.inner.findById(projectId);
    return project && project.orgId === this.orgId ? project : null;   // cross-tenant → null
  }

  async findMany(filter: { orgId: OrgId; archived?: boolean }): Promise<Project[]> {
    return this.inner.findMany({ ...filter, orgId: this.orgId });      // orgId always overridden
  }

  async create(input: Omit<Project, "projectId" | "createdAt" | "archivedAt">): Promise<Project> {
    return this.inner.create({ ...input, orgId: this.orgId });
  }

  async archive(projectId: ProjectId, at: Date): Promise<Project | null> {
    if (!(await this.findById(projectId))) return null;                // scope check first
    return this.inner.archive(projectId, at);
  }
}

// ════════════════════════════════════════════════════════════════════════════
// 6. The singleton composition root
// ════════════════════════════════════════════════════════════════════════════

// Split deliberately: infrastructure is faked in tests, services are not.
export interface Infrastructure {
  readonly config:   AppConfig;
  readonly clock:    Clock;
  readonly logger:   Logger;
  readonly metrics:  Metrics;
  readonly users:    UserRepository;
  readonly projects: ProjectRepository;
  readonly hasher:   PasswordHasher;
  readonly tokens:   TokenIssuer;
  readonly mailer:   Mailer;
}

export interface AppContainer extends Infrastructure {
  readonly authService: AuthService;
}

export function buildInfrastructure(config: AppConfig): Infrastructure {
  const logger  = createJsonLogger(config.nodeEnv === "production" ? "info" : "debug");
  const clock: Clock = { now: () => new Date() };
  const metrics = createStatsdMetrics(logger);

  return {
    config, clock, logger, metrics,
    users:    new MongoUserRepository(),
    projects: new MongoProjectRepository(),
    hasher:   new BcryptHasher(config.bcryptRounds),
    tokens:   new JwtTokenIssuer(config.jwtSecret, clock),
    mailer:   config.nodeEnv === "production"
                ? new SendGridMailer(config.sendgridKey, logger)
                : new ConsoleMailer(logger),
  };
}

export function buildContainer(infra: Infrastructure): AppContainer {
  // AuthService is stateless and org-agnostic → safe as a singleton.
  const authService = new AuthService(
    infra.users, infra.hasher, infra.tokens, infra.mailer, infra.clock, infra.logger,
  );
  return { ...infra, authService };
}

// ════════════════════════════════════════════════════════════════════════════
// 7. The per-request container
// ════════════════════════════════════════════════════════════════════════════

export interface RequestContext {
  readonly requestId: string;
  readonly startedAt: Date;
  readonly userId:    UserId | null;
  readonly orgId:     OrgId | null;
  readonly role:      User["role"] | null;
}

export interface RequestContainer {
  readonly ctx:            RequestContext;
  readonly logger:         Logger;               // bound to requestId
  readonly authService:    AuthService;          // reused singleton
  readonly projectService: ProjectService;       // rebuilt, tenant-scoped
}

export function buildRequestContainer(app: AppContainer, ctx: RequestContext): RequestContainer {
  const logger = app.logger.child({ requestId: ctx.requestId, userId: ctx.userId ?? "anon" });

  // Scoped repository only when we know the tenant; unauthenticated routes get the raw one
  // and must not touch project data (guarded by requireAuth below).
  const projects: ProjectRepository = ctx.orgId
    ? new TenantScopedProjectRepository(app.projects, ctx.orgId)
    : app.projects;

  return {
    ctx,
    logger,
    authService: app.authService,
    projectService: new ProjectService(
      projects, app.users, app.mailer, app.clock, logger, app.config,
    ),
  };
}

// ════════════════════════════════════════════════════════════════════════════
// 8. Express wiring — no service is ever imported by a route file
// ════════════════════════════════════════════════════════════════════════════

// See doc 53 for the module-augmentation details of this declaration.
declare global {
  // eslint-disable-next-line @typescript-eslint/no-namespace
  namespace Express {
    interface Request { container: RequestContainer; }
  }
}

export function createApp(app: AppContainer): express.Express {
  const server = express();
  server.use(express.json());

  // ── Middleware: build the per-request container ───────────────────────────
  server.use(async (req: Request, _res: Response, next: NextFunction) => {
    const header    = req.headers.authorization;
    const authToken = header?.startsWith("Bearer ") ? header.slice(7) : null;
    const claims    = authToken ? await app.tokens.verify(authToken) : null;

    const ctx: RequestContext = {
      requestId: randomUUID(),
      startedAt: app.clock.now(),
      userId:    claims?.userId ?? null,
      orgId:     claims?.orgId ?? null,
      role:      claims?.role ?? null,
    };
    req.container = buildRequestContainer(app, ctx);
    next();
  });

  // ── Middleware: latency metrics, using the injected Metrics ───────────────
  server.use((req: Request, res: Response, next: NextFunction) => {
    res.on("finish", () => {
      const ms = app.clock.now().getTime() - req.container.ctx.startedAt.getTime();
      app.metrics.timing("http.request", ms, {
        route:  req.route?.path ?? "unknown",
        status: String(res.statusCode),
      });
    });
    next();
  });

  // ── Guard: narrows ctx.orgId from `OrgId | null` to `OrgId` for handlers ──
  const requireAuth = (req: Request, res: Response, next: NextFunction): void => {
    if (!req.container.ctx.orgId || !req.container.ctx.userId) {
      res.status(401).json({ ok: false, error: "UNAUTHENTICATED" } satisfies ApiResponse<never>);
      return;
    }
    next();
  };

  // ── Routes: thin. Parse input, call a service, shape a response. ──────────
  server.post("/auth/register", async (req: Request, res: Response) => {
    const result = await req.container.authService.register({
      email:       String(req.body.email),
      password:    String(req.body.password),
      displayName: String(req.body.displayName),
      orgId:       parseOrgId(String(req.body.orgId)),
    });
    res.status(result.ok ? 201 : 400).json(result);
  });

  server.post("/auth/login", async (req: Request, res: Response) => {
    const result = await req.container.authService.login(
      String(req.body.email), String(req.body.password),
    );
    res.status(result.ok ? 200 : 401).json(result);
  });

  server.get("/projects", requireAuth, async (req: Request, res: Response) => {
    const orgId = req.container.ctx.orgId!;             // guaranteed by requireAuth
    const projects = await req.container.projectService.list(orgId);
    res.json({ ok: true, data: projects } satisfies ApiResponse<Project[]>);
  });

  server.post("/projects", requireAuth, async (req: Request, res: Response) => {
    const { orgId, userId } = req.container.ctx;
    const result = await req.container.projectService.create(
      orgId!, userId!, String(req.body.name),
    );
    res.status(result.ok ? 201 : 400).json(result);
  });

  server.post("/projects/:projectId/archive", requireAuth, async (req: Request, res: Response) => {
    // Tenant scoping is automatic — the repository this service holds is org-bound.
    const result = await req.container.projectService.archive(
      parseProjectId(req.params.projectId),
    );
    res.status(result.ok ? 200 : 404).json(result);
  });

  return server;
}

// ════════════════════════════════════════════════════════════════════════════
// 9. Entry point — the ONLY place process.env, connections, and listen() appear
// ════════════════════════════════════════════════════════════════════════════

async function main(): Promise<void> {
  const config = loadConfig(process.env);
  const infra  = buildInfrastructure(config);
  const app    = buildContainer(infra);

  const server = createApp(app).listen(config.port, () =>
    app.logger.info("server.listening", { port: config.port, env: config.nodeEnv }),
  );

  // Graceful shutdown — the container makes it obvious what needs closing.
  const shutdown = async (signal: string): Promise<void> => {
    app.logger.info("server.shutdown", { signal });
    server.close();
    process.exit(0);
  };
  process.on("SIGTERM", () => void shutdown("SIGTERM"));
  process.on("SIGINT",  () => void shutdown("SIGINT"));
}

// void main();

// ════════════════════════════════════════════════════════════════════════════
// 10. Tests — the whole app, in-process, in milliseconds
// ════════════════════════════════════════════════════════════════════════════

export function buildTestInfrastructure(overrides: Partial<Infrastructure> = {}): Infrastructure {
  const clock = new FakeClock();
  const silentLogger: Logger = {
    info() {}, warn() {}, error() {}, child() { return silentLogger; },
  };
  const base: Infrastructure = {
    config: {
      nodeEnv: "test", port: 0, jwtSecret: "test-secret",
      bcryptRounds: 4, sendgridKey: "test", appBaseUrl: "http://test.local",
    },
    clock,
    logger:  silentLogger,
    metrics: { increment() {}, timing() {} },
    users:    new InMemoryUserRepository(),
    projects: new InMemoryProjectRepository(),
    // A fake hasher: 4000× faster than bcrypt and perfectly adequate for logic tests.
    hasher: {
      async hash(plain) { return `fake:${plain}`; },
      async verify(plain, hashed) { return hashed === `fake:${plain}`; },
    },
    tokens: new FakeTokenIssuer(),
    mailer: new RecordingMailer(),
  };
  return { ...base, ...overrides };
}

// import assert from "node:assert/strict";
// import { test } from "node:test";
//
// test("registering sends exactly one welcome email and returns a token", async () => {
//   const mailer = new RecordingMailer();
//   const infra  = buildTestInfrastructure({ mailer });
//   const app    = buildContainer(infra);
//
//   const result = await app.authService.register({
//     email: "ada@example.com", password: "correct-horse-battery",
//     displayName: "Ada Lovelace", orgId: parseOrgId("org_1"),
//   });
//
//   assert.equal(result.ok, true);
//   assert.equal(mailer.sent.length, 1);
//   assert.equal(mailer.sent[0]?.to, "ada@example.com");
// });
//
// test("a request scoped to org_1 cannot archive a project owned by org_2", async () => {
//   const infra = buildTestInfrastructure();
//   const app   = buildContainer(infra);
//   const foreign = await infra.projects.create({
//     orgId: parseOrgId("org_2"), name: "Secret", ownerId: parseUserId("usr_9"),
//   });
//
//   const req = buildRequestContainer(app, {
//     requestId: "req_1", startedAt: infra.clock.now(),
//     userId: parseUserId("usr_1"), orgId: parseOrgId("org_1"), role: "admin",
//   });
//
//   const result = await req.projectService.archive(foreign.projectId);
//   assert.deepEqual(result, { ok: false, error: "PROJECT_NOT_FOUND" });   // scoped away
// });
//
// test("a slow mailer does not fail registration", async () => {
//   const infra = buildTestInfrastructure({ mailer: new FlakyMailer(3) });
//   const app   = buildContainer(infra);
//   const result = await app.authService.register({ /* ... */ } as never);
//   assert.equal(result.ok, true);       // best-effort email, logged not thrown
// });
```

---

## Going deeper

### `satisfies` for wiring: keep the check, keep the inference

`const clock: Clock = {...}` widens the value to `Clock`, losing anything extra. `satisfies` checks the constraint but preserves the literal type — useful for containers where you want both.

```ts
// With annotation: `container.clock` is exactly `Clock` — extra members are erased.
const container: AppContainer = { clock: new FakeClock(), /* ... */ };
// container.clock.advance(1000);   // ❌ 'advance' does not exist on type 'Clock'

// With `satisfies`: checked against AppContainer, but the concrete types survive.
const testRig = {
  clock:  new FakeClock(),
  mailer: new RecordingMailer(),
  users:  new InMemoryUserRepository(),
  /* ...the rest... */
} satisfies Partial<Infrastructure>;

testRig.clock.advance(60_000);          // ✅ FakeClock is preserved
testRig.mailer.sent.length;             // ✅ RecordingMailer is preserved
```

This is the cleanest way to write test wiring: one object, constraint-checked, with full access to fake-only helpers like `advance()` and `sent`.

### Typing the dependency graph so misordering is impossible

A plain `buildContainer` lets you reference a `const` before it exists (a TDZ error at runtime, not a type error). Structuring the root in explicit levels makes ordering visible:

```ts
// Each level's function can only receive the previous levels' output.
interface Level0 { readonly config: AppConfig; readonly clock: Clock; readonly logger: Logger }
interface Level1 extends Level0 {
  readonly users: UserRepository; readonly tokens: TokenIssuer; readonly mailer: Mailer;
}
interface Level2 extends Level1 { readonly authService: AuthService }

const buildLevel0 = (config: AppConfig): Level0 => ({
  config, clock: { now: () => new Date() }, logger: createJsonLogger("info"),
});
const buildLevel1 = (l0: Level0): Level1 => ({
  ...l0,
  users:  new MongoUserRepository(),
  tokens: new JwtTokenIssuer(l0.config.jwtSecret, l0.clock),
  mailer: new SendGridMailer(l0.config.sendgridKey, l0.logger),
});
const buildLevel2 = (l1: Level1): Level2 => ({
  ...l1,
  authService: new AuthService(l1.users, hasher, l1.tokens, l1.mailer, l1.clock, l1.logger),
});

export const buildContainer = (config: AppConfig): Level2 =>
  buildLevel2(buildLevel1(buildLevel0(config)));
```

Overkill for ten dependencies; genuinely helpful at fifty, and it makes cycles structurally impossible — a level can only depend downward.

### Async dependencies

Some dependencies cannot be constructed synchronously (a connected DB pool, a fetched secret, a warmed cache). Do **not** hide the async inside a lazy getter — make it explicit.

```ts
// ❌ Lazy async: every call site now deals with a promise, and errors surface late.
class Repo {
  private db?: Db;
  private async getDb(): Promise<Db> { return (this.db ??= await connect()); }
}

// ✅ Async composition root: everything is connected before the server accepts traffic.
export async function buildInfrastructure(config: AppConfig): Promise<Infrastructure> {
  const [mongo, redis, secrets] = await Promise.all([
    mongoose.connect(config.mongoUri),
    createRedisClient(config.redisUrl),
    fetchSecrets(config.secretsArn),
  ]);
  return {
    config,
    clock:  { now: () => new Date() },
    logger: createJsonLogger("info"),
    users:  new MongoUserRepository(mongo.connection),
    cache:  new RedisCache(redis),
    tokens: new JwtTokenIssuer(secrets.jwtSecret, { now: () => new Date() }),
    /* ... */
  } as Infrastructure;
}
// Services stay synchronous: `new AuthService(infra.users, ...)`. Fail fast at boot.
```

### Disposal: what a manual root gives you that a container hides

```ts
export interface Disposable { close(): Promise<void>; }

export interface Infrastructure {
  /* ...deps... */
  readonly disposables: readonly Disposable[];    // recorded during construction
}

// In buildInfrastructure:
// disposables: [redisClient, { close: () => mongoose.disconnect() }, metricsFlusher]

export async function shutdown(infra: Infrastructure): Promise<void> {
  // Reverse order: last constructed, first closed.
  for (const resource of [...infra.disposables].reverse()) {
    await resource.close().catch(() => { /* keep closing the rest */ });
  }
}
```

Twelve lines. Every DI container has an equivalent feature with subtler semantics and worse failure modes.

### Testing the composition root itself

The one thing DI can't test is the wiring. So test that too:

```ts
test("the production container builds without throwing", () => {
  const config = loadConfig({
    NODE_ENV: "production", JWT_SECRET: "x".repeat(32),
    SENDGRID_API_KEY: "SG.test", MONGO_URI: "mongodb://localhost/test",
    REDIS_URL: "redis://localhost", PORT: "3000",
  });
  const infra = buildInfrastructure(config);
  const app   = buildContainer(infra);
  assert.ok(app.authService);
});

test("loadConfig rejects a missing JWT_SECRET", () => {
  assert.throws(() => loadConfig({ NODE_ENV: "production" }), /JWT_SECRET/);
});
```

With a decorator container, the analogous test is `container.resolve(AuthService)` — and it will only catch registrations reachable from that one root.

### Over-injection: the smell that DI makes visible

```ts
// ❌ Eleven dependencies. This is not a DI problem, it's a class-design problem.
class OrderService {
  constructor(
    private readonly orders: OrderRepository,
    private readonly users: UserRepository,
    private readonly products: ProductRepository,
    private readonly inventory: InventoryService,
    private readonly payments: PaymentGateway,
    private readonly shipping: ShippingProvider,
    private readonly tax: TaxCalculator,
    private readonly mailer: Mailer,
    private readonly metrics: Metrics,
    private readonly clock: Clock,
    private readonly logger: Logger,
  ) {}
}

// ✅ Split by use case, each with the two or three things it actually needs.
class PlaceOrderUseCase {
  constructor(
    private readonly orders: OrderRepository,
    private readonly pricing: PricingService,     // wraps products + tax
    private readonly clock: Clock,
  ) {}
}
class FulfilOrderUseCase {
  constructor(
    private readonly orders: OrderRepository,
    private readonly shipping: ShippingProvider,
    private readonly mailer: Mailer,
  ) {}
}
```

Six of one class's dependencies being unused by half its methods is the signal. Injection is what makes it countable.

### Don't inject what isn't a seam

```ts
// ❌ Injecting pure functions. They have no I/O, no state, no alternative implementation.
class PriceService {
  constructor(private readonly round: (n: number) => number) {}   // pointless indirection
}

// ✅ Just import it.
import { roundToCents } from "../lib/money";

// ❌ Injecting the config VALUES individually.
class TokenIssuer { constructor(private readonly secret: string, private readonly ttl: number) {} }
// Fine for two; at six it becomes positional soup. Inject a typed slice instead:
interface TokenConfig { readonly secret: string; readonly ttlSeconds: number; readonly issuer: string }
class TokenIssuerB { constructor(private readonly config: TokenConfig, private readonly clock: Clock) {} }
```

The rule of thumb: inject anything that touches **the network, the disk, the clock, randomness, or the process environment**. Import everything else directly. Those five categories are exactly what makes tests slow, flaky, or impossible.

### `abstract class` as a token *and* a contract

If you want one declaration that is both a compile-time contract and a runtime value (usable as a DI key), an `abstract class` is the trick — it emits a class at runtime but can be used as a type.

```ts
export abstract class Mailer {
  abstract sendWelcome(to: string, displayName: string): Promise<void>;
}

// It's a type:
class UserService { constructor(private readonly mailer: Mailer) {} }
// AND a runtime value usable as a key:
container.register(Mailer, () => new SendGridMailer(config.sendgridKey));
// AND still structurally satisfiable — no `extends` needed on the implementation:
const fake: Mailer = { async sendWelcome() {} };     // ✅ TypeScript is structural
```

This is what Angular does. The cost is an emitted class you never instantiate, and losing the "interfaces are free" property. Worth it only if you're already using a runtime container.

### The `Partial<Infrastructure>` override helper, properly typed

```ts
// A builder that requires you to be explicit about what you're faking.
export function buildTestInfrastructure<O extends Partial<Infrastructure>>(
  overrides: O = {} as O,
): Infrastructure & O {
  const base = defaultTestInfrastructure();
  return { ...base, ...overrides };
}

// The intersection preserves the concrete override types:
const infra = buildTestInfrastructure({ mailer: new RecordingMailer(), clock: new FakeClock() });
infra.mailer.sent;        // ✅ RecordingMailer["sent"] — not erased to Mailer
infra.clock.advance(500); // ✅ FakeClock["advance"]
infra.users.findById;     // ✅ still the default InMemoryUserRepository
```

That generic + intersection trick is the single most useful piece of typing in a DI-heavy test suite.

---

## Common mistakes

### Mistake 1 — Injecting the concrete class instead of the contract

```ts
// ❌ Injected, but nothing is decoupled. Every consumer transitively imports mongoose,
//    tests still need a database, and the "abstraction" is a lie.
import { MongoUserRepository } from "../infra/mongo-user.repository";

class UserService {
  constructor(private readonly users: MongoUserRepository) {}
}
// new UserService(new InMemoryUserRepository());
//   ❌ Type 'InMemoryUserRepository' is missing properties from 'MongoUserRepository'
//      (the private fields make it nominally incompatible, even if the methods match)

// ✅ Depend on the interface. Nothing infrastructural is imported.
import type { UserRepository } from "../domain/contracts";

class UserService {
  constructor(private readonly users: UserRepository) {}
}
// new UserService(new InMemoryUserRepository());   // ✅
// new UserService(new MongoUserRepository());      // ✅
```

Note the specific reason the first version fails: **classes with `private` members are compared nominally**. `MongoUserRepository`'s private fields are unique to it, so no other type is ever assignable — the very thing that makes `private` useful makes concrete-class injection a dead end.

### Mistake 2 — A singleton service holding per-request state

```ts
// ❌ A cross-request data leak, served with a straight face.
class TenantService {
  private orgId: OrgId | null = null;            // ONE field, ALL concurrent requests
  setOrg(orgId: OrgId): void { this.orgId = orgId; }
  async listProjects(): Promise<Project[]> {
    return this.projects.findMany({ orgId: this.orgId! });
  }
}
const tenantService = new TenantService(projectRepo);   // process-wide singleton

// Request A: setOrg("org_alice"); await someIo();
// Request B: setOrg("org_bob");
// Request A resumes → listProjects() returns Bob's projects. Shipped incident.

// ✅ Option A — the tenant is a parameter. Stateless, obviously correct.
class TenantService {
  constructor(private readonly projects: ProjectRepository) {}
  async listProjects(orgId: OrgId): Promise<Project[]> {
    return this.projects.findMany({ orgId });
  }
}

// ✅ Option B — a per-request instance whose tenant is fixed at construction.
class ScopedTenantService {
  constructor(
    private readonly projects: ProjectRepository,
    private readonly orgId: OrgId,               // readonly, set once, per request
  ) {}
  async listProjects(): Promise<Project[]> {
    return this.projects.findMany({ orgId: this.orgId });
  }
}
// const service = new ScopedTenantService(app.projects, ctx.orgId);  // in middleware
```

`readonly` on the field is doing real work in Option B: there is no `setOrg`, so there is no window in which the instance means something different.

### Mistake 3 — Service locator instead of injection

```ts
// ❌ The service reaches INTO a global container. This is not DI; it's a global variable
//    with extra steps.
import { container } from "./container";

class OrderService {
  async placeOrder(userId: UserId): Promise<Order> {
    const orders = container.resolve<OrderRepository>("OrderRepository");  // hidden dep
    const mailer = container.resolve<Mailer>("Mailer");                    // hidden dep
    /* ... */
    throw new Error();
  }
}
// Problems: the constructor lies about what this class needs; a missing registration is a
// runtime error, not a compile error; tests must populate a global container and reset it
// between cases; and `resolve<T>("string")` is an unchecked cast — the string and the type
// parameter can disagree freely.

// ✅ Real injection: dependencies are in the signature, checked, and visible.
class OrderService {
  constructor(
    private readonly orders: OrderRepository,
    private readonly mailer: Mailer,
  ) {}
}
```

The tell: if a class imports the container, it is a service locator. The container should be imported by exactly one file — the entry point.

### Mistake 4 — Wiring inside route handlers instead of at the root

```ts
// ❌ A new repository, mailer, and service per request — plus a new connection pool
//    per request if any of them opens one. Also: the route file now imports Mongo.
app.post("/users", async (req, res) => {
  const service = new UserService(
    new MongoUserRepository(UserModel),
    new SendGridMailer(process.env.SENDGRID_API_KEY!),
    { now: () => new Date() },
    logger,
  );
  const result = await service.register(req.body);
  res.json(result);
});

// ✅ Wire once; hand the route what it needs.
export function registerUserRoutes(app: express.Express, container: AppContainer): void {
  app.post("/users", async (req, res) => {
    const result = await container.authService.register(req.body);
    res.json(result);
  });
}
// Even better — the handler asks for the narrowest thing it uses:
export const makeRegisterHandler =
  (deps: { authService: AuthService }) =>
  async (req: Request, res: Response): Promise<void> => {
    res.json(await deps.authService.register(req.body));
  };
```

### Mistake 5 — Optional dependencies with `?`

```ts
// ❌ Now every use needs a guard, and "was it wired?" is a runtime question.
class NotificationService {
  constructor(
    private readonly mailer?: Mailer,
    private readonly sms?: SmsSender,
  ) {}
  async notify(user: User, message: string): Promise<void> {
    await this.mailer?.sendWelcome(user.email, message);   // silently does nothing if unwired
    await this.sms?.send(user.phone, message);             // and you'll never know
  }
}
// new NotificationService();   // ✅ compiles. Notifies nobody. Forever.

// ✅ Make the dependency required and inject an explicit null implementation.
class NoopMailer implements Mailer {
  async sendWelcome(): Promise<void> {}
  async sendProjectInvite(): Promise<void> {}
}

class NotificationService {
  constructor(
    private readonly mailer: Mailer,           // always present
    private readonly sms: SmsSender,
  ) {}
  async notify(user: User, message: string): Promise<void> {
    await this.mailer.sendWelcome(user.email, message);    // no guard needed
    await this.sms.send(user.phone, message);
  }
}
// The decision "we don't send email in dev" is made ONCE, in the composition root:
// mailer: config.nodeEnv === "production" ? new SendGridMailer(key, logger) : new NoopMailer(),
```

The Null Object pattern converts an ongoing runtime question into a one-line wiring decision.

### Mistake 6 — Circular constructor dependencies "solved" with `any` or `!`

```ts
// ❌ Both of these compile and both defer an inevitable crash.
class OrderService {
  private invoices!: InvoiceService;          // definite assignment assertion = a promise you made up
  constructor(private readonly orders: OrderRepository) {}
}
class InvoiceServiceBad {
  constructor(private readonly orders: any) {}   // `any` erases the cycle AND the type safety
}

// ✅ Break the cycle by narrowing the contract (usually one side needs far less).
interface OrderLookup { findById(orderId: OrderId): Promise<Order | null>; }

class InvoiceService {
  constructor(private readonly orders: OrderLookup) {}   // needs the REPO, not the service
}
class OrderService {
  constructor(
    private readonly orders: OrderRepository,
    private readonly invoices: InvoiceService,
  ) {}
}
// Wiring is now a straight line: repo → InvoiceService → OrderService.
```

### Mistake 7 — Constructing dependencies at module scope

```ts
// ❌ Runs at import time, before config is loaded, in every test file, and in any tool
//    that merely imports this module (a migration script, a type-check, a bundler).
export const mailer = new SendGridMailer(process.env.SENDGRID_API_KEY!);
export const pool   = new Pg.Pool({ connectionString: process.env.DATABASE_URL! });
// Import this from a unit test → the test opens a real Postgres connection.
// `!` on a missing env var → a live client with `undefined` credentials, failing later.

// ✅ Export a factory. Nothing happens until the composition root calls it.
export function createMailer(config: AppConfig, logger: Logger): Mailer {
  return new SendGridMailer(config.sendgridKey, logger);
}
export function createPool(config: AppConfig): Pg.Pool {
  return new Pg.Pool({ connectionString: config.databaseUrl, max: config.poolSize });
}
```

---

## Practice exercises

### Exercise 1 — easy

Refactor a hard-wired service into an injected one and write a test for it, with no mocking library.

Requirements:

- Define contracts: `SessionRepository` (`findByToken(authToken: string): Promise<Session | null>`, `create(input): Promise<Session>`, `revoke(sessionId: SessionId): Promise<boolean>`), `Clock` (`now(): Date`), and `TokenGenerator` (`generate(): string`).
- Define `Session = { sessionId, userId, authToken, createdAt, expiresAt, revokedAt: Date | null }` with branded `SessionId` and `UserId`, all fields `readonly`.
- Write `class SessionService` using **parameter properties** for all three dependencies (`private readonly`) plus a `ttlMinutes: number` with a default of `30`.
- It needs three methods: `issue(userId): Promise<Session>` (expiry = `now + ttl`), `validate(authToken): Promise<Session | null>` (returns `null` if revoked or expired), and `logout(authToken): Promise<boolean>`.
- Write `InMemorySessionRepository`, `FakeClock` (with an `advance(ms)` method), and a `SequentialTokenGenerator` that returns `"tok_1"`, `"tok_2"`, ….
- Write three assertions using only `node:assert`: a fresh session validates; after `clock.advance(31 * 60_000)` it does not; after `logout` it does not.
- Finally, show the exact compile error you get if you try `new SessionService(fakeClock, repo, generator)` (arguments swapped).

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed composition root with both singleton and per-request scopes, and prove the scoping works.

Requirements:

- Contracts: `Clock`, `Logger` (with `child(bindings)`), `AuditRepository` (`append(entry): Promise<void>`, `findByOrg(orgId): Promise<AuditEntry[]>`), `DocumentRepository` (`findById`, `findMany({ orgId })`, `create`, `delete`), and `Metrics` (`timing(name, ms)`).
- Split the container into two interfaces: `Infrastructure` (config, clock, logger, metrics, the two repositories) and `AppContainer extends Infrastructure` (adds one singleton `AuditService`).
- Write `buildInfrastructure(config: AppConfig): Infrastructure` and `buildContainer(infra): AppContainer`.
- Define `RequestContext = { requestId, orgId: OrgId | null, userId: UserId | null, startedAt: Date }` and `RequestContainer` containing a request-bound child `Logger`, a `TenantScopedDocumentRepository`, and a `DocumentService` built from them.
- `TenantScopedDocumentRepository` must implement `DocumentRepository`, force `orgId` into every filter, and return `null`/throw on cross-tenant `findById` and `delete`.
- Write `buildRequestContainer(app, ctx): RequestContainer` reusing every singleton and rebuilding only the scoped parts.
- Write `buildTestInfrastructure<O extends Partial<Infrastructure>>(overrides?: O): Infrastructure & O` so that `buildTestInfrastructure({ clock: new FakeClock() }).clock.advance(1000)` type-checks (the concrete `FakeClock` must survive).
- Write a test proving that a request container scoped to `org_a` gets `null` from `findById` for a document belonging to `org_b`, and a second test proving that two request containers built from the same `AppContainer` share the same `AuditRepository` instance but have different `Logger` instances.
- Add a fourth requirement: make it a **compile error** for a `DocumentService` method to be called without a tenant, by making `RequestContainer` two types — `AuthenticatedRequestContainer` (with `orgId: OrgId`) and `AnonymousRequestContainer` (no `documentService` at all) — and a type guard that narrows between them.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a small, fully type-safe runtime DI container with tokens, lifetimes, and cycle detection — then argue against using it.

Requirements:

- Implement `class Token<T> { declare readonly _type: T; constructor(readonly name: string) {} }` and prove `container.resolve(MailerToken)` infers `Mailer` (not `unknown`, no cast at the call site).
- Implement `Container` with:
  - `register<T>(token: Token<T>, factory: (c: Container) => T, lifetime: "singleton" | "transient" | "scoped"): this`
  - `resolve<T>(token: Token<T>): T` — caches singletons, never caches transients.
  - `createScope(): Container` — a child container that inherits registrations, shares the parent's singletons, and caches `scoped` instances per child.
  - `registerValue<T>(token: Token<T>, value: T): this` for config and pre-built instances.
- Add **cycle detection**: keep a resolution stack and throw `Circular dependency: Mailer -> Logger -> Mailer` (with the real chain) instead of blowing the stack.
- Add `dispose(): Promise<void>` that closes anything registered with an `onDispose` hook, in reverse construction order, and continues on error. A disposed container must throw on further `resolve` calls.
- Make `register` **fail to compile** when the factory's return type does not match the token's type parameter, and demonstrate the error in a comment.
- Add a `resolveAll<T>(token: Token<T>): T[]` for multi-registration (several `EventHandler` implementations under one token), and show how the type stays `T[]`.
- Wire a realistic graph through it: `AppConfig` (value), `Clock`, `Logger`, `UserRepository`, `PasswordHasher`, `TokenIssuer`, `Mailer`, `AuthService`, plus a `scoped` `RequestContext` and a `scoped` tenant-scoped `ProjectRepository`.
- Then write the **same graph as a hand-written composition root** (a plain function returning a typed object), and write a short comparison covering: total lines of code, where each version fails when you add a new required dependency to `AuthService`, whether that failure is compile-time or runtime, what happens on a typo in a token name, and what `Go to Definition` does in each. Conclude with the specific project size or requirement at which you would actually adopt the container.
- Bonus: attempt automatic constructor injection using `experimentalDecorators` + `emitDecoratorMetadata` + `reflect-metadata`, and document exactly which dependencies it can resolve without an explicit token and which it cannot — and why.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Constructor injection with parameter properties ─────────────────────────
class UserService {
  constructor(
    private readonly users:  UserRepository,   // declared + annotated + assigned
    private readonly mailer: Mailer,
    private readonly clock:  Clock,
    private readonly logger: Logger,
  ) {}
}
// NOTE: without an access modifier, a constructor parameter is NOT a field.

// ── Contracts, not classes ──────────────────────────────────────────────────
interface Clock  { now(): Date }
interface Mailer { sendWelcome(to: string, displayName: string): Promise<void> }
const fixedClock: Clock = { now: () => new Date("2026-07-22T00:00:00Z") };  // structural

// ── Factory-function style (no classes) ─────────────────────────────────────
export function createAuthService(deps: { users: UserRepository; tokens: TokenIssuer }) {
  return { async login(email: string, password: string) { /* ... */ } };
}
export type AuthService = ReturnType<typeof createAuthService>;

// ── Composition root: the one file that names concrete classes ──────────────
export interface AppContainer { readonly users: UserRepository; readonly mailer: Mailer }
export function buildContainer(config: AppConfig): AppContainer {
  const logger: Logger = createJsonLogger(config.nodeEnv);
  return { users: new MongoUserRepository(), mailer: new SendGridMailer(config.sendgridKey, logger) };
}

// ── Per-request container (tenant scoping, request logger, DataLoader) ──────
export function buildRequestContainer(app: AppContainer, ctx: RequestContext) {
  return {
    ctx,
    logger:   app.logger.child({ requestId: ctx.requestId }),
    projects: new TenantScopedProjectRepository(app.projects, ctx.orgId),
  };
}

// ── Typed injection token (interfaces don't exist at runtime) ───────────────
class Token<T> { declare readonly _type: T; constructor(readonly name: string) {} }
const MailerToken = new Token<Mailer>("Mailer");
// container.resolve(MailerToken) → Mailer

// ── Test wiring with `satisfies` — constraint checked, fakes preserved ──────
const rig = { clock: new FakeClock(), mailer: new RecordingMailer() } satisfies Partial<Infrastructure>;
rig.clock.advance(60_000);      // ✅ FakeClock survives
rig.mailer.sent.length;         // ✅ RecordingMailer survives

// ── Override helper that preserves concrete fake types ──────────────────────
function buildTestInfra<O extends Partial<Infrastructure>>(o: O = {} as O): Infrastructure & O {
  return { ...defaultTestInfra(), ...o };
}
```

| Rule | Detail |
|---|---|
| Inject the interface, never the class | Concrete classes with `private` fields are compared **nominally** — nothing else is assignable |
| Use `private readonly` parameter properties | No modifier = no field; `readonly` stops mid-flight reassignment |
| One composition root | Only the entry point may `new` a concrete implementation or read `process.env` |
| Never import the container into a service | That is a service locator: hidden deps, runtime failures, unchecked casts |
| Singletons must be stateless | Per-request state in a singleton field is a cross-tenant data leak |
| Per-request: context, tenant scope, DataLoader, child logger, tx client | Everything else is a singleton |
| Inject only real seams | Network, disk, clock, randomness, env. Import pure functions directly |
| No optional deps (`mailer?: Mailer`) | Use a required dep + a `NoopMailer`; decide once, in the root |
| Break cycles by narrowing the contract | `OrderLookup` instead of `OrderService`; then events; lazy thunks last |
| `import type` breaks type-only cycles | Erased at compile time — no runtime module cycle |
| Fakes are compile-checked | Add a method to `Mailer` → every fake fails to compile, precisely |
| `satisfies` for test wiring | Checks the constraint, keeps `advance()` / `sent` on the fakes |
| Interfaces are erased | So automatic container injection needs tokens; `Token<T>` keeps inference |
| `emitDecoratorMetadata` only sees classes | Interfaces emit as `Object`, primitives as `Number`/`String` — hence `@inject("...")` |
| Prefer manual wiring | ~100 lines, zero deps, zero flags, all errors at compile time |

---

## Connected topics

- **14 — Interfaces** — the contract you inject against; structural typing is why an object literal is a valid dependency.
- **19 — Implementing interfaces in classes** — `implements` moves errors from the wiring site to the implementation file.
- **26 — What are generics** — `Token<T>`, `buildTestInfra<O>`, and `resolve<T>` all depend on type parameters.
- **32 — Utility types** — `Partial<Infrastructure>`, `Omit`, `ReturnType<typeof createAuthService>` shape every container helper.
- **34 — Access modifiers** — `private` on a class field makes that class **nominally** typed, which is exactly why injecting concrete classes fails.
- **36 — Parameter properties** — the syntax this whole doc rests on; read it if `constructor(private readonly x: X) {}` is still surprising.
- **38 — Class implementing interface** — repositories, mailers, and clocks all `implements` their contracts.
- **42 — Discriminated unions** — `ApiResponse<T>` as `{ ok: true } | { ok: false }` is what injected services return.
- **48 — Modules in TypeScript** — module cycles, `import type`, and why module-scope `new` is a trap.
- **52 — Typing Express routes** and **54 — Typed middleware** — where the request container is built and consumed.
- **53 — Extending the Express Request** — the module augmentation that makes `req.container` type-safe.
- **55 — Typing environment variables** — `loadConfig(process.env)` is the first line of every composition root.
- **58 — Typing API responses** — the `ApiResponse<T>` returned by every injected service.
- **65 — Readonly and immutability patterns** — `readonly` dependencies and `readonly` context objects prevent whole classes of scope bugs.
- **67 — Repository pattern** — the prequel: this doc wires the repositories that doc defines, and swaps the in-memory fake in tests.
