# 28 — Generic Interfaces

## What is this?

A **generic interface** is an interface that declares one or more **type parameters** — just like a generic function, but applied to an entire interface shape. Every property, method, and nested type in the interface can reference those type parameters.

Generic interfaces let you define a **contract once** and specialize it for many different data shapes. The `ApiResponse<T>`, `Repository<T>`, `Paginated<T>`, and `EventHandler<T>` patterns you see in every real TypeScript backend are all generic interfaces.

## Why does it matter?

Without generic interfaces, you'd need a separate interface per data type:

```ts
interface UserApiResponse    { data: User;    status: number; message: string; }
interface PostApiResponse    { data: Post;    status: number; message: string; }
interface OrderApiResponse   { data: Order;   status: number; message: string; }
interface UserListResponse   { data: User[];  status: number; message: string; }
```

Every time you add an endpoint, you add another interface. They all share the same shape — only `data` changes. Generic interfaces collapse all of this into one:

```ts
interface ApiResponse<TData> {
  data: TData;
  status: number;
  message: string;
}

type UserApiResponse  = ApiResponse<User>;
type PostApiResponse  = ApiResponse<Post>;
type UserListResponse = ApiResponse<User[]>;
```

Beyond reducing duplication, generic interfaces enable **type-safe contracts** — a `Repository<User>` can only store and return `User` objects, and TypeScript enforces it throughout the entire codebase.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no interfaces at all:
// You just return whatever shape you want, and callers guess.

function getUser(id) {
  return fetch(`/api/users/${id}`).then(r => r.json());
  // Returns ??? — nobody knows
}
```

```ts
// TypeScript with generic interfaces — the contract is explicit and enforced:

interface ApiResponse<TData> {
  data: TData;
  status: number;
  message: string;
  timestamp: string;
}

async function getUser(id: number): Promise<ApiResponse<User>> {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

const response = await getUser(1);
response.data.name;      // ✅ TypeScript knows data is User
response.data.password;  // ❌ Error — User has no 'password' field
response.status;         // ✅ number
response.timestamp.toUpperCase(); // ✅ string
```

---

## Syntax

```ts
// ── Basic generic interface ───────────────────────────────────────────────
interface Container<T> {
  value: T;
}

// ── Multiple type parameters ──────────────────────────────────────────────
interface KeyValuePair<TKey, TValue> {
  key: TKey;
  value: TValue;
}

// ── Generic method in an interface ───────────────────────────────────────
interface Transformer<TInput, TOutput> {
  transform(input: TInput): TOutput;
  transformAll(inputs: TInput[]): TOutput[];
}

// ── Constraint on a type parameter ───────────────────────────────────────
interface Repository<TEntity extends { id: number }> {
  findById(id: number): Promise<TEntity | null>;
  findAll(): Promise<TEntity[]>;
  create(entity: Omit<TEntity, "id">): Promise<TEntity>;
  update(id: number, partial: Partial<TEntity>): Promise<TEntity | null>;
  delete(id: number): Promise<boolean>;
}

// ── Extending a generic interface ────────────────────────────────────────
interface PaginatedRepository<TEntity extends { id: number }>
  extends Repository<TEntity> {
  findPage(page: number, pageSize: number): Promise<Paginated<TEntity>>;
}

// ── Specializing (instantiating) a generic interface ─────────────────────
type UserRepository       = Repository<User>;
type PostRepository       = Repository<Post>;
type UserApiResponse      = ApiResponse<User>;
type UserListApiResponse  = ApiResponse<User[]>;

// ── Default type parameter ────────────────────────────────────────────────
interface ApiResponse<TData = unknown> {
  data: TData;
  status: number;
  message: string;
  timestamp: string;
}
```

---

## How it works — concept by concept

### Type parameters flow through the entire interface

Every member of the interface can reference the type parameters. The types are resolved when you use the interface with a concrete type:

```ts
interface ServiceResult<TData, TError = Error> {
  success: boolean;
  data: TData | null;         // TData used in a property
  error: TError | null;       // TError used in a property
  unwrap(): TData;            // TData used in a method return type
  map<TResult>(fn: (data: TData) => TResult): ServiceResult<TResult, TError>;
  // ↑ TData used in a callback param, TResult introduces a new param for just this method
}
```

When you write `ServiceResult<User>`:
- `data` becomes `User | null`
- `error` becomes `Error | null` (default)
- `unwrap()` returns `User`
- `map(fn)` takes `(data: User) => TResult` and returns `ServiceResult<TResult, Error>`

### Implementing a generic interface in a class

A class can implement a generic interface in two ways:

**1. Keep the type parameter open** — the class itself is generic:
```ts
class InMemoryRepo<T extends { id: number }> implements Repository<T> {
  private store = new Map<number, T>();

  async findById(id: number): Promise<T | null> {
    return this.store.get(id) ?? null;
  }
  // ... other methods
}

const userRepo = new InMemoryRepo<User>();   // now locked to User
const postRepo = new InMemoryRepo<Post>();   // now locked to Post
```

**2. Close the type parameter** — the class specializes the interface for a specific type:
```ts
class UserRepository implements Repository<User> {
  // All methods now work with User specifically:
  async findById(id: number): Promise<User | null> { ... }
  async findAll(): Promise<User[]> { ... }
  // User-specific extra methods:
  async findByEmail(email: string): Promise<User | null> { ... }
}
```

### Extending generic interfaces

Generic interfaces can extend other generic interfaces — passing, transforming, or adding type parameters:

```ts
interface Entity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

interface Repository<T extends Entity> {
  findById(id: number): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: number): Promise<boolean>;
}

