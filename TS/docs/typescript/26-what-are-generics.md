# 26 — What Are Generics

## What is this?

Generics are TypeScript's way of writing **type-safe code that works with multiple types** without duplicating logic. Instead of hardcoding a specific type like `number` or `User`, you write a **type placeholder** — typically named `T` — that TypeScript fills in with the actual type at the call site.

Think of generics like a function parameter, but for types. Just as a function parameter lets one function work with many *values*, a type parameter lets one function/class/interface work with many *types* — without losing type safety.

## Why does it matter?

Without generics, you have two bad options:

1. **Use `any`** — everything works but you lose all type safety. TypeScript becomes pointless.
2. **Duplicate code** — write `getFirstNumber`, `getFirstString`, `getFirstUser`, etc. for every type.

Generics give you a third option: write the logic once, let TypeScript infer and check the types automatically. This is the backbone of TypeScript's standard library — `Array<T>`, `Promise<T>`, `Map<K, V>`, `Record<K, V>`, `Set<T>` are all generic.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — one function, works for any type:
function getFirst(arr) {
  return arr[0];
}

const first = getFirst([1, 2, 3]);   // first could be anything — no type info
const name  = getFirst(["Parsh"]);   // same — no safety
```

```ts
// TypeScript WITHOUT generics — forced to choose one type or use any:
function getFirstNumber(arr: number[]): number  { return arr[0]; }
function getFirstString(arr: string[]): string  { return arr[0]; }
// ...duplicated forever ❌

// TypeScript WITH generics — one function, full type safety:
function getFirst<T>(arr: T[]): T {
  return arr[0];
}

const first = getFirst([1, 2, 3]);        // T inferred as number  → first: number
const name  = getFirst(["Parsh", "Dev"]); // T inferred as string  → name: string
const user  = getFirst([{ id: 1 }]);      // T inferred as { id: number } → user: { id: number }

name.toUpperCase();  // ✅ TypeScript knows name is string
first.toFixed(2);    // ✅ TypeScript knows first is number
user.id;             // ✅ TypeScript knows user has .id
```

---

## Syntax

```ts
// ── Generic function ──────────────────────────────────────────────────────
function identity<T>(value: T): T {
  return value;
}

// ── Multiple type parameters ──────────────────────────────────────────────
function pair<TFirst, TSecond>(a: TFirst, b: TSecond): [TFirst, TSecond] {
  return [a, b];
}

// ── Generic arrow function ────────────────────────────────────────────────
const wrap = <T>(value: T): { data: T } => ({ data: value });
// Note: In .tsx files, use <T,> or <T extends unknown> to avoid JSX ambiguity

// ── Explicit type argument (when inference fails) ─────────────────────────
const result = identity<string>("hello");  // T = string (explicit)
const result2 = identity("hello");         // T = string (inferred — prefer this)

// ── Generic interface ─────────────────────────────────────────────────────
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

// ── Generic type alias ────────────────────────────────────────────────────
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type Pair<TFirst, TSecond> = { first: TFirst; second: TSecond };

// ── Generic class ─────────────────────────────────────────────────────────
class Repository<T> {
  private items: T[] = [];
  add(item: T): void { this.items.push(item); }
  getAll(): T[] { return this.items; }
}

// ── Constraint: T must extend a shape ────────────────────────────────────
function getById<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);
}
```

---

## How it works — concept by concept

### Type parameters are placeholders filled at call time

```ts
function wrap<T>(value: T): { data: T } {
  return { data: value };
}

// Each call fills in T differently:
wrap(42);           // T = number  → { data: number }
wrap("admin");      // T = string  → { data: string }
wrap({ id: 1 });    // T = { id: number } → { data: { id: number } }
```

TypeScript infers `T` from the argument. You don't need to write `wrap<number>(42)` — TypeScript figures it out. Only write the explicit type argument when inference fails or you want to enforce a specific type.

### The `T` convention — what to name type parameters

`T` is the default single-letter convention, but meaningful names are better when there are multiple:

| Convention | Meaning |
|-----------|---------|
| `T` | General "Type" — single type param |
| `TData`, `TResult` | Descriptive names — preferred for clarity |
| `K` | Key (especially in key-value pairs) |
| `V` | Value |
| `E` | Error or Element |
| `TInput`, `TOutput` | Function transformation input/output |

```ts
// Single T — fine for simple cases:
function first<T>(arr: T[]): T | undefined { return arr[0]; }

