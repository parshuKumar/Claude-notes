# 27 — Generic Functions

## What is this?

A **generic function** is a function that declares one or more **type parameters** — placeholders that TypeScript fills in with real types at the call site. You saw the concept introduced in topic 26. This topic goes deep on the mechanics: how multiple type parameters interact, how constraints narrow what `T` can be, how `keyof` works inside constraints, how TypeScript infers type arguments from complex argument shapes, and when to use multiple type parameters vs a single one.

Generic functions are where most day-to-day TypeScript generic usage happens — utility functions, data-transformation pipelines, typed wrappers, and safe accessor helpers all rely on the patterns in this topic.

## Why does it matter?

A generic function is the tool you reach for whenever you write the same logic for multiple types. Without it:

- You use `any` and lose type safety.
- You write overloads (one per type) and maintain duplicated code.
- You lose return type inference — callers have to cast manually.

Generic functions solve all three problems in one shot. They also compose: a generic function calling another generic function propagates type information automatically — which is how TypeScript's standard library works internally.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — one function, works for anything, but no type info:
function pick(obj, keys) {
  const result = {};
  for (const key of keys) result[key] = obj[key];
  return result;
}

const subset = pick({ id: 1, name: "Parsh", role: "admin" }, ["id", "name"]);
// subset — no type info, TypeScript can't help with what properties exist
```

```ts
// TypeScript generic version — typed and safe:
function pick<TObj, TKey extends keyof TObj>(
  obj: TObj,
  keys: TKey[],
): Pick<TObj, TKey> {
  const result = {} as Pick<TObj, TKey>;
  for (const key of keys) result[key] = obj[key];
  return result;
}

const subset = pick({ id: 1, name: "Parsh", role: "admin" }, ["id", "name"]);
// subset: { id: number; name: string }  — TypeScript knows exactly what's in it!

subset.id;    // ✅ number
subset.name;  // ✅ string
subset.role;  // ❌ Error: Property 'role' does not exist — it wasn't in the keys array
```

---

## Syntax

```ts
// ── Single type param ─────────────────────────────────────────────────────
function identity<T>(value: T): T { return value; }

// ── Multiple type params ──────────────────────────────────────────────────
function merge<TFirst, TSecond>(a: TFirst, b: TSecond): TFirst & TSecond {
  return { ...a as object, ...b as object } as TFirst & TSecond;
}

// ── Constraint: T must extend a shape ────────────────────────────────────
function getById<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);
}

// ── keyof constraint: TKey must be a key of TObj ─────────────────────────
function getField<TObj, TKey extends keyof TObj>(obj: TObj, key: TKey): TObj[TKey] {
  return obj[key];
}

// ── Default type parameter ────────────────────────────────────────────────
function createResponse<T = unknown>(data: T, status = 200): ApiResponse<T> {
  return { data, status, message: "ok", timestamp: new Date().toISOString() };
}

// ── Generic arrow function ────────────────────────────────────────────────
const wrap = <T>(value: T): { data: T } => ({ data: value });

// ── Generic arrow in .tsx — use trailing comma to avoid JSX ambiguity: ───
const wrap = <T,>(value: T): { data: T } => ({ data: value });

// ── Explicit type argument (when inference fails) ─────────────────────────
const result = identity<User>(someValue as User);
```

---

## How it works — rule by rule

### Rule 1 — Type inference flows from arguments to return type

TypeScript reads the argument types to infer type parameters, then propagates those inferred types through the return type and other parameters:

```ts
function transform<TInput, TOutput>(
  items: TInput[],
  fn: (item: TInput) => TOutput,
): TOutput[] {
  return items.map(fn);
}

// TypeScript infers:
//   TInput = { id: number; email: string }
//   TOutput = string
const emails = transform(
  [{ id: 1, email: "a@dev.io" }, { id: 2, email: "b@dev.io" }],
  user => user.email,
);
// emails: string[]  — inferred from the lambda's return type

// Another call — different types, same function:
const doubled = transform([1, 2, 3], n => n * 2);
// TInput = number, TOutput = number → doubled: number[]
```

### Rule 2 — Constraints add requirements without stripping properties

`T extends X` means "T must have at least the shape of X". It doesn't change `T` — the full original type is preserved in the return type and elsewhere:

```ts
function enrichWithTimestamp<T extends { id: number }>(entity: T): T & { fetchedAt: Date } {
  return { ...entity, fetchedAt: new Date() };
}

