# 23 — Function Overloads

## What is this?

**Function overloads** let you declare multiple distinct call signatures for a single function. Each overload signature describes one valid way to call the function — different argument types, different argument counts, different return types. The actual function body is written once under the **implementation signature**, which must be broad enough to handle every overload case.

```ts
// Overload signatures (no body — just the type contracts):
function findUser(id: number): Promise<User | null>;
function findUser(email: string): Promise<User | null>;

// Implementation signature (has the body — not visible to callers):
function findUser(criteria: number | string): Promise<User | null> {
  if (typeof criteria === "number") return repo.findById(criteria);
  return repo.findByEmail(criteria);
}
```

TypeScript shows **only the overload signatures** to callers — the implementation signature is internal. Callers see `findUser(id: number)` and `findUser(email: string)` but not `findUser(criteria: number | string)`.

## Why does it matter?

Sometimes a function genuinely behaves differently based on what arguments it receives — not just running the same logic with different types, but producing a **different return type** or accepting a **different argument shape**. A union type in the parameter or return type alone can't capture this. For example:

```ts
// ❌ Union type — caller can't know which return type they get:
function find(criteria: number | string): User | null | Promise<User | null> { ... }
```

With overloads, TypeScript can **correlate** the input type to the output type — if you pass a `number`, you get back `Promise<User | null>`. If you pass a `string[]`, you get back `Promise<User[]>`. This correlation is impossible to express with a single union signature.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — one function, handles multiple call shapes at runtime
// Callers have no idea what return type to expect
function findUser(criteria) {
  if (typeof criteria === "number") return db.findById(criteria);
  if (typeof criteria === "string") return db.findByEmail(criteria);
  if (Array.isArray(criteria))      return db.findManyByIds(criteria);
}
// Called: findUser(1), findUser("a@b.io"), findUser([1,2,3])
// No compile-time verification, no return type info for callers
```

```ts
// TypeScript — overloads express exact input → output correlations
function findUser(id: number): Promise<User | null>;
function findUser(email: string): Promise<User | null>;
function findUser(ids: number[]): Promise<User[]>;
function findUser(criteria: number | string | number[]): Promise<User | null | User[]> {
  if (typeof criteria === "number") return repo.findById(criteria);
  if (typeof criteria === "string") return repo.findByEmail(criteria);
  return repo.findManyByIds(criteria);
}

// Callers get precise return types:
const user1 = await findUser(1);           // type: User | null  ✅
const user2 = await findUser("a@dev.io");  // type: User | null  ✅
const users = await findUser([1, 2, 3]);   // type: User[]       ✅

// TypeScript knows each call's return type based on the overload matched:
const count = (await findUser([1, 2])).length; // ✅ .length — it's User[]
```

---

## Syntax

```ts
// ── BASIC OVERLOAD PATTERN ─────────────────────────────────────────────────
// 1. Write overload signatures (no body):
function myFn(a: string): string;
function myFn(a: number): number;

// 2. Write implementation signature (body, not visible to callers):
function myFn(a: string | number): string | number {
  if (typeof a === "string") return a.toUpperCase();
  return a * 2;
}

// ── OVERLOADS ON A METHOD (class or object) ────────────────────────────────
class UserRepository {
  find(id: number): Promise<User | null>;
  find(email: string): Promise<User | null>;
  find(criteria: number | string): Promise<User | null> {
    if (typeof criteria === "number") return this.findById(criteria);
    return this.findByEmail(criteria);
  }

  private findById(id: number): Promise<User | null> { return Promise.resolve(null); }
  private findByEmail(email: string): Promise<User | null> { return Promise.resolve(null); }
}

// ── OVERLOADS IN AN INTERFACE ──────────────────────────────────────────────
interface Formatter {
  format(value: string): string;
  format(value: number, decimals: number): string;
  format(value: Date, locale: string): string;
}
```

---

## How it works — rule by rule

### The implementation signature is invisible to callers

```ts
function process(input: string): string;
function process(input: number): number;
function process(input: string | number): string | number {
  // ↑ This signature is the IMPLEMENTATION — callers cannot use it directly
  if (typeof input === "string") return input.toUpperCase();
  return input * 2;
}

// What callers see:
process("hello");  // ✅ → overload 1 → return type: string
process(42);       // ✅ → overload 2 → return type: number
process(true);     // ❌ — boolean doesn't match any overload signature

// Callers cannot use the implementation union:
const x: string | number = process("hello"); // ✅ but verbose
// TypeScript narrows correctly:
const s: string = process("hello");   // ✅ — overload 1 returns string
const n: number = process(42);        // ✅ — overload 2 returns number
```

### Implementation signature must be compatible with all overloads

The implementation signature must be broad enough to accept every overload's arguments AND return every overload's return type:

```ts
// ❌ BROKEN — implementation signature too narrow:
function getValue(key: string): string;
function getValue(key: number): number;
function getValue(key: string): string {  // ❌ doesn't handle number input
  return key.toUpperCase();
}

