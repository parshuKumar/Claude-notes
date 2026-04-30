# 05 — Type Annotations

## What is this?

Type annotations are the `: type` syntax you write after a variable name, function parameter, or function to tell TypeScript exactly what kind of value is allowed there. They are the core building block of TypeScript — everything else (interfaces, generics, utility types) is built on top of this idea. At compile time the annotations are checked; at runtime they are completely erased and your JavaScript has no trace of them.

## Why does it matter?

In JavaScript, a function parameter can silently receive the wrong type of data and cause a runtime crash far away from the actual mistake. Type annotations put the error at the source — the moment you call the function with wrong data, TypeScript tells you. This means bugs are caught in your editor, not in production logs at 3 AM.

## The JavaScript way vs the TypeScript way

### JavaScript — no annotations, silent failures

```js
// JS: nothing stops you passing the wrong thing
function createAuthToken(userId, expiresInDays) {
  // What is userId? A number? A string? An object?
  // What is expiresInDays? A number? A string "30d"?
  // You have to read the whole function body to know.
  return `token-${userId}-exp${expiresInDays * 24 * 60 * 60}`;
}

// All of these "work" in JS — no errors, but wrong results:
createAuthToken("abc", 7);       // "token-abc-expNaN"  ← bug, no error
createAuthToken(42, "seven");    // "token-42-expNaN"   ← bug, no error
createAuthToken(42);             // "token-42-expNaN"   ← bug, no error
```

### TypeScript — annotations catch mistakes immediately

```ts
// TS: the contract is explicit
function createAuthToken(userId: number, expiresInDays: number): string {
  return `token-${userId}-exp${expiresInDays * 24 * 60 * 60}`;
}

createAuthToken("abc", 7);    // ❌ Error: Argument of type 'string' is not assignable to parameter of type 'number'
createAuthToken(42, "seven"); // ❌ Error: Argument of type 'string' is not assignable to parameter of type 'number'
createAuthToken(42);          // ❌ Error: Expected 2 arguments, but got 1
createAuthToken(42, 7);       // ✅ works perfectly
```

---

## Syntax

```ts
// ── VARIABLE ANNOTATIONS ──────────────────────────────────────────

let userName: string = "Parsh";          // variable: type = value
let userId: number = 101;
let isActive: boolean = true;
let lastLogin: Date = new Date();        // built-in JS types work too

// ── FUNCTION PARAMETER ANNOTATIONS ───────────────────────────────

function greet(name: string): string {   // (param: type): returnType
  return `Hello, ${name}`;
}

// ── RETURN TYPE ANNOTATIONS ───────────────────────────────────────

function add(a: number, b: number): number {  // return type after ()
  return a + b;
}

function logMessage(msg: string): void {      // void = returns nothing
  console.log(msg);
}

// ── OBJECT ANNOTATIONS ────────────────────────────────────────────

let user: { id: number; name: string; email: string } = {
  id: 1,
  name: "Parsh",
  email: "parsh@dev.io",
};

// ── ARRAY ANNOTATIONS ─────────────────────────────────────────────

let userIds: number[] = [1, 2, 3];            // array of numbers
let tags: string[] = ["admin", "verified"];   // array of strings

// ── ARROW FUNCTION ANNOTATIONS ────────────────────────────────────

const multiply = (a: number, b: number): number => a * b;

// ── ASYNC FUNCTION ANNOTATIONS ────────────────────────────────────

async function fetchUser(userId: number): Promise<string> {
  return `user-${userId}`;   // Promise wraps the return type
}
```

---

## How it works — line by line

### Variable annotation

```ts
let userName: string = "Parsh";
//  ^^^^^^^^^  ^^^^^^   ^^^^^^^
//  variable   type     initial value
```

- The `: string` tells TypeScript: "this variable may only ever hold a string."
- After this line, `userName = 42` causes an error — the annotation is permanent for that variable.
- The annotation is **optional** when you assign a value immediately (TypeScript can infer it), but required when you declare without assigning:

```ts
let userName = "Parsh";     // ✅ TypeScript infers string — annotation not needed
let pendingName: string;    // ✅ No value yet — annotation IS required here
pendingName = "Parsh";      // ✅ Fine — it's a string
pendingName = 42;           // ❌ Error
```

### Function parameter annotation

```ts
function processRequest(requestBody: { userId: number; action: string }): void {
//                      ^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                      parameter    type (an inline object shape)
```

- Every parameter should have a type annotation.
- If you skip one and `noImplicitAny` is on (which it is with `strict: true`), TypeScript errors.

### Return type annotation

```ts
function getUser(userId: number): { id: number; name: string } | null {
//                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                               return type — either a user object or null
  if (userId <= 0) return null;
  return { id: userId, name: "Parsh" };
}
```

