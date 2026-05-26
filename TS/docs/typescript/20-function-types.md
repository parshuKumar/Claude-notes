# 20 — Function Types

## What is this?

In TypeScript, functions are first-class values — you can assign them to variables, pass them as arguments, return them from other functions, and store them in objects. **Function types** describe the shape of a function: what parameters it accepts (and their types) and what it returns.

TypeScript lets you annotate:
- **Individual function declarations** — parameter types + return type inline
- **Function type aliases** — name a function shape with `type`
- **Callback parameters** — describe the shape of a function you receive
- **Return types** — including `void` (no meaningful return) and `never` (never returns)

## Why does it matter?

JavaScript functions are completely untyped. You call `processRequest(handler)` and have no guarantee that `handler` is even callable, let alone that it accepts the right arguments. In TypeScript, function types give you:

- **Call-site safety** — TypeScript verifies every argument at every call
- **Callback contracts** — describe exactly what shape a callback must have before accepting it
- **Return type enforcement** — TypeScript verifies you actually return what you promised
- **Autocomplete** — inside a typed callback, TypeScript knows the parameter types so you get full IDE support
- **Refactor safety** — change a function signature and TypeScript immediately flags every call site that's now broken

## The JavaScript way vs the TypeScript way

```js
// JavaScript — zero safety on function signatures
function processUsers(users, transformer, onError) {
  // transformer could be undefined, not a function, wrong number of args — no warning
  return users.map(user => {
    try {
      return transformer(user);
    } catch (err) {
      onError(err);  // onError might not exist, might not accept an error arg
    }
  });
}

// Called completely wrong — no errors until runtime:
processUsers(users, "not a function", null);
```

```ts
// TypeScript — every function shape is a contract
type UserTransformer<T> = (user: User) => T;
type ErrorHandler = (err: Error) => void;

function processUsers<T>(
  users: User[],
  transformer: UserTransformer<T>,
  onError: ErrorHandler,
): T[] {
  return users.flatMap(user => {
    try {
      return [transformer(user)];
    } catch (err) {
      onError(err instanceof Error ? err : new Error(String(err)));
      return [];
    }
  });
}

// ❌ Wrong argument types caught immediately:
processUsers(users, "not a function", null);
// Error: Argument of type 'string' is not assignable to parameter of type 'UserTransformer<unknown>'
```

---

## Syntax

```ts
// ── FUNCTION DECLARATION — inline parameter + return type annotation ───────
function add(a: number, b: number): number {
  return a + b;
}

// ── FUNCTION EXPRESSION ────────────────────────────────────────────────────
const multiply = function(a: number, b: number): number {
  return a * b;
};

// ── ARROW FUNCTION ─────────────────────────────────────────────────────────
const divide = (a: number, b: number): number => a / b;

// ── FUNCTION TYPE ALIAS ────────────────────────────────────────────────────
type MathOperation = (a: number, b: number) => number;
const subtract: MathOperation = (a, b) => a - b;  // TypeScript infers param types

// ── FUNCTION AS INTERFACE METHOD ──────────────────────────────────────────
interface Calculator {
  add(a: number, b: number): number;       // method syntax
  subtract: (a: number, b: number) => number; // property function syntax
}

// ── VOID RETURN TYPE ───────────────────────────────────────────────────────
function logRequest(method: string, path: string): void {
  console.log(`${method} ${path}`);
  // No return value — or return; with no expression
}

// ── NEVER RETURN TYPE ──────────────────────────────────────────────────────
function throwHttpError(status: number, message: string): never {
  throw new Error(`HTTP ${status}: ${message}`);
  // TypeScript knows execution never reaches past this function
}

function infiniteLoop(): never {
  while (true) { /* runs forever */ }
}

// ── OPTIONAL PARAMETERS ────────────────────────────────────────────────────
function createUser(email: string, name: string, role?: "admin" | "viewer"): User {
  return { id: Date.now(), email, name, role: role ?? "viewer", createdAt: new Date(), updatedAt: new Date() };
}

// ── DEFAULT PARAMETERS ─────────────────────────────────────────────────────
function paginate(page: number = 1, pageSize: number = 20): { skip: number; take: number } {
  return { skip: (page - 1) * pageSize, take: pageSize };
}

// ── REST PARAMETERS ────────────────────────────────────────────────────────
function logAll(level: string, ...messages: string[]): void {
  messages.forEach(msg => console.log(`[${level}] ${msg}`));
}
```

---

## How it works — concept by concept

### Parameter types

