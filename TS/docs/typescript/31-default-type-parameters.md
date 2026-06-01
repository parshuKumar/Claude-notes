# 31 — Default Type Parameters

## What is this?

A **default type parameter** gives a type parameter a fallback type that TypeScript uses when no explicit type argument is provided *and* inference cannot determine the type from the arguments. You write it as `<T = DefaultType>` — the same syntax as a default function parameter, but at the type level.

Without a default, omitting a type argument on a generic that cannot be inferred gives you `unknown` (or triggers an error in strict contexts). With a default, omitting the type argument gives you the default type instead — making the type argument effectively optional.

## Why does it matter?

Three concrete problems defaults solve:

1. **Empty collections** — `first<T>([])` can't infer `T` from an empty array. Without a default, `T = unknown`. With `<T = never>` or a sensible domain default, callers get the type they expect.
2. **Optional generic parameters** — A generic interface like `ApiResponse<TData, TError = Error>` lets callers write `ApiResponse<User>` instead of `ApiResponse<User, Error>` every time — removing noise when the second parameter is almost always `Error`.
3. **Ergonomic factory functions** — `createStore()` with no argument can default `T` to a sensible base type, while `createStore<AdminUser>()` overrides it explicitly.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no concept of type parameters at all:
function createResponse(data, status = 200) {
  return { data, status, timestamp: new Date().toISOString() };
}
// Returns an object, but callers have no idea what 'data' contains.
```

```ts
// TypeScript WITHOUT a default type parameter:
interface ApiResponse<TData> {
  data: TData;
  status: number;
  timestamp: string;
}

// Every usage must specify TData — even for simple void/unknown responses:
function noContent(): ApiResponse<null> { ... }     // forced to write <null>
function unknownResp(): ApiResponse<unknown> { ... } // forced to write <unknown>

// TypeScript WITH a default type parameter:
interface ApiResponse<TData = unknown> {
  data: TData;
  status: number;
  timestamp: string;
}

// Default kicks in when TData is omitted:
function noContent(): ApiResponse { ... }          // TData = unknown (default)
function typed(): ApiResponse<User> { ... }        // TData = User   (explicit)

// Callers of noContent():
const r = noContent();
r.data;            // unknown — safe, TypeScript forces checking before use
r.status;          // number  — always typed correctly
```

---

## Syntax

```ts
// ── Default on an interface ───────────────────────────────────────────────
interface Container<T = string> {
  value: T;
}
// Container         → Container<string>  (default)
// Container<number> → Container<number>  (explicit)

// ── Default on a type alias ───────────────────────────────────────────────
type Nullable<T = unknown> = T | null;
// Nullable          → unknown | null
// Nullable<User>    → User | null

// ── Default on a generic function ────────────────────────────────────────
function createList<T = User>(): T[] {
  return [];
}
// createList()         → User[]   (default — explicit arg needed to override)
// createList<Post>()   → Post[]   (explicit)

// ── Multiple params — defaults must come after non-defaulted params ───────
interface ServiceResult<TData, TError = Error> {
  ok: boolean;
  data?: TData;
  error?: TError;
}
// ServiceResult<User>         → TData=User, TError=Error  (default)
// ServiceResult<User, string> → TData=User, TError=string (explicit)

// ── Default that references a previous param ─────────────────────────────
interface Transform<TInput, TOutput = TInput> {
  // If TOutput not given, it defaults to TInput (identity transform)
  execute(input: TInput): TOutput;
}
// Transform<string>         → input: string, output: string
// Transform<string, number> → input: string, output: number

// ── Default on a class ───────────────────────────────────────────────────
class EventEmitter<TPayload = Record<string, unknown>> {
  private handlers: Array<(payload: TPayload) => void> = [];
  on(handler: (payload: TPayload) => void): void { this.handlers.push(handler); }
  emit(payload: TPayload): void { this.handlers.forEach(h => h(payload)); }
}
// new EventEmitter()        → EventEmitter<Record<string, unknown>>
// new EventEmitter<UserEvent>() → EventEmitter<UserEvent>
```

---

## How it works — concept by concept

### When does the default apply?

The default only kicks in when **both** conditions are true:
1. The type argument is not provided explicitly.
2. TypeScript cannot infer the type from the function arguments.

If TypeScript *can* infer `T` from the arguments, the inferred type wins — the default is ignored:

```ts
function wrap<T = string>(value: T): { data: T } {
  return { data: value };
}

