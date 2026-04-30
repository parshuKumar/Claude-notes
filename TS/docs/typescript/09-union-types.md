# 09 — Union Types

## What is this?

A union type says: "this value can be **one of these types**." You write it with the pipe symbol `|` between types: `string | number`. A variable typed as `string | number` can hold either a string or a number — but nothing else. Union types are one of TypeScript's most used features because real-world data is rarely one clean type: a user ID might come in as a string from a URL param or a number from a database, a field might be a value or `null`, a function might return data or an error.

## Why does it matter?

In JavaScript you constantly deal with values that could be one of several types — URL parameters are always strings, but you need them as numbers. Database fields can be `null`. API responses succeed or fail. Without union types, you're back to `any` and guessing. With union types, TypeScript knows every possible type a value can be and forces you to handle each case before using the value.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — a function that takes a string or number ID
function findUser(userId) {
  // Is userId a string? A number? Who knows at the call site.
  // You have to read the function body to find out.
  // And if you pass the wrong thing, you find out at runtime.
  if (typeof userId === "string") {
    return db.findByEmail(userId);
  }
  return db.findById(userId);
}

findUser(42);         // works
findUser("parsh@dev.io");  // works
findUser(true);       // JS accepts this — bug, runtime crash
findUser(null);       // JS accepts this — bug, runtime crash
```

```ts
// TypeScript — the union type documents exactly what's allowed
function findUser(userId: string | number): User | null {
  if (typeof userId === "string") {
    return db.findByEmail(userId);  // TypeScript knows: userId is string here
  }
  return db.findById(userId);       // TypeScript knows: userId is number here
}

findUser(42);                  // ✅
findUser("parsh@dev.io");      // ✅
findUser(true);                // ❌ not string | number
findUser(null);                // ❌ not string | number
```

---

## Syntax

```ts
// ── BASIC UNION ───────────────────────────────────────────────────────────
let userId: string | number;           // either type is valid
let apiKey: string | null;             // value or intentionally absent
let timeout: number | undefined;       // value or not provided

// ── UNION IN FUNCTION PARAMS ──────────────────────────────────────────────
function parseId(raw: string | number): number { ... }

// ── UNION AS RETURN TYPE ──────────────────────────────────────────────────
function findUser(id: number): User | null { ... }

// ── UNION WITH OBJECTS ────────────────────────────────────────────────────
type StringOrNumber = string | number;  // reusable union alias

// ── UNION WITH MORE THAN TWO TYPES ───────────────────────────────────────
type Primitive = string | number | boolean | null | undefined;

// ── UNION OF OBJECT TYPES ─────────────────────────────────────────────────
type AdminUser = { role: "admin"; permissions: string[] };
type RegularUser = { role: "user"; lastLogin: Date };
type AnyUser = AdminUser | RegularUser;  // either shape
```

---

## How it works — narrowing unions

When you have a union type, TypeScript doesn't let you use the value until you **narrow** it — prove which type it is. This is called type narrowing.

### Narrowing with `typeof`

```ts
function formatId(id: string | number): string {
  // Before narrowing — TypeScript refuses:
  id.toUpperCase();     // ❌ Property 'toUpperCase' does not exist on type 'number'
  id.toFixed(2);        // ❌ Property 'toFixed' does not exist on type 'string'

  // After narrowing with typeof:
  if (typeof id === "string") {
    return id.toUpperCase();   // ✅ TypeScript knows: id is string here
  }
  return id.toFixed(0);        // ✅ TypeScript knows: id is number here
  //     ^^  TypeScript automatically narrows to number after the if block
}
```

### Narrowing with null/undefined checks

```ts
function sendWelcomeEmail(email: string | null): void {
  if (email === null) {
    console.log("No email — skipping");
    return;
  }
  // TypeScript narrows: email is string here
  email.toLowerCase();   // ✅
}

// Optional chaining shorthand (doesn't narrow, just safely returns undefined):
function getEmailDomain(email: string | null): string | undefined {
  return email?.split("@")[1];   // returns undefined if email is null
}
```

### Narrowing with `instanceof`

```ts
function processError(error: Error | string): string {
  if (error instanceof Error) {
    return error.message;   // ✅ Error object has .message
  }
  return error;             // ✅ TypeScript knows it's a string here
}
```

### Narrowing with `in` operator

```ts
type ApiSuccess = { data: object; status: "success" };
type ApiError = { error: string; status: "error" };
type ApiResult = ApiSuccess | ApiError;

