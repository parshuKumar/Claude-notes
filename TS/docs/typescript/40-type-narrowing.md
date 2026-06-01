# 40 — Type Narrowing in Depth

## What is this?

**Type narrowing** is TypeScript's ability to refine a variable's type inside a block of code based on a runtime check. When you write an `if` statement that checks `typeof value === "string"`, TypeScript *narrows* the type of `value` to `string` inside that branch — giving you access to string methods with full type safety.

Narrowing is not one thing — it is a collection of distinct mechanisms TypeScript understands:

| Mechanism | Example |
|---|---|
| `typeof` guard | `typeof x === "string"` |
| `instanceof` guard | `x instanceof Date` |
| `in` operator | `"email" in payload` |
| Equality narrowing | `x === null`, `x === "admin"` |
| Truthiness narrowing | `if (x) { }` |
| Assignment narrowing | `x = "hello"` → x is string in scope |
| Control flow analysis | TypeScript tracks what's impossible after a `return`, `throw`, or `break` |

## Why does it matter?

Every real backend function receives data that could be one of several types — a string or number ID, a validated body or an error, a `User | null` from the database. Without narrowing you'd need explicit casts (`as string`) everywhere, losing all type safety.

With narrowing, TypeScript does the work: you write the same runtime checks you'd write in JavaScript, and TypeScript automatically changes the type in each branch. No casts, no `as any`, and the compiler still catches mistakes.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — runtime checks exist, but no compile-time benefit:
function handleId(id) {
  if (typeof id === "string") {
    return id.toUpperCase();  // works at runtime if id is actually a string
  }
  return id * 2;  // works at runtime if id is actually a number
  // Nothing stops you from calling id.toUpperCase() in the number branch — silent bug
}
```

```ts
// TypeScript — same runtime checks, but the type shrinks in each branch:
function handleId(id: string | number): string | number {
  if (typeof id === "string") {
    // Inside this block: id is narrowed to string
    return id.toUpperCase(); // ✅ string method available
    // id * 2 would be a type error here — TS knows id is string
  }
  // Outside the if: id is narrowed to number (string was handled above)
  return id * 2;  // ✅ number arithmetic available
  // id.toUpperCase() would be a type error here
}
```

---

## Syntax

```ts
// typeof:
if (typeof value === "string")  { /* value: string */ }
if (typeof value === "number")  { /* value: number */ }
if (typeof value === "boolean") { /* value: boolean */ }
if (typeof value === "object")  { /* value: object | null — careful! */ }
if (typeof value === "function"){ /* value: Function */ }
if (typeof value === "undefined"){ /* value: undefined */ }

// instanceof:
if (error instanceof Error)     { /* error: Error */ }
if (date  instanceof Date)      { /* date:  Date  */ }
if (req   instanceof MyRequest) { /* req:   MyRequest */ }

// in operator:
if ("email" in payload)         { /* payload has email property */ }
if ("statusCode" in error)      { /* error has statusCode */ }

// equality narrowing:
if (status === "active")        { /* status: "active" */ }
if (value === null)             { /* value: null */ }
if (value === undefined)        { /* value: undefined */ }
if (value == null)              { /* value: null | undefined */ }

// truthiness:
if (user)   { /* user is not null/undefined/0/""/false */ }
if (!token) { /* token is null | undefined | "" | 0 | false */ }

// assignment:
let value: string | number;
value = "hello"; // from here: value is string (until reassigned)
```

---

## How it works — mechanism by mechanism

### Mechanism 1 — `typeof` narrowing

`typeof` narrows to JavaScript's primitive type strings. TypeScript recognises all six `typeof` checks:

```ts
function formatValue(value: string | number | boolean | null | undefined): string {
  if (typeof value === "string")    return `"${value}"`;
  if (typeof value === "number")    return value.toFixed(2);
  if (typeof value === "boolean")   return value ? "true" : "false";
  if (typeof value === "undefined") return "<undefined>";
  // Remaining type: null (typeof null === "object" — JS quirk — but TS still narrows)
  return "<null>";
}

