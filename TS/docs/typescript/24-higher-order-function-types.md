# 24 — Higher-Order Function Types

## What is this?

A **higher-order function** is a function that either:
- **Takes a function as a parameter** (a callback), or
- **Returns a function**, or
- **Both**

You already use these constantly in JavaScript: `Array.map`, `Array.filter`, `Array.reduce`, Express middleware factories, `setTimeout`, event handlers — all higher-order functions. TypeScript lets you type them precisely: you declare exactly what shape of function a parameter must be, and exactly what shape of function is returned.

This topic covers:
- Typing callback parameters
- Typing functions that return functions (factory functions, currying, decorators)
- Generic higher-order functions that preserve the types of whatever they wrap

## Why does it matter?

In JavaScript, higher-order functions are untyped. You call `users.map(transformFn)` and TypeScript has no idea what `transformFn` returns, so the result is `any[]`. You write a middleware factory and the returned function has no type — no autocomplete, no safety inside the callback.

With typed higher-order functions:
- `users.map(transformFn)` returns the correct typed array based on `transformFn`'s return type
- Middleware factories return `Middleware` — their inner function is fully typed
- Generic wrappers like `withRetry(fn)` return a function with the **exact same signature** as `fn` — no type information lost
- Callbacks get full autocomplete inside their body because TypeScript infers parameter types from the function type

## The JavaScript way vs the TypeScript way

```js
// JavaScript — callbacks and returned functions have no type info
function withAuth(handler) {
  return (req, res) => {
    if (!req.user) { res.status(401).json({ error: "Unauthorized" }); return; }
    handler(req, res); // handler could be anything
  };
}

function mapUsers(users, transform) {
  return users.map(transform); // transform: unknown, result: any[]
}
```

```ts
// TypeScript — every function shape is a precise contract
type RouteHandler = (req: Request, res: Response) => Promise<void>;
type Middleware = (req: Request, res: Response, next: NextFunction) => Promise<void>;

function withAuth(handler: RouteHandler): RouteHandler {
  return async (req, res) => {
    const authedReq = req as Request & { userId?: number };
    if (!authedReq.userId) {
      res.status(401).json({ error: "Unauthorized" });
      return;
    }
    await handler(req, res);
  };
}

function mapUsers<T>(users: User[], transform: (user: User) => T): T[] {
  return users.map(transform);
}

const emails = mapUsers(users, u => u.email);    // type: string[]    ✅
const ids    = mapUsers(users, u => u.id);       // type: number[]    ✅
const roles  = mapUsers(users, u => u.role);     // type: ("admin" | "editor" | "viewer")[] ✅
```

---

## Syntax

```ts
// ── FUNCTION TAKING A CALLBACK ─────────────────────────────────────────────
function applyToAll<T, U>(items: T[], fn: (item: T) => U): U[] {
  return items.map(fn);
}

// ── FUNCTION RETURNING A FUNCTION ─────────────────────────────────────────
function multiplier(factor: number): (value: number) => number {
  return (value) => value * factor;
}
const double = multiplier(2);  // type: (value: number) => number
double(5);                     // 10

// ── FUNCTION TYPE ALIAS FOR A CALLBACK ─────────────────────────────────────
type Predicate<T> = (item: T, index: number) => boolean;
type Mapper<T, U> = (item: T) => U;
type Reducer<T, U> = (accumulator: U, current: T, index: number) => U;
type AsyncMapper<T, U> = (item: T) => Promise<U>;

// ── MIDDLEWARE FACTORY — returns a function ────────────────────────────────
type ExpressMiddleware = (req: Request, res: Response, next: NextFunction) => void;

function rateLimit(requestsPerMinute: number): ExpressMiddleware {
  const counts = new Map<string, number>();
  return (req, res, next) => {
    const ip = req.ip ?? "unknown";
    const count = (counts.get(ip) ?? 0) + 1;
    counts.set(ip, count);
    if (count > requestsPerMinute) {
      res.status(429).json({ error: "Rate limit exceeded" });
      return;
    }
    next();
  };
}

// ── GENERIC WRAPPER PRESERVING SIGNATURE ───────────────────────────────────
function withLogging<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => TReturn,
  label: string,
): (...args: TArgs) => TReturn {
  return (...args) => {
    console.log(`Calling ${label}`);
    const result = fn(...args);
    console.log(`${label} returned`);
    return result;
  };
}
```

---

## How it works — pattern by pattern

### Pattern 1 — Typed callback parameters

The simplest higher-order function: accepts a callback and calls it:

```ts
type ErrorHandler = (err: Error, context?: Record<string, unknown>) => void;
type SuccessHandler<T> = (data: T) => void;

function executeOperation<T>(
  operation: () => T,
  onSuccess: SuccessHandler<T>,
  onError: ErrorHandler,
): void {
  try {
    const result = operation();
    onSuccess(result);
  } catch (err) {
    onError(err instanceof Error ? err : new Error(String(err)));
  }
}

// TypeScript infers callback param types from the function type:
executeOperation(
  () => ({ userId: 1, email: "p@dev.io" }),
  (data) => {                   // TypeScript infers data: { userId: number; email: string }
    console.log(data.email);    // ✅ full autocomplete
    console.log(data.nonExist); // ❌ property doesn't exist
  },
  (err) => console.error(err.message), // err: Error ✅
);
```

### Pattern 2 — Factory functions (functions that return functions)

A factory produces a configured function. TypeScript must type both the factory parameters and the returned function:

```ts
// Factory: takes configuration, returns a configured handler
type RouteHandler = (req: Request, res: Response) => Promise<void>;

function createPaginatedListHandler<T>(
  fetchFn: (page: number, pageSize: number) => Promise<{ items: T[]; total: number }>,
  defaultPageSize: number = 20,
): RouteHandler {
  return async (req, res) => {
    const page = parseInt(req.query.page as string, 10) || 1;
    const pageSize = parseInt(req.query.pageSize as string, 10) || defaultPageSize;
    const result = await fetchFn(page, pageSize);
    const totalPages = Math.ceil(result.total / pageSize);
    res.json({
      success: true,
      data: result.items,
      pagination: { page, pageSize, total: result.total, totalPages },
    });
  };
}

// Each call produces a correctly typed route handler:
const listUsersHandler = createPaginatedListHandler(
  (page, pageSize) => userRepo.findAll(page, pageSize),
);

const listOrdersHandler = createPaginatedListHandler(
  (page, pageSize) => orderRepo.findAll(page, pageSize),
  10,  // override default page size
);
```

### Pattern 3 — Generic wrappers preserving the original signature

This is the most powerful pattern — wrap any function and return a function with the **identical type signature**, adding behaviour around it:

```ts
// The key: TArgs extends unknown[] captures the full parameter tuple
// The return type exactly mirrors the original function's return type

function withCache<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => TReturn,
  keyFn: (...args: TArgs) => string,  // computes a cache key from the args
  ttlMs: number = 60_000,
): (...args: TArgs) => TReturn {
  const cache = new Map<string, { value: TReturn; expiresAt: number }>();
  return (...args: TArgs): TReturn => {
    const key = keyFn(...args);
    const entry = cache.get(key);
    if (entry && Date.now() < entry.expiresAt) return entry.value;
    const value = fn(...args);
    cache.set(key, { value, expiresAt: Date.now() + ttlMs });
    return value;
  };
}

function getUserName(userId: number, includeRole: boolean): string {
  return `User ${userId}${includeRole ? " (admin)" : ""}`;
}

const cachedGetUserName = withCache(
  getUserName,
  (userId, includeRole) => `${userId}:${includeRole}`,
);
// type of cachedGetUserName: (userId: number, includeRole: boolean) => string
// Exact same signature as getUserName — no information lost

cachedGetUserName(1, true);   // ✅ → "User 1 (admin)" (cached after first call)
cachedGetUserName("1", true); // ❌ string not assignable to number — preserved!
```

### Pattern 4 — Currying (returning a function from a function from a function)

```ts
// Two-step curried function:
function curriedFilter<T>(predicate: Predicate<T>): (items: T[]) => T[] {
  return (items) => items.filter(predicate);
}

type Predicate<T> = (item: T) => boolean;

const getActiveUsers = curriedFilter<User>(u => !u.deletedAt);
// type: (items: User[]) => User[]

const admins = curriedFilter<User>(u => u.role === "admin");
// type: (items: User[]) => User[]

const activeUsers  = getActiveUsers(allUsers);   // User[]
const adminUsers   = admins(allUsers);            // User[]

// Three-level curry:
function createQueryFilter<T>(
  field: keyof T,
): (operator: "eq" | "gt" | "lt") => (value: T[keyof T]) => Predicate<T> {
  return (operator) => (value) => (item) => {
    switch (operator) {
      case "eq": return item[field] === value;
      case "gt": return (item[field] as number) > (value as number);
      case "lt": return (item[field] as number) < (value as number);
    }
  };
}

const filterByRole = createQueryFilter<User>("role");
const filterByAge  = createQueryFilter<User>("id");

const isAdmin    = filterByRole("eq")("admin");   // Predicate<User>
const isHighId   = filterByAge("gt")(100);         // Predicate<User>
```