const user = { id: 1, name: "Parsh", email: "p@dev.io", role: "admin" as const };
const enriched = enrichWithTimestamp(user);

// T was inferred as { id: number; name: string; email: string; role: "admin" }
// enriched: { id: number; name: string; email: string; role: "admin"; fetchedAt: Date }
// All original properties preserved — constraint only added a requirement, didn't shrink the type

enriched.name;      // ✅ string
enriched.role;      // ✅ "admin"
enriched.fetchedAt; // ✅ Date
```

### Rule 3 — `keyof` constraint for safe property access

`TKey extends keyof TObj` ensures `TKey` is a valid key of `TObj`, and `TObj[TKey]` gives you the exact type of that property:

```ts
// getField — returns the exact type of the property, not just 'any' or 'unknown':
function getField<TObj, TKey extends keyof TObj>(obj: TObj, key: TKey): TObj[TKey] {
  return obj[key];
}

interface UserRecord {
  id: number;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
}

const user: UserRecord = { id: 1, name: "Parsh", email: "p@dev.io", role: "admin" };

const userId    = getField(user, "id");    // userId: number
const userName  = getField(user, "name");  // userName: string
const userRole  = getField(user, "role");  // userRole: "admin" | "editor" | "viewer"

getField(user, "password"); // ❌ Error: Argument of type '"password"' is not assignable to keyof UserRecord
```

### Rule 4 — Multiple type parameters can be independent or related

```ts
// Independent — TKey and TValue don't constrain each other:
function createRecord<TKey extends string, TValue>(key: TKey, value: TValue): Record<TKey, TValue> {
  return { [key]: value } as Record<TKey, TValue>;
}

// Related — TValue is constrained by TKey through TMap:
function getFromMap<TMap extends Record<string, unknown>, TKey extends keyof TMap>(
  map: TMap,
  key: TKey,
): TMap[TKey] {
  return map[key];
}

const config = { timeout: 5000, retries: 3, baseUrl: "https://api.dev.io" };
const timeout = getFromMap(config, "timeout");  // number
const baseUrl = getFromMap(config, "baseUrl");  // string
// TypeScript knows the exact type of each field — not just 'unknown'
```

### Rule 5 — Inferring from callback parameters

TypeScript can infer type parameters from the parameter types of callback arguments:

```ts
function withEach<TItem, TResult>(
  items: TItem[],
  callback: (item: TItem, index: number) => TResult,
): TResult[] {
  return items.map(callback);
}

// TypeScript infers TItem from the array, TResult from the callback's return type:
const lengths = withEach(["hello", "world"], (s, i) => s.length + i);
// TItem = string, TResult = number → lengths: number[]

// s and i are fully typed in the callback — no need to annotate them:
withEach(["a", "b"], (s, i) => {
  s.toUpperCase(); // ✅ s: string
  i.toFixed(0);   // ✅ i: number
});
```

### Rule 6 — Default type parameters

A default gives TypeScript something to fall back on when inference fails (e.g., empty array, no argument):

```ts
interface ApiResponse<TData = unknown> {
  data: TData;
  status: number;
  message: string;
}

// With data — T inferred:
function ok<T>(data: T): ApiResponse<T> {
  return { data, status: 200, message: "ok" };
}

// No-data response — uses default:
function noContent(): ApiResponse {  // T = unknown (default)
  return { data: undefined as unknown, status: 204, message: "No Content" };
}

// Empty array edge case — default prevents T = never:
function emptyList<T = User>(): T[] { return []; }
const users = emptyList();        // users: User[]  — uses default
const posts = emptyList<Post>();  // posts: Post[]  — explicit override
```

### Rule 7 — When to use one type param vs many

Use **one** `T` when all positions share the same type:
```ts
function clone<T>(value: T): T { ... }
function first<T>(arr: T[]): T | undefined { ... }
```

Use **multiple** when positions have different but related types:
```ts
function map<TInput, TOutput>(arr: TInput[], fn: (x: TInput) => TOutput): TOutput[] { ... }
function zip<TLeft, TRight>(a: TLeft[], b: TRight[]): Array<[TLeft, TRight]> { ... }
```

Use **`keyof` + two params** when you need a property accessor pattern:
```ts
function pluck<TObj, TKey extends keyof TObj>(items: TObj[], key: TKey): TObj[TKey][] { ... }
```

---

## Example 1 — basic

```ts
// A small library of typed array/object utilities