// Descriptive names — better for complex generics:
function transform<TInput, TOutput>(
  items: TInput[],
  fn: (item: TInput) => TOutput,
): TOutput[] {
  return items.map(fn);
}
```

### Type inference — TypeScript fills T automatically

```ts
// TypeScript reads the argument type to infer T:
function echo<T>(value: T): T { return value; }

echo(42);           // TypeScript sees 42: number → T = number
echo("hello");      // T = string
echo([1, 2, 3]);    // T = number[]
echo({ id: 1 });    // T = { id: number }

// For return types — TypeScript propagates T through:
const result = echo({ id: 1, role: "admin" });
// result: { id: number; role: string }  — full type preserved!
```

### Generic `any` is NOT the same as `any`

```ts
// any — completely loses type information:
function badWrap(value: any): any {
  return { data: value };
}
const r1 = badWrap(42);
r1.data.toFixed(2);   // No error, but r1.data is any — TypeScript can't help you
r1.data.nonExistent;  // Also no error — dangerous!

// Generic T — preserves type information:
function goodWrap<T>(value: T): { data: T } {
  return { data: value };
}
const r2 = goodWrap(42);
r2.data.toFixed(2);   // ✅ r2.data is number
r2.data.nonExistent;  // ❌ Error: Property 'nonExistent' does not exist on type 'number'
```

### Constraints: `T extends SomeType`

Without constraints, TypeScript treats `T` as completely unknown — you can't access any properties:

```ts
function logId<T>(item: T): void {
  console.log(item.id);  // ❌ Error: Property 'id' does not exist on type 'T'
}

// ✅ Constrain T to have the required shape:
function logId<T extends { id: number }>(item: T): void {
  console.log(item.id);  // ✅ TypeScript knows T has .id
}

// T must still be the full type — constraints only ADD requirements, not strip properties:
logId({ id: 1, name: "Parsh", role: "admin" });  // ✅ T = { id, name, role }
// The full object type is preserved in the return type and inference
```

### Multiple type parameters

```ts
function mapRecord<TKey extends string | number | symbol, TValue, TResult>(
  record: Record<TKey, TValue>,
  fn: (value: TValue, key: TKey) => TResult,
): Record<TKey, TResult> {
  const result = {} as Record<TKey, TResult>;
  for (const key in record) {
    result[key as TKey] = fn(record[key as TKey], key as TKey);
  }
  return result;
}

const prices = { apple: 1.5, banana: 0.75, mango: 2.0 };
const doubled = mapRecord(prices, v => v * 2);
// doubled: Record<string, number> — { apple: 3, banana: 1.5, mango: 4 }
```

### Generics in interfaces and type aliases

```ts
// Generic interface — used everywhere for API responses:
interface ApiResponse<TData> {
  data: TData;
  status: number;
  message: string;
  timestamp: string;
}

// Now specialize it for different endpoints:
type UserListResponse  = ApiResponse<User[]>;
type UserResponse      = ApiResponse<User>;
type AuthTokenResponse = ApiResponse<{ accessToken: string; expiresIn: number }>;

// A response handler that works for any data shape:
function handleResponse<TData>(
  response: ApiResponse<TData>,
  onSuccess: (data: TData) => void,
  onError: (message: string) => void,
): void {
  if (response.status >= 200 && response.status < 300) {
    onSuccess(response.data);
  } else {
    onError(response.message);
  }
}
```

---

## Example 1 — basic

```ts
// Generic utility functions — the building blocks

// 1. Safe array access with default:
function getOrDefault<T>(arr: T[], index: number, defaultValue: T): T {
  return index >= 0 && index < arr.length ? arr[index] : defaultValue;
}

const userIds = [101, 202, 303];
const firstId   = getOrDefault(userIds, 0, -1);   // 101 — T = number
const missingId = getOrDefault(userIds, 99, -1);  // -1  — T = number, default used

// 2. Wrap a value in a standard result container:
interface Result<TValue, TError = Error> {
  ok: boolean;
  value?: TValue;
  error?: TError;
}

function ok<T>(value: T): Result<T> {
  return { ok: true, value };
}

function fail<T, E = Error>(error: E): Result<T, E> {
  return { ok: false, error };
}