wrap(42);           // T inferred as number — default NOT used → { data: number }
wrap("hello");      // T inferred as string — default NOT used → { data: string }
wrap<boolean>(true);// T explicit as boolean — default NOT used → { data: boolean }

// Default only applies here — no argument to infer from:
function createEmpty<T = User>(): T[] {
  return [];
}
createEmpty();        // T cannot be inferred → T = User (default) → User[]
createEmpty<Post>();  // T = Post (explicit) → Post[]
```

### Default ordering rule — defaults must come last

Just like default function parameters must come after required ones, default type parameters must come after non-defaulted ones:

```ts
// ✅ Non-defaulted before defaulted:
interface Result<TValue, TError = Error> { ... }

// ❌ Defaulted before non-defaulted — Error:
interface Result<TError = Error, TValue> { ... }
// Error: Required type parameters may not follow optional type parameters

// ✅ All with defaults — any order is fine since they're all optional:
interface Config<TBase = object, TExtra = unknown> { ... }
```

### Defaults can reference earlier type parameters

A later type parameter's default can reference an earlier type parameter:

```ts
// TOutput defaults to TInput — a no-op / identity transform by default:
interface Mapper<TInput, TOutput = TInput> {
  map(input: TInput): TOutput;
}

// When only TInput is given, TOutput = TInput:
const identity: Mapper<string> = {
  map: s => s,  // input: string, output: string
};

// When both are given, TOutput overrides:
const parser: Mapper<string, number> = {
  map: s => parseInt(s, 10),  // input: string, output: number
};

// More complex example — default builds on TEntity:
interface Repository<
  TEntity extends { id: number },
  TCreateInput = Omit<TEntity, "id" | "createdAt" | "updatedAt">,
  TUpdateInput = Partial<TCreateInput>,
> {
  create(input: TCreateInput): Promise<TEntity>;
  update(id: number, input: TUpdateInput): Promise<TEntity | null>;
}

// With User:
// TCreateInput defaults to Omit<User, "id"|"createdAt"|"updatedAt">
// TUpdateInput defaults to Partial<Omit<User, "id"|"createdAt"|"updatedAt">>

interface User { id: number; name: string; email: string; createdAt: Date; updatedAt: Date; }

type UserRepo = Repository<User>;
// UserRepo.create(input: { name: string; email: string }) — id/timestamps excluded automatically
// UserRepo.update(id, input: { name?: string; email?: string }) — all optional
```

### Using `never` as a default for impossible states

`never` as a default is used when the type parameter represents "no error" or "no extra data" — a state that should never be instantiated:

```ts
// Result type — TError defaults to never (success has no error):
type Result<TValue, TError = never> =
  | { ok: true;  value: TValue }
  | { ok: false; error: TError };

// Success — TError = never (you can't create an error branch):
type SuccessResult<T> = Result<T>;          // Result<T, never>
const ok: Result<User> = { ok: true, value: { id: 1 } as User };

// Failure with specific error type:
type FailResult = Result<never, Error>;     // { ok: false; error: Error }

// Full both sides:
type ApiResult = Result<User, { code: number; message: string }>;
```

---

## Example 1 — basic

```ts
// Practical default type parameter patterns for a backend service

// ── 1. ServiceResult with a default error type ────────────────────────────

interface ServiceResult<TData, TError = string> {
  ok: boolean;
  data?: TData;
  error?: TError;
}

function succeed<T>(data: T): ServiceResult<T> {
  return { ok: true, data };
}

function fail<T, E = string>(error: E): ServiceResult<T, E> {
  return { ok: false, error };
}

// Default error = string:
const r1: ServiceResult<User> = succeed({ id: 1, name: "Parsh" } as User);
// ServiceResult<User, string> — TError defaults to string

// Custom error type:
interface ApiError { code: number; message: string; details: string[]; }
const r2: ServiceResult<User, ApiError> = fail<User, ApiError>({ code: 404, message: "Not found", details: [] });

// ── 2. ApiResponse with optional metadata ─────────────────────────────────