- The return type goes **after the closing parenthesis** of the parameters, before the `{`.
- If your function returns different things from different branches, TypeScript checks **all branches** match the declared return type.
- Annotating return types is best practice — it acts as documentation and catches mistakes if you refactor.

### `void` return type

```ts
function sendEmail(to: string, subject: string): void {
  // void means: this function intentionally returns nothing useful
  console.log(`Sending email to ${to}: ${subject}`);
  // return;       ← allowed
  // return null;  ← NOT allowed with void
  // return 42;    ← NOT allowed with void
}
```

### `Promise<T>` return type for async functions

```ts
// An async function always returns a Promise.
// You annotate the type INSIDE the Promise:
async function getUserById(userId: number): Promise<{ id: number; name: string }> {
  const user = await db.findUser(userId);  // pretend this exists
  return user;
}

// If the async function returns nothing:
async function deleteUser(userId: number): Promise<void> {
  await db.deleteUser(userId);
}
```

---

## Where to annotate vs where to let TypeScript infer

TypeScript can **infer** types in many situations. You don't need to annotate everything — that becomes noise. Here's the rule:

| Situation | Annotate? | Why |
|-----------|-----------|-----|
| Variable with immediate assignment | No | TypeScript infers it |
| Variable declared without assignment | Yes | TypeScript can't infer yet |
| Function parameters | Yes | TypeScript cannot infer these |
| Function return type | Recommended | Documents intent, catches mistakes |
| Object property in a class | Yes | Needed for class property typing |
| `let` vs `const` | `const` infers narrow type; `let` infers wider | See below |

```ts
// TypeScript infers these correctly — no annotation needed:
const port = 3000;               // inferred as: 3000 (literal type with const)
let port2 = 3000;                // inferred as: number (widenened with let)
const name = "Parsh";            // inferred as: "Parsh" (literal type with const)
const user = { id: 1 };          // inferred as: { id: number }

// Annotation IS needed here:
let authToken: string;           // declared without value — must annotate
function fn(x) { return x; }    // ❌ x has implicit any — must annotate
```

---

## Annotating complex objects inline vs with a type alias

```ts
// Inline annotation — fine for one-off objects, gets messy with repetition
function createUser(user: { id: number; name: string; email: string }): { id: number; name: string; email: string } {
  return user;
}

// Better: use a type alias (covered in topic 12) to name the shape
type User = {
  id: number;
  name: string;
  email: string;
};

function createUser(user: User): User {   // clean and reusable
  return user;
}
```

---

## Example 1 — basic

```ts
// Annotated variables
let serverPort: number = 3000;
let serverHost: string = "localhost";
let isProduction: boolean = false;

// Annotated function — all params and return type
function buildConnectionString(
  host: string,
  port: number,
  dbName: string
): string {
  return `mongodb://${host}:${port}/${dbName}`;
}

// Call it correctly
const connectionString = buildConnectionString(serverHost, serverPort, "usersdb");
console.log(connectionString);  // "mongodb://localhost:3000/usersdb"

// TypeScript catches these mistakes immediately:
// buildConnectionString(3000, "localhost", "usersdb");  // ❌ wrong order
// buildConnectionString("localhost", 3000);             // ❌ missing 3rd arg
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response } from 'express';

// Define the shape of the expected request body
type CreateUserBody = {
  name: string;
  email: string;
  role: string;
};

// Define the shape of the user stored in the DB
type User = {
  id: number;
  name: string;
  email: string;
  role: string;
  createdAt: Date;
};

// Define the standard API response shape
type ApiResponse<T> = {
  success: boolean;
  data: T | null;
  error: string | null;
};

// A typed route handler
async function createUserHandler(
  req: Request<{}, {}, CreateUserBody>,   // Request<Params, ResBody, ReqBody>
  res: Response<ApiResponse<User>>        // Response body type
): Promise<void> {
  const { name, email, role } = req.body; // TypeScript knows these are strings

  // Validation — TypeScript can't validate at runtime, but knows the types
  if (!name || !email) {
    res.status(400).json({
      success: false,
      data: null,
      error: "name and email are required",
    });
    return;
  }

  // Simulate saving to DB
  const newUser: User = {
    id: Date.now(),
    name,
    email,
    role: role || "user",
    createdAt: new Date(),
  };

  res.status(201).json({
    success: true,
    data: newUser,
    error: null,
  });
}
```

---

## Common mistakes

### Mistake 1 — Annotating when TypeScript can already infer

```ts
// ❌ WRONG — redundant annotations create noise
const userId: number = 42;                    // TypeScript already knows it's a number
const isActive: boolean = true;               // TypeScript already knows it's boolean
const serverName: string = "my-api";          // TypeScript already knows it's a string

// ✅ RIGHT — let TypeScript infer when the value is obvious
const userId = 42;            // inferred: number
const isActive = true;        // inferred: boolean
const serverName = "my-api";  // inferred: string