const successResult = ok({ id: 1, name: "Parsh" });
// successResult: Result<{ id: number; name: string }>

const failResult = fail<User>(new Error("Not found"));
// failResult: Result<User, Error>

// 3. Pick the first truthy value from a list:
function firstTruthy<T>(...values: Array<T | null | undefined>): T | undefined {
  return values.find((v): v is T => v !== null && v !== undefined);
}

const authToken = firstTruthy<string>(null, undefined, "Bearer xyz");
// authToken: string | undefined → "Bearer xyz"

// 4. Group an array by a key:
function groupBy<TItem, TKey extends string | number | symbol>(
  items: TItem[],
  getKey: (item: TItem) => TKey,
): Partial<Record<TKey, TItem[]>> {
  const result: Partial<Record<TKey, TItem[]>> = {};
  for (const item of items) {
    const key = getKey(item);
    if (!result[key]) result[key] = [];
    result[key]!.push(item);
  }
  return result;
}

interface LogEntry { level: "info" | "warn" | "error"; message: string; timestamp: Date; }
const logs: LogEntry[] = [
  { level: "info",  message: "Server started",  timestamp: new Date() },
  { level: "error", message: "DB connection lost", timestamp: new Date() },
  { level: "info",  message: "Request received", timestamp: new Date() },
];

const byLevel = groupBy(logs, log => log.level);
// byLevel: Partial<Record<"info"|"warn"|"error", LogEntry[]>>
// byLevel.info  → LogEntry[]
// byLevel.error → LogEntry[]
byLevel.info?.[0].message;  // ✅ fully typed
```

---

## Example 2 — real world backend use case

```ts
// Generic repository pattern — write once, use for every database entity

interface Entity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

interface Repository<TEntity extends Entity, TCreateInput, TUpdateInput> {
  findById(id: number): Promise<TEntity | null>;
  findAll(options?: { limit?: number; offset?: number }): Promise<TEntity[]>;
  create(input: TCreateInput): Promise<TEntity>;
  update(id: number, input: TUpdateInput): Promise<TEntity | null>;
  delete(id: number): Promise<boolean>;
}

// Domain models:
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

// Input types (no id/createdAt/updatedAt — those are set by the DB):
type CreateUserInput  = Pick<User, "name" | "email" | "role">;
type UpdateUserInput  = Partial<Pick<User, "name" | "email" | "role">>;
type CreatePostInput  = Pick<Post, "title" | "body" | "authorId">;
type UpdatePostInput  = Partial<Pick<Post, "title" | "body" | "published">>;