TypeScript checks every argument at every call site. The parameter name in the type annotation is irrelevant — only the position and type matter:

```ts
function sendEmail(to: string, subject: string, body: string): Promise<void> {
  // ...
  return Promise.resolve();
}

sendEmail("user@dev.io", "Welcome", "Hello!");   // ✅
sendEmail(42, "Welcome", "Hello!");               // ❌ number not assignable to string
sendEmail("user@dev.io", "Welcome");              // ❌ missing third argument
```

### Return type annotations

You can let TypeScript infer the return type, or declare it explicitly. Explicit is better for public functions — it documents intent and catches errors when your implementation doesn't match:

```ts
// Inferred — TypeScript figures out the return type:
function getPort() {
  return 3000;  // inferred: number
}

// Explicit — you declare what it should return:
function getUserById(id: number): User | null {
  // TypeScript verifies the return statement matches User | null
  return null;  // ✅
}

// ❌ Mismatch caught immediately:
function getCount(): number {
  return "many"; // ❌ 'string' is not assignable to type 'number'
}
```

### `void` vs `undefined`

These are subtly different:

```ts
// void — the function doesn't return a meaningful value
// Callers should not use the return value
function logError(err: Error): void {
  console.error(err.message);
  // Can return nothing, or return undefined explicitly
}

// A void function CAN return undefined:
function doSomething(): void {
  return undefined;  // ✅ fine
  return;            // ✅ fine
  return 42;         // ❌ '42' not assignable to type 'void'
}

// undefined — the function explicitly returns the value undefined:
function findIndex(arr: string[], target: string): number | undefined {
  const i = arr.indexOf(target);
  return i === -1 ? undefined : i;  // explicitly returns undefined as a meaningful signal
}

// Key difference in callback context:
type VoidCallback = () => void;
const cb: VoidCallback = () => 42;  // ✅ — void callbacks ignore return value
// When a callback is typed void, TypeScript doesn't enforce what the function returns
// This lets you pass array.forEach a function that happens to return something
[1, 2, 3].forEach((n): void => n * 2);  // ✅ forEach ignores return value
```

### `never` — the bottom type

`never` means a function **never produces a value at the call site** — either it throws, or it loops forever. It's the bottom of TypeScript's type hierarchy: `never` is assignable to everything, but nothing is assignable to `never`.

```ts
// Throws unconditionally:
function throwError(message: string): never {
  throw new Error(message);
}

// Infinite loop:
function startServer(): never {
  while (true) {
    // process requests forever
  }
}

// never in exhaustiveness checking — the key backend use case:
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

function handleMethod(method: HttpMethod): string {
  switch (method) {
    case "GET":    return "read";
    case "POST":   return "create";
    case "PUT":    return "update";
    case "DELETE": return "delete";
    default:
      // If all cases are handled, method is 'never' here
      // If you add "PATCH" to HttpMethod but forget to add a case, TypeScript errors here:
      const exhaustivenessCheck: never = method;
      throw new Error(`Unhandled method: ${method}`);
  }
}
```

### Function type aliases as parameters (callbacks)

```ts
// Type the callback shape:
type RouteHandler = (req: Request, res: Response) => void | Promise<void>;
type Middleware = (req: Request, res: Response, next: () => void) => void | Promise<void>;
type Predicate<T> = (item: T) => boolean;
type Mapper<T, U> = (item: T, index: number) => U;
type AsyncMapper<T, U> = (item: T) => Promise<U>;

// Using them:
function registerRoute(method: string, path: string, handler: RouteHandler): void {
  // TypeScript knows handler is (req, res) => void | Promise<void>
}

async function mapAsync<T, U>(items: T[], fn: AsyncMapper<T, U>): Promise<U[]> {
  return Promise.all(items.map(fn));
}

// Filter with a typed predicate:
function filterUsers(users: User[], predicate: Predicate<User>): User[] {
  return users.filter(predicate);
}

const adminUsers = filterUsers(users, user => user.role === "admin");
//                                     ^^^^ TypeScript knows user: User here
```

### Function overloads

When a function can accept different argument shapes and returns different types based on them, use overload signatures:

```ts
// Overload signatures (no body):
function findUser(id: number): Promise<User | null>;
function findUser(email: string): Promise<User | null>;
function findUser(criteria: { id?: number; email?: string }): Promise<User | null>;

// Implementation signature (must handle all cases, not exported as a type):
function findUser(
  criteria: number | string | { id?: number; email?: string }
): Promise<User | null> {
  if (typeof criteria === "number") {
    return userRepo.findById(criteria);
  }
  if (typeof criteria === "string") {
    return userRepo.findByEmail(criteria);
  }
  if (criteria.id) return userRepo.findById(criteria.id);
  if (criteria.email) return userRepo.findByEmail(criteria.email);
  return Promise.resolve(null);
}

// TypeScript shows only the overload signatures to callers:
findUser(1);                        // ✅ overload 1
findUser("admin@dev.io");           // ✅ overload 2
findUser({ id: 1 });               // ✅ overload 3
findUser(true);                     // ❌ no overload matches boolean
```

---

## Example 1 — basic

```ts
// A collection of typed utility functions for a backend

// Type aliases for common function shapes:
type Predicate<T> = (item: T) => boolean;
type Comparator<T> = (a: T, b: T) => number;
type Transformer<T, U> = (item: T) => U;
type AsyncOperation<T, U> = (input: T) => Promise<U>;
type ErrorHandler = (err: Error, context?: Record<string, unknown>) => void;

// Generic filter with typed predicate:
function filterItems<T>(items: T[], predicate: Predicate<T>): T[] {
  return items.filter(predicate);
}

// Generic sort with typed comparator:
function sortItems<T>(items: T[], comparator: Comparator<T>): T[] {
  return [...items].sort(comparator);
}

// Generic transform pipeline:
function transformAll<T, U>(items: T[], transformer: Transformer<T, U>): U[] {
  return items.map(transformer);
}

// Async operation with error handling:
async function tryOperation<T, U>(
  input: T,
  operation: AsyncOperation<T, U>,
  onError: ErrorHandler,
): Promise<U | null> {
  try {
    return await operation(input);
  } catch (err) {
    onError(err instanceof Error ? err : new Error(String(err)), { input });
    return null;
  }
}

// Concrete usage with User type:
const activeUsers = filterItems(users, u => !u.deletedAt);
const sortedByName = sortItems(users, (a, b) => a.name.localeCompare(b.name));
const emails = transformAll(users, u => u.email);  // string[]

const userId = await tryOperation(
  42,
  async (id) => userRepo.findById(id),
  (err, ctx) => logger.error(err.message, ctx),
);
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response, NextFunction } from 'express';

// ── Core function type aliases ──────────────────────────────────────────────
type RouteHandler = (req: Request, res: Response) => Promise<void>;
type Middleware = (req: Request, res: Response, next: NextFunction) => Promise<void>;
type RequestValidator<T> = (body: unknown) => T;  // throws on invalid, returns T on valid
type ResponseSerializer<T> = (data: T) => Record<string, unknown>;

// ── Middleware factory functions ────────────────────────────────────────────

// Returns a Middleware — the factory itself has a clear return type:
function requireAuth(
  verifyToken: (token: string) => Promise<number | null>
): Middleware {
  return async (req, res, next) => {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith("Bearer ")) {
      res.status(401).json({ error: "Missing or malformed auth token" });
      return;
    }
    const userId = await verifyToken(authHeader.slice(7));
    if (userId === null) {
      res.status(401).json({ error: "Invalid or expired token" });
      return;
    }
    (req as Request & { userId: number }).userId = userId;
    next();
  };
}

function validateBody<T>(validator: RequestValidator<T>): Middleware {
  return async (req, res, next) => {
    try {
      (req as Request & { validatedBody: T }).validatedBody = validator(req.body);
      next();
    } catch (err) {
      res.status(400).json({ error: (err as Error).message });
    }
  };
}

// ── Route handler factories — return RouteHandler ─────────────────────────

function makeGetUserHandler(
  findUser: (id: number) => Promise<User | null>
): RouteHandler {
  return async (req, res) => {
    const id = parseInt(req.params.id, 10);
    if (isNaN(id)) {
      res.status(400).json({ error: "Invalid user ID" });
      return;
    }
    const user = await findUser(id);
    if (!user) {
      res.status(404).json({ error: `User ${id} not found` });
      return;
    }
    res.json({ success: true, data: user });
  };
}

function makeCreateUserHandler(
  createUser: (data: CreateUserBody) => Promise<User>,
  validate: RequestValidator<CreateUserBody>,
): RouteHandler {
  return async (req, res) => {
    try {
      const body = validate(req.body);
      const user = await createUser(body);
      res.status(201).json({ success: true, data: user });
    } catch (err) {
      res.status(400).json({ error: (err as Error).message });
    }
  };
}

// ── never — exhaustiveness guard on status codes ───────────────────────────
type ApiErrorCode = "NOT_FOUND" | "UNAUTHORIZED" | "VALIDATION_ERROR" | "INTERNAL_ERROR";

function errorCodeToStatus(code: ApiErrorCode): number {
  switch (code) {
    case "NOT_FOUND":         return 404;
    case "UNAUTHORIZED":      return 401;
    case "VALIDATION_ERROR":  return 400;
    case "INTERNAL_ERROR":    return 500;
    default:
      const _exhaustive: never = code;
      throw new Error(`Unhandled error code: ${code}`);
  }
}

// ── void — logging functions that intentionally return nothing ─────────────
function logIncomingRequest(req: Request): void {
  console.log(`→ ${req.method} ${req.path}`, {
    ip: req.ip,
    userAgent: req.headers["user-agent"],
  });
}

function logOutgoingResponse(req: Request, statusCode: number, duration: number): void {
  console.log(`← ${req.method} ${req.path} ${statusCode} (${duration}ms)`);
}
```