### Pattern 5 — Composing and piping

```ts
// Compose: right-to-left
function compose<T>(...fns: Array<(x: T) => T>): (x: T) => T {
  return (x) => fns.reduceRight((acc, fn) => fn(acc), x);
}

// Pipe: left-to-right (more readable for data pipelines)
function pipe<T>(...fns: Array<(x: T) => T>): (x: T) => T {
  return (x) => fns.reduce((acc, fn) => fn(acc), x);
}

// Heterogeneous pipe — each step can change the type:
function typedPipe<A, B, C>(
  f1: (a: A) => B,
  f2: (b: B) => C,
): (a: A) => C {
  return (a) => f2(f1(a));
}

function typedPipe3<A, B, C, D>(
  f1: (a: A) => B,
  f2: (b: B) => C,
  f3: (c: C) => D,
): (a: A) => D {
  return (a) => f3(f2(f1(a)));
}

// Usage:
const processRequest = typedPipe3(
  (raw: string) => JSON.parse(raw) as unknown,
  (parsed: unknown) => parsed as CreateUserBody,
  (body: CreateUserBody) => ({ ...body, role: body.role ?? "viewer" }),
);
// type: (raw: string) => CreateUserBody & { role: string }
```

---

## Example 1 — basic

```ts
// A functional utility library for typed backend data pipelines

type Predicate<T> = (item: T) => boolean;
type Mapper<T, U> = (item: T) => U;
type Reducer<T, U> = (acc: U, item: T) => U;

// Generic filter with typed predicate:
function where<T>(items: T[], predicate: Predicate<T>): T[] {
  return items.filter(predicate);
}

// Generic map:
function select<T, U>(items: T[], mapper: Mapper<T, U>): U[] {
  return items.map(mapper);
}

// Generic reduce:
function aggregate<T, U>(items: T[], reducer: Reducer<T, U>, initial: U): U {
  return items.reduce(reducer, initial);
}

// Group by — returns a factory that groups items:
function groupBy<T>(keyFn: (item: T) => string): (items: T[]) => Record<string, T[]> {
  return (items) => {
    return items.reduce<Record<string, T[]>>((groups, item) => {
      const key = keyFn(item);
      (groups[key] ??= []).push(item);
      return groups;
    }, {});
  };
}

// Usage:
const activeAdmins = where(users, u => u.role === "admin" && !u.deletedAt);
const emailList    = select(users, u => u.email);         // string[]
const userCount    = aggregate(users, (acc) => acc + 1, 0); // number
const byRole       = groupBy<User>(u => u.role)(users);   // Record<string, User[]>

// Composing predicates:
function and<T>(...predicates: Predicate<T>[]): Predicate<T> {
  return (item) => predicates.every(p => p(item));
}

function or<T>(...predicates: Predicate<T>[]): Predicate<T> {
  return (item) => predicates.some(p => p(item));
}

function not<T>(predicate: Predicate<T>): Predicate<T> {
  return (item) => !predicate(item);
}

const isAdmin     : Predicate<User> = u => u.role === "admin";
const isActive    : Predicate<User> = u => u.deletedAt === null;
const isNotDeleted: Predicate<User> = not(u => u.deletedAt !== null);

const activeAdminPredicate = and(isAdmin, isActive);
const activeAdminsFiltered = where(users, activeAdminPredicate);
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response, NextFunction } from 'express';

type RouteHandler  = (req: Request, res: Response) => Promise<void>;
type Middleware     = (req: Request, res: Response, next: NextFunction) => Promise<void>;
type AuthedRequest  = Request & { userId: number; userRole: string };

// ── Middleware factories — functions returning middleware ──────────────────

function requireRole(...allowedRoles: string[]): Middleware {
  return async (req, res, next) => {
    const authedReq = req as AuthedRequest;
    if (!authedReq.userId) {
      res.status(401).json({ error: "Unauthorized" });
      return;
    }
    if (!allowedRoles.includes(authedReq.userRole)) {
      res.status(403).json({ error: "Forbidden", required: allowedRoles });
      return;
    }
    next();
  };
}

function withRequestId(): Middleware {
  return async (req, res, next) => {
    (req as Request & { requestId: string }).requestId = crypto.randomUUID();
    next();
  };
}

function withTiming(label: string): Middleware {
  return async (req, res, next) => {
    const start = Date.now();
    res.on("finish", () => {
      console.log(`[${label}] ${req.method} ${req.path} → ${res.statusCode} (${Date.now() - start}ms)`);
    });
    next();
  };
}

// ── Route handler factories — functions returning route handlers ────────────

function createGetHandler<T>(
  fetchFn: (id: number) => Promise<T | null>,
  entityName: string,
): RouteHandler {
  return async (req, res) => {
    const id = parseInt(req.params.id, 10);
    if (isNaN(id)) {
      res.status(400).json({ error: `Invalid ${entityName} ID` });
      return;
    }
    const entity = await fetchFn(id);
    if (!entity) {
      res.status(404).json({ error: `${entityName} ${id} not found` });
      return;
    }
    res.json({ success: true, data: entity });
  };
}

function createCreateHandler<TBody, TResult>(
  createFn: (body: TBody) => Promise<TResult>,
  validateFn: (body: unknown) => TBody,
): RouteHandler {
  return async (req, res) => {
    try {
      const body = validateFn(req.body);
      const result = await createFn(body);
      res.status(201).json({ success: true, data: result });
    } catch (err) {
      res.status(400).json({ success: false, error: (err as Error).message });
    }
  };
}

// ── Async wrapper — preserves exact signature ──────────────────────────────
function withErrorBoundary<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => Promise<TReturn>,
): (...args: TArgs) => Promise<TReturn | null> {
  return async (...args) => {
    try {
      return await fn(...args);
    } catch (err) {
      console.error("Unhandled error:", (err as Error).message);
      return null;
    }
  };
}

// ── Wiring everything together ─────────────────────────────────────────────
const getUserHandler    = createGetHandler(id => userRepo.findById(id), "User");
const getOrderHandler   = createGetHandler(id => orderRepo.findById(id), "Order");
const createUserHandler = createCreateHandler(
  body => userService.createUser(body),
  body => body as CreateUserBody,  // in production: use zod/yup/joi
);

const safeGetUser = withErrorBoundary(async (id: number) => userRepo.findById(id));
// type: (id: number) => Promise<User | null | null>

// Middleware usage:
const adminOnly  = requireRole("admin");
const editorPlus = requireRole("admin", "editor");
const timing     = withTiming("UserAPI");
const requestId  = withRequestId();
```