// Generic base implementation:
class InMemoryRepository<TEntity extends Entity, TCreate, TUpdate>
  implements Repository<TEntity, TCreate, TUpdate>
{
  protected store = new Map<number, TEntity>();
  private nextId = 1;

  async findById(id: number): Promise<TEntity | null> {
    return this.store.get(id) ?? null;
  }

  async findAll(options: { limit?: number; offset?: number } = {}): Promise<TEntity[]> {
    const all = Array.from(this.store.values());
    const start = options.offset ?? 0;
    const end = options.limit ? start + options.limit : undefined;
    return all.slice(start, end);
  }

  async create(input: TCreate): Promise<TEntity> {
    const entity = {
      ...input,
      id: this.nextId++,
      createdAt: new Date(),
      updatedAt: new Date(),
    } as TEntity;
    this.store.set(entity.id, entity);
    return entity;
  }

  async update(id: number, input: TUpdate): Promise<TEntity | null> {
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

// Concrete repositories — just specify the type params, zero new logic:
class UserRepository extends InMemoryRepository<User, CreateUserInput, UpdateUserInput> {
  // Domain-specific queries:
  async findByEmail(email: string): Promise<User | null> {
    for (const user of this.store.values()) {
      if (user.email === email) return user;
    }
    return null;
  }

  async findByRole(role: User["role"]): Promise<User[]> {
    return Array.from(this.store.values()).filter(u => u.role === role);
  }
}

class PostRepository extends InMemoryRepository<Post, CreatePostInput, UpdatePostInput> {
  async findByAuthor(authorId: number): Promise<Post[]> {
    return Array.from(this.store.values()).filter(p => p.authorId === authorId);
  }

  async findPublished(): Promise<Post[]> {
    return Array.from(this.store.values()).filter(p => p.published);
  }
}

// Usage — fully typed throughout:
const userRepo = new UserRepository();
const postRepo = new PostRepository();

const user = await userRepo.create({ name: "Parsh", email: "p@dev.io", role: "admin" });
// user: User — id, createdAt, updatedAt auto-added

const post = await postRepo.create({ title: "TS Generics", body: "...", authorId: user.id });
// post: Post

const found = await userRepo.findById(1);
// found: User | null

const admins = await userRepo.findByRole("admin");
// admins: User[]

// Generic service layer on top:
async function findOrFail<TEntity extends Entity, TCreate, TUpdate>(
  repo: Repository<TEntity, TCreate, TUpdate>,
  id: number,
): Promise<TEntity> {
  const entity = await repo.findById(id);
  if (!entity) throw new Error(`Entity with id ${id} not found`);
  return entity;
}

const foundUser = await findOrFail(userRepo, 1);
// foundUser: User — TypeScript knows the exact type
```

---

## Common mistakes

### Mistake 1 — Using `any` instead of a generic

```ts
// ❌ any — type information is lost at the call site:
function firstAny(arr: any[]): any {
  return arr[0];
}

const result = firstAny([1, 2, 3]);
result.toFixed(2);    // No error — but if arr is strings, this crashes at runtime
result.nonExistent;  // Also no error — dangerous

// ✅ Generic — type information flows through:
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const result2 = first([1, 2, 3]);
result2?.toFixed(2);    // ✅ TypeScript knows result2: number | undefined
result2?.nonExistent;  // ❌ Error: Property 'nonExistent' does not exist on type 'number'
```

### Mistake 2 — Unnecessary explicit type arguments

```ts
// ❌ Redundant — TypeScript infers T from the argument:
const wrapped = wrap<string>("hello");
const items   = first<number>([1, 2, 3]);

// ✅ Let TypeScript infer — cleaner and equivalent:
const wrapped = wrap("hello");
const items   = first([1, 2, 3]);

// Only use explicit type arguments when inference fails:
const empty = first<User>([]);
// ✅ Here it's needed — TypeScript can't infer T from an empty array
// Without <User>: first<unknown>([]) → result: unknown | undefined
```

### Mistake 3 — Trying to use properties of T without a constraint

```ts
// ❌ T is unknown — TypeScript can't access any properties:
function logUserId<T>(entity: T): void {
  console.log(entity.id);
  // Error: Property 'id' does not exist on type 'T'
}

// ❌ Workaround with any — defeats the purpose:
function logUserId<T>(entity: T): void {
  console.log((entity as any).id);  // unsafe
}

// ✅ Constrain T to the required shape:
function logUserId<T extends { id: number }>(entity: T): void {
  console.log(entity.id);  // ✅ TypeScript knows T has .id
}

// Even better — name the constraint:
interface HasId { id: number; }
function logUserId<T extends HasId>(entity: T): void {
  console.log(entity.id);  // ✅
}
// T still preserves the full type — only adds the constraint, doesn't strip properties
```

---

## Practice exercises

### Exercise 1 — easy

1. Write a generic function `last<T>(arr: T[]): T | undefined` that returns the last element of an array. Test it with `number[]`, `string[]`, and an array of objects `{ userId: number }[]`.

2. Write a generic function `toMap<T extends { id: number }>(items: T[]): Map<number, T>` that converts an array of entities into a `Map` keyed by their `id`.

3. Write a generic type alias `Paginated<T>` that represents:
   ```ts
   { items: T[]; total: number; page: number; pageSize: number; hasNext: boolean }
   ```
   Then write a function `paginateArray<T>(items: T[], page: number, pageSize: number): Paginated<T>` that implements it.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a generic `Cache<T>` class with a TTL (time-to-live) expiry system:

```ts
class Cache<T> {
  constructor(private defaultTtlMs: number) {}

  set(key: string, value: T, ttlMs?: number): void
  get(key: string): T | undefined            // returns undefined if missing OR expired
  has(key: string): boolean                  // returns false if expired
  delete(key: string): boolean
  clear(): void
  size(): number                             // only counts non-expired entries
}
```

Requirements:
- Each entry stores the value + its expiry timestamp
- `get` must check expiry and return `undefined` for expired entries (also delete them)
- `has` must return `false` for expired entries
- `size()` must only count non-expired entries

Then use the cache:
```ts
const userCache = new Cache<User>(5 * 60 * 1000);    // 5 min default TTL
const tokenCache = new Cache<string>(15 * 60 * 1000); // 15 min default TTL

userCache.set("user:1", { id: 1, name: "Parsh", email: "p@dev.io", role: "admin", createdAt: new Date(), updatedAt: new Date() });
const cachedUser = userCache.get("user:1");  // User | undefined — typed!
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a type-safe generic event bus:

```ts
// Step 1: Define the event map interface:
interface AppEventMap {
  "user.registered":  { userId: number; email: string; role: string };
  "user.loggedIn":    { userId: number; ipAddress: string; timestamp: Date };
  "user.loggedOut":   { userId: number };
  "order.created":    { orderId: number; userId: number; totalCents: number };
  "order.fulfilled":  { orderId: number; shippedAt: Date };
  "payment.received": { orderId: number; amountCents: number; provider: string };
}

// Step 2: Build EventBus<TEventMap> class:
class EventBus<TEventMap extends Record<string, unknown>> {
  subscribe<K extends keyof TEventMap>(
    event: K,
    handler: (payload: TEventMap[K]) => void,
  ): () => void   // returns an unsubscribe function

  publish<K extends keyof TEventMap>(
    event: K,
    payload: TEventMap[K],
  ): void

  subscribeOnce<K extends keyof TEventMap>(
    event: K,
    handler: (payload: TEventMap[K]) => void,
  ): () => void   // auto-unsubscribes after first call

  listenerCount(event: keyof TEventMap): number
}

// Step 3: Use it:
const bus = new EventBus<AppEventMap>();

const unsubscribe = bus.subscribe("user.registered", ({ userId, email, role }) => {
  // userId: number, email: string, role: string — fully typed
  console.log(`Welcome ${email}!`);
});

bus.publish("user.registered", { userId: 1, email: "p@dev.io", role: "admin" });
// ❌ bus.publish("user.registered", { userId: 1 }) — TypeScript should error (missing email, role)
// ❌ bus.publish("nonexistent.event", {}) — TypeScript should error (unknown event)

bus.subscribeOnce("order.created", ({ orderId, totalCents }) => {
  console.log(`First order: #${orderId} for $${(totalCents / 100).toFixed(2)}`);
});