interface ApiResponse<TData = unknown, TMeta = Record<string, never>> {
  data: TData;
  status: number;
  message: string;
  timestamp: string;
  meta?: TMeta;
}

interface PaginationMeta {
  total: number;
  page: number;
  pageSize: number;
  hasNext: boolean;
}

function response<T>(data: T, status = 200, message = "ok"): ApiResponse<T> {
  return { data, status, message, timestamp: new Date().toISOString() };
}

function pagedResponse<T>(
  data: T[],
  meta: PaginationMeta,
): ApiResponse<T[], PaginationMeta> {
  return {
    data,
    status: 200,
    message: "ok",
    timestamp: new Date().toISOString(),
    meta,
  };
}

// No meta — TMeta defaults to Record<string, never>:
const r3 = response({ id: 1, name: "Parsh" });
r3.data.id;    // ✅ number
r3.meta;       // ✅ Record<string, never> | undefined — minimal

// With pagination meta:
const r4 = pagedResponse([{ id: 1 }], { total: 100, page: 1, pageSize: 20, hasNext: true });
r4.meta?.total;    // ✅ number
r4.meta?.hasNext;  // ✅ boolean

// ── 3. Transform pipeline with TOutput defaulting to TInput ───────────────

interface PipelineStep<TInput, TOutput = TInput> {
  name: string;
  execute(input: TInput): Promise<TOutput>;
  rollback?(input: TInput): Promise<void>;
}

// Identity step (no transformation — TOutput = TInput):
const validateStep: PipelineStep<CreateOrderInput> = {
  name: "validate",
  async execute(input) {
    // Validates and returns the same type — no transformation
    if (!input.userId) throw new Error("userId required");
    return input;  // TInput = TOutput = CreateOrderInput
  },
};

// Transform step (changes the type — TOutput ≠ TInput):
interface CreateOrderInput { userId: number; items: Array<{ productId: number; quantity: number }>; totalCents: number; }
interface CreatedOrder { orderId: number; userId: number; status: "pending"; totalCents: number; createdAt: Date; }

const createOrderStep: PipelineStep<CreateOrderInput, CreatedOrder> = {
  name: "createOrder",
  async execute(input) {
    return {
      orderId: Math.floor(Math.random() * 100000),
      userId: input.userId,
      status: "pending",
      totalCents: input.totalCents,
      createdAt: new Date(),
    };
  },
};
```

---

## Example 2 — real world backend use case

```ts
// Generic Repository base with defaulted input types — write once, customize only when needed

interface Entity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

// TCreateInput defaults to "everything except id/createdAt/updatedAt"
// TUpdateInput defaults to "all create fields made optional"
// This covers 90% of repositories with zero extra work:
interface Repository<
  TEntity extends Entity,
  TCreateInput = Omit<TEntity, "id" | "createdAt" | "updatedAt">,
  TUpdateInput = Partial<TCreateInput>,
> {
  findById(id: number): Promise<TEntity | null>;
  findAll(filter?: Partial<TEntity>): Promise<TEntity[]>;
  create(input: TCreateInput): Promise<TEntity>;
  update(id: number, input: TUpdateInput): Promise<TEntity | null>;
  delete(id: number): Promise<boolean>;
}

// ── Domain models ──────────────────────────────────────────────────────────

interface User extends Entity {
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
}

interface Post extends Entity {
  title: string;
  body: string;
  authorId: number;
  published: boolean;
}

interface Session extends Entity {
  userId: number;
  token: string;
  expiresAt: Date;
  ipAddress: string;
}

// ── In-memory implementation ───────────────────────────────────────────────

class InMemoryRepository<
  TEntity extends Entity,
  TCreateInput = Omit<TEntity, "id" | "createdAt" | "updatedAt">,
  TUpdateInput = Partial<TCreateInput>,
