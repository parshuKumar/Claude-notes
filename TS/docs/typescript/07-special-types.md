# 07 — Special Types: any, unknown, never, void

## What is this?

TypeScript has four special types that don't represent a specific value like `string` or `number` do — they represent **type-level concepts**: a type you've given up checking (`any`), a type you haven't identified yet (`unknown`), a type that can never exist (`never`), and a function that returns nothing meaningful (`void`). These four come up constantly, and using the wrong one is one of the most damaging mistakes a new TypeScript developer makes.

## Why does it matter?

`any` is the most dangerous keyword in TypeScript. It turns off type checking entirely for that value — silently, with no warning. Developers who come from JavaScript often reach for `any` the moment TypeScript pushes back, effectively destroying the safety they installed TypeScript to get. Understanding `unknown`, `never`, and `void` gives you the right tool for every situation instead of defaulting to `any`.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — everything is implicitly "any"
// No types exist, so JS behaves as if every variable is any:
function parseResponse(response) {
  return response.data.user.email;  // if any of these are undefined → crash at runtime
}
```

```ts
// TypeScript — you choose what each special type means
function parseResponse(response: unknown): string {  // unknown = "I don't know the type yet"
  // TypeScript FORCES you to check before accessing
  if (
    typeof response === "object" &&
    response !== null &&
    "data" in response
  ) {
    // Now TypeScript trusts you have an object with data
    // (full narrowing shown in topic 40)
  }
  return "";
}

// vs the dangerous shortcut:
function parseResponseBad(response: any): string {  // any = "pretend this is fine"
  return response.data.user.email;  // TypeScript says ✅ — but runtime says 💥
}
```

---

## `any` — the escape hatch you should almost never use

### What it does

`any` tells TypeScript: "stop checking this value entirely." You can assign it to anything, read any property off it, call it as a function — TypeScript will not complain. It is a complete bypass of the type system.

```ts
let payload: any = fetchSomething();

// TypeScript allows ALL of these — no errors:
payload.nonExistentProperty;      // ✅ (in TS's eyes) — runtime: undefined
payload.user.email.toLowerCase(); // ✅ (in TS's eyes) — runtime: crash if null
payload();                        // ✅ (in TS's eyes) — runtime: crash if not a function
payload = 42;                     // ✅ reassign to anything
payload = "hello";                // ✅
payload = { a: 1 };               // ✅

// any is CONTAGIOUS — values derived from any are also any:
const email = payload.email;      // email is type 'any'
const upper = email.toUpperCase(); // upper is type 'any' — no checking at all
```

### When `any` appears in the wild

```ts
// 1. Explicit any annotation (avoid)
let userData: any = await fetchUser();

// 2. Implicit any — function param without type (caught by noImplicitAny)
function process(data) { ... }    // data is implicitly any

// 3. JSON.parse() always returns any
const config = JSON.parse(rawJson);  // type: any — TypeScript can't know the shape

// 4. Third-party libraries with poor/missing types
import { legacyLib } from 'old-package';  // might be typed as any
```

### The one legitimate use of `any`

```ts
// Migrating a JavaScript codebase to TypeScript gradually:
// You CAN use any temporarily during migration, but set a deadline to remove it.
// Use // @ts-ignore or any ONLY on files you haven't converted yet.

// Another acceptable use: when you genuinely need to accept anything and
// IMMEDIATELY narrow it — but unknown is better for this (see below)
```

---

## `unknown` — the safe alternative to `any`

### What it does

`unknown` also means "I don't know the type" — but unlike `any`, TypeScript **forces you to narrow** the type before you can use the value. You cannot call methods on `unknown`, cannot access properties, cannot assign it to a typed variable — until you prove what it is.

```ts
let payload: unknown = fetchSomething();

// TypeScript PREVENTS all of these:
payload.email;            // ❌ Object is of type 'unknown'
payload();                // ❌ Object is of type 'unknown'
payload.toLowerCase();    // ❌ Object is of type 'unknown'

// You must narrow first:
if (typeof payload === "string") {
  payload.toLowerCase();  // ✅ TypeScript now knows it's a string
}