unsubscribe(); // unsubscribe from user.registered
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Generic function:
function fn<T>(arg: T): T { return arg; }

// Generic arrow function:
const fn = <T>(arg: T): T => arg;
// In .tsx — avoid JSX ambiguity:
const fn = <T,>(arg: T): T => arg;

// Generic interface:
interface Container<T> { value: T; }

// Generic type alias:
type Nullable<T> = T | null;

// Generic class:
class Box<T> { constructor(public value: T) {} }

// Constraint:
function fn<T extends { id: number }>(item: T): T { return item; }

// Multiple type params:
function fn<TKey, TValue>(key: TKey, value: TValue): [TKey, TValue] {
  return [key, value];
}

// Default type parameter:
interface Result<T = unknown> { data: T; status: number; }

// Explicit type argument (when inference fails):
const result = fn<User>(someUnknown as User);
```

| Concept | Rule |
|---------|------|
| `T` in function | Inferred from arguments — explicit usually not needed |
| `T extends X` | Constraint — T must have shape X, but still preserves full type |
| Multiple params | Use descriptive names: `TInput`, `TOutput`, `TKey`, `TValue` |
| `any` vs `T` | `any` loses type info. `T` preserves it — always prefer generics |
| Default type param | `<T = string>` — used when T can be omitted |
| Generic classes | Same `<T>` on the class name, available to all methods |

---

## Connected topics

- **27 — Generic constraints** — `T extends X`, `keyof`, `T extends keyof U` — restricting what T can be.
- **28 — Generic interfaces and classes** — applying generics to interfaces, class implementations, and constructor signatures.
- **29 — Generic utility types** — `Partial<T>`, `Required<T>`, `Pick<T,K>`, `Omit<T,K>` — all built on generics.
- **32 — Utility types deep dive** — `ReturnType<T>`, `Parameters<T>`, `Awaited<T>`, `ThisParameterType<T>`.
- **40 — Conditional types** — `T extends U ? X : Y` — generics combined with conditional logic.
- **24 — Higher-order function types** — generic function signatures for callbacks and factories.