// 1. pluck — extract one property from every item in an array:
function pluck<TObj, TKey extends keyof TObj>(items: TObj[], key: TKey): TObj[TKey][] {
  return items.map(item => item[key]);
}

interface UserSummary {
  id: number;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
}

const users: UserSummary[] = [
  { id: 1, name: "Parsh", email: "p@dev.io",   role: "admin" },
  { id: 2, name: "Dev",   email: "d@dev.io",   role: "editor" },
  { id: 3, name: "Asha",  email: "a@dev.io",   role: "viewer" },
];

const ids    = pluck(users, "id");    // ids: number[]
const names  = pluck(users, "name");  // names: string[]
const emails = pluck(users, "email"); // emails: string[]
pluck(users, "password");  // ❌ Error — 'password' is not a key of UserSummary

// 2. partition — split an array into two based on a predicate:
function partition<T>(
  items: T[],
  predicate: (item: T) => boolean,
): [T[], T[]] {  // [matching, not-matching]
  const yes: T[] = [];
  const no: T[]  = [];
  for (const item of items) {
    (predicate(item) ? yes : no).push(item);
  }
  return [yes, no];
}

const [admins, nonAdmins] = partition(users, u => u.role === "admin");
// admins: UserSummary[]  nonAdmins: UserSummary[]

// 3. zip — combine two arrays element-by-element:
function zip<TLeft, TRight>(left: TLeft[], right: TRight[]): Array<[TLeft, TRight]> {
  const len = Math.min(left.length, right.length);
  return Array.from({ length: len }, (_, i) => [left[i], right[i]]);
}

const zipped = zip([1, 2, 3], ["a", "b", "c"]);
// zipped: Array<[number, string]>
zipped[0][0].toFixed(0);  // ✅ number
zipped[0][1].toUpperCase(); // ✅ string

// 4. mapKeys — transform every value, preserving keys exactly:
function mapValues<TObj extends Record<string, unknown>, TResult>(
  obj: TObj,
  fn: (value: TObj[keyof TObj], key: keyof TObj) => TResult,
): Record<keyof TObj, TResult> {
  const result = {} as Record<keyof TObj, TResult>;
  for (const key in obj) {
    result[key] = fn(obj[key], key);
  }
  return result;
}

const config = { timeout: 5000, retries: 3, maxConnections: 10 };
const asStrings = mapValues(config, v => String(v));
// asStrings: Record<"timeout" | "retries" | "maxConnections", string>
```

---

## Example 2 — real world backend use case

```ts
// Type-safe HTTP client wrapper — generic function chains for API calls

interface ApiResponse<TData> {
  data: TData;
  status: number;
  message: string;
  timestamp: string;
}

interface PaginatedResponse<TItem> {
  items: TItem[];
  total: number;
  page: number;
  pageSize: number;
  hasNext: boolean;
}

// Generic fetch wrapper:
async function apiGet<TResponse>(
  url: string,
  options?: { headers?: Record<string, string>; timeout?: number },
): Promise<ApiResponse<TResponse>> {
  const response = await fetch(url, {
    method: "GET",
    headers: { "Content-Type": "application/json", ...options?.headers },
    signal: options?.timeout ? AbortSignal.timeout(options.timeout) : undefined,
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  return response.json() as Promise<ApiResponse<TResponse>>;
}

async function apiPost<TRequestBody, TResponse>(
  url: string,
  body: TRequestBody,
  options?: { headers?: Record<string, string> },
): Promise<ApiResponse<TResponse>> {
  const response = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json", ...options?.headers },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  return response.json() as Promise<ApiResponse<TResponse>>;
}

// Domain types:
interface User {
  id: number;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  createdAt: string;
}

interface CreateUserInput {
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  password: string;
}

// Type-safe API layer built on the generic functions:
const UserApi = {
  async getAll(page = 1, pageSize = 20): Promise<PaginatedResponse<User>> {
    const response = await apiGet<PaginatedResponse<User>>(
      `/api/users?page=${page}&pageSize=${pageSize}`,
    );
    return response.data;
    // response.data: PaginatedResponse<User>
    // response.data.items: User[]
    // response.data.total: number
    // all fully typed — no casting needed
  },

  async getById(userId: number): Promise<User | null> {
    try {
      const response = await apiGet<User>(`/api/users/${userId}`);
      return response.data; // User
    } catch {
      return null;
    }
  },

  async create(input: CreateUserInput): Promise<User> {
    const response = await apiPost<CreateUserInput, User>("/api/users", input);
    return response.data; // User
  },
};

// Usage — every return type is correctly typed:
const allUsers = await UserApi.getAll(1, 20);
// allUsers.items: User[]
// allUsers.total: number

const singleUser = await UserApi.getById(1);
// singleUser: User | null

const newUser = await UserApi.create({
  name: "Parsh",
  email: "p@dev.io",
  role: "admin",
  password: "securepass",
});
// newUser: User — id, createdAt etc. included from the response

// Generic retry wrapper — works for any async function:
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts: number,
  delayMs: number,
): Promise<T> {
  let lastError: unknown;
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      if (attempt < maxAttempts) {
        await new Promise(resolve => setTimeout(resolve, delayMs * attempt));
      }
    }
  }
  throw lastError;
}