// Careful: typeof null === "object" — always check for null before assuming object:
function processBody(body: object | null): void {
  if (typeof body === "object" && body !== null) {
    // body: object — null is ruled out by the second check
    console.log(Object.keys(body));
  }
}
```

### Mechanism 2 — `instanceof` narrowing

`instanceof` narrows to a class type. Works with any class, including custom ones and built-ins:

```ts
class ApiError extends Error {
  constructor(public readonly statusCode: number, message: string) {
    super(message);
    this.name = "ApiError";
  }
}

class ValidationError extends Error {
  constructor(public readonly fields: Record<string, string>) {
    super("Validation failed");
    this.name = "ValidationError";
  }
}

function handleError(error: unknown): { status: number; message: string; fields?: Record<string, string> } {
  if (error instanceof ApiError) {
    // error: ApiError — statusCode available
    return { status: error.statusCode, message: error.message };
  }

  if (error instanceof ValidationError) {
    // error: ValidationError — fields available
    return { status: 422, message: error.message, fields: error.fields };
  }

  if (error instanceof Error) {
    // error: Error — standard message available
    return { status: 500, message: error.message };
  }

  // error: unknown — none of the above matched
  return { status: 500, message: "An unknown error occurred" };
}
```

### Mechanism 3 — `in` operator narrowing

`"property" in object` narrows the object to a type that has that property. Useful for discriminating between object shapes that share no common discriminant:

```ts
interface EmailNotification  { email: string; subject: string; body: string; }
interface SmsNotification    { phone: string; message: string; }
interface PushNotification   { deviceToken: string; title: string; body: string; }

type Notification = EmailNotification | SmsNotification | PushNotification;

function sendNotification(notification: Notification): void {
  if ("email" in notification) {
    // notification: EmailNotification
    sendEmail(notification.email, notification.subject, notification.body);
    return;
  }

  if ("phone" in notification) {
    // notification: SmsNotification
    sendSms(notification.phone, notification.message);
    return;
  }

  // notification: PushNotification (only option left)
  sendPush(notification.deviceToken, notification.title, notification.body);
}
```

### Mechanism 4 — Equality narrowing

Comparing with `===` or `!==` narrows to the specific literal value. Comparing with `== null` narrows to `null | undefined`:

```ts
type UserRole = "admin" | "moderator" | "user";

function getPermissions(role: UserRole): string[] {
  if (role === "admin") {
    // role: "admin"
    return ["read", "write", "delete", "manage_users"];
  }
  if (role === "moderator") {
    // role: "moderator"
    return ["read", "write", "delete"];
  }
  // role: "user" — only option left
  return ["read"];
}

// null/undefined check with == null (catches both):
function processUserId(userId: number | null | undefined): number {
  if (userId == null) {
    // userId: null | undefined
    throw new Error("userId is required");
  }
  // userId: number — both null and undefined ruled out
  return userId * 2;
}

// Strict null check:
function getUsername(user: { name: string } | null): string {
  if (user === null) return "anonymous";
  // user: { name: string }
  return user.name;
}
```

### Mechanism 5 — Truthiness narrowing

Checking a value in an `if` condition filters out all falsy values (`null`, `undefined`, `0`, `""`, `false`, `NaN`):

```ts
function processToken(token: string | null | undefined): string {
  if (!token) {
    // token: string | null | undefined AND falsy
    // i.e. null, undefined, or empty string ""
    throw new Error("Token is required");
  }
  // token: string — null and undefined ruled out, and it's truthy (non-empty string)
  return token.trim().toUpperCase();
}

// Careful — truthiness narrows out 0 and "" even when valid:
function processCount(count: number | null): number {
  if (!count) {
    // count: null | 0 — 0 is falsy, accidentally treated as missing
    throw new Error("Count required");  // ← BUG: throws when count is 0
  }
  return count;
}