> implements Repository<TEntity, TCreateInput, TUpdateInput> {
  protected store = new Map<number, TEntity>();
  private seq = 1;

  async findById(id: number): Promise<TEntity | null> {
    return this.store.get(id) ?? null;
  }

  async findAll(filter?: Partial<TEntity>): Promise<TEntity[]> {
    const all = [...this.store.values()];
    if (!filter) return all;
    return all.filter(entity =>
      Object.entries(filter).every(([k, v]) => (entity as Record<string, unknown>)[k] === v),
    );
  }

  async create(input: TCreateInput): Promise<TEntity> {
    const entity = {
      ...(input as object),
      id: this.seq++,
      createdAt: new Date(),
      updatedAt: new Date(),
    } as TEntity;
    this.store.set(entity.id, entity);
    return entity;
  }

  async update(id: number, input: TUpdateInput): Promise<TEntity | null> {
    const existing = this.store.get(id);
    if (!existing) return null;
    const updated = { ...existing, ...(input as object), updatedAt: new Date() } as TEntity;
    this.store.set(id, updated);
    return updated;
  }

  async delete(id: number): Promise<boolean> {
    return this.store.delete(id);
  }
}

// ── Standard repositories — zero extra boilerplate for create/update types ─

// UserRepository: TCreateInput and TUpdateInput are auto-derived from User via defaults
class UserRepository extends InMemoryRepository<User> {
  // create() expects: { name: string; email: string; role: "admin"|"editor"|"viewer" }
  // update() expects: { name?: string; email?: string; role?: "admin"|"editor"|"viewer" }
  // All correct — zero extra type declarations needed

  async findByEmail(email: string): Promise<User | null> {
    return (await this.findAll({ email } as Partial<User>))[0] ?? null;
  }
}

class PostRepository extends InMemoryRepository<Post> {
  // create() expects: { title: string; body: string; authorId: number; published: boolean }
  // update() expects: { title?: string; body?: string; authorId?: number; published?: boolean }

  async findPublished(): Promise<Post[]> {
    return this.findAll({ published: true } as Partial<Post>);
  }
}

// ── Custom inputs — override defaults when needed ─────────────────────────

// Password is required at creation but must NOT appear in TEntity (it's hashed separately).
// We need a custom TCreateInput:
interface UserCreateInput {
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  password: string;  // raw password — hashed before saving
}

interface UserUpdateInput {
  name?: string;
  email?: string;
  role?: "admin" | "editor" | "viewer";
  newPassword?: string;  // optional new password
}

// Override TCreateInput and TUpdateInput — default no longer applies here:
class SecureUserRepository extends InMemoryRepository<User, UserCreateInput, UserUpdateInput> {
  override async create(input: UserCreateInput): Promise<User> {
    const { password, ...rest } = input;
    const passwordHash = `hashed_${password}`;  // stub — use bcrypt in production
    return super.create({ ...rest } as Omit<User, "id" | "createdAt" | "updatedAt">);
  }
}

// ── Usage ──────────────────────────────────────────────────────────────────

const userRepo = new UserRepository();
const postRepo = new PostRepository();
const secureRepo = new SecureUserRepository();

// Standard UserRepository — inferred create/update types:
const user = await userRepo.create({ name: "Parsh", email: "p@dev.io", role: "admin" });
// ✅ { name: string; email: string; role: ... } — auto-derived from User

await userRepo.update(1, { role: "editor" });
// ✅ Partial<{ name; email; role }> — all optional, auto-derived

await userRepo.create({ name: "Parsh", email: "p@dev.io", role: "admin", id: 99 });
// ❌ Error: 'id' is not expected — it's in Omit<..., "id"> (the default)

// Secure repo — custom create input:
await secureRepo.create({ name: "Parsh", email: "p@dev.io", role: "admin", password: "s3cr3t" });
// ✅ password is part of UserCreateInput (explicit override)
```

---

## Common mistakes

### Mistake 1 — Expecting the default to apply when inference succeeds

```ts
function wrap<T = string>(value: T): { data: T } {
  return { data: value };
}

// ❌ WRONG expectation: T should be "string" because of the default
const r = wrap(42);
// T is inferred as NUMBER from the argument — default is NOT used
// r: { data: number }  — not { data: string }

// ✅ CORRECT: Default only applies when inference has nothing to work with:
function createEmpty<T = string>(): T[] { return []; }
const arr = createEmpty();      // T = string (default — nothing to infer from)
const arr2 = createEmpty<number>(); // T = number (explicit override)
```

### Mistake 2 — Putting a defaulted param before a non-defaulted one

```ts
// ❌ Error — defaulted before non-defaulted:
interface Transformer<TOutput = string, TInput> {
  transform(input: TInput): TOutput;
}
// Error: Required type parameters may not follow optional type parameters

