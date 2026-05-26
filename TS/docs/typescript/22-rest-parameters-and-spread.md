# 22 — Rest Parameters and Spread in TypeScript

## What is this?

**Rest parameters** (`...args`) let a function accept any number of trailing arguments, collecting them into a typed array. **Spread** (`...value`) unpacks an array or tuple into individual arguments at a call site.

JavaScript has both since ES6. TypeScript adds:
- **Typed rest parameters** — `...args: string[]` vs `...args: any[]`
- **Tuple rest parameters** — `...args: [string, number, boolean]` — fixed-length typed spread
- **Spread type checking** — TypeScript verifies that spread arguments match the function's parameter types
- **`as const` + spread** — turning a `readonly` tuple into spread arguments

## Why does it matter?

In JavaScript, `arguments` and rest parameters have no type information. You can spread anything into any function with no compile-time safety. In TypeScript, rest and spread are fully type-checked:

- A variadic logger function can enforce that every argument is a `string`
- A function that wraps another function can spread its args with full type preservation
- Tuple-typed rest params let you express "exactly this fixed sequence of argument types"
- TypeScript catches you spreading an `unknown[]` into a function expecting `[string, number]`

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no type checking on variadic arguments
function log(level, ...messages) {
  // messages is any[] — could be strings, numbers, objects, anything
  console.log(`[${level}]`, ...messages);
}

log("INFO", "Server started", 42, { port: 3000 }); // no warnings, chaos

function applyFn(fn, ...args) {
  return fn(...args); // TypeScript has no idea if args matches fn's params
}
```

```ts
// TypeScript — rest parameters are fully typed
function log(level: "INFO" | "WARN" | "ERROR", ...messages: string[]): void {
  console.log(`[${level}]`, ...messages);
}

log("INFO", "Server started", "port 3000");  // ✅
log("INFO", "Server started", 42);            // ❌ 42 is not a string
log("DEBUG", "test");                          // ❌ "DEBUG" not in the union

// Tuple-typed rest — fixed sequence of types:
function createRoute(...args: [string, string, RouteHandler]): Route {
  const [method, path, handler] = args;
  return { method, path, handler };
}
```

---

## Syntax

```ts
// ── REST PARAMETER — array type ─────────────────────────────────────────────
function logAll(prefix: string, ...lines: string[]): void {
  lines.forEach(line => console.log(`${prefix}: ${line}`));
}

// ── REST PARAMETER — union element type ─────────────────────────────────────
function writeLog(level: string, ...args: (string | number)[]): void {
  console.log(`[${level}]`, ...args);
}

// ── REST PARAMETER — tuple type (fixed-length, typed positionally) ──────────
function createEndpoint(...args: [method: string, path: string, handler: RouteHandler]): void {
  const [method, path, handler] = args;
}

// ── SPREAD INTO A FUNCTION CALL ─────────────────────────────────────────────
const nums: number[] = [1, 2, 3];
Math.max(...nums);  // ✅ TypeScript checks: Math.max accepts ...number[]

// ── SPREAD A TUPLE — exact positional match ──────────────────────────────────
function connect(host: string, port: number, ssl: boolean): void {}
const args: [string, number, boolean] = ["localhost", 5432, true];
connect(...args);  // ✅ tuple spread matches exactly

// ── SPREAD AN ARRAY — requires matching rest param ───────────────────────────
const hosts: string[] = ["a.io", "b.io"];
function greetAll(...names: string[]): void {}
greetAll(...hosts);  // ✅ string[] spreads into ...string[]
```

---

## How it works — rule by rule

### Rest parameter must be last

```ts
// ✅ Rest parameter at the end:
function send(to: string, ...attachments: string[]): void {}

// ❌ Rest parameter in the middle — not allowed:
function bad(...headers: string[], body: string): void {}
// Error: A rest parameter must be last in a parameter list

// ❌ Multiple rest parameters — not allowed:
function alsoBad(...a: string[], ...b: number[]): void {}
```

### The type of `args` inside the function is `T[]`

```ts
function sum(...nums: number[]): number {
  // nums is number[] inside the function
  return nums.reduce((acc, n) => acc + n, 0);
}