// ✅ CORRECT — implementation covers all overload cases:
function getValue(key: string): string;
function getValue(key: number): number;
function getValue(key: string | number): string | number {
  if (typeof key === "string") return key.toUpperCase();
  return key * 10;
}
```

### Return type correlates with input type

This is the main reason to use overloads — when the return type **depends on** the input type and you need TypeScript to know this:

```ts
// Without overloads — caller gets the union, must narrow manually:
function parse(input: string, asJson?: boolean): string | object {
  return asJson ? JSON.parse(input) : input;
}
const result = parse('{"a":1}', true);
result.a; // ❌ Error — 'a' doesn't exist on 'string | object'
(result as { a: number }).a; // ugly cast required

// With overloads — TypeScript knows the return type from the call:
function parse(input: string, asJson: true): object;
function parse(input: string, asJson?: false): string;
function parse(input: string, asJson?: boolean): string | object {
  return asJson ? JSON.parse(input) : input;
}

const obj = parse('{"a":1}', true);   // type: object ✅
const str = parse("hello");            // type: string ✅
obj; str; // no casts needed
```

### Minimum two overload signatures required

You need at least 2 overload signatures. If you only have one, just annotate the function normally:

```ts
// ❌ Pointless — single overload, should just be a normal function:
function greet(name: string): string;
function greet(name: string): string {
  return `Hello, ${name}!`;
}

// ✅ Just write it normally:
function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

### Overloads are matched top to bottom — order matters

TypeScript picks the **first** overload signature that matches:

```ts
function format(value: string): string;
function format(value: string | null): string | null;
function format(value: string | null): string | null {
  return value?.toUpperCase() ?? null;
}

// TypeScript matches overload 1 for string — returns string (not string | null):
const s = format("hello");    // type: string  ← first overload matched
const n = format(null);       // ❌ — null matches overload 2, but overload 1 came first
                               // Actually: null doesn't match overload 1 (string only)
                               //           null matches overload 2 ✅ → string | null

// Put MORE SPECIFIC overloads first, LESS SPECIFIC last:
function format(value: string): string;           // more specific first
function format(value: string | null): string | null; // broader second
```

### When to use overloads vs alternatives

```ts
// ── SCENARIO 1: Use overloads — return type depends on input ─────────────
function serialize(data: string): Buffer;
function serialize(data: Buffer): string;
function serialize(data: string | Buffer): string | Buffer {
  return typeof data === "string" ? Buffer.from(data) : data.toString();
}

// ── SCENARIO 2: Use generics instead — logic is the same, type varies ────
// DON'T DO THIS with overloads:
function identity(x: string): string;
function identity(x: number): number;
function identity(x: string | number): string | number { return x; }

// DO THIS with generics:
function identity<T>(x: T): T { return x; }

// ── SCENARIO 3: Use optional param instead — second form just omits an arg:
// DON'T DO THIS with overloads:
function log(msg: string): void;
function log(msg: string, level: string): void;
function log(msg: string, level?: string): void { console.log(level ? `[${level}] ${msg}` : msg); }

// DO THIS with optional param:
function log(msg: string, level?: string): void {
  console.log(level ? `[${level}] ${msg}` : msg);
}
```

---

## Example 1 — basic

```ts
// A typed event system with overloads for different event payloads

interface UserCreatedEvent { userId: number; email: string; name: string; }
interface UserDeletedEvent { userId: number; deletedAt: Date; }
interface OrderPlacedEvent { orderId: number; userId: number; total: number; }

// Overloaded emit — return type could vary, but here we use for type-safe payload:
function emitEvent(event: "user.created", payload: UserCreatedEvent): void;
function emitEvent(event: "user.deleted", payload: UserDeletedEvent): void;
function emitEvent(event: "order.placed", payload: OrderPlacedEvent): void;
function emitEvent(event: string, payload: unknown): void {
  console.log(`EVENT: ${event}`, payload);
  // In production: eventBus.publish(event, payload)
}

emitEvent("user.created", { userId: 1, email: "p@dev.io", name: "Parsh" });  // ✅
emitEvent("user.deleted", { userId: 1, deletedAt: new Date() });              // ✅
emitEvent("user.created", { userId: 1 });  // ❌ missing email and name
emitEvent("user.created", { userId: 1, email: "p@dev.io", name: "Parsh", extra: true }); // ❌ extra field

// Overloaded find — different input types, different (but related) return types:
function findEntity(type: "user",    id: number): Promise<User | null>;
function findEntity(type: "order",   id: number): Promise<Order | null>;
function findEntity(type: "product", id: number): Promise<Product | null>;
function findEntity(type: string, id: number): Promise<User | Order | Product | null> {
  switch (type) {
    case "user":    return userRepo.findById(id);
    case "order":   return orderRepo.findById(id);
    case "product": return productRepo.findById(id);
    default: return Promise.resolve(null);
  }
}

const user    = await findEntity("user",    1);  // type: User | null    ✅
const order   = await findEntity("order",   1);  // type: Order | null   ✅
const product = await findEntity("product", 1);  // type: Product | null ✅
```