// Adds pagination on top:
interface PaginatedRepository<T extends Entity> extends Repository<T> {
  findPage(page: number, pageSize: number): Promise<Paginated<T>>;
  count(filter?: Partial<T>): Promise<number>;
}

// Adds soft-delete on top:
interface SoftDeletableRepository<T extends Entity & { deletedAt: Date | null }>
  extends Repository<T> {
  softDelete(id: number): Promise<T | null>;
  findDeleted(): Promise<T[]>;
  restore(id: number): Promise<T | null>;
}
```

### Using generic interfaces as function parameter types

This is one of the most powerful patterns — a function that accepts *any* implementation of a generic interface:

```ts
// Works with ANY Repository<User> — InMemoryRepo, PostgresRepo, MongoRepo, etc.:
async function seedUsers(
  repo: Repository<User>,
  users: Array<Omit<User, "id" | "createdAt" | "updatedAt">>,
): Promise<User[]> {
  const results: User[] = [];
  for (const user of users) {
    const created = await repo.create(user);
    results.push(created);
  }
  return results;
}

// Works with ANY Repository<T> that has an id — fully generic:
async function findOrCreate<T extends Entity>(
  repo: Repository<T>,
  id: number,
  factory: () => Omit<T, "id" | "createdAt" | "updatedAt">,
): Promise<T> {
  const existing = await repo.findById(id);
  if (existing) return existing;
  return repo.create(factory() as Omit<T, "id">);
}
```

### Generic interfaces for callbacks and handlers

```ts
// Typed event handler contract:
interface EventHandler<TPayload> {
  handle(payload: TPayload): Promise<void>;
  canHandle(payload: unknown): payload is TPayload;
}

// Typed middleware contract:
interface Middleware<TContext> {
  process(context: TContext, next: () => Promise<void>): Promise<void>;
}

// Typed pipeline step:
interface PipelineStep<TInput, TOutput = TInput> {
  execute(input: TInput): Promise<TOutput>;
  rollback?(input: TInput): Promise<void>;  // optional compensation
}
```

---

## Example 1 — basic

```ts
// The ApiResponse<T> pattern — used across every endpoint in a real backend

interface ApiResponse<TData = unknown> {
  data: TData;
  status: number;
  message: string;
  timestamp: string;
}