sum(1, 2, 3);           // ✅ → 6
sum(1, 2, 3, 4, 5);     // ✅ → 15
sum();                   // ✅ → 0 (empty array)
sum(1, "two");           // ❌ "two" is not a number
```

### Spread into a function — TypeScript checks compatibility

```ts
function addUser(email: string, name: string, role: "admin" | "viewer"): void {}

// ── Spread a tuple (exact match) ─────────────────────────────────────────
const userArgs: [string, string, "admin" | "viewer"] = ["a@b.io", "Alice", "admin"];
addUser(...userArgs);  // ✅

// ── Spread an array — TypeScript may reject it ────────────────────────────
const args = ["a@b.io", "Alice", "admin"];  // inferred as string[]
addUser(...args);
// ❌ A spread argument must either have a tuple type or be passed to a rest parameter
// TypeScript can't verify string[3] matches (string, string, "admin" | "viewer")

// ── Fix 1: use a tuple with 'as const' ───────────────────────────────────
const args2 = ["a@b.io", "Alice", "admin"] as const;
addUser(...args2);  // ✅ type is readonly ["a@b.io", "Alice", "admin"] — known positions

// ── Fix 2: annotate the array as a tuple ─────────────────────────────────
const args3: [string, string, "admin" | "viewer"] = ["a@b.io", "Alice", "admin"];
addUser(...args3);  // ✅
```

### Tuple rest parameters — fixed-length variadic functions

When you want a function to accept a fixed sequence of typed arguments (like `Promise.all` or `Array.from`), use a tuple as the rest type:

```ts
// This function accepts exactly 3 args of known types:
function runMigration(...args: [id: number, name: string, up: () => Promise<void>]): void {
  const [id, name, up] = args;
  console.log(`Running migration ${id}: ${name}`);
  up();
}

runMigration(1, "add_users_table", async () => { /* ... */ });  // ✅
runMigration(1, "add_users_table");  // ❌ missing third arg
runMigration("1", "add_users_table", async () => {});  // ❌ "1" not number
```

### Generic rest parameters — forwarding arguments

A common pattern: wrapping a function and forwarding all its typed arguments:

```ts
// Wrap console.log with a prefix — preserves all argument types:
function createLogger(prefix: string) {
  return function<T extends unknown[]>(...args: T): void {
    console.log(`[${prefix}]`, ...args);
  };
}

const logger = createLogger("API");
logger("Request received", { method: "GET", path: "/users" });  // ✅ any types, any count

// Wrapping an async function with timing:
function withTiming<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => Promise<TReturn>,
  label: string,
): (...args: TArgs) => Promise<TReturn> {
  return async (...args: TArgs): Promise<TReturn> => {
    const start = Date.now();
    const result = await fn(...args);
    console.log(`${label} took ${Date.now() - start}ms`);
    return result;
  };
}

// The wrapper preserves the exact signature of the wrapped function:
async function findUser(id: number, includeDeleted: boolean): Promise<User | null> {
  return null;
}

const timedFindUser = withTiming(findUser, "findUser");
// timedFindUser has type: (id: number, includeDeleted: boolean) => Promise<User | null>
timedFindUser(1, false);   // ✅ TypeScript infers both parameter types
timedFindUser("1", false); // ❌ string not assignable to number — preserved!
```

### Rest with `as const` — spreading read-only tuples

```ts
// Problem: arrays lose positional type information
const dbArgs = ["localhost", 5432, "mydb"]; // string | number)[] — loses order info

// Solution: as const preserves the tuple type
const dbArgs2 = ["localhost", 5432, "mydb"] as const;
// type: readonly ["localhost", 5432, "mydb"]

function connectDb(host: string, port: number, name: string): void {}
connectDb(...dbArgs);   // ❌ array spread doesn't match (string, number, string)
connectDb(...dbArgs2);  // ✅ tuple spread matches exactly
```

---

## Example 1 — basic

```ts
// Typed variadic utilities for backend use