if (typeof payload === "object" && payload !== null) {
  // TypeScript knows: it's a non-null object
  // But still doesn't know what properties it has
}
```

### `unknown` vs `any` — the key difference

```ts
// any: TypeScript trusts you blindly
function processAny(value: any) {
  return value.toUpperCase();   // TypeScript: ✅   Runtime: 💥 if value is a number
}

// unknown: TypeScript makes you prove it
function processUnknown(value: unknown) {
  return value.toUpperCase();   // ❌ TypeScript: Object is of type 'unknown'

  // ✅ Correct pattern:
  if (typeof value === "string") {
    return value.toUpperCase(); // ✅ proven to be string
  }
  return "";
}
```

### When to use `unknown`

```ts
// 1. catch blocks — errors in JS catch are always unknown in strict TS
try {
  const data = await fetch('/api/users');
  const json = await data.json();  // json is any — consider asserting type
} catch (error: unknown) {
  // error is unknown — could be Error, string, network failure, anything
  if (error instanceof Error) {
    console.error(error.message);  // ✅ now TypeScript knows it's an Error
  } else {
    console.error(String(error));  // fallback
  }
}

// 2. Validating external data (API responses, JSON.parse, user input)
function parseApiResponse(raw: unknown): { userId: number; email: string } | null {
  if (
    typeof raw === "object" &&
    raw !== null &&
    "userId" in raw &&
    "email" in raw
  ) {
    // At this point TypeScript has narrowed what it can
    // For full validation, use a library like zod
    return raw as { userId: number; email: string };
  }
  return null;
}

// 3. Generic "catch-all" function parameters that you'll validate inside
function logAnything(value: unknown): void {
  console.log(JSON.stringify(value));  // JSON.stringify accepts unknown fine
}
```

---

## `never` — the impossible type

### What it does

`never` represents a value that **can never exist**. A function has return type `never` when it can never finish — either because it throws unconditionally or runs forever. A type is `never` when TypeScript determines a code path is unreachable.

```ts
// A function that always throws — it never returns
function throwHttpError(statusCode: number, message: string): never {
  throw new Error(`HTTP ${statusCode}: ${message}`);
  // No return statement — TypeScript enforces this.
  // 'never' promises: this function will NEVER return a value.
}

// An infinite loop — also never returns
function keepAlive(): never {
  while (true) {
    // poll something
  }
}
```

### `never` in exhaustive checks — the most powerful use

```ts
// never shines when doing exhaustive switch statements on union types
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

function handleMethod(method: HttpMethod): string {
  switch (method) {
    case "GET":    return "Reading resource";
    case "POST":   return "Creating resource";
    case "PUT":    return "Updating resource";
    case "DELETE": return "Deleting resource";
    default:
      // If all cases are handled, 'method' here is type 'never'
      // This is an exhaustive check — if you add "PATCH" to HttpMethod
      // and forget to add a case here, TypeScript will error:
      // "Type 'string' is not assignable to type 'never'"
      const exhaustiveCheck: never = method;
      throw new Error(`Unhandled method: ${exhaustiveCheck}`);
  }
}

// TypeScript catches the missing case:
type HttpMethodWithPatch = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

function handleMethodBad(method: HttpMethodWithPatch): string {
  switch (method) {
    case "GET":    return "Reading";
    case "POST":   return "Creating";
    case "PUT":    return "Updating";
    case "DELETE": return "Deleting";
    // PATCH is missing!
    default:
      const exhaustiveCheck: never = method;  // ❌ Error: Type 'string' is not assignable to type 'never'
      // TypeScript is telling you: you didn't handle all cases
      throw new Error(`Unhandled: ${exhaustiveCheck}`);
  }
}
```

### `never` in type narrowing

```ts
// When TypeScript narrows a type completely, the remaining type is never
function processValue(value: string | number): string {
  if (typeof value === "string") {
    return value.toUpperCase();  // value: string
  }
  if (typeof value === "number") {
    return value.toString();     // value: number
  }
  // TypeScript knows: if we reach here, value is type 'never'
  // because string | number is fully exhausted
  const impossibleValue: never = value;  // ✅ TypeScript agrees this is never
  return impossibleValue;  // never is assignable to any type
}
```

---

## `void` — functions that return nothing

### What it does

`void` is the return type of a function that **intentionally returns no useful value**. It's what you put when a function's job is to cause side effects (logging, sending a response, updating state) rather than compute and return a value.

```ts
// void = "this function runs for its side effects, not its return value"
function sendWelcomeEmail(email: string): void {
  console.log(`Sending welcome email to ${email}`);
  // No return statement needed
  // return;          ← allowed
  // return undefined; ← allowed
  // return "ok";     ← ❌ Not allowed — void means no return value
}