interface PaginatedData<TItem> {
  items: TItem[];
  total: number;
  page: number;
  pageSize: number;
  hasNext: boolean;
  hasPrev: boolean;
}

// Convenience type aliases for common response shapes:
type ApiListResponse<TItem>      = ApiResponse<PaginatedData<TItem>>;
type ApiSingleResponse<TItem>    = ApiResponse<TItem>;
type ApiCreatedResponse<TItem>   = ApiResponse<TItem>;  // status 201
type ApiNoContentResponse        = ApiResponse<null>;   // status 204
type ApiErrorResponse            = ApiResponse<never>;  // data is never present on errors

// Helper factory functions:
function successResponse<T>(data: T, status = 200, message = "ok"): ApiResponse<T> {
  return { data, status, message, timestamp: new Date().toISOString() };
}

function paginatedResponse<T>(
  items: T[],
  total: number,
  page: number,
  pageSize: number,
): ApiListResponse<T> {
  return successResponse({
    items,
    total,
    page,
    pageSize,
    hasNext: page * pageSize < total,
    hasPrev: page > 1,
  });
}

function errorResponse(message: string, status: number): ApiErrorResponse {
  return { data: undefined as never, status, message, timestamp: new Date().toISOString() };
}

// Domain types:
interface User {
  id: number;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  createdAt: Date;
  updatedAt: Date;
}

interface Post {
  id: number;
  title: string;
  body: string;
  authorId: number;
  published: boolean;
  createdAt: Date;
  updatedAt: Date;
}

// Express route handlers typed with ApiResponse<T>:
function getUserHandler(userId: number): ApiSingleResponse<User> {
  const user: User = {
    id: userId,
    name: "Parsh",
    email: "p@dev.io",
    role: "admin",
    createdAt: new Date(),
    updatedAt: new Date(),
  };
  return successResponse(user);
}

function getUsersHandler(page: number, pageSize: number): ApiListResponse<User> {
  const users: User[] = [];
  const total = 0;
  return paginatedResponse(users, total, page, pageSize);
}

// Usage — callers get full type info:
const response = getUserHandler(1);
response.data.name;      // ✅ string
response.data.email;     // ✅ string
response.data.password;  // ❌ Error — not on User
response.status;         // ✅ number

const listResponse = getUsersHandler(1, 20);
listResponse.data.items;           // ✅ User[]
listResponse.data.items[0].name;   // ✅ string
listResponse.data.hasNext;         // ✅ boolean
listResponse.data.total;           // ✅ number
```

---

## Example 2 — real world backend use case

```ts
// Full typed service layer using generic interfaces throughout

// ── Core interfaces ────────────────────────────────────────────────────────

interface Entity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

interface Repository<T extends Entity> {
  findById(id: number): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, "id" | "createdAt" | "updatedAt">): Promise<T>;
  update(id: number, data: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>): Promise<T | null>;
  delete(id: number): Promise<boolean>;
}

interface ServiceResult<TData, TError = string> {
  ok: boolean;
  data?: TData;
  error?: TError;
}

interface CrudService<TEntity extends Entity, TCreateInput, TUpdateInput> {
  getById(id: number): Promise<ServiceResult<TEntity>>;
  getAll(): Promise<ServiceResult<TEntity[]>>;
  create(input: TCreateInput): Promise<ServiceResult<TEntity>>;
  update(id: number, input: TUpdateInput): Promise<ServiceResult<TEntity>>;
  remove(id: number): Promise<ServiceResult<boolean>>;
}

// ── Domain models ──────────────────────────────────────────────────────────

interface User extends Entity {
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  passwordHash: string;
}

type CreateUserInput = { name: string; email: string; role: User["role"]; password: string };
type UpdateUserInput = Partial<{ name: string; email: string; role: User["role"] }>;

interface Post extends Entity {
  title: string;
  body: string;
  authorId: number;
  published: boolean;
  tags: string[];
}