// ── Multi-target event emitter ─────────────────────────────────────────────
type EventLevel = "debug" | "info" | "warn" | "error";

function broadcast(level: EventLevel, event: string, ...recipients: string[]): void {
  if (recipients.length === 0) {
    console.log(`[${level}] ${event} → all`);
    return;
  }
  recipients.forEach(r => console.log(`[${level}] ${event} → ${r}`));
}

broadcast("info", "user.created");                         // → all
broadcast("info", "user.created", "admin@dev.io");         // → one recipient
broadcast("warn", "login.failed", "sec@dev.io", "ops@dev.io"); // → two recipients

// ── SQL-style tagged query builder ─────────────────────────────────────────
function buildWhereClause(operator: "AND" | "OR", ...conditions: string[]): string {
  if (conditions.length === 0) return "1=1";
  return conditions.map(c => `(${c})`).join(` ${operator} `);
}

const clause = buildWhereClause(
  "AND",
  "deleted_at IS NULL",
  "role = 'admin'",
  "email LIKE '%@company.io'",
);
// "(deleted_at IS NULL) AND (role = 'admin') AND (email LIKE '%@company.io')"

// ── Pipeline composer — spread functions into a pipeline ───────────────────
type Transform<T> = (input: T) => T;

function compose<T>(...transforms: Transform<T>[]): Transform<T> {
  return (input: T): T => transforms.reduce((acc, fn) => fn(acc), input);
}

const sanitizeEmail: Transform<string> = s => s.toLowerCase().trim();
const validateEmail: Transform<string> = s => {
  if (!s.includes("@")) throw new Error(`Invalid email: ${s}`);
  return s;
};
const normalizeEmail: Transform<string> = s => s.replace(/\+.*@/, "@");

const processEmail = compose(sanitizeEmail, validateEmail, normalizeEmail);
processEmail("  USER+tag@Dev.IO  "); // "user@dev.io"

// ── Merge objects with spread ──────────────────────────────────────────────
function mergeConfigs<T extends object>(...configs: Partial<T>[]): Partial<T> {
  return Object.assign({}, ...configs);
}

const base = { host: "localhost", port: 3000, debug: false };
const env  = { port: 8080 };
const user = { debug: true };
const merged = mergeConfigs(base, env, user);
// { host: "localhost", port: 8080, debug: true }
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response, NextFunction } from 'express';

// ── Middleware composer — spread middlewares into an array ─────────────────
type ExpressMiddleware = (req: Request, res: Response, next: NextFunction) => void | Promise<void>;

function composeMiddlewares(...middlewares: ExpressMiddleware[]): ExpressMiddleware {
  return async (req, res, next) => {
    let index = 0;
    async function runNext(): Promise<void> {
      if (index >= middlewares.length) {
        next();
        return;
      }
      const current = middlewares[index++];
      await current(req, res, runNext);
    }
    await runNext();
  };
}

// Usage — spread individual middlewares:
const requireAuth: ExpressMiddleware = (req, res, next) => { /* ... */ next(); };
const validateBody: ExpressMiddleware = (req, res, next) => { /* ... */ next(); };
const rateLimit: ExpressMiddleware = (req, res, next) => { /* ... */ next(); };

const protectedRoute = composeMiddlewares(requireAuth, rateLimit, validateBody);

// ── Generic function wrapper preserving full type signature ─────────────────
function withErrorHandling<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => Promise<TReturn>,
  fallback: TReturn,
): (...args: TArgs) => Promise<TReturn> {
  return async (...args: TArgs): Promise<TReturn> => {
    try {
      return await fn(...args);
    } catch (err) {
      console.error("Caught error:", (err as Error).message);
      return fallback;
    }
  };
}

async function getUserById(id: number, includeDeleted: boolean): Promise<User | null> {
  // ... db query
  return null;
}

// Wrapped function preserves exact signature:
const safeGetUserById = withErrorHandling(getUserById, null);
// type: (id: number, includeDeleted: boolean) => Promise<User | null>
await safeGetUserById(1, false);   // ✅

