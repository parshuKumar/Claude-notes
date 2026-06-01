# 29 — Generic Classes

## What is this?

A **generic class** is a class that declares one or more **type parameters** on its class declaration. Those type parameters are available to every property, constructor parameter, and method in the class body — just like a generic interface, but with full class mechanics: constructors, inheritance, `implements`, static vs instance members, and method-level type parameters on top of class-level ones.

Generic classes are the backbone of reusable data structures and service layers: `Map<K, V>`, `Set<T>`, `Promise<T>`, `Array<T>` are all generic classes in TypeScript's standard library. In a backend codebase you write them for repositories, caches, queues, builders, and service wrappers.

## Why does it matter?

Without generic classes you either:
- Write one class per type (`UserRepository`, `PostRepository`, `OrderRepository` — each identical except for the entity type), or
- Use `any` everywhere and lose all type safety.

A generic class lets you write the logic once, then instantiate it for each type. The instance is **locked** to that specific type — TypeScript enforces it at every method call. You get the reusability of a shared base with the safety of a fully typed concrete class.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — class works fine but has no type information:
class Stack {
  #items = [];
  push(item)   { this.#items.push(item); }
  pop()        { return this.#items.pop(); }
  peek()       { return this.#items.at(-1); }
  get size()   { return this.#items.length; }
}

const requestIdStack = new Stack();
requestIdStack.push(42);
requestIdStack.push("wrong"); // silently accepted — no type guard
const top = requestIdStack.peek(); // unknown type — callers must guess
```

```ts
// TypeScript generic class — locked to a specific type at instantiation:
class Stack<T> {
  private items: T[] = [];

  push(item: T): void        { this.items.push(item); }
  pop(): T | undefined       { return this.items.pop(); }
  peek(): T | undefined      { return this.items.at(-1); }
  get size(): number         { return this.items.length; }
  isEmpty(): boolean         { return this.items.length === 0; }
}

const requestIdStack = new Stack<number>();
requestIdStack.push(42);       // ✅
requestIdStack.push("wrong");  // ❌ Error: Argument of type 'string' is not assignable to 'number'

const top = requestIdStack.peek(); // top: number | undefined — fully typed
top?.toFixed(0);               // ✅ TypeScript knows top is number | undefined
```

---

## Syntax

```ts
// ── Basic generic class ───────────────────────────────────────────────────
class Box<T> {
  constructor(public value: T) {}
  getValue(): T { return this.value; }
}

// ── Multiple type parameters ──────────────────────────────────────────────
class Pair<TFirst, TSecond> {
  constructor(
    public first: TFirst,
    public second: TSecond,
  ) {}
  swap(): Pair<TSecond, TFirst> {
    return new Pair(this.second, this.first);
  }
}

// ── Constrained type parameter ────────────────────────────────────────────
class Repository<T extends { id: number }> {
  protected store = new Map<number, T>();
  findById(id: number): T | undefined { return this.store.get(id); }
  save(entity: T): T { this.store.set(entity.id, entity); return entity; }
}

// ── Generic class implementing a generic interface ────────────────────────
interface Queue<T> { enqueue(item: T): void; dequeue(): T | undefined; }
class ArrayQueue<T> implements Queue<T> {
  private items: T[] = [];
  enqueue(item: T): void     { this.items.push(item); }
  dequeue(): T | undefined   { return this.items.shift(); }
}

// ── Closing the generic — concrete subclass ───────────────────────────────
class UserRepository extends Repository<User> {
  findByEmail(email: string): User | undefined {
    return [...this.store.values()].find(u => u.email === email);
  }
}

// ── Method-level type param on top of class-level ─────────────────────────
class EventEmitter<TEventMap extends Record<string, unknown>> {
  // K is a method-level type param; TEventMap is the class-level param:
  emit<K extends keyof TEventMap>(event: K, payload: TEventMap[K]): void { ... }
}

// ── Instantiation ─────────────────────────────────────────────────────────
const numBox   = new Box<number>(42);           // locked to number
const strBox   = new Box("hello");              // T inferred as string
const userRepo = new UserRepository();          // closed — always works with User
```

---

## How it works — concept by concept

### Class-level type parameters are shared by all instance members

```ts
class Container<T> {
  private value: T;      // T from class declaration
  private history: T[] = [];

  constructor(initial: T) {   // constructor receives T
    this.value = initial;
  }

  set(newValue: T): void {    // method accepts T
    this.history.push(this.value);
    this.value = newValue;
  }

  get(): T {                  // method returns T
    return this.value;
  }

  getHistory(): T[] {         // method returns T[]
    return [...this.history];
  }
}

// Once instantiated, T is locked:
const c = new Container<string>("initial");
c.set("second");   // ✅ string accepted
c.set(42);         // ❌ Error: number not assignable to string
const v = c.get(); // v: string — TypeScript knows
```

### Static members cannot use the class type parameter

Static members belong to the class itself, not to any instance. Because `T` is determined per instance, static members cannot reference `T`:

```ts
class Registry<T> {
  private items: T[] = [];

  // ✅ Instance method — can use T:
  add(item: T): void { this.items.push(item); }

  // ❌ Static method — cannot use T:
  static create<T>(): Registry<T> {   // T here is a NEW method-level param, not the class T
    return new Registry<T>();
  }

  // ✅ Static method with its own type param:
  static of<TItem>(item: TItem): Registry<TItem> {
    const reg = new Registry<TItem>();
    reg.add(item);
    return reg;
  }
}

const reg = Registry.of({ id: 1, name: "Parsh" });
// reg: Registry<{ id: number; name: string }>
```

### Constraints on class type parameters

```ts
class Repository<T extends { id: number; createdAt: Date; updatedAt: Date }> {
  protected store = new Map<number, T>();

  save(entity: T): T {
    // Can access .id because of the constraint:
    this.store.set(entity.id, entity);
    return entity;
  }

  findById(id: number): T | undefined {
    return this.store.get(id);
  }

  findAll(): T[] {
    return [...this.store.values()];
  }

  // Can sort by createdAt because of the constraint:
  findAllSortedByDate(): T[] {
    return this.findAll().sort(
      (a, b) => a.createdAt.getTime() - b.createdAt.getTime(),
    );
  }
}
```

### Extending a generic class — open and closed

**Keep it open** — the subclass adds more logic but stays generic:
```ts
class CachingRepository<T extends { id: number; createdAt: Date; updatedAt: Date }>
  extends Repository<T>
{
  private cache = new Map<number, T>();
  private ttlMs: number;

  constructor(ttlMs = 60_000) {
    super();
    this.ttlMs = ttlMs;
  }

  override findById(id: number): T | undefined {
    // Check cache first, fall back to store:
    return this.cache.get(id) ?? super.findById(id);
  }
}

// Still generic — specialized later:
const userCacheRepo = new CachingRepository<User>(30_000);
```

**Close it** — the subclass fixes `T` to a concrete type:
```ts
class UserRepository extends Repository<User> {
  // All inherited methods now work with User specifically.
  // Add domain-specific methods:
  findByEmail(email: string): User | undefined {
    return this.findAll().find(u => u.email === email);
  }

  findByRole(role: User["role"]): User[] {
    return this.findAll().filter(u => u.role === role);
  }
}

const repo = new UserRepository(); // no type arg needed — already closed to User
```

### Method-level type parameters on top of class-level ones

A method can introduce its own type parameters independent of the class type parameter:

```ts
class DataStore<TRecord extends { id: number }> {
  private records: TRecord[] = [];

  add(record: TRecord): this {
    this.records.push(record);
    return this;
  }

  // Class param: TRecord  |  Method param: TResult (new, method-only)
  map<TResult>(fn: (record: TRecord) => TResult): TResult[] {
    return this.records.map(fn);
  }

  // Class param: TRecord  |  Method param: TKey (new, method-only)
  groupBy<TKey extends string | number | symbol>(
    getKey: (record: TRecord) => TKey,
  ): Partial<Record<TKey, TRecord[]>> {
    const result: Partial<Record<TKey, TRecord[]>> = {};
    for (const record of this.records) {
      const key = getKey(record);
      if (!result[key]) result[key] = [];
      result[key]!.push(record);
    }
    return result;
  }
}

const store = new DataStore<User>();
store.add({ id: 1, name: "Parsh", email: "p@dev.io", role: "admin", createdAt: new Date(), updatedAt: new Date() });

const emails = store.map(u => u.email);     // TResult = string → string[]
const byRole = store.groupBy(u => u.role);  // TKey = "admin"|"editor"|"viewer"
```

---

## Example 1 — basic

```ts
// Generic typed in-memory data store with full CRUD + query capabilities

interface Entity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

class InMemoryStore<T extends Entity> {
  private store = new Map<number, T>();
  private nextId = 1;

  // INSERT
  insert(data: Omit<T, "id" | "createdAt" | "updatedAt">): T {
    const entity = {
      ...data,
      id: this.nextId++,
      createdAt: new Date(),
      updatedAt: new Date(),
    } as T;
    this.store.set(entity.id, entity);
    return entity;
  }

  // SELECT
  findById(id: number): T | undefined {
    return this.store.get(id);
  }

  findAll(): T[] {
    return [...this.store.values()];
  }

  findWhere(predicate: (entity: T) => boolean): T[] {
    return this.findAll().filter(predicate);
  }

  findOne(predicate: (entity: T) => boolean): T | undefined {
    return this.findAll().find(predicate);
  }

  // UPDATE
  update(id: number, changes: Partial<Omit<T, "id" | "createdAt">>): T | undefined {
    const existing = this.store.get(id);
    if (!existing) return undefined;
    const updated = { ...existing, ...changes, updatedAt: new Date() } as T;
    this.store.set(id, updated);
    return updated;
  }

  // DELETE
  delete(id: number): boolean {
    return this.store.delete(id);
  }

  // AGGREGATE
  count(predicate?: (entity: T) => boolean): number {
    return predicate ? this.findAll().filter(predicate).length : this.store.size;
  }

  // TRANSFORM — method-level type param:
  pluck<TKey extends keyof T>(key: TKey): T[TKey][] {
    return this.findAll().map(e => e[key]);
  }

  groupBy<TKey extends string | number | symbol>(
    getKey: (entity: T) => TKey,
  ): Partial<Record<TKey, T[]>> {
    const result: Partial<Record<TKey, T[]>> = {};
    for (const entity of this.findAll()) {
      const key = getKey(entity);
      if (!result[key]) result[key] = [];
      result[key]!.push(entity);
    }
    return result;
  }
}

// Domain types:
interface User extends Entity {
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
}

interface Product extends Entity {
  name: string;
  priceCents: number;
  category: "electronics" | "clothing" | "food";
  inStock: boolean;
}

// Two completely independent, fully-typed stores from one class:
const userStore    = new InMemoryStore<User>();
const productStore = new InMemoryStore<Product>();

const user1 = userStore.insert({ name: "Parsh", email: "p@dev.io", role: "admin" });
const user2 = userStore.insert({ name: "Dev",   email: "d@dev.io", role: "editor" });
// user1: User — id, createdAt, updatedAt auto-populated

const admins  = userStore.findWhere(u => u.role === "admin");  // User[]
const emails  = userStore.pluck("email");                      // string[]
const byRole  = userStore.groupBy(u => u.role);                // Partial<Record<"admin"|"editor"|"viewer", User[]>>
const updated = userStore.update(1, { role: "editor" });       // User | undefined
const count   = userStore.count(u => u.role === "admin");      // number

// productStore is completely separate and typed to Product:
productStore.insert({ name: "Laptop", priceCents: 99900, category: "electronics", inStock: true });
const inStock = productStore.findWhere(p => p.inStock);   // Product[]
const prices  = productStore.pluck("priceCents");         // number[]
```

---

## Example 2 — real world backend use case

```ts
// Generic async job queue — processes typed jobs one at a time

interface Job<TPayload> {
  jobId: string;
  payload: TPayload;
  attempts: number;
  maxAttempts: number;
  createdAt: Date;
  scheduledAt: Date;
  status: "pending" | "running" | "completed" | "failed";
}

interface JobResult<TOutput> {
  jobId: string;
  output: TOutput;
  completedAt: Date;
  durationMs: number;
}

class JobQueue<TPayload, TOutput> {
  private pending: Job<TPayload>[] = [];
  private results  = new Map<string, JobResult<TOutput>>();
  private failures = new Map<string, { job: Job<TPayload>; error: string }>();
  private running  = false;

  constructor(
    private readonly processor: (payload: TPayload) => Promise<TOutput>,
    private readonly concurrency = 1,
  ) {}

  enqueue(
    payload: TPayload,
    options: { maxAttempts?: number; delayMs?: number } = {},
  ): string {
    const jobId = `job_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`;
    const job: Job<TPayload> = {
      jobId,
      payload,
      attempts: 0,
      maxAttempts: options.maxAttempts ?? 3,
      createdAt: new Date(),
      scheduledAt: new Date(Date.now() + (options.delayMs ?? 0)),
      status: "pending",
    };
    this.pending.push(job);
    return jobId;
  }

  async drain(): Promise<void> {
    if (this.running) return;
    this.running = true;

    while (this.pending.length > 0) {
      const batch = this.pending.splice(0, this.concurrency);
      await Promise.all(batch.map(job => this.processJob(job)));
    }

    this.running = false;
  }

  private async processJob(job: Job<TPayload>): Promise<void> {
    const now = Date.now();
    if (job.scheduledAt.getTime() > now) {
      this.pending.push(job);  // re-queue if not yet due
      return;
    }

    job.status = "running";
    job.attempts++;

    try {
      const startMs = Date.now();
      const output = await this.processor(job.payload);  // output: TOutput
      this.results.set(job.jobId, {
        jobId: job.jobId,
        output,
        completedAt: new Date(),
        durationMs: Date.now() - startMs,
      });
      job.status = "completed";
    } catch (err) {
      const shouldRetry = job.attempts < job.maxAttempts;
      if (shouldRetry) {
        job.status = "pending";
        this.pending.push(job);
      } else {
        job.status = "failed";
        this.failures.set(job.jobId, { job, error: String(err) });
      }
    }
  }

  getResult(jobId: string): JobResult<TOutput> | undefined {
    return this.results.get(jobId);  // JobResult<TOutput> — TOutput fully typed
  }

  getFailure(jobId: string): { job: Job<TPayload>; error: string } | undefined {
    return this.failures.get(jobId);
  }

  stats(): { pending: number; completed: number; failed: number } {
    return {
      pending:   this.pending.length,
      completed: this.results.size,
      failed:    this.failures.size,
    };
  }
}

// ── Email job queue ────────────────────────────────────────────────────────

interface EmailPayload {
  to: string;
  subject: string;
  templateId: string;
  variables: Record<string, string>;
}

interface EmailSendResult {
  messageId: string;
  provider: "sendgrid" | "ses";
  accepted: boolean;
}

async function sendEmail(payload: EmailPayload): Promise<EmailSendResult> {
  // In production: call email provider SDK
  console.log(`Sending "${payload.subject}" to ${payload.to}`);
  return { messageId: `msg_${Date.now()}`, provider: "sendgrid", accepted: true };
}

const emailQueue = new JobQueue<EmailPayload, EmailSendResult>(sendEmail, 5);

const jobId = emailQueue.enqueue({
  to: "p@dev.io",
  subject: "Welcome to the platform",
  templateId: "welcome-v2",
  variables: { userName: "Parsh", activationLink: "https://app.dev.io/activate/abc123" },
});

await emailQueue.drain();

const result = emailQueue.getResult(jobId);
// result: JobResult<EmailSendResult> | undefined
result?.output.messageId;   // ✅ string
result?.output.provider;    // ✅ "sendgrid" | "ses"
result?.output.accepted;    // ✅ boolean
result?.durationMs;         // ✅ number

// ── Image processing queue — completely different types, same class ─────────

interface ImageProcessingPayload {
  sourceUrl: string;
  operations: Array<{ type: "resize" | "crop" | "compress"; params: Record<string, number> }>;
  outputFormat: "webp" | "jpg" | "png";
}

interface ImageProcessingResult {
  outputUrl: string;
  originalBytes: number;
  outputBytes: number;
  processingMs: number;
}

async function processImage(payload: ImageProcessingPayload): Promise<ImageProcessingResult> {
  // In production: call image processing service
  return { outputUrl: "https://cdn.dev.io/img.webp", originalBytes: 500_000, outputBytes: 120_000, processingMs: 340 };
}

const imageQueue = new JobQueue<ImageProcessingPayload, ImageProcessingResult>(processImage, 2);
// Completely independent from emailQueue — different payload, different result, same class
```

---

## Common mistakes

### Mistake 1 — Annotating the variable with the open generic type

```ts
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
}

// ❌ Using the open generic as the variable type loses the specific T:
let stack: Stack<any> = new Stack<number>();
stack.push("oops");    // No error — Stack<any> accepts anything

// ❌ Assigning wrong specialization:
let stack2: Stack<string> = new Stack<string>();
stack2 = new Stack<number>();  // Error — but this reveals a misuse: you mixed types

// ✅ Let TypeScript infer the type from the constructor:
const stack3 = new Stack<number>();
// stack3: Stack<number> — T is locked to number
stack3.push(42);      // ✅
stack3.push("wrong"); // ❌ Error caught by TypeScript
```

### Mistake 2 — Trying to use `T` in a static method

```ts
class Container<T> {
  private value: T;
  constructor(value: T) { this.value = value; }

  // ❌ Cannot use class T in a static method:
  static empty(): Container<T> {
    // Error: Static members cannot reference class type parameters
    return new Container<T>(undefined as T);
  }

  // ✅ Static method needs its own type parameter:
  static empty<TItem>(): Container<TItem | undefined> {
    return new Container<TItem | undefined>(undefined);
  }

  // ✅ Or hard-code a specific type:
  static emptyString(): Container<string> {
    return new Container<string>("");
  }
}
```

### Mistake 3 — Forgetting the constraint and falling back to `as` casts

```ts
// ❌ No constraint — T is unknown, so you cast everything:
class Sorter<T> {
  sort(items: T[]): T[] {
    return items.sort((a, b) => (a as any) - (b as any));  // unsafe
  }
}

// ✅ Add a constraint — TypeScript knows T is comparable:
class NumericSorter<T extends number | bigint> {
  sort(items: T[]): T[] {
    return [...items].sort((a, b) => Number(a) - Number(b));
  }
}

// ✅ For string comparison:
class StringSorter<T extends string> {
  sort(items: T[]): T[] {
    return [...items].sort((a, b) => a.localeCompare(b));
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

1. Write a generic `Queue<T>` class with:
   - `enqueue(item: T): void`
   - `dequeue(): T | undefined`
   - `peek(): T | undefined`
   - `size: number` (getter)
   - `isEmpty(): boolean`
   - `toArray(): T[]`

   Test it with `Queue<number>` (request IDs) and `Queue<string>` (job names).

2. Write a generic `Result<TValue, TError = Error>` class (not interface — a class) with:
   - Static factory: `Result.ok<T>(value: T): Result<T, never>`
   - Static factory: `Result.fail<E>(error: E): Result<never, E>`
   - Instance methods: `isOk(): boolean`, `isFail(): boolean`
   - `getValue(): TValue` — throws if failed
   - `getError(): TError` — throws if ok
   - `map<TResult>(fn: (value: TValue) => TResult): Result<TResult, TError>`

3. Instantiate `Result.ok({ userId: 1, email: "p@dev.io" })` and `Result.fail(new Error("Not found"))`. Verify the types are correctly narrowed.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a generic `TypedMap<TKey extends string, TValue>` class that wraps `Map<TKey, TValue>` and adds:

```ts
class TypedMap<TKey extends string, TValue> {
  // All standard Map operations — get, set, has, delete, clear, size
  get(key: TKey): TValue | undefined
  set(key: TKey, value: TValue): this
  has(key: TKey): boolean
  delete(key: TKey): boolean
  clear(): void
  get size(): number

  // Extra typed utilities:
  getOrDefault(key: TKey, defaultValue: TValue): TValue
  getOrThrow(key: TKey, errorMessage?: string): TValue
  setMany(entries: Array<[TKey, TValue]>): this
  toRecord(): Record<TKey, TValue>
  keys(): TKey[]
  values(): TValue[]
  entries(): Array<[TKey, TValue]>
  map<TResult>(fn: (value: TValue, key: TKey) => TResult): TypedMap<TKey, TResult>
  filter(predicate: (value: TValue, key: TKey) => boolean): TypedMap<TKey, TValue>
}
```

Use it:
```ts
type ConfigKey = "timeout" | "retries" | "baseUrl" | "apiVersion";
const config = new TypedMap<ConfigKey, string | number>();

config
  .set("timeout", 5000)
  .set("retries", 3)
  .set("baseUrl", "https://api.dev.io")
  .set("apiVersion", "v2");

config.getOrThrow("baseUrl");           // string | number, no undefined
config.getOrDefault("timeout", 3000);  // string | number
config.toRecord();                     // Record<ConfigKey, string | number>
config.keys();                         // ConfigKey[]
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a generic `Pipeline<TContext>` class that chains async middleware-style handlers:

```ts
type Handler<TContext> = (
  ctx: TContext,
  next: () => Promise<void>,
) => Promise<void>;

class Pipeline<TContext> {
  // Add a handler to the pipeline:
  use(handler: Handler<TContext>): this

  // Add a handler that only runs if a condition is met:
  useIf(condition: (ctx: TContext) => boolean, handler: Handler<TContext>): this

  // Add a handler that only runs for specific 'method' values (if TContext has method):
  useFor<TKey extends keyof TContext>(
    key: TKey,
    value: TContext[TKey],
    handler: Handler<TContext>,
  ): this

  // Execute the pipeline with a context object:
  execute(ctx: TContext): Promise<void>

  // Branch — create a sub-pipeline and merge back:
  branch(
    condition: (ctx: TContext) => boolean,
    trueBranch: (pipeline: Pipeline<TContext>) => void,
    falseBranch?: (pipeline: Pipeline<TContext>) => void,
  ): this
}
```

Then define a typed request context and build a full request-processing pipeline:

```ts
interface RequestContext {
  requestId: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  path: string;
  headers: Record<string, string>;
  body: unknown;
  userId?: number;
  startTime: number;
  response: {
    statusCode: number;
    body: unknown;
    headers: Record<string, string>;
  };
  errors: string[];
}

const pipeline = new Pipeline<RequestContext>()
  .use(async (ctx, next) => {
    console.log(`[${ctx.requestId}] ${ctx.method} ${ctx.path}`);
    await next();
    console.log(`[${ctx.requestId}] completed in ${Date.now() - ctx.startTime}ms`);
  })
  .useFor("method", "POST", async (ctx, next) => {
    // Only runs for POST requests — validate body
    if (!ctx.body) ctx.errors.push("Body required for POST");
    await next();
  })
  .use(async (ctx, next) => {
    // Auth middleware
    const token = ctx.headers["authorization"];
    if (token) ctx.userId = 1;  // stub: parse JWT
    await next();
  })
  .branch(
    ctx => ctx.path.startsWith("/admin"),
    adminPipeline => adminPipeline.use(async (ctx, next) => {
      if (!ctx.userId) { ctx.response.statusCode = 401; return; }
      await next();
    }),
  );

const ctx: RequestContext = {
  requestId: "req_abc123",
  method: "POST",
  path: "/admin/users",
  headers: { authorization: "Bearer token123" },
  body: { name: "Parsh" },
  startTime: Date.now(),
  response: { statusCode: 200, body: null, headers: {} },
  errors: [],
};

await pipeline.execute(ctx);
console.log(ctx.response.statusCode);  // typed — number
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Generic class:
class Name<T> { constructor(public value: T) {} }

// Constrained:
class Repo<T extends { id: number }> { store = new Map<number, T>(); }

// Multiple params:
class Pair<TFirst, TSecond> { constructor(public first: TFirst, public second: TSecond) {} }

// Method-level param (extra param beyond class-level):
class Store<T> {
  map<TResult>(fn: (item: T) => TResult): TResult[] { return []; }
}

// Open extension (still generic):
class ExtendedRepo<T extends { id: number }> extends Repo<T> { ... }

// Closed extension (fixes T to a concrete type):
class UserRepo extends Repo<User> { ... }

// Implements a generic interface:
class ArrayStack<T> implements Stack<T> { ... }

// Instantiation:
const box = new Box<number>(42);        // explicit T
const box2 = new Box("hello");          // T inferred as string
```

| Pattern | When to use |
|---------|------------|
| Open generic class | Reusable data structure / infrastructure (stores, queues, caches) |
| Closed (concrete) subclass | Domain-specific class built on a generic base (`UserRepository`) |
| Constrained `<T extends X>` | When methods need to access properties of `T` |
| Method-level `<TResult>` | When a method transforms `T` into a different type |
| Static factory with own `<T>` | When static creation methods need type flexibility |
| `implements Interface<T>` | When the class must fulfill a typed contract |

---

## Connected topics

- **28 — Generic interfaces** — generic interfaces that classes implement; `Repository<T>`, `EventHandler<T>`.
- **27 — Generic functions** — method-level type parameters use the same rules as generic functions.
- **19 — Implementing interfaces** — `implements` keyword; class-interface contracts.
- **33 — Classes in TypeScript** — class fundamentals: access modifiers, `readonly`, constructors, inheritance.
- **34 — Abstract classes** — combining abstract methods with generic type parameters.
- **29 → 30 — Generic constraints** — deep dive into `extends`, `keyof`, conditional constraints.