// ✅ Non-defaulted first, defaulted last:
interface Transformer<TInput, TOutput = TInput> {
  transform(input: TInput): TOutput;
}
```

### Mistake 3 — Using `unknown` as a default when `never` is more appropriate

```ts
// ❌ Using unknown as default for "no error" — forces callers to handle unknown error:
interface Result<TValue, TError = unknown> {
  ok: boolean;
  value?: TValue;
  error?: TError;
}

const r: Result<User> = { ok: true, value: user };
r.error; // TError = unknown — even on success, TypeScript thinks error could be unknown

// ✅ Use never for "this branch cannot exist" — TypeScript knows error is absent on success:
type Result<TValue, TError = never> =
  | { ok: true;  value: TValue  }
  | { ok: false; error: TError  };

const success: Result<User> = { ok: true, value: user };
// success.error — ❌ doesn't exist on the success branch
// TError = never means there literally cannot be an error value
```

---

## Practice exercises

### Exercise 1 — easy

1. Write a generic `Wrapper<T = string>` interface with `{ value: T; label: string; serialize(): string }`. Implement it as a class. Verify that `new Wrapper("hello")` gives `Wrapper<string>` and `new Wrapper(42)` gives `Wrapper<number>`, but `new Wrapper()` (no arg) requires an explicit type arg or won't compile without one — explain why.

2. Write `createPair<TFirst = string, TSecond = TFirst>(first: TFirst, second: TSecond): [TFirst, TSecond]`. Test with:
   - No explicit args, both inferred
   - `createPair<number>()` — what happens to TSecond?
   - `createPair<number, boolean>(1, true)`

3. Write a `Cache<TValue = string>` interface and implement it. Show that `new Cache()` gives `Cache<string>` and `new Cache<User>()` gives `Cache<User>`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed HTTP client where the error type is optional:

```ts
interface HttpResult<TData, TError = HttpError> {
  ok: boolean;
  status: number;
  data?: TData;
  error?: TError;
  headers: Record<string, string>;
}

interface HttpError {
  status: number;
  message: string;
  code?: string;
}

class HttpClient {
  async get<TData, TError = HttpError>(url: string): Promise<HttpResult<TData, TError>>
  async post<TData, TRequestBody = unknown, TError = HttpError>(
    url: string,
    body: TRequestBody,
  ): Promise<HttpResult<TData, TError>>
  async put<TData, TRequestBody = unknown, TError = HttpError>(
    url: string,
    body: TRequestBody,
  ): Promise<HttpResult<TData, TError>>
  async delete<TData = null, TError = HttpError>(url: string): Promise<HttpResult<TData, TError>>
}
```

Implement the class (stubs are fine for the actual fetch calls — return fake data). Then:
1. Call `client.get<User>("/api/users/1")` — TError defaults to `HttpError`
2. Call `client.get<User, ValidationError>("/api/users/1")` — TError overridden
3. Call `client.delete("/api/users/1")` — TData defaults to `null`
4. Call `client.post<Order, CreateOrderInput>("/api/orders", input)` — TRequestBody explicit, TError defaults

Show that accessing `.error` on each result has the right type.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a type-safe generic state machine where the state and event types have defaults:

```ts
type Transition<TState extends string, TEvent extends string> = {
  from: TState | TState[];
  event: TEvent;
  to: TState;
  guard?: (context: unknown) => boolean;
  action?: (context: unknown) => void;
};

class StateMachine<
  TState extends string = string,
  TEvent extends string = string,
  TContext = Record<string, unknown>,
> {
  constructor(
    initialState: TState,
    transitions: Array<Transition<TState, TEvent>>,
    context?: TContext,
  )

  // Send an event — returns the new state or throws if transition invalid:
  send(event: TEvent): TState

  // Current state:
  get state(): TState

  // Current context:
  get context(): TContext

  // Update context:
  setContext(updater: (ctx: TContext) => TContext): void

  // Check if a transition is valid from current state:
  canSend(event: TEvent): boolean

  // Subscribe to state changes:
  onTransition(listener: (from: TState, event: TEvent, to: TState, ctx: TContext) => void): () => void
}
```

Implement the `StateMachine` class fully, then use it for an order workflow:

```ts
type OrderState = "pending" | "confirmed" | "processing" | "shipped" | "delivered" | "cancelled";
type OrderEvent = "confirm" | "startProcessing" | "ship" | "deliver" | "cancel";