// ✅ Better — explicit null check:
function processCount(count: number | null): number {
  if (count === null) {
    throw new Error("Count required");  // only throws for null, not for 0
  }
  // count: number — includes 0
  return count;
}
```

### Mechanism 6 — Control flow analysis (narrowing after early exit)

TypeScript tracks that after a `return`, `throw`, or `continue`, the narrowed-out type is gone in the rest of the function — no nesting needed:

```ts
interface DatabaseUser {
  id: number;
  email: string;
  role: "admin" | "user";
  deletedAt: Date | null;
}

async function requireActiveUser(userId: number): Promise<DatabaseUser> {
  const user = await db.findUserById(userId);

  if (!user) {
    throw ApiError.notFound("User");
  }
  // user: DatabaseUser — null is ruled out above

  if (user.deletedAt !== null) {
    throw ApiError.forbidden("Account is deactivated");
  }
  // user.deletedAt: null — non-null is ruled out above

  if (user.role !== "admin") {
    throw ApiError.forbidden("Admin access required");
  }
  // user.role: "admin" — only "user" was ruled out above

  return user; // user: DatabaseUser with role "admin" and deletedAt null
}
```

### Mechanism 7 — Assignment narrowing

When you assign a value to a variable, TypeScript narrows the type at the point of assignment:

```ts
let requestBody: string | Buffer;

// Based on content-type header:
if (req.headers["content-type"] === "application/octet-stream") {
  requestBody = req.rawBody; // Buffer
  // requestBody: Buffer here
  requestBody.byteLength; // ✅
} else {
  requestBody = req.body as string; // string
  // requestBody: string here
  requestBody.toUpperCase(); // ✅
}

// After both branches — back to string | Buffer (the declared type):
requestBody; // string | Buffer
```

### Mechanism 8 — Combining narrowing with `never` for exhaustiveness

After narrowing every branch of a union, the remaining type is `never`. You can use this to enforce that all union members are handled:

```ts
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

function getMethodColor(method: HttpMethod): string {
  switch (method) {
    case "GET":    return "blue";
    case "POST":   return "green";
    case "PUT":    return "orange";
    case "PATCH":  return "yellow";
    case "DELETE": return "red";
    default: {
      // method: never — all cases handled
      // If you add "OPTIONS" to HttpMethod but forget a case here, this line is a compile error:
      const _exhaustive: never = method;
      throw new Error(`Unhandled HTTP method: ${_exhaustive}`);
    }
  }
}
```

---

## Example 1 — basic

```ts
// Request body parsing — narrowing from unknown to concrete types

type JsonPrimitive = string | number | boolean | null;
type JsonValue = JsonPrimitive | JsonValue[] | { [key: string]: JsonValue };

function parseRequestValue(value: unknown): JsonValue {
  // typeof narrowing:
  if (typeof value === "string")  return value;
  if (typeof value === "number")  return value;
  if (typeof value === "boolean") return value;
  if (value === null)             return null;

  // Array check — typeof [] === "object", so use Array.isArray:
  if (Array.isArray(value)) {
    // value: unknown[] after Array.isArray check
    return value.map(parseRequestValue);
  }

  // typeof "object" + not null + not array → plain object:
  if (typeof value === "object") {
    const result: { [key: string]: JsonValue } = {};
    for (const [k, v] of Object.entries(value as Record<string, unknown>)) {
      result[k] = parseRequestValue(v);
    }
    return result;
  }

  // Everything else (undefined, function, symbol) — not valid JSON:
  throw new TypeError(`Cannot serialize value of type: ${typeof value}`);
}

// Route parameter coercion — narrowing from string | undefined:
function parseIntParam(param: string | undefined, name: string): number {
  if (param === undefined) {
    throw ApiError.badRequest(`Missing required parameter: ${name}`);
  }
  // param: string

  const parsed = parseInt(param, 10);
  if (Number.isNaN(parsed) || parsed <= 0) {
    throw ApiError.badRequest(`Parameter '${name}' must be a positive integer, got: ${param}`);
  }
  // parsed: number — valid positive integer
  return parsed;
}

// Usage:
const userId    = parseIntParam(req.params.id,   "id");       // number
const productId = parseIntParam(req.params.pid,  "pid");      // number
const body      = parseRequestValue(req.body);                // JsonValue
```

---

## Example 2 — real world backend use case

```ts
// Middleware pipeline — narrowing `unknown` errors to structured responses