// withRetry works for any return type — T is inferred from fn:
const user = await withRetry(() => UserApi.getById(1), 3, 500);
// user: User | null  — T inferred from the lambda's return type
```

---

## Common mistakes

### Mistake 1 — Widening T by annotating the argument

```ts
function wrap<T>(value: T): { data: T } {
  return { data: value };
}

// ❌ Annotating the argument forces T to widen to the annotation type:
const role: string = "admin";
const result = wrap(role);
// T = string → result: { data: string }
// You lost the literal type "admin"

// ✅ Let TypeScript infer from the literal:
const result2 = wrap("admin" as const);
// T = "admin" → result2: { data: "admin" }

// ✅ Or use explicit type argument:
const result3 = wrap<"admin">("admin");
// T = "admin" → result3: { data: "admin" }
```

### Mistake 2 — Forgetting constraints, then casting with `as`

```ts
// ❌ No constraint — TypeScript doesn't know T has .id, so you cast:
function logId<T>(entity: T): void {
  console.log((entity as any).id);  // unsafe — defeats generics
}

// ✅ Add a constraint — no cast needed:
function logId<T extends { id: number | string }>(entity: T): void {
  console.log(entity.id);  // ✅ typed — string | number
}
```

### Mistake 3 — Too many type parameters — combine when possible

```ts
// ❌ Over-parameterized — TResponse is always ApiResponse<TData>, redundant:
async function fetchData<TData, TResponse extends ApiResponse<TData>>(
  url: string,
): Promise<TResponse> { ... }

// ✅ One parameter is enough — derive the second from it:
async function fetchData<TData>(url: string): Promise<ApiResponse<TData>> { ... }

// ❌ Separate params when one is always derived from the other:
function getProperty<TObj, TKey, TValue>(obj: TObj, key: TKey): TValue { ... }
// TValue is always TObj[TKey] — redundant extra param

// ✅ Use indexed access:
function getProperty<TObj, TKey extends keyof TObj>(obj: TObj, key: TKey): TObj[TKey] {
  return obj[key];
}
```

---

## Practice exercises

### Exercise 1 — easy

1. Write a generic function `clamp<T extends number>(value: T, min: T, max: T): T` that clamps a number to a range. Call it to clamp a request timeout to `[100, 30000]`.

2. Write `omit<TObj, TKey extends keyof TObj>(obj: TObj, keys: TKey[]): Omit<TObj, TKey>` — the inverse of `pick`. Verify it removes only the specified keys and the return type reflects that.

3. Write `findFirst<T>(items: T[], predicate: (item: T) => boolean): T | undefined` — a typed wrapper around `Array.find`. Use it to find a user by email from a `User[]`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed data-transformation pipeline:

```ts
function pipeline<TInput, TOutput>(
  input: TInput,
  ...steps: ???  // each step receives the output of the previous step
): TOutput
```

The challenge is typing the `steps` array so that:
- Step 1 takes `TInput` and returns some intermediate type
- Step 2 takes what step 1 returned
- Step N returns `TOutput`

Implement it for a fixed number of steps using overloads (up to 4 steps):

```ts
// 1-step:
function pipeline<A, B>(input: A, step1: (a: A) => B): B;
// 2-step:
function pipeline<A, B, C>(input: A, step1: (a: A) => B, step2: (b: B) => C): C;
// 3-step, 4-step — you fill in
```

Then use it to build a data transformation for an incoming request body:

```ts
interface RawRequestBody { userId: string; amount: string; currency: string; }
interface ParsedBody    { userId: number; amount: number; currency: string; }
interface ValidatedBody  { userId: number; amount: number; currency: "USD" | "EUR" | "GBP"; }
interface EnrichedBody   { userId: number; amount: number; currency: "USD" | "EUR" | "GBP"; amountCents: number; }