type CreatePostInput = { title: string; body: string; authorId: number; tags?: string[] };
type UpdatePostInput = Partial<{ title: string; body: string; published: boolean; tags: string[] }>;

// ── Generic base service — implements CrudService<T, TCreate, TUpdate> ─────

class BaseCrudService<
  TEntity extends Entity,
  TCreateInput,
  TUpdateInput,
> implements CrudService<TEntity, TCreateInput, TUpdateInput> {
  constructor(protected repo: Repository<TEntity>) {}

  async getById(id: number): Promise<ServiceResult<TEntity>> {
    const entity = await this.repo.findById(id);
    if (!entity) return { ok: false, error: `Not found: id=${id}` };
    return { ok: true, data: entity };
  }

  async getAll(): Promise<ServiceResult<TEntity[]>> {
    const entities = await this.repo.findAll();
    return { ok: true, data: entities };
  }

  async create(input: TCreateInput): Promise<ServiceResult<TEntity>> {
    try {
      const entity = await this.repo.create(
        input as unknown as Omit<TEntity, "id" | "createdAt" | "updatedAt">,
      );
      return { ok: true, data: entity };
    } catch (err) {
      return { ok: false, error: String(err) };
    }
  }

  async update(id: number, input: TUpdateInput): Promise<ServiceResult<TEntity>> {
    const entity = await this.repo.update(
      id,
      input as Partial<Omit<TEntity, "id" | "createdAt" | "updatedAt">>,
    );
    if (!entity) return { ok: false, error: `Not found: id=${id}` };
    return { ok: true, data: entity };
  }

  async remove(id: number): Promise<ServiceResult<boolean>> {
    const deleted = await this.repo.delete(id);
    if (!deleted) return { ok: false, error: `Not found: id=${id}` };
    return { ok: true, data: true };
  }
}

// ── Concrete services — extend base, add domain-specific logic ─────────────

class UserService extends BaseCrudService<User, CreateUserInput, UpdateUserInput> {
  async findByEmail(email: string): Promise<ServiceResult<User>> {
    const users = await this.repo.findAll({ email } as Partial<User>);
    const user = users[0];
    if (!user) return { ok: false, error: `No user with email: ${email}` };
    return { ok: true, data: user };
  }

  async promote(id: number, newRole: User["role"]): Promise<ServiceResult<User>> {
    return this.update(id, { role: newRole });
  }
}

class PostService extends BaseCrudService<Post, CreatePostInput, UpdatePostInput> {
  async publish(id: number): Promise<ServiceResult<Post>> {
    return this.update(id, { published: true });
  }

  async getByAuthor(authorId: number): Promise<ServiceResult<Post[]>> {
    const posts = await this.repo.findAll({ authorId } as Partial<Post>);
    return { ok: true, data: posts };
  }
}

// ── Generic controller helper — handles ServiceResult → HTTP response ──────

function toHttpResponse<T>(
  result: ServiceResult<T>,
  successStatus = 200,
): ApiResponse<T | null> {
  if (result.ok) {
    return successResponse(result.data ?? null, successStatus);
  }
  return { data: null, status: 404, message: result.error ?? "Unknown error", timestamp: new Date().toISOString() };
}

// Usage:
// const userService = new UserService(userRepo);
// const result = await userService.findByEmail("p@dev.io");
// result.data?.name   // ✅ User | undefined — fully typed
// result.error        // ✅ string | undefined
// const httpResponse = toHttpResponse(result);
// httpResponse.data   // ✅ User | null
```

---

## Common mistakes

### Mistake 1 — Forgetting to pass the type argument (getting the default instead)

```ts
interface ApiResponse<TData = unknown> {
  data: TData;
  status: number;
}

// ❌ Forgot the type argument — data is 'unknown', you lose all type safety:
function getUser(): ApiResponse {    // ApiResponse<unknown>
  return { data: { id: 1 }, status: 200 };
}