// DO annotate when there's no initial value:
let currentUserId: number;    // ✅ no value yet — annotation needed
```

### Mistake 2 — Forgetting the return type on functions that should always return

```ts
// ❌ WRONG — no return type annotation
// TypeScript infers the return as string | undefined, silently
function getUserEmail(userId: number) {
  const users = [{ id: 1, email: "parsh@dev.io" }];
  const user = users.find(u => u.id === userId);
  if (user) {
    return user.email;
  }
  // Falls through here returning undefined — is that intentional?
}

// ✅ RIGHT — annotate the return type explicitly
function getUserEmail(userId: number): string | null {
  const users = [{ id: 1, email: "parsh@dev.io" }];
  const user = users.find(u => u.id === userId);
  if (user) return user.email;
  return null;   // ✅ explicit, intentional
}
```

### Mistake 3 — Using `: any` as the "fix" for annotation errors

```ts
// ❌ WRONG — any silences errors but removes all type safety
function handleWebhookPayload(payload: any) {
  // TypeScript lets you do ANYTHING with 'any' — no checking
  const userId = payload.user.id;       // no error even if this crashes
  const amount = payload.paymnt.total;  // typo "paymnt" — no error
}

// ✅ RIGHT — define the actual shape
type WebhookPayload = {
  user: { id: number; email: string };
  payment: { total: number; currency: string };
};

function handleWebhookPayload(payload: WebhookPayload) {
  const userId = payload.user.id;         // ✅ TypeScript checks this
  const amount = payload.paymnt.total;    // ❌ TypeScript catches the typo immediately
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a TypeScript function called `buildApiUrl` that takes three parameters: `baseUrl` (string), `endpoint` (string), and `version` (number). It should return a string formatted as `"https://[baseUrl]/v[version]/[endpoint]"`. Annotate all parameters and the return type. Then call it with correct arguments and store the result in a variable with an appropriate type annotation.

```ts
// Write your code here
```

### Exercise 2 — medium

You're writing a utility for a payment service. Write these three typed functions:

1. `calculateTax(price: number, taxRate: number): number` — returns price multiplied by (1 + taxRate/100), rounded to 2 decimal places.
2. `formatCurrency(amount: number, currency: string): string` — returns a string like `"USD 29.99"`.
3. `buildInvoiceLine(itemName: string, quantity: number, unitPrice: number, taxRate: number): string` — uses the two functions above and returns `"[itemName] x[quantity] — USD [totalWithTax]"`.

Annotate every parameter and every return type. Create the variables needed to call `buildInvoiceLine` and log the result.

```ts
// Write your code here
```

### Exercise 3 — hard

You're building a typed request validation utility for an Express backend. Write the following:

1. A type `RequestValidationResult` with two properties: `isValid: boolean` and `errors: string[]`.
2. A function `validateCreateUserBody` that takes a parameter `body` typed as `{ name: unknown; email: unknown; age: unknown }` (using `unknown` because `req.body` in the real world is untyped) and returns `RequestValidationResult`. It should:
   - Check `name` is a non-empty string — add `"name is required and must be a string"` to errors if not.
   - Check `email` is a string containing `"@"` — add `"valid email is required"` if not.
   - Check `age` is a number between 1 and 120 — add `"age must be a number between 1 and 120"` if not.
   - Return `{ isValid: errors.length === 0, errors }`.
3. A function `formatValidationErrors(result: RequestValidationResult): string` that returns a joined error string like `"Validation failed: [error1], [error2]"` or `"Validation passed"` if valid.

Annotate every type. Call both functions with at least two test inputs — one valid, one invalid.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Example |
|--------|---------|
| Variable annotation | `let userId: number = 1` |
| Without initial value | `let authToken: string` |
| Function parameter | `function fn(name: string) {}` |
| Return type | `function fn(): string {}` |
| Void return | `function fn(): void {}` |
| Async return | `async function fn(): Promise<User> {}` |
| Object inline | `let u: { id: number; name: string }` |
| Array | `let ids: number[]` |
| Arrow function | `const fn = (x: number): number => x * 2` |
| Infer (no annotation needed) | `const port = 3000` |
| Annotation needed | `let port: number` (no value yet) |

| Rule | Guidance |
|------|---------|
| Annotate function params | Always — TypeScript can't infer these |
| Annotate return types | Recommended — documents intent |
| Annotate variables | Only when there's no initial value |
| Never use `any` to silence errors | Fix the type instead |
| `void` vs `undefined` | Use `void` for functions that return nothing |
| `Promise<void>` | Async functions that return nothing |

## Connected topics

- **06 — Primitive types** — the actual types you use in annotations: `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`.
- **07 — Special types** — `any`, `unknown`, `never`, `void` — the unusual ones seen in annotations.
- **08 — Type inference** — when TypeScript figures out the type so you don't have to annotate.
- **12 — Type aliases** — naming complex annotation shapes with `type User = { ... }`.