function handleResult(result: ApiResult): void {
  if ("data" in result) {
    // TypeScript narrows to ApiSuccess
    console.log(result.data);    // ✅
  } else {
    // TypeScript narrows to ApiError
    console.log(result.error);   // ✅
  }
}
```

---

## Discriminated unions — the most powerful pattern

A discriminated union is a union of object types where each member has a **shared property with a unique literal type** — the "discriminant". This lets TypeScript narrow the union completely just by checking that one field.

```ts
// Each type has a 'kind' field with a unique literal value — the discriminant
type PendingOrder = {
  kind: "pending";
  orderId: number;
  createdAt: Date;
};

type ProcessingOrder = {
  kind: "processing";
  orderId: number;
  startedAt: Date;
  assignedTo: string;
};

type CompletedOrder = {
  kind: "completed";
  orderId: number;
  completedAt: Date;
  trackingNumber: string;
};

type Order = PendingOrder | ProcessingOrder | CompletedOrder;

// TypeScript narrows completely based on 'kind':
function describeOrder(order: Order): string {
  switch (order.kind) {
    case "pending":
      return `Order ${order.orderId} waiting since ${order.createdAt.toDateString()}`;
      // TypeScript knows: order is PendingOrder
      // order.trackingNumber  // ❌ doesn't exist on PendingOrder

    case "processing":
      return `Order ${order.orderId} being handled by ${order.assignedTo}`;
      // TypeScript knows: order is ProcessingOrder

    case "completed":
      return `Order ${order.orderId} delivered — tracking: ${order.trackingNumber}`;
      // TypeScript knows: order is CompletedOrder

    default:
      // Exhaustive check — if you add a new Order type and forget a case:
      const exhaustiveCheck: never = order;
      throw new Error(`Unknown order kind: ${exhaustiveCheck}`);
  }
}
```

---

## Union type assignability rules

```ts
// A union type is WIDER than any of its member types
let value: string | number;

value = "hello";  // ✅ string is assignable to string | number
value = 42;       // ✅ number is assignable to string | number
value = true;     // ❌ boolean is not in the union

// You cannot assign a union to a specific type without narrowing:
let specific: string = value;  // ❌ string | number is not assignable to string
// (value might be a number)

// After narrowing, you can assign:
if (typeof value === "string") {
  let specific: string = value;  // ✅
}
```

---

## Common union patterns

### `T | null` — nullable value

```ts
// The most common union — a value that might not exist
type UserId = number | null;

function findUserById(id: number): User | null {
  const user = db.find(id);
  return user ?? null;   // explicit null if not found
}

// Consumers must handle null:
const user = findUserById(42);
if (user !== null) {
  console.log(user.email);   // ✅ TypeScript knows user is User
}

// Or use optional chaining:
console.log(user?.email);   // ✅ undefined if user is null
```

### `T | undefined` — optional value

```ts
// Often used for optional function params and optional object properties
function buildQuery(filter?: string): string {
  // filter is string | undefined
  return filter ? `WHERE ${filter}` : "";
}
```

### String literal unions — enumerating valid values

```ts
// Union of string literals — safer and more flexible than enums
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
type UserRole = "admin" | "editor" | "viewer";
type Environment = "development" | "staging" | "production";

function setEnvironment(env: Environment): void {
  process.env.NODE_ENV = env;
}

setEnvironment("production");   // ✅
setEnvironment("prod");         // ❌ "prod" is not assignable to Environment
```

---

## Example 1 — basic

```ts
// A flexible ID type used throughout the app
type UserId = string | number;

// Parse whatever comes in and normalize to number
function normalizeUserId(id: UserId): number {
  if (typeof id === "string") {
    const parsed = parseInt(id, 10);
    if (isNaN(parsed)) throw new Error(`Invalid user ID: "${id}"`);
    return parsed;
  }
  return id;  // already a number
}

// A function that returns data or null
function getUserName(userId: number): string | null {
  const users: Record<number, string> = { 1: "Parsh", 2: "Alice" };
  return users[userId] ?? null;
}

// Usage:
const id1 = normalizeUserId("42");    // 42
const id2 = normalizeUserId(99);      // 99

const name = getUserName(1);
if (name !== null) {
  console.log(name.toUpperCase());    // "PARSH" — TypeScript knows name is string
}
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response } from 'express';