---

## Common mistakes

### Mistake 1 — Using `Function` as a callback type (too broad)

```ts
// ❌ BAD — Function type loses all type information:
function runCallback(callback: Function): void {
  callback(42, "extra");  // TypeScript has no idea what callback expects
}

// ❌ ALSO BAD — explicit any:
function runCallback(callback: (...args: any[]) => any): void {}

// ✅ GOOD — type the callback precisely:
function runCallback(callback: (value: number) => void): void {
  callback(42);
}

// ✅ GOOD — generic if callback shape varies:
function runCallback<T>(value: T, callback: (v: T) => void): void {
  callback(value);
}
```

### Mistake 2 — Losing the return type of a wrapper by annotating it too broadly

```ts
async function findUserById(id: number): Promise<User | null> {
  return userRepo.findById(id);
}

// ❌ BAD — wrapper annotated too broadly — returns Promise<any>:
function withLogging(fn: (...args: any[]) => any) {
  return (...args: any[]) => {
    console.log("calling");
    return fn(...args);
  };
}

const loggedFind = withLogging(findUserById);
const result = await loggedFind(1);
// result: any — all type information lost

// ✅ GOOD — generic wrapper preserves exact signature:
function withLogging<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => TReturn,
): (...args: TArgs) => TReturn {
  return (...args) => {
    console.log("calling");
    return fn(...args);
  };
}

const loggedFind2 = withLogging(findUserById);
const result2 = await loggedFind2(1);
// result2: User | null — preserved! ✅
```

### Mistake 3 — Not typing the return function signature of a factory

```ts
// ❌ BAD — return type of factory inferred as Function (or implicit any):
function createHandler(repo: UserRepository) {
  return function(req: any, res: any) {  // 'any' in callback — no safety
    repo.findById(req.params.id).then(u => res.json(u));
  };
}

// ✅ GOOD — explicitly type what the factory returns:
type RouteHandler = (req: Request, res: Response) => Promise<void>;

function createHandler(repo: UserRepository): RouteHandler {
  return async (req, res) => {
    const user = await repo.findById(parseInt(req.params.id, 10));
    if (!user) { res.status(404).json({ error: "Not found" }); return; }
    res.json({ success: true, data: user });
  };
}
// Now callers know exactly what createHandler returns — a fully typed RouteHandler
```

---

## Practice exercises

### Exercise 1 — easy

Write these typed higher-order functions:

1. `once<T extends (...args: unknown[]) => unknown>(fn: T): T`
   — wraps any function so it can only be called once; subsequent calls return the first result silently