---

## Common mistakes

### Mistake 1 — Confusing `void` and `undefined` as return types

```ts
// ❌ WRONG — annotating a function as returning undefined when void is correct:
function logMessage(msg: string): undefined {
  console.log(msg);
  // Error: Function lacks ending return statement and return type does not include 'undefined'
  // You'd have to add: return undefined;
}

// ✅ RIGHT — use void when the return value is not meaningful:
function logMessage(msg: string): void {
  console.log(msg);
  // No return needed
}

// ✅ RIGHT — use undefined when it IS the meaningful return signal:
function findFirst(arr: string[], target: string): string | undefined {
  return arr.find(item => item === target);  // Array.find returns T | undefined
}
```

### Mistake 2 — Using `never` when you mean `void`

```ts
// ❌ WRONG — never means the function NEVER returns
// If you throw conditionally, the return type is NOT never:
function maybeThrow(value: string | null): never {
  if (value === null) {
    throw new Error("Value is null");
  }
  // ❌ Error: Function lacks ending return statement — code CAN return here (when value is not null)
  // TypeScript correctly rejects this because the function sometimes returns normally
}

// ✅ RIGHT — void (returns undefined when value is not null):
function maybeThrow(value: string | null): void {
  if (value === null) throw new Error("Value is null");
  // implicitly returns undefined — fine for void
}

// ✅ RIGHT — never for unconditional throws only:
function assertNonNull(value: string | null, fieldName: string): never {
  throw new Error(`${fieldName} must not be null`);
  // Always throws — never returns
}
```

### Mistake 3 — Not annotating callback types, losing autocomplete

```ts
// ❌ MISSING — callback parameter has no type:
function processItems(items: unknown[], callback: Function): unknown[] {
  // 'Function' is basically any — no type safety, no autocomplete inside callback
  return items.map(callback);
}

// ❌ ALSO MISSING — inline with 'any':
users.forEach((user: any) => {
  user.eamil;  // typo — no error because type is 'any'
});

// ✅ RIGHT — type the callback parameter precisely:
function processItems<T, U>(items: T[], callback: (item: T, index: number) => U): U[] {
  return items.map(callback);
}

// ✅ RIGHT — typed inline callback:
users.forEach((user: User) => {
  user.eamil;  // ❌ TypeScript catches: Property 'eamil' does not exist on type 'User'
});

// ✅ EVEN BETTER — TypeScript infers 'user: User' from the array type:
const users: User[] = [];
users.forEach(user => {
  user.eamil;  // ❌ Still caught — TypeScript infers user: User from users: User[]
});
```

---

## Practice exercises

### Exercise 1 — easy

Write these typed functions:

1. `clamp(value: number, min: number, max: number): number` — returns value constrained between min and max
2. `formatCurrency(amount: number, currency: "USD" | "EUR" | "GBP" | "INR"): string` — returns `"$42.00"` style strings
3. `chunk<T>(arr: T[], size: number): T[][]` — splits array into chunks of given size
4. `debounce<T extends (...args: unknown[]) => void>(fn: T, delayMs: number): T` — returns a debounced version of fn (just type the signature correctly — implementation can be a stub)
5. `assertNever(value: never, message?: string): never` — throws an error (use for exhaustiveness checking)

For each function, write at least two call examples: one valid, one that TypeScript should reject.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed event emitter from scratch using function types:

```
EventMap — a Record<string, unknown> describing all event names and their payload types
Listener<T> — a function type: (payload: T) => void
```