const res = getUser();
res.data.id;  // ❌ Error: 'id' does not exist on type 'unknown'
res.data.name;  // also unknown — no help from TypeScript

// ✅ Always pass the type argument:
function getUser(): ApiResponse<User> {
  return { data: { id: 1, name: "Parsh", email: "p@dev.io", role: "admin", createdAt: new Date(), updatedAt: new Date() }, status: 200 };
}

const res2 = getUser();
res2.data.name;  // ✅ string
res2.data.id;    // ✅ number
```

### Mistake 2 — Using `any` in the implementation instead of the type parameter

```ts
interface Repository<T extends { id: number }> {
  findById(id: number): Promise<T | null>;
}

// ❌ Using any breaks the contract — TypeScript stops enforcing the type:
class BadRepo<T extends { id: number }> implements Repository<T> {
  async findById(id: number): Promise<any> {   // returns any, not T | null
    return null;
  }
}

const repo = new BadRepo<User>();
const user = await repo.findById(1);
user.nonExistent.deeply.nested;  // No error — any is infectious

// ✅ Return T | null as declared:
class GoodRepo<T extends { id: number }> implements Repository<T> {
  async findById(id: number): Promise<T | null> {
    return null;  // TypeScript is happy — null satisfies T | null
  }
}
```

### Mistake 3 — Re-declaring type params on methods that already have them from the interface

```ts
interface Transformer<TInput, TOutput> {
  transform(input: TInput): TOutput;
}

// ❌ Don't re-declare TInput and TOutput on the method — they come from the class:
class StringToNumber implements Transformer<string, number> {
  transform<TInput, TOutput>(input: TInput): TOutput {  // wrong — these are NEW params
    return Number(input) as unknown as TOutput;          // forced cast — loss of safety
  }
}