// ── Batch operation — rest params of typed IDs ─────────────────────────────
async function deleteUsers(...userIds: number[]): Promise<{ deleted: number; failed: number[] }> {
  let deleted = 0;
  const failed: number[] = [];

  await Promise.all(
    userIds.map(async id => {
      try {
        // await db.delete(id);
        deleted++;
      } catch {
        failed.push(id);
      }
    })
  );

  return { deleted, failed };
}

await deleteUsers(1, 2, 3, 4, 5);

// Spread from an array of IDs:
const staleUserIds: number[] = [10, 11, 12];
await deleteUsers(...staleUserIds);  // ✅ number[] spreads into ...number[]

// ── SQL IN clause builder ──────────────────────────────────────────────────
function sqlIn(column: string, ...values: (string | number)[]): string {
  if (values.length === 0) return "FALSE";  // no values = always false
  const placeholders = values.map((_, i) => `$${i + 1}`).join(", ");
  return `${column} IN (${placeholders})`;
}

sqlIn("user_id", 1, 2, 3);           // "user_id IN ($1, $2, $3)"
sqlIn("status", "active", "pending"); // "status IN ($1, $2)"
sqlIn("id");                           // "FALSE"

const ids = [1, 2, 3, 4, 5];
sqlIn("user_id", ...ids);             // ✅ spread number[] into ...(string | number)[]
```

---

## Common mistakes

### Mistake 1 — Spreading a plain array into a non-rest function

```ts
function connect(host: string, port: number): void {}

const args = ["localhost", 3000]; // inferred as (string | number)[]

connect(...args);
// ❌ Error: A spread argument must either have a tuple type or be passed to a rest parameter
// TypeScript can't verify (string | number)[] positionally matches (string, number)

// ✅ Fix 1 — annotate as a tuple:
const args2: [string, number] = ["localhost", 3000];
connect(...args2);  // ✅

// ✅ Fix 2 — use 'as const':
const args3 = ["localhost", 3000] as const;
connect(...args3);  // ✅ type: readonly ["localhost", 3000]
```

### Mistake 2 — Typing rest as `any[]` and losing safety

```ts
// ❌ BAD — rest typed as any[] — defeats the purpose:
function logEvent(event: string, ...data: any[]): void {
  console.log(event, ...data);  // data can be anything
}

logEvent("user.created", undefined, null, Symbol());  // no errors — any is infectious

// ✅ GOOD — type the rest array element type specifically:
function logEvent(event: string, ...data: (string | number | boolean | Record<string, unknown>)[]): void {
  console.log(event, ...data);
}

logEvent("user.created", 1, "admin@dev.io", { role: "admin" });  // ✅
logEvent("user.created", Symbol());  // ❌ caught — Symbol not in union
```

### Mistake 3 — Forgetting rest parameters result in an array (not individual values)

```ts
function processItems(...items: string[]): void {
  // items IS an array — items: string[]

  // ❌ WRONG — treating items as a single string:
  console.log(items.toUpperCase());
  // Error: Property 'toUpperCase' does not exist on type 'string[]'

  // ✅ CORRECT — iterate or use array methods:
  items.forEach(item => console.log(item.toUpperCase()));
  const result = items.join(", ");
}

// Also: passing an array to a rest parameter gives you an array of arrays:
function wrap(...args: string[]): string[][] {
  return [args];  // args is string[] — one flat array of all args
}