interface ApiErrorShape {
  statusCode: number;
  message:    string;
  code:       string;
}

interface ValidationErrorShape {
  statusCode:  number;
  message:     string;
  fieldErrors: Record<string, string[]>;
}

// Type predicates (preview of topic 41) — used here for clarity:
function isApiErrorShape(value: unknown): value is ApiErrorShape {
  return (
    typeof value === "object" &&
    value !== null &&
    "statusCode" in value &&
    "code"       in value &&
    typeof (value as ApiErrorShape).statusCode === "number"
  );
}

function isValidationErrorShape(value: unknown): value is ValidationErrorShape {
  return (
    typeof value === "object" &&
    value !== null &&
    "fieldErrors" in value &&
    typeof (value as ValidationErrorShape).fieldErrors === "object"
  );
}

// Express-style global error handler — full narrowing chain:
function globalErrorHandler(
  error:   unknown,
  req:     ExpressRequest,
  res:     ExpressResponse,
  _next:   ExpressNextFn,
): void {
  const requestId = req.headers["x-request-id"] as string ?? "unknown";

  // 1. instanceof checks for our custom error classes:
  if (error instanceof ValidationError) {
    res.status(422).json({
      statusCode:  422,
      message:     error.message,
      fieldErrors: error.fieldErrors,
      requestId,
    });
    return;
  }

  if (error instanceof ApiError) {
    res.status(error.statusCode).json({
      statusCode: error.statusCode,
      message:    error.message,
      code:       error.code,
      requestId,
    });
    return;
  }

  // 2. in operator — check for shaped objects thrown by third-party libraries:
  if (isApiErrorShape(error)) {
    res.status(error.statusCode).json({ ...error, requestId });
    return;
  }

  if (isValidationErrorShape(error)) {
    res.status(422).json({ ...error, requestId });
    return;
  }

  // 3. instanceof Error — standard error:
  if (error instanceof Error) {
    console.error("Unhandled Error:", error.stack);
    res.status(500).json({
      statusCode: 500,
      message:    process.env.NODE_ENV === "production" ? "Internal server error" : error.message,
      requestId,
    });
    return;
  }

  // 4. typeof checks — primitive thrown (bad practice but handle gracefully):
  if (typeof error === "string") {
    console.error("String thrown:", error);
    res.status(500).json({ statusCode: 500, message: error, requestId });
    return;
  }

  // 5. Fallback — completely unknown shape:
  console.error("Unknown error type thrown:", error);
  res.status(500).json({ statusCode: 500, message: "Internal server error", requestId });
}

// Auth middleware — narrowing from unknown request body:
async function loginHandler(req: ExpressRequest, res: ExpressResponse): Promise<void> {
  const body = req.body as unknown; // treat as unknown for safe narrowing

  // Narrow the body shape:
  if (typeof body !== "object" || body === null) {
    throw ApiError.badRequest("Request body must be a JSON object");
  }
  // body: object (non-null)

  if (!("email" in body) || !("password" in body)) {
    throw ApiError.badRequest("Body must contain email and password");
  }
  // body has email and password properties

  const { email, password } = body as { email: unknown; password: unknown };

  if (typeof email !== "string" || typeof password !== "string") {
    throw ApiError.badRequest("email and password must be strings");
  }
  // email: string, password: string

  if (email.trim().length === 0 || password.length === 0) {
    throw ApiError.badRequest("email and password cannot be empty");
  }
  // email and password are non-empty strings — safe to use

  const result = await authService.login(email.trim().toLowerCase(), password);
  res.status(200).json(result);
}
```

---

## Common mistakes

### Mistake 1 — Using truthiness to check for `null` when `0` or `""` are valid values

```ts
function processPort(port: number | null): number {
  if (!port) {
    // ❌ This also triggers when port === 0 — 0 is falsy
    throw new Error("Port is required");
  }
  return port;
}

processPort(0);    // ❌ throws — but 0 might be a valid port in some contexts