// ✅ Method inherits TInput = string, TOutput = number from the class declaration:
class StringToNumber implements Transformer<string, number> {
  transform(input: string): number {  // correct — no extra type params needed
    return Number(input);
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

1. Write a generic interface `Stack<T>` with:
   - `push(item: T): void`
   - `pop(): T | undefined`
   - `peek(): T | undefined`
   - `isEmpty(): boolean`
   - `size(): number`

   Implement it as `class ArrayStack<T> implements Stack<T>`. Test it with `Stack<number>` (push request IDs) and `Stack<string>` (push auth tokens).

2. Write a generic interface `Cache<T>` with `get(key: string): T | undefined`, `set(key: string, value: T): void`, `has(key: string): boolean`, `delete(key: string): void`. Implement it as `class MapCache<T> implements Cache<T>`.

3. Extend `ApiResponse<TData>` into `ApiListResponse<TItem>` where `data` is `{ items: TItem[]; total: number; page: number }`. Write a helper `listResponse<T>(items: T[], total: number, page: number): ApiListResponse<T>`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed event-driven architecture using generic interfaces:

```ts
interface DomainEvent<TPayload> {
  eventId: string;
  eventType: string;
  occurredAt: Date;
  payload: TPayload;
}

interface EventHandler<TEvent extends DomainEvent<unknown>> {
  eventType: string;
  handle(event: TEvent): Promise<void>;
}

interface EventBus {
  publish<TPayload>(event: DomainEvent<TPayload>): Promise<void>;
  subscribe<TEvent extends DomainEvent<unknown>>(handler: EventHandler<TEvent>): void;
}
```

1. Implement `InMemoryEventBus implements EventBus`.
2. Define these domain events (using the generic interface):
   - `UserRegisteredEvent` — payload: `{ userId: number; email: string; role: string }`
   - `OrderPlacedEvent` — payload: `{ orderId: number; userId: number; totalCents: number }`
   - `PaymentFailedEvent` — payload: `{ orderId: number; reason: string }`
3. Write handlers:
   - `WelcomeEmailHandler implements EventHandler<UserRegisteredEvent>` — logs a welcome message
   - `InventoryReserveHandler implements EventHandler<OrderPlacedEvent>` — logs reservation
   - `PaymentRetryHandler implements EventHandler<PaymentFailedEvent>` — logs retry schedule
4. Wire them together and publish all three events. Verify TypeScript catches wrong payload types at compile time.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a full generic CRUD HTTP controller factory:

```ts
interface HttpRequest<TBody = unknown, TParams = Record<string, string>> {
  body: TBody;
  params: TParams;
  query: Record<string, string>;
  headers: Record<string, string>;
}

interface HttpResponse<TData = unknown> {
  status(code: number): this;
  json(data: ApiResponse<TData>): void;
}

interface CrudController<TEntity extends Entity, TCreateInput, TUpdateInput> {
  getOne(req: HttpRequest<never, { id: string }>, res: HttpResponse<TEntity>): Promise<void>;
  getAll(req: HttpRequest, res: HttpResponse<PaginatedData<TEntity>>): Promise<void>;
  createOne(req: HttpRequest<TCreateInput>, res: HttpResponse<TEntity>): Promise<void>;
  updateOne(req: HttpRequest<TUpdateInput, { id: string }>, res: HttpResponse<TEntity>): Promise<void>;
  deleteOne(req: HttpRequest<never, { id: string }>, res: HttpResponse<null>): Promise<void>;
}
```

1. Write `createCrudController<TEntity, TCreateInput, TUpdateInput>(service: CrudService<TEntity, TCreateInput, TUpdateInput>): CrudController<TEntity, TCreateInput, TUpdateInput>` — a factory function that returns an object implementing the controller interface. Each method calls the service and writes the appropriate `ApiResponse` to `res`.

2. Instantiate controllers for `User` (using `UserService`) and `Post` (using `PostService`). TypeScript must correctly type each controller's request and response types.

3. Write a helper `registerRoutes<TEntity, TCreate, TUpdate>(path: string, controller: CrudController<TEntity, TCreate, TUpdate>): void` that logs the 5 CRUD routes for the given path.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Generic interface:
interface Name<T> { value: T; }

// Multiple type params:
interface Pair<TKey, TValue> { key: TKey; value: TValue; }

// Constrained type param:
interface Repo<T extends { id: number }> {
  findById(id: number): Promise<T | null>;
}

// Default type param:
interface ApiResponse<TData = unknown> { data: TData; status: number; }

// Extending a generic interface:
interface PagedRepo<T extends { id: number }> extends Repo<T> {
  findPage(page: number, size: number): Promise<T[]>;
}

// Specializing (instantiating):
type UserRepo = Repo<User>;
type UserResponse = ApiResponse<User>;

// Class implements open generic:
class InMemory<T extends { id: number }> implements Repo<T> { ... }

// Class closes the generic:
class UserRepo implements Repo<User> { ... }
```

| Pattern | Use case |
|---------|---------|
| `ApiResponse<T>` | Standard HTTP response envelope — specialize per endpoint |
| `Repository<T extends Entity>` | Data access layer — one interface, many entity types |
| `ServiceResult<T, E = string>` | Service return value — encapsulates success/failure |
| `EventHandler<TEvent>` | Typed event handler contract |
| `Middleware<TContext>` | Typed middleware pipeline step |
| `PipelineStep<TIn, TOut>` | Transform step with optional rollback |
| `CrudService<T, TCreate, TUpdate>` | Full CRUD contract — decouples controller from impl |

---

## Connected topics

- **26 — What are generics** — foundational concept: type placeholders, inference, `any` vs `T`.
- **27 — Generic functions** — type parameters on functions; constraints, `keyof`, multiple params.
- **29 — Generic utility types** — `Partial<T>`, `Pick<T,K>`, `Omit<T,K>` — generic interfaces from the standard library.
- **19 — Implementing interfaces** — `implements` keyword; how classes fulfill interface contracts.
- **18 — Extending interfaces** — `extends` for interfaces; combining constraints from multiple interfaces.
- **33 — Classes in TypeScript** — generic classes; class type parameters vs method type parameters.