---

## Example 2 — real world backend use case

```ts
import { Response } from 'express';

// ── Overloaded response builder — return type narrows based on status ──────

function sendResponse(res: Response, status: 200, data: unknown): void;
function sendResponse(res: Response, status: 201, data: unknown): void;
function sendResponse(res: Response, status: 400, error: string): void;
function sendResponse(res: Response, status: 401, error: string): void;
function sendResponse(res: Response, status: 403, error: string): void;
function sendResponse(res: Response, status: 404, error: string): void;
function sendResponse(res: Response, status: 500, error: string): void;
function sendResponse(res: Response, status: number, payload: unknown): void {
  if (status >= 400) {
    res.status(status).json({ success: false, error: payload });
  } else {
    res.status(status).json({ success: true, data: payload });
  }
}

// Callers get type-checked payloads per status:
sendResponse(res, 200, { userId: 1 });   // ✅ data: unknown
sendResponse(res, 201, { userId: 1 });   // ✅ data: unknown
sendResponse(res, 400, "Invalid email"); // ✅ error: string
sendResponse(res, 400, { userId: 1 });   // ❌ object not assignable to string
sendResponse(res, 200, "Bad string");    // ✅ (unknown accepts string — but intent is clear)

// ── Overloaded query builder — sync vs async based on options ─────────────

interface SyncQueryOptions  { async: false; }
interface AsyncQueryOptions { async: true; timeoutMs?: number; }

function runQuery(sql: string, params: SqlParam[], options: SyncQueryOptions): QueryRow[];
function runQuery(sql: string, params: SqlParam[], options: AsyncQueryOptions): Promise<QueryRow[]>;
function runQuery(sql: string, params: SqlParam[], options: SyncQueryOptions | AsyncQueryOptions): QueryRow[] | Promise<QueryRow[]> {
  if (options.async) {
    return new Promise<QueryRow[]>((resolve) => {
      // simulate async db query
      resolve([]);
    });
  }
  // simulate sync in-memory query
  return [];
}

const syncRows  = runQuery("SELECT 1", [], { async: false }); // type: QueryRow[]           ✅
const asyncRows = runQuery("SELECT 1", [], { async: true });  // type: Promise<QueryRow[]>  ✅

// ── Overloaded cache get — with/without default value ─────────────────────

class TypedCache<T> {
  private store = new Map<string, T>();

  get(key: string): T | undefined;
  get(key: string, defaultValue: T): T;
  get(key: string, defaultValue?: T): T | undefined {
    const value = this.store.get(key);
    if (value !== undefined) return value;
    return defaultValue;
  }

  set(key: string, value: T): void {
    this.store.set(key, value);
  }
}

const cache = new TypedCache<User>();
const maybeUser = cache.get("user_1");           // type: User | undefined  ✅
const definiteUser = cache.get("user_1", adminUser); // type: User           ✅
// No need to null-check definiteUser — the overload guarantees it's always User
```

---

## Common mistakes

### Mistake 1 — Writing the implementation signature only (no overload signatures)

```ts
// ❌ No overload signatures — callers see the broad union, not the narrow types:
function find(criteria: number | string): User | null {
  // ...
  return null;
}

const result = find(1);
// result type: User | null  — TypeScript doesn't know it's non-null for number input
// No type narrowing based on argument type

// ✅ Add overload signatures so callers get narrow return types:
function find(id: number): User | null;
function find(email: string): User | null;
function find(criteria: number | string): User | null {
  // ...
  return null;
}
// Now TypeScript uses the overload to resolve the call-site type
```

### Mistake 2 — Making the implementation signature too narrow

```ts
// ❌ Implementation doesn't cover all overloads:
function encode(value: string): Buffer;
function encode(value: Buffer): string;
function encode(value: string): Buffer | string {  // ❌ Only handles string — ignores Buffer overload
  return Buffer.from(value);
}
// Error: Overload signature is not compatible with its implementation signature

// ✅ Implementation covers both:
function encode(value: string): Buffer;
function encode(value: Buffer): string;
function encode(value: string | Buffer): Buffer | string {  // ✅ handles both
  return typeof value === "string" ? Buffer.from(value) : value.toString();
}
```