// ✅ Use strict equality:
function processPort(port: number | null): number {
  if (port === null) {
    throw new Error("Port is required");
  }
  return port; // port: number — includes 0
}

// ✅ Or check for null and invalid range together:
function processPort(port: number | null): number {
  if (port === null || port < 0 || port > 65535) {
    throw new Error(`Invalid port: ${port}`);
  }
  return port;
}
```

### Mistake 2 — `typeof null === "object"` — the JS quirk that bites TypeScript too

```ts
function processConfig(config: object | null): void {
  if (typeof config === "object") {
    // ❌ config is still null | object here — typeof null === "object" in JS
    console.log(Object.keys(config)); // TypeError at runtime if config is null
  }
}

// ✅ Always add a null check alongside typeof "object":
function processConfig(config: object | null): void {
  if (typeof config === "object" && config !== null) {
    // config: object — null ruled out
    console.log(Object.keys(config)); // ✅
  }
}

// ✅ Or use instanceof (doesn't match null):
function processConfig(config: AppConfig | null): void {
  if (config instanceof AppConfig) {
    // config: AppConfig — cannot be null
    config.getPort(); // ✅
  }
}
```

### Mistake 3 — Narrowing inside a callback loses the narrowed type

```ts
function processUser(userId: number | null): void {
  if (userId === null) return;
  // userId: number here — correct

  // ❌ But inside a callback, TypeScript can't guarantee userId wasn't reassigned:
  setTimeout(() => {
    console.log(userId.toFixed()); // ❌ TypeScript may warn: userId is number | null in some versions
    // because callbacks execute later — userId could theoretically be reassigned
  }, 1000);
}