wrap("a", "b", "c");         // args = ["a", "b", "c"]
wrap(..."abc".split(""));    // spread — args = ["a", "b", "c"]  ✅
```

---

## Practice exercises

### Exercise 1 — easy

Write these typed variadic functions:

1. `max(...nums: number[]): number` — returns the largest number (throws if called with 0 args)
2. `joinPaths(...segments: string[]): string` — joins path segments with `/`, removes double slashes
3. `pick<T extends object, K extends keyof T>(obj: T, ...keys: K[]): Pick<T, K>` — returns a new object with only the specified keys
4. `mergeHeaders(...headerMaps: Record<string, string>[]): Record<string, string>` — later maps override earlier ones

For each, write 3 call examples. For `pick`, verify TypeScript rejects a key that doesn't exist on the object.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed function wrapper library using rest parameters and generics:

Write these higher-order functions:

1. `memoize<TArgs extends unknown[], TReturn>(fn: (...args: TArgs) => TReturn): (...args: TArgs) => TReturn`
   — caches results by serialized args (use `JSON.stringify(args)` as cache key); return type must match original fn

2. `retry<TArgs extends unknown[], TReturn>(fn: (...args: TArgs) => Promise<TReturn>, attempts: number, delayMs: number): (...args: TArgs) => Promise<TReturn>`
   — wraps any async function with retry logic; returned function has exact same signature as original

3. `timeout<TArgs extends unknown[], TReturn>(fn: (...args: TArgs) => Promise<TReturn>, ms: number): (...args: TArgs) => Promise<TReturn>`
   — wraps any async function; rejects with `new Error("Timeout")` if it doesn't resolve within `ms`

Demonstrate each with a real example (e.g., wrap `findUserById`, `fetchRemoteData`, a slow DB query).

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed SQL query builder using rest parameters and tuples.

Define:
```ts
type SqlValue = string | number | boolean | null;
type SqlParam = SqlValue;

interface QueryResult {
  sql: string;
  params: SqlParam[];
}
```

Write these functions:

1. `sql(template: TemplateStringsArray, ...values: SqlParam[]): QueryResult`
   — a tagged template literal that builds parameterized queries:
   ```ts
   const result = sql`SELECT * FROM users WHERE id = ${userId} AND role = ${"admin"}`;
   // result.sql   → "SELECT * FROM users WHERE id = $1 AND role = $2"
   // result.params → [userId, "admin"]
   ```

2. `sqlAnd(...conditions: QueryResult[]): QueryResult`
   — combines multiple `QueryResult`s with AND, re-indexing parameters:
   ```ts
   const q = sqlAnd(
     sql`status = ${"active"}`,
     sql`role = ${"admin"}`,
     sql`deleted_at IS NULL`,
   );
   // q.sql    → "(status = $1) AND (role = $2) AND (deleted_at IS NULL)"
   // q.params → ["active", "admin"]
   ```

3. `sqlOr(...conditions: QueryResult[]): QueryResult` — same but with OR

4. `sqlIn(column: string, ...values: SqlParam[]): QueryResult`
   — generates `column IN ($1, $2, $3)` with proper param indexing

5. `buildSelect(table: string, where: QueryResult, ...fields: string[]): QueryResult`
   — builds `SELECT fields FROM table WHERE conditions`; if no fields provided, uses `*`

Write 4 complete queries using these builders and log their `sql` and `params`.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Meaning |
|--------|---------|
| `...args: string[]` | Rest param — collects trailing args into `string[]` |
| `...args: (string \| number)[]` | Rest with union element type |
| `...args: [string, number, boolean]` | Tuple rest — fixed-length, positionally typed |
| `fn(...array)` | Spread array as arguments |
| `[...a, ...b]` | Spread in array literal — concatenation |
| `{ ...obj }` | Spread in object literal — shallow merge |
| `as const` + spread | Preserves tuple type for positional spread into non-rest function |

| Rule | Notes |
|------|-------|
| Rest must be last | Can't have params after `...rest` |
| Only one rest param | Can't have two `...rest` params |
| Spread array → rest param | ✅ `string[]` spreads into `...string[]` |
| Spread array → positional params | ❌ unless tuple or `as const` |
| `T extends unknown[]` for generic rest | The correct constraint for variadic generics |
| Rest result is always an array | `items: string[]` inside the function — iterate, don't use scalar methods |

## Connected topics

- **20 — Function types** — rest types `...args: T[]` are part of the function type signature.
- **21 — Optional and default parameters** — rest params are always optional (can be zero args).
- **13 — Arrays and tuples** — tuple types are used as typed rest parameters.
- **26 — Generic functions** — `<TArgs extends unknown[]>` — the variadic generic pattern.
- **23 — Function overloads** — an alternative to variadic params when argument sets are distinct.
