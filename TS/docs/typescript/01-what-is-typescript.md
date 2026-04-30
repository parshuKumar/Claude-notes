# 01 — What is TypeScript

## What is this?

TypeScript is JavaScript with a type system bolted on top. Every valid JavaScript file is already valid TypeScript — TS is a **superset** of JS. The TypeScript compiler (`tsc`) checks your types at compile time and then strips them away, outputting plain JavaScript that runs anywhere JS runs (Node.js, browsers, Deno).

## Why does it matter?

In JavaScript, bugs hide until runtime. You ship code, a user hits a path you didn't test, and `Cannot read properties of undefined` crashes your server at 3 AM. TypeScript catches these bugs **before** your code ever runs — at the moment you write it. Companies like Microsoft, Google, Airbnb, and Stripe use TypeScript because it eliminates entire categories of bugs, makes refactoring safe, and gives developers autocomplete that actually knows what your data looks like.

## The JavaScript way vs the TypeScript way

### JavaScript — no safety net

```js
// You write a function. Nothing stops you from passing wrong data.
function getUser(userId) {
  // userId could be a number, string, object, undefined... JS doesn't care
  return { id: userId, name: "Parsh" };
}

// This works fine
const user = getUser(42);

// This also "works" — but it's a bug. No error until runtime.
const broken = getUser(undefined);
console.log(broken.id.toString()); // 💥 Runtime crash: Cannot read properties of undefined
```

### TypeScript — caught before it runs

```ts
// You declare: userId MUST be a number. Period.
function getUser(userId: number) {
  return { id: userId, name: "Parsh" };
}

// This works fine
const user = getUser(42);

// This is caught IMMEDIATELY — red underline in your editor
const broken = getUser(undefined);
// ❌ Error: Argument of type 'undefined' is not assignable to parameter of type 'number'
```

The bug never reaches production. You see the error the moment you type it.

## Syntax

```ts
// TypeScript adds type annotations after a colon
let userName: string = "Parsh";       // variable: type = value
let userId: number = 101;             // only numbers allowed
let isActive: boolean = true;         // only true/false

// Function parameters get types, and so does the return value
function greet(name: string): string {
  return `Hello, ${name}`;
}

// Objects can have their shape defined
let user: { id: number; name: string } = { id: 1, name: "Parsh" };
```

## How it works — line by line

1. `let userName: string = "Parsh"` — The `: string` after the variable name is a **type annotation**. It tells TypeScript: "this variable can ONLY hold a string." If you later try `userName = 42`, TypeScript refuses.

2. `function greet(name: string): string` — The `name: string` types the parameter. The `: string` after the parentheses types the **return value**. TypeScript guarantees this function always returns a string.

3. `let user: { id: number; name: string }` — This is an **inline object type**. It defines the exact shape the object must have. Missing a property or adding an extra one causes an error.

4. TypeScript is **compiled** — you run `tsc yourfile.ts` and it outputs `yourfile.js` with all type annotations removed. The JavaScript runtime never sees the types. They exist only during development.

## Example 1 — basic

```ts
// Declare a variable with a type
let authToken: string = "eyJhbGciOiJIUzI1NiJ9";

// TypeScript won't let you reassign to a wrong type
// authToken = 123; // ❌ Error: Type 'number' is not assignable to type 'string'

// A simple typed function
function formatUserId(userId: number): string {
  // Must return a string because we said so
  return `USER-${userId}`;
}

// Works perfectly
const formatted = formatUserId(42); // "USER-42"

// Caught at compile time — not at 3 AM in production
// formatUserId("abc"); // ❌ Error: Argument of type 'string' is not assignable to parameter of type 'number'
```

## Example 2 — real world backend use case