// Common in Express route handlers
import { Request, Response } from 'express';

function healthCheck(req: Request, res: Response): void {
  res.json({ status: "ok" });
  // return;  ← fine with void
  // The function "returns" nothing — it sent a response as a side effect
}
```

### `void` vs `undefined`

```ts
// void and undefined are related but NOT the same:

// A function returning void CAN return undefined:
function logMessage(msg: string): void {
  console.log(msg);
  return undefined;   // ✅ allowed with void
}

// A function typed to return undefined MUST return undefined explicitly:
function noop(): undefined {
  // return;          // ❌ Error: Function missing return statement
  return undefined;   // ✅ must be explicit
}

// In practice: always use void for side-effect functions.
// Use undefined only when you need to explicitly return undefined as a value.
```

### `Promise<void>` for async functions with no return value

```ts
// Async functions that do work but don't produce a return value:
async function deleteUserFromDb(userId: number): Promise<void> {
  await db.query("DELETE FROM users WHERE id = $1", [userId]);
  // No return — just side effects
}

async function sendNotification(userId: number, message: string): Promise<void> {
  await notificationService.send({ userId, message });
  // void means callers can't accidentally use the return value
}

// Usage:
await deleteUserFromDb(42);         // ✅ — don't need to capture the return
const result = await deleteUserFromDb(42);  // result is void — useless, but not an error
```

---

## Side-by-side comparison

| Type | Means | Can access properties? | Use when |
|------|-------|----------------------|----------|
| `any` | "Disable type checking" | Yes — no checks | Almost never. Migrating legacy code only |
| `unknown` | "I don't know yet" | No — must narrow first | External data, catch blocks, JSON.parse |
| `never` | "Cannot exist" | N/A | Throw-only functions, exhaustive checks |
| `void` | "Returns nothing" | N/A | Side-effect functions, event handlers |

---

## Example 1 — basic

```ts
// Demonstrating all four special types in one file

// void — a logging utility
function logRequest(method: string, path: string, statusCode: number): void {
  const timestamp = new Date().toISOString();
  console.log(`[${timestamp}] ${method} ${path} → ${statusCode}`);
}

// never — an error factory that always throws
function throwValidationError(field: string, reason: string): never {
  throw new Error(`Validation failed on '${field}': ${reason}`);
}

// unknown — safely handling JSON.parse output
function safeParseJson(raw: string): unknown {
  try {
    return JSON.parse(raw);
  } catch {
    return null;
  }
}

// any — shown for contrast (don't do this)
function unsafeParseJson(raw: string): any {
  return JSON.parse(raw);   // TypeScript trusts the result — dangerous
}

// Usage:
logRequest("GET", "/api/users", 200);

const parsed = safeParseJson('{"userId": 42}');
if (typeof parsed === "object" && parsed !== null && "userId" in parsed) {
  console.log("Got userId:", (parsed as { userId: number }).userId);
}
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response, NextFunction } from 'express';

// ── 1. void — Express route handlers return nothing ──────────────────────────

async function getUserHandler(
  req: Request<{ userId: string }>,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    const userId = parseInt(req.params.userId, 10);
    // ... fetch user from DB
    res.json({ success: true, data: { id: userId, name: "Parsh" } });
  } catch (error) {
    next(error);   // pass to error handler — no return value needed
  }
}