### Mistake 3 — Using overloads where generics or union types are cleaner

```ts
// ❌ OVERENGINEERED with overloads — same logic, same return type, just different input:
function wrap(value: string): string[];
function wrap(value: number): number[];
function wrap(value: boolean): boolean[];
function wrap(value: string | number | boolean): (string | number | boolean)[] {
  return [value];
}
// 6 lines for something a generic handles in 1

// ✅ SIMPLE with generics:
function wrap<T>(value: T): T[] {
  return [value];
}
// wrap("hello")  → string[]
// wrap(42)       → number[]
// wrap(true)     → boolean[]
```

---

## Practice exercises

### Exercise 1 — easy

Write overloaded versions of these functions:

1. `toString(value: number): string` and `toString(value: boolean): string` and `toString(value: Date): string`
   — converts each type to a meaningful string representation (number → fixed-2 decimals, boolean → "yes"/"no", Date → ISO string)

2. `getConfig(key: "port"): number` and `getConfig(key: "host"): string` and `getConfig(key: "debug"): boolean`
   — returns a config value typed by which key you request (use hardcoded values in the implementation)

For both, demonstrate that TypeScript infers the correct return type at each call site and rejects invalid inputs.

```ts
// Write your code here
```

### Exercise 2 — medium

Build an overloaded `request` function for a typed HTTP client:

```
request(url: string): Promise<string>
request(url: string, options: { method: "GET"; responseType: "text" }): Promise<string>
request(url: string, options: { method: "GET"; responseType: "json" }): Promise<unknown>
request(url: string, options: { method: "POST"; body: unknown }): Promise<unknown>
request(url: string, options: { method: "DELETE" }): Promise<void>
```

Implementation: `request(url: string, options?: { method?: string; body?: unknown; responseType?: string }): Promise<string | unknown | void>`

In the implementation body:
- Simulate the request with a stub (no real fetch needed)
- Return `"response text"` for text, `{ status: "ok" }` for json, `undefined` for DELETE
- Log the url and options to console

Demonstrate all 5 overload signatures are callable and TypeScript correctly infers each return type.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed repository factory using overloads to return different repository types based on a string discriminant:

Define these interfaces (stubs — no need to fully implement):
```ts
interface UserRepo    { findById(id: number): Promise<User | null>;    findByEmail(email: string): Promise<User | null>; }
interface OrderRepo   { findById(id: number): Promise<Order | null>;   findByUserId(userId: number): Promise<Order[]>; }
interface ProductRepo { findById(id: number): Promise<Product | null>; findBySku(sku: string): Promise<Product | null>; }
```

Write an overloaded `createRepository` function:
```
createRepository(type: "user"):    UserRepo
createRepository(type: "order"):   OrderRepo
createRepository(type: "product"): ProductRepo
```

Implement it with an in-memory stub for each. The implementation returns `UserRepo | OrderRepo | ProductRepo`.

Then write a `ServiceContainer` class that:
- Lazily instantiates repositories (only creates them on first access)
- Exposes `getRepo(type: "user"): UserRepo`, `getRepo(type: "order"): OrderRepo`, `getRepo(type: "product"): ProductRepo` — each overloaded
- Internally uses `createRepository` to create instances

Wire it up: get all three repos from the container, call one method on each, verify TypeScript knows the exact return types.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Rule | Notes |
|------|-------|
| Overload signatures have no body | Just the type contract — no `{ }` |
| Implementation signature has the body | Not visible to callers |
| Implementation must be compatible | Must accept all overload inputs and return all overload types |
| At least 2 overload signatures required | Single signature → just annotate the function normally |
| Overloads matched top to bottom | Put more specific signatures first |
| Implementation signature is NOT callable | Callers can only use the named overload signatures |
| Overloads on methods too | Same syntax inside class body |

| Use overloads when | Use generics/union instead when |
|--------------------|---------------------------------|
| Return type depends on input type | Return type is always the same regardless of input |
| Input shapes are structurally distinct | Input differences are just subtypes/supertypes |
| You need to express input → output correlation | Same logic runs for all input variants |
| Wrapping a third-party API with multiple call forms | Simple identity-style functions |

## Connected topics

- **20 — Function types** — the foundation: all overloads are function types.
- **26 — Generic functions** — often a cleaner alternative to overloads when return type doesn't vary.
- **09 — Union types** — implementation signatures use unions; overloads narrow them for callers.
- **19 — Implementing interfaces in classes** — overloaded interface methods must be implemented with compatible overloads.
- **24 — Higher-order functions** — when wrapping overloaded functions, generics + overloads often work together.