```ts
// Imagine you're building a REST API. In JS, you have no guarantee
// about what shape your data is. In TS, you define it explicitly.

// Define the shape of a User object
type User = {
  id: number;
  email: string;
  role: string;
  createdAt: Date;
};

// Define what the API response looks like
type ApiResponse = {
  success: boolean;
  data: User | null;
  error: string | null;
};

// A function that builds a consistent API response
function buildUserResponse(user: User | null, errorMessage: string | null): ApiResponse {
  return {
    success: user !== null,
    data: user,
    error: errorMessage,
  };
}

// Using it — TypeScript ensures you pass the right shape
const response = buildUserResponse(
  { id: 1, email: "parsh@dev.io", role: "admin", createdAt: new Date() },
  null
);

// If you forget a field, TypeScript catches it instantly:
// buildUserResponse({ id: 1, email: "parsh@dev.io" }, null);
// ❌ Error: Property 'role' is missing in type '{ id: number; email: string; }'
```

## Common mistakes

### Mistake 1 — Thinking TypeScript runs in production

```ts
// ❌ WRONG thinking: "TypeScript adds runtime type checking"
// TypeScript types are ERASED after compilation. They don't exist at runtime.

// This TS code:
let port: number = 3000;

// Compiles to this JS:
// let port = 3000;
// No type information remains. TS is a DEVELOPMENT tool, not a runtime tool.
```

### Mistake 2 — Using `any` to silence errors

```ts
// ❌ WRONG: Slapping `any` on everything to shut TypeScript up
let userData: any = fetchUser();
userData.nmae; // No error! But "nmae" is a typo. You just defeated the point of TS.

// ✅ RIGHT: Define the actual type
type UserData = { id: number; name: string; email: string };
let userData: UserData = fetchUser();
userData.nmae; // ❌ TypeScript catches the typo immediately
```

### Mistake 3 — Thinking you need to rewrite all JS to use TypeScript

```ts
// ❌ WRONG: "I need to convert my whole project at once"
// TypeScript is a SUPERSET of JavaScript. You can rename .js to .ts
// and it already works. Add types gradually, file by file.

// This is valid TypeScript (it's just JavaScript):
function add(a, b) {
  return a + b;
}
// TypeScript will infer what it can and warn you gently about the rest.

// ✅ RIGHT approach: Migrate one file at a time. Add types where they help most first.
```

## Practice exercises

### Exercise 1 — easy

Write a TypeScript function called `calculateTotal` that takes two parameters: `price` (number) and `quantity` (number), and returns a number. Add a variable `orderTotal` that stores the result of calling this function.

```ts
// Write your code here
```

### Exercise 2 — medium

Write a TypeScript type called `ServerConfig` that has these properties: `host` (string), `port` (number), `isSecure` (boolean), and `dbConnectionString` (string). Then write a function `createServer` that accepts a `ServerConfig` and returns a string message like `"Server running on host:port"`.

```ts
// Write your code here
```

### Exercise 3 — hard

You're building a backend logger. Write a type called `LogEntry` with properties: `timestamp` (Date), `level` (string), `message` (string), `userId` (number or null — because some logs are system-level with no user). Write a function `createLogEntry` that takes `level`, `message`, and `userId` as parameters and returns a `LogEntry` object (it should auto-generate the timestamp). Then write a function `formatLog` that takes a `LogEntry` and returns a formatted string like `"[ERROR] 2026-05-01T10:00:00 | User: 42 | Something broke"`. If userId is null, print `"User: SYSTEM"` instead.

```ts
// Write your code here
```

## Quick reference cheat sheet

| Concept | Summary |
|---------|---------|
| TypeScript is... | A superset of JavaScript that adds static types |
| Types exist at... | Compile time only — erased in output JS |
| Compiler command | `tsc filename.ts` → outputs `filename.js` |
| Type annotation | `let x: string = "hello"` |
| Function typing | `function fn(param: type): returnType {}` |
| Object typing | `let obj: { key: type; key2: type }` |
| Why use it? | Catch bugs before runtime, better autocomplete, safer refactoring |
| Can I mix JS and TS? | Yes — TS is a superset, all JS is valid TS |
| Does TS run in Node? | No — it compiles to JS first, then JS runs in Node |

## Connected topics

- **02 — Setting up TypeScript** — How to install and configure TypeScript so you can actually run these examples.
- **05 — Type annotations** — Deep dive into the `: type` syntax you saw here.
- **07 — Special types (any, unknown, never, void)** — Understanding the escape hatches and when to never use them.