// ── Discriminated union for API responses ────────────────────────────────────

type SuccessResponse<T> = {
  status: "success";
  data: T;
  statusCode: number;
};

type ErrorResponse = {
  status: "error";
  message: string;
  statusCode: number;
};

type ApiResponse<T> = SuccessResponse<T> | ErrorResponse;

// Function that always returns a consistent shape:
function createResponse<T>(data: T, statusCode: number): ApiResponse<T>;
function createResponse(error: string, statusCode: number): ApiResponse<never>;
function createResponse<T>(dataOrError: T | string, statusCode: number): ApiResponse<T> {
  if (typeof dataOrError === "string" && statusCode >= 400) {
    return { status: "error", message: dataOrError, statusCode };
  }
  return { status: "success", data: dataOrError as T, statusCode };
}

// ── Discriminated union for auth state ───────────────────────────────────────

type AuthenticatedRequest = {
  kind: "authenticated";
  userId: number;
  role: "admin" | "user";
  authToken: string;
};

type AnonymousRequest = {
  kind: "anonymous";
};

type RequestIdentity = AuthenticatedRequest | AnonymousRequest;

function getRequestIdentity(req: Request): RequestIdentity {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return { kind: "anonymous" };
  }
  // Pretend we validated the token:
  return { kind: "authenticated", userId: 1, role: "admin", authToken: authHeader };
}

// Route handler using discriminated union:
async function getUserProfileHandler(req: Request, res: Response): Promise<void> {
  const identity = getRequestIdentity(req);

  if (identity.kind === "anonymous") {
    res.status(401).json(createResponse("Unauthorized", 401));
    return;
  }

  // TypeScript knows: identity is AuthenticatedRequest
  const { userId, role } = identity;   // ✅ TypeScript narrows completely

  res.json(createResponse({ userId, role }, 200));
}

// ── Union for route param validation ─────────────────────────────────────────

type ParsedParam = { valid: true; value: number } | { valid: false; error: string };

function parseUserIdParam(raw: string): ParsedParam {
  const n = parseInt(raw, 10);
  if (isNaN(n) || n <= 0) {
    return { valid: false, error: `"${raw}" is not a valid user ID` };
  }
  return { valid: true, value: n };
}

async function deleteUserHandler(req: Request, res: Response): Promise<void> {
  const parsed = parseUserIdParam(req.params.userId);

  if (!parsed.valid) {
    res.status(400).json({ error: parsed.error });   // TypeScript knows: parsed has .error
    return;
  }

  // TypeScript knows: parsed has .value
  const userId = parsed.value;
  // await db.deleteUser(userId);
  res.status(204).send();
}
```

---

## Common mistakes

### Mistake 1 — Accessing a property before narrowing

```ts
// ❌ WRONG — accessing a property that doesn't exist on all union members
function processUser(user: AdminUser | RegularUser): string {
  return user.permissions.join(", ");
  // ❌ Property 'permissions' does not exist on type 'AdminUser | RegularUser'
  // 'permissions' only exists on AdminUser, not RegularUser
}

// ✅ RIGHT — narrow first
function processUser(user: AdminUser | RegularUser): string {
  if (user.role === "admin") {
    return user.permissions.join(", ");  // ✅ TypeScript knows: AdminUser
  }
  return `Last login: ${user.lastLogin.toDateString()}`;  // ✅ TypeScript knows: RegularUser
}
```

### Mistake 2 — Using a non-discriminant field to narrow

```ts
// ❌ WRONG — trying to narrow by a field that doesn't uniquely identify the type
type Cat = { name: string; sound: "meow" };
type Dog = { name: string; sound: "woof"; breed: string };
type Pet = Cat | Dog;

function describePet(pet: Pet): string {
  if (pet.name === "Rex") {
    return pet.breed;  // ❌ TypeScript won't narrow — 'name' is not a discriminant
    // A Cat could also be named "Rex"
  }
  return pet.name;
}

// ✅ RIGHT — use the field that uniquely identifies each type
function describePet(pet: Pet): string {
  if (pet.sound === "woof") {
    return `${pet.name} is a ${pet.breed}`;  // ✅ TypeScript knows: Dog
  }
  return `${pet.name} says ${pet.sound}`;    // ✅ TypeScript knows: Cat
}
```

### Mistake 3 — Forgetting that union works on the WHOLE type, not one at a time

```ts
// ❌ WRONG — thinking you can call methods that exist on only one member
function processValue(val: string | number): void {
  val.toUpperCase();  // ❌ Property 'toUpperCase' does not exist on type 'number'
  val.toFixed(2);     // ❌ Property 'toFixed' does not exist on type 'string'
  // TypeScript sees the FULL union — you must handle each member
}