// ✅ Capture in a local const — TypeScript knows a const cannot be reassigned:
function processUser(userId: number | null): void {
  if (userId === null) return;
  const safeId = userId; // safeId: number — const, never reassigned

  setTimeout(() => {
    console.log(safeId.toFixed()); // ✅ safeId is definitely number
  }, 1000);
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a `formatApiValue` function that accepts `string | number | boolean | null | undefined` and returns a formatted string:
- `string` → wrapped in double quotes: `"hello"` → `'"hello"'`
- `number` → formatted to 2 decimal places: `3.14159` → `"3.14"`
- `boolean` → uppercase: `true` → `"TRUE"`
- `null` → `"NULL"`
- `undefined` → `"UNDEFINED"`

Then write `parseEnvVar(name: string, type: "string" | "number" | "boolean")`:
- Reads `process.env[name]`
- If missing (undefined), throws: `EnvVarMissingError`
- Narrows to the requested type (parse number, parse boolean from `"true"`/`"false"`)
- Returns `string | number | boolean` accordingly

Use only built-in narrowing — no type predicates, no `as` casts.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a `handleWebhookEvent` function for a payment webhook system. The incoming event body is `unknown`. Narrow it through these shapes:

```ts
type WebhookEventType =
  | "payment.succeeded"
  | "payment.failed"
  | "refund.created"
  | "dispute.opened";

interface PaymentSucceededEvent {
  type:      "payment.succeeded";
  paymentId: string;
  amountCents: number;
  userId:    number;
  timestamp: string;
}

interface PaymentFailedEvent {
  type:       "payment.failed";
  paymentId:  string;
  errorCode:  string;
  userId:     number;
  timestamp:  string;
}

interface RefundCreatedEvent {
  type:       "refund.created";
  refundId:   string;
  paymentId:  string;
  amountCents: number;
  timestamp:  string;
}

interface DisputeOpenedEvent {
  type:       "dispute.opened";
  disputeId:  string;
  paymentId:  string;
  reason:     string;
  timestamp:  string;
}
```

Requirements:
- The function signature is `handleWebhookEvent(rawBody: unknown): WebhookResult`
- First narrow that it's a non-null object with a `type` field that's a string
- Then narrow on the `type` string using a `switch` or `if/else` chain
- Each branch handles its event and returns a `WebhookResult`
- The `default` branch must use the `never` exhaustiveness trick — if someone adds a new event type but forgets to handle it, it's a compile error
- Use only equality narrowing, `typeof`, `in`, and control flow — no type predicates yet

```ts
// Write your code here
```

### Exercise 3 — hard

Build a `safeJsonParse<T>` utility and a `validateRequestBody` function that together narrow `unknown` → `T` through a chain of runtime checks:

```ts
// Part 1 — safeJsonParse:
// Parses a JSON string safely. Returns:
// { ok: true; data: unknown } on success
// { ok: false; error: string } on failure
// Use truthiness + typeof + control flow — no try/catch-as-cast

// Part 2 — Schema-based validator (manual, without zod/yup):
// Define a minimal "schema" type:
type Schema =
  | { type: "string"; minLength?: number; maxLength?: number; pattern?: RegExp }
  | { type: "number"; min?: number; max?: number; integer?: boolean }
  | { type: "boolean" }
  | { type: "object"; fields: Record<string, Schema> }
  | { type: "array";  items: Schema; minItems?: number; maxItems?: number }
  | { type: "union";  variants: Schema[] };

// Write: validate(value: unknown, schema: Schema): ValidationOutcome
// type ValidationOutcome =
//   | { valid: true;  value: unknown }
//   | { valid: false; errors: string[] }

// Part 3 — validateRequestBody<T>(body: unknown, schema: Schema): T
// - Uses validate() internally
// - If valid: returns body narrowed to T (use a single `as T` only here, justified by the full validation chain)
// - If invalid: throws ApiError.badRequest() with the validation errors joined

// Demonstrate with a CreateUserDto schema:
const createUserSchema: Schema = {
  type: "object",
  fields: {
    email:    { type: "string", minLength: 3, maxLength: 254, pattern: /@/ },
    password: { type: "string", minLength: 8 },
    role:     { type: "union", variants: [
      { type: "string" }, // accept any string — further validation in service layer
    ]},
    age:      { type: "number", min: 13, max: 120, integer: true },
  },
};

const dto = validateRequestBody<CreateUserDto>(req.body, createUserSchema);
// dto: CreateUserDto — fully narrowed from unknown
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// typeof — primitives:
if (typeof x === "string")    { /* x: string */ }
if (typeof x === "number")    { /* x: number */ }
if (typeof x === "boolean")   { /* x: boolean */ }
if (typeof x === "undefined") { /* x: undefined */ }
if (typeof x === "object" && x !== null) { /* x: object (not null) */ }

// instanceof — class instances:
if (x instanceof Error)   { /* x: Error */ }
if (x instanceof MyClass) { /* x: MyClass */ }

// in — property existence:
if ("email" in x)  { /* x has email */ }
if ("id" in x)     { /* x has id */ }

// equality — literal values:
if (x === null)    { /* x: null */ }
if (x === "admin") { /* x: "admin" */ }
if (x == null)     { /* x: null | undefined */ }

// truthiness — falsy filter:
if (x)  { /* x is not null/undefined/0/""/false/NaN */ }
if (!x) { /* x is null | undefined | 0 | "" | false | NaN */ }

// exhaustiveness with never:
function handle(x: "a" | "b"): void {
  if (x === "a") return;
  if (x === "b") return;
  const _: never = x; // compile error if "a" | "b" is extended and not handled
}

// GOTCHAS:
// typeof null === "object"  → always add `&& x !== null`
// 0 and "" are falsy        → prefer === null over !x when 0/"" are valid
// Callbacks may lose narrowing → capture in a const before the callback
```

---

## Connected topics

- **41 — Type guards** — custom functions that return `value is Type` — the next step when built-in narrowing isn't enough.
- **42 — Discriminated unions** — using a `kind`/`type` field to drive equality narrowing cleanly across complex unions.
- **09 — Union types** — what you're narrowing from; understanding the union is a prerequisite.
- **07 — Special types** — `unknown` vs `any` — narrowing is how you safely use `unknown`.
- **44 — Conditional types** — type-level narrowing (`T extends U ? X : Y`), the compile-time counterpart to runtime narrowing.
- **32 — Utility types** — `NonNullable<T>` is a utility type that does at the type level what `if (x != null)` does at runtime.