interface OrderContext {
  orderId: number;
  userId: number;
  totalCents: number;
  shippedAt?: Date;
  deliveredAt?: Date;
  cancelledAt?: Date;
  cancellationReason?: string;
}

const orderMachine = new StateMachine<OrderState, OrderEvent, OrderContext>(
  "pending",
  [
    { from: "pending",     event: "confirm",         to: "confirmed"  },
    { from: "confirmed",   event: "startProcessing", to: "processing" },
    { from: "processing",  event: "ship",            to: "shipped",
      action: (ctx) => { (ctx as OrderContext).shippedAt = new Date(); } },
    { from: "shipped",     event: "deliver",         to: "delivered",
      action: (ctx) => { (ctx as OrderContext).deliveredAt = new Date(); } },
    { from: ["pending", "confirmed", "processing"], event: "cancel", to: "cancelled",
      action: (ctx) => { (ctx as OrderContext).cancelledAt = new Date(); } },
  ],
  { orderId: 42, userId: 1, totalCents: 9900 },
);

orderMachine.send("confirm");          // ✅ state = "confirmed"
orderMachine.send("startProcessing");  // ✅ state = "processing"
orderMachine.send("confirm");          // ❌ throws — "confirm" not valid from "processing"
orderMachine.canSend("ship");          // ✅ true
orderMachine.send("ship");
orderMachine.context.shippedAt;       // ✅ Date — set by action
```

Also demonstrate the machine with default type parameters:
```ts
// Simple string-based machine — uses defaults:
const trafficLight = new StateMachine("red", [
  { from: "red",    event: "next", to: "green"  },
  { from: "green",  event: "next", to: "yellow" },
  { from: "yellow", event: "next", to: "red"    },
]);
// TState = string, TEvent = string, TContext = Record<string, unknown>
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Default on interface:
interface Box<T = string> { value: T; }
// Box       → Box<string>
// Box<User> → Box<User>

// Default on type alias:
type Nullable<T = unknown> = T | null;

// Default on function:
function createList<T = User>(): T[] { return []; }
// createList()        → User[]  (default — inference has nothing to work with)
// createList<Post>()  → Post[]  (explicit)
// createList([])      → never[] (inferred from argument — default NOT used)

// Default on class:
class Emitter<T = Record<string, unknown>> { ... }

// Default referencing earlier param:
interface Transform<TIn, TOut = TIn> { ... }
// Transform<string>         → TIn=string, TOut=string
// Transform<string, number> → TIn=string, TOut=number

// Multiple defaults — non-defaulted BEFORE defaulted:
interface Result<TData, TError = Error, TMeta = null> { ... }
// Result<User>                → TData=User,  TError=Error,  TMeta=null
// Result<User, ApiError>      → TData=User,  TError=ApiError, TMeta=null
// Result<User, ApiError, {}>  → TData=User,  TError=ApiError, TMeta={}
```

| Rule | Details |
|------|---------|
| When default is used | Type arg omitted AND TypeScript cannot infer from arguments |
| When default is NOT used | TypeScript infers `T` from arguments — inference always wins |
| Ordering | Non-defaulted params must come before defaulted params |
| Default can ref earlier param | `<TIn, TOut = TIn>` — TOut defaults to TIn |
| `never` as default | "This branch cannot exist" — better than `unknown` for absent error types |
| `unknown` as default | "Could be anything" — safe but requires checking before use |

---

## Connected topics

- **26 — What are generics** — type parameters, inference, the `T` placeholder.
- **27 — Generic functions** — where default function type params are most used.
- **28 — Generic interfaces** — defaulted interface type params like `ApiResponse<T = unknown>`.
- **29 — Generic classes** — defaulted class type params like `EventEmitter<T = Record<string,unknown>>`.
- **30 — Generic constraints** — constraints can be combined with defaults: `<T extends Entity = User>`.
- **40 — Conditional types** — defaults interact with conditional types in advanced utility type patterns.