Define a generic `EventEmitter<TEvents extends Record<string, unknown>>` class with:
- `on<K extends keyof TEvents>(event: K, listener: Listener<TEvents[K]>): void`
- `off<K extends keyof TEvents>(event: K, listener: Listener<TEvents[K]>): void`
- `emit<K extends keyof TEvents>(event: K, payload: TEvents[K]): void`
- `once<K extends keyof TEvents>(event: K, listener: Listener<TEvents[K]>): void`

Define a concrete `AppEvents` type:
```ts
type AppEvents = {
  "user.created": { userId: number; email: string };
  "user.deleted": { userId: number };
  "order.placed": { orderId: number; userId: number; total: number };
  "payment.failed": { orderId: number; reason: string };
};
```

Instantiate `EventEmitter<AppEvents>`, subscribe to 3 events, emit them, and verify TypeScript catches wrong payload shapes.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed middleware pipeline using function types (no classes this time — pure functions and type aliases):

1. Define these types:
   - `RequestContext` — `{ requestId: string; method: string; path: string; userId?: number; body: unknown; startTime: number; errors: string[] }`
   - `MiddlewareFn` — `(ctx: RequestContext, next: () => Promise<void>) => Promise<void>`
   - `Pipeline` — `{ use(fn: MiddlewareFn): Pipeline; run(ctx: RequestContext): Promise<RequestContext> }`

2. Write a `createPipeline(): Pipeline` factory function (no class — just returns an object satisfying the `Pipeline` interface). Internally, store an array of `MiddlewareFn[]`. `run` should chain them: call middleware[0] with a `next` that calls middleware[1] with a `next` that calls middleware[2], etc. (recursive or index-based iteration).

3. Write five `MiddlewareFn` functions:
   - `requestIdMiddleware` — sets `ctx.requestId = crypto.randomUUID()` then calls `next()`
   - `timingMiddleware` — records start, calls `next()`, then pushes elapsed time into `ctx.errors` (use errors array as a general log for this exercise)
   - `authMiddleware` — if `ctx.userId` is undefined, pushes `"Unauthorized"` to `ctx.errors` and does NOT call `next()`; otherwise calls `next()`
   - `bodyValidatorMiddleware` — if `ctx.body` is null or undefined, pushes `"Missing body"` to `ctx.errors` and does NOT call `next()`; otherwise calls `next()`
   - `loggerMiddleware` — pushes `"[LOG] METHOD PATH"` to `ctx.errors` (log array) then calls `next()`

4. Compose a pipeline from all five, run it twice:
   - Once with `{ userId: 99, body: { name: "Parsh" } }` — should flow all the way through
   - Once with `{ userId: undefined, body: null }` — should be stopped by `authMiddleware`
   - Log `ctx.errors` for both runs to show what happened

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Meaning |
|--------|---------|
| `(a: string): number` | Inline function type |
| `type F = (a: string) => number` | Named function type alias |
| `void` | No meaningful return value |
| `never` | Function never returns (always throws or loops) |
| `param?: string` | Optional parameter — `string \| undefined` |
| `param = "default"` | Default parameter |
| `...items: string[]` | Rest parameter — collects remaining args |
| `Function` | Avoid — use specific function type instead |

| Return type | When to use |
|-------------|-------------|
| `void` | Side-effect function, return value not used |
| `undefined` | Function explicitly returns `undefined` as a signal |
| `never` | Always throws or runs forever — never returns |
| `T \| null` | May return a value or null (not found, etc.) |
| `Promise<T>` | Async function returning T |
| `Promise<void>` | Async function with no meaningful return |

| Rule | Notes |
|------|-------|
| Annotate public function return types explicitly | Documents intent, catches implementation mismatches |
| Never use `Function` as a type | Too broad — use `(args) => returnType` instead |
| `void` callbacks are permissive | A `() => number` satisfies `() => void` |
| `never` = exhaustiveness check | Assign unhandled union member to `never` in default branch |
| Infer simple private/internal functions | Return type inference is fine for short internal functions |

## Connected topics

- **21 — Optional and default parameters** — `param?` and `param = default` in depth.
- **22 — Rest parameters and spread** — `...args: T[]` and how spread interacts with tuples.
- **23 — Function overloads** — multiple call signatures for one function.
- **24 — Higher-order functions** — functions that take and return functions, fully typed.
- **26 — Generic functions** — `<T>(items: T[]): T` — the generic version of function types.
- **13 — Arrays and tuples** — tuples frequently appear as typed rest/spread in function signatures.