const result = pipeline(
  rawBody,
  parseBody,    // RawRequestBody → ParsedBody
  validateBody, // ParsedBody → ValidatedBody
  enrichBody,   // ValidatedBody → EnrichedBody
);
// result: EnrichedBody — fully typed
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a type-safe generic `QueryBuilder<T>` that produces filtered, sorted, paginated results from an in-memory dataset:

```ts
class QueryBuilder<TEntity extends { id: number }> {
  constructor(private dataset: TEntity[]) {}

  // Filter by any field:
  where<TKey extends keyof TEntity>(
    key: TKey,
    operator: "eq" | "neq" | "gt" | "lt" | "contains",
    value: TEntity[TKey],
  ): this

  // Select only certain fields:
  select<TKey extends keyof TEntity>(
    ...keys: TKey[]
  ): QueryBuilder<Pick<TEntity, TKey>>

  // Sort by any field:
  orderBy<TKey extends keyof TEntity>(
    key: TKey,
    direction?: "asc" | "desc",
  ): this

  // Paginate:
  paginate(page: number, pageSize: number): this

  // Execute:
  execute(): TEntity[]

  // Count:
  count(): number
}
```

Requirements:
- `where` filters must be chainable
- `select` must return a new `QueryBuilder` typed to the picked fields — so further `.where` calls only accept the selected keys
- `orderBy` handles strings and numbers correctly
- `execute()` applies all accumulated filters, sorts, and pagination in order

Use it:
```ts
interface Product {
  id: number;
  name: string;
  category: "electronics" | "clothing" | "food";
  priceCents: number;
  inStock: boolean;
}

const products: Product[] = [...]; // sample data

const result = new QueryBuilder(products)
  .where("inStock", "eq", true)
  .where("category", "eq", "electronics")
  .where("priceCents", "lt", 50000)
  .orderBy("priceCents", "asc")
  .paginate(1, 10)
  .execute();
// result: Product[]

// select — narrows the type:
const names = new QueryBuilder(products)
  .where("inStock", "eq", true)
  .select("id", "name", "priceCents")
  .execute();
// names: Pick<Product, "id" | "name" | "priceCents">[]
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Single type param — inferred from argument:
function fn<T>(arg: T): T { return arg; }

// Constraint:
function fn<T extends { id: number }>(arg: T): T { return arg; }

// keyof constraint — safe property access:
function get<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }

// Multiple params — input → output transformation:
function map<TIn, TOut>(arr: TIn[], fn: (x: TIn) => TOut): TOut[] { return arr.map(fn); }

// Default type param:
function createList<T = User>(): T[] { return []; }

// Arrow generic (.tsx safe):
const fn = <T,>(arg: T): T => arg;

// Explicit type argument (when inference fails):
const result = fn<SpecificType>(value);
```

| Pattern | Use case |
|---------|----------|
| `<T>` | Any single-type operation (wrap, clone, first) |
| `<T extends X>` | Need to access properties of T |
| `<T, K extends keyof T>` | Property accessor, pluck, pick |
| `<TIn, TOut>` | Transformations (map, filter, reduce) |
| `<T = Default>` | Optional type param with fallback |
| Explicit `<Type>()` | Empty arrays, no-arg functions, when inference fails |

---

## Connected topics

- **26 — What are generics** — the foundational concept; type params, basic inference, `any` vs `T`.
- **28 — Generic interfaces and classes** — applying the same patterns to interfaces and classes.
- **29 — Generic utility types** — `Partial<T>`, `Required<T>`, `Pick<T,K>`, `Omit<T,K>` — all use `keyof` constraints.
- **40 — Conditional types** — `T extends U ? X : Y` — advanced generic branching logic.
- **41 — Mapped types** — iterating over `keyof T` to transform every property.
- **43 — Template literal types** — combining generics with string manipulation at the type level.