// ── 2. never — error factory used throughout the app ──────────────────────────

function throwAppError(statusCode: number, message: string): never {
  const error = new Error(message) as Error & { statusCode: number };
  error.statusCode = statusCode;
  throw error;
  // TypeScript knows: nothing after this line is reachable
}

// Usage in a controller:
async function requireAuth(authToken: string | undefined): Promise<{ userId: number }> {
  if (!authToken) {
    throwAppError(401, "Authorization token is required");
    // TypeScript knows throwAppError is never — so it knows
    // the function continues ONLY if authToken exists
  }
  // ... validate token
  return { userId: 42 };
}

// ── 3. unknown — safely handling catch blocks ─────────────────────────────────

function handleError(error: unknown): { statusCode: number; message: string } {
  if (error instanceof Error) {
    // Narrow: it's a real Error object
    const appError = error as Error & { statusCode?: number };
    return {
      statusCode: appError.statusCode ?? 500,
      message: appError.message,
    };
  }

  if (typeof error === "string") {
    return { statusCode: 500, message: error };
  }

  return { statusCode: 500, message: "An unknown error occurred" };
}

// ── 4. unknown — validating API request body ──────────────────────────────────

type CreateUserBody = { name: string; email: string };

function isCreateUserBody(value: unknown): value is CreateUserBody {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof (value as CreateUserBody).name === "string" &&
    typeof (value as CreateUserBody).email === "string"
  );
}

async function createUserHandler(req: Request, res: Response): Promise<void> {
  const body: unknown = req.body;   // treat req.body as unknown for safety

  if (!isCreateUserBody(body)) {
    res.status(400).json({ success: false, error: "Invalid request body" });
    return;
  }

  // TypeScript now knows body is { name: string; email: string }
  const { name, email } = body;
  res.status(201).json({ success: true, data: { name, email } });
}
```

---

## Common mistakes

### Mistake 1 — Using `any` to "fix" TypeScript errors

```ts
// ❌ WRONG — using any to silence a legitimate type error
async function loadConfig(): Promise<any> {    // any defeats the purpose
  const raw = await fs.readFile('config.json', 'utf8');
  return JSON.parse(raw);   // returns any — no type safety downstream
}

const config = await loadConfig();
const port = config.prot;   // typo "prot" — TypeScript says ✅ — runtime: undefined

// ✅ RIGHT — define the shape and assert or validate
type AppConfig = { port: number; dbUrl: string; jwtSecret: string };

async function loadConfig(): Promise<AppConfig> {
  const raw = await fs.readFile('config.json', 'utf8');
  return JSON.parse(raw) as AppConfig;   // assertion — or use zod for full validation
}

const config = await loadConfig();
const port = config.prot;   // ❌ TypeScript: Property 'prot' does not exist on type 'AppConfig'
```

### Mistake 2 — Putting `never` as a return type on normal functions

```ts
// ❌ WRONG — never on a function that sometimes returns
function findUser(userId: number): never {   // ❌ wrong — this function returns a User
  const user = db.find(userId);
  if (!user) throw new Error("Not found");
  return user;  // ❌ Error: Type 'User' is not assignable to type 'never'
}

// ✅ RIGHT — never only for functions that ALWAYS throw or loop forever
function assertUserExists(user: User | null): never | User {  // still wrong
  if (!user) throw new Error("User required");
  return user;
}

// Better pattern: use asserts (topic 63) or return the type normally
function getUserOrThrow(userId: number): User {   // ✅ return type is User
  const user = db.find(userId);
  if (!user) throw new Error(`User ${userId} not found`);  // might throw
  return user;  // ✅ TypeScript is fine — the throw path exits before this
}
```

### Mistake 3 — Thinking `void` and `undefined` are the same

```ts
// ❌ WRONG — using undefined where void is expected (and vice versa)
const callbacks: Array<() => void> = [];

// A function returning undefined is assignable to () => void — that's fine:
callbacks.push(() => undefined);   // ✅ OK

// But a function typed to return void cannot be used where undefined is needed:
function getVoidValue(): void {
  return;
}