2. `debounce<T extends (...args: unknown[]) => void>(fn: T, delayMs: number): T`
   — returns a debounced version of fn (use `setTimeout`; clear previous timer on each new call)

3. `tap<T>(fn: (value: T) => void): (value: T) => T`
   — calls fn with the value as a side effect, then returns the value unchanged (useful for logging in pipelines)

4. `negate<T>(predicate: (item: T) => boolean): (item: T) => boolean`
   — returns a predicate that is the logical opposite of the input predicate

Demonstrate each with a concrete example.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed middleware composition system using higher-order functions:

Define:
```ts
type Ctx = { requestId: string; userId?: number; role?: string; body: unknown; errors: string[]; result?: unknown };
type MiddlewareFn = (ctx: Ctx, next: () => Promise<void>) => Promise<void>;
```

Write these middleware factories (each returns a `MiddlewareFn`):

1. `withRequestId(): MiddlewareFn` — sets `ctx.requestId = crypto.randomUUID()` then calls `next()`
2. `requireAuth(validUserIds: number[]): MiddlewareFn` — if `ctx.userId` is not in `validUserIds`, pushes error and stops; otherwise calls `next()`
3. `requireRole(allowedRoles: string[]): MiddlewareFn` — same for `ctx.role`
4. `withBodyValidation<T>(validator: (body: unknown) => T): MiddlewareFn` — validates `ctx.body`, on error pushes to `ctx.errors` and stops; on success calls `next()`
5. `withErrorCatch(fallback: unknown): MiddlewareFn` — wraps `next()` in try/catch, pushes error message to `ctx.errors` and sets `ctx.result = fallback` if caught

Write a `compose(...middlewares: MiddlewareFn[]): MiddlewareFn` function that chains them in order.

Run a composed pipeline twice — once through, once blocked — and log `ctx`.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a fully typed functional data pipeline library. Define:

```ts
type Pipeline<T> = {
  map<U>(fn: (item: T) => U): Pipeline<U>;
  filter(predicate: (item: T) => boolean): Pipeline<T>;
  flatMap<U>(fn: (item: T) => U[]): Pipeline<U>;
  reduce<U>(fn: (acc: U, item: T) => U, initial: U): U;
  tap(fn: (item: T) => void): Pipeline<T>;
  take(n: number): Pipeline<T>;
  skip(n: number): Pipeline<T>;
  toArray(): T[];
  first(): T | undefined;
  count(): number;
};
```

Write a `pipeline<T>(items: T[]): Pipeline<T>` factory function that returns an object satisfying the `Pipeline<T>` interface. Each `map`, `filter`, `flatMap`, `tap`, `take`, and `skip` should be lazy-evaluated — build up an array of transformation steps, only execute them all when a terminal operation (`toArray`, `first`, `reduce`, `count`) is called.

Demonstrate the pipeline with a `User[]` dataset:
1. Filter active users → map to email → take first 10 → toArray
2. Filter admins → map to `{ id, name }` → toArray
3. All users → reduce to count by role → `Record<string, number>`
4. Filter by role → tap(log each) → map to name → skip 2 → take 5 → toArray

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Pattern | Type syntax |
|---------|-------------|
| Accept a callback | `fn: (item: T) => U` |
| Accept an async callback | `fn: (item: T) => Promise<U>` |
| Return a function | `(): (x: T) => U` |
| Generic wrapper (preserve signature) | `<TArgs extends unknown[], TReturn>(fn: (...args: TArgs) => TReturn)` |
| Predicate | `(item: T) => boolean` |
| Mapper | `(item: T) => U` |
| Reducer | `(acc: U, item: T) => U` |
| Middleware factory | `(config: Config): Middleware` |

| Rule | Notes |
|------|-------|
| Never use `Function` as callback type | Too broad — loses all param and return type info |
| Never use `any` in generic wrappers | Use `TArgs extends unknown[]` to preserve types |
| Type the factory's return value | Explicitly annotate what the factory returns |
| Callbacks get inferred param types | TypeScript infers types from the callback parameter annotation — no need to re-annotate inside |
| `TArgs extends unknown[]` for variadic wrappers | The correct constraint for functions wrapping other functions with unknown signatures |

## Connected topics

- **20 — Function types** — the foundation: every callback and returned function is a function type.
- **22 — Rest parameters and spread** — `TArgs extends unknown[]` uses rest + spread to capture full parameter tuples.
- **26 — Generic functions** — generics are the backbone of all type-preserving wrappers.
- **19 — Implementing interfaces** — factory functions returning objects that implement interfaces.
- **40 — Mapped types** — the functional programming equivalent in the type system.