// ✅ RIGHT — you can only call methods that exist on ALL members:
function processValue(val: string | number): string {
  return val.toString();  // ✅ both string and number have .toString()
}

// Or narrow first to access type-specific methods:
function processValue2(val: string | number): string {
  if (typeof val === "string") return val.toUpperCase();
  return val.toFixed(2);
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `formatRouteParam(param: string | number | undefined): string` that:
- Returns `"missing"` if `param` is `undefined`
- Returns the string directly if it's already a string
- Returns the number converted to a string if it's a number

Then write a type `HttpStatus = 200 | 201 | 400 | 401 | 403 | 404 | 500` (a numeric literal union) and a function `getStatusMessage(status: HttpStatus): string` that returns a human-readable message for each status code. All cases must be handled — use a `never` exhaustive check in the default branch.

```ts
// Write your code here
```

### Exercise 2 — medium

Model a database query result as a discriminated union:

```ts
type QuerySuccess<T> = { ... }   // status: "success", rows: T[], rowCount: number
type QueryError = { ... }        // status: "error", code: string, message: string
type QueryResult<T> = QuerySuccess<T> | QueryError
```

Write a function `executeQuery<T>(sql: string): QueryResult<T>` that randomly returns a success or error (simulate it — no real DB needed). Then write a function `handleQueryResult<T>(result: QueryResult<T>): T[]` that returns the rows if successful, or throws an `Error` with the error message if the query failed. TypeScript must narrow correctly in both branches.

```ts
// Write your code here
```

### Exercise 3 — hard

You're building a typed notification system. Define these types:

```ts
type EmailNotification = { kind: "email"; to: string; subject: string; body: string }
type SmsNotification = { kind: "sms"; phoneNumber: string; message: string }
type PushNotification = { kind: "push"; deviceToken: string; title: string; payload: object }
type Notification = EmailNotification | SmsNotification | PushNotification
```

Write:
1. `sendNotification(notification: Notification): Promise<void>` — uses a switch on `kind` with an exhaustive `never` check. Each case logs a formatted message (e.g., `"[EMAIL] → parsh@dev.io: Subject"`). TypeScript must know the exact shape in each case.
2. `buildNotification(userId: number, type: "email" | "sms" | "push", data: Record<string, string>): Notification` — builds the correct notification shape based on `type`. Throw a `never`-typed error for unhandled types.
3. A `NotificationQueue` class with a `private queue: Notification[]` property, an `enqueue(n: Notification): void` method, and a `processAll(): Promise<void>` method that calls `sendNotification` for each item and clears the queue.

All types fully annotated. No `any`.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Pattern | Syntax |
|---------|--------|
| Basic union | `string \| number` |
| Nullable | `User \| null` |
| Optional | `string \| undefined` or `param?: string` |
| String literals | `"GET" \| "POST" \| "PUT"` |
| Numeric literals | `200 \| 201 \| 404` |
| Object union | `AdminUser \| RegularUser` |
| Discriminated union | each member has `kind: "literal"` field |

| Narrowing technique | When to use |
|--------------------|------------|
| `typeof x === "string"` | Primitive unions (`string \| number`) |
| `x === null` | Nullable unions (`T \| null`) |
| `x instanceof Error` | Class instance unions |
| `"field" in x` | Object unions where one has a unique field |
| `x.kind === "value"` | Discriminated unions (best pattern) |
| `switch (x.kind)` | Discriminated unions with multiple cases |

| Rule | Notes |
|------|-------|
| Can only use shared methods | `(string \| number).toString()` ✅ |
| Must narrow before type-specific methods | `typeof x === "string"` first |
| Discriminant must be a literal type | `"pending"` not `string` |
| Exhaustive check | `const _: never = x` in default branch |

## Connected topics

- **10 — Intersection types** — combining types with `&` (opposite of union).
- **11 — Literal types** — the `"GET"` and `200` literal types that make discriminated unions work.
- **40 — Type narrowing in depth** — every narrowing technique in full detail.
- **42 — Discriminated unions** — deep dive into the tag/kind pattern and state machines.