const val: undefined = getVoidValue();  // ❌ Error: Type 'void' is not assignable to type 'undefined'

// ✅ Rule of thumb:
// void — use for function return types (side-effect functions)
// undefined — use when you specifically need the value undefined
```

---

## Practice exercises

### Exercise 1 — easy

Write three functions:
1. `logApiCall(method: string, endpoint: string, durationMs: number): void` — logs a formatted string to the console. No return value.
2. `throwNotFound(resourceName: string): never` — throws a `new Error(\`${resourceName} not found\`)`. Mark the return type as `never`.
3. `parseUnknown(value: unknown): string` — returns `"string"` if value is a string, `"number"` if a number, `"boolean"` if boolean, `"null"` if null, `"object"` if a non-null object, and `"other"` for anything else.

Call all three functions and verify TypeScript accepts the calls.

```ts
// Write your code here
```

### Exercise 2 — medium

Write a function `safeJsonParse<T>(raw: string, validator: (value: unknown) => value is T): T | null`. It should:
- Parse `raw` with `JSON.parse` (return type is `any` — immediately treat it as `unknown`)
- Pass the parsed value to `validator`
- Return the value as `T` if validation passes, `null` otherwise
- Wrap everything in a try/catch — return `null` if parsing fails

Then write a type `UserConfig = { port: number; dbUrl: string }` and a validator function `isUserConfig(value: unknown): value is UserConfig` that checks both fields exist and have the right types.

Call `safeJsonParse` with a valid JSON string and an invalid one, storing results with correct types.

```ts
// Write your code here
```

### Exercise 3 — hard

You're building a typed error handling system for an Express API. Write:

1. A type `AppErrorCode` as a union of string literals: `"NOT_FOUND"` | `"UNAUTHORIZED"` | `"FORBIDDEN"` | `"VALIDATION_ERROR"` | `"INTERNAL_ERROR"`.
2. A class `AppError` that extends `Error` with additional properties: `statusCode: number` and `errorCode: AppErrorCode`. Constructor takes `message`, `statusCode`, `errorCode`.
3. A function `throwAppError(message: string, statusCode: number, errorCode: AppErrorCode): never` that creates and throws an `AppError`.
4. A function `handleCaughtError(error: unknown): { statusCode: number; errorCode: string; message: string }` that:
   - If `error` is an `AppError`, returns its properties
   - If `error` is a generic `Error`, returns `{ statusCode: 500, errorCode: "INTERNAL_ERROR", message: error.message }`
   - Otherwise returns `{ statusCode: 500, errorCode: "INTERNAL_ERROR", message: "Unknown error" }`
5. A function `getStatusText(errorCode: AppErrorCode): string` using a **switch statement with an exhaustive `never` check** for the default case.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Type | When to use | When NOT to use |
|------|------------|----------------|
| `any` | Migrating JS to TS (temporary only) | Silencing TypeScript errors |
| `unknown` | External data, catch blocks, JSON.parse | When you know the type — use the real type |
| `never` | Always-throw functions, exhaustive switch | Normal functions that sometimes return |
| `void` | Side-effect function return types | When you need to return undefined as a value |

| Pattern | Code |
|---------|------|
| Safe catch block | `catch (error: unknown) { if (error instanceof Error) { ... } }` |
| JSON parse safely | `const data: unknown = JSON.parse(raw)` |
| Exhaustive switch | `default: const x: never = value; throw new Error(...)` |
| Throw-only function | `function fail(): never { throw new Error(...) }` |
| Async void handler | `async function handler(req, res): Promise<void>` |
| Narrow unknown | `if (typeof x === "string") { ... }` |

## Connected topics

- **08 — Type inference** — how TypeScript infers types so you rarely need to write `any`.
- **09 — Union types** — the `string | number | null` pattern used with `unknown` narrowing.
- **40 — Type narrowing in depth** — the full set of techniques for narrowing `unknown`.
- **60 — Error handling in TypeScript** — deep dive into catch blocks and the `unknown` error type.
