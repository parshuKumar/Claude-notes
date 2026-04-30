# 08 — Type Inference

## What is this?

Type inference is TypeScript's ability to **figure out the type of a value automatically** without you writing a type annotation. When you write `const port = 3000`, TypeScript already knows `port` is a `number` — you don't need to write `const port: number = 3000`. The compiler looks at the value on the right side of the assignment and determines the type from it. Inference runs everywhere: variables, function return values, array literals, object literals, and more.

## Why does it matter?

Without inference you'd have to annotate everything, making TypeScript extremely verbose. With inference, you only annotate where TypeScript genuinely cannot figure out the type on its own (mainly function parameters). Understanding where TypeScript infers — and what it infers — helps you write clean code that's still fully type-safe. It also explains some surprising behavior like why `const` and `let` infer different types from the same value.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no types at all, everything is dynamic
const port = 3000;           // could be reassigned to anything
let userName = "Parsh";      // could be reassigned to a number tomorrow
const config = { retries: 3, timeout: 5000 };  // shape unknown to any tool
```

```ts
// TypeScript — same code, but now every value has a known type
const port = 3000;           // TypeScript infers: 3000 (literal type — see below)
let userName = "Parsh";      // TypeScript infers: string (widened)
const config = { retries: 3, timeout: 5000 };  // TypeScript infers: { retries: number; timeout: number }

// No annotations written — TypeScript figured it all out
// But now your editor knows:
port.toFixed(2);             // ✅ TypeScript knows port is a number
userName.toUpperCase();      // ✅ TypeScript knows userName is a string
config.retries.toString();   // ✅ TypeScript knows retries is a number
config.missing;              // ❌ Property 'missing' does not exist — caught!
```

---

## How inference works — the core rules

### Rule 1: Inference from assignment

TypeScript looks at the right-hand side and infers the type:

```ts
let count = 0;              // inferred: number
let label = "active";       // inferred: string
let isEnabled = true;       // inferred: boolean
let createdAt = new Date(); // inferred: Date
let tags = ["admin", "verified"];  // inferred: string[]
let scores = [95, 87, 100];        // inferred: number[]
let mixed = [1, "hello", true];    // inferred: (string | number | boolean)[]
```

### Rule 2: `const` vs `let` — widening vs narrowing

This is one of the most important nuances in TypeScript inference:

```ts
// const — TypeScript infers the LITERAL type (narrow)
// because a const can never be reassigned, the value IS the type
const port = 3000;           // inferred: 3000     (not number — literally 3000)
const method = "GET";        // inferred: "GET"    (not string — literally "GET")
const isActive = true;       // inferred: true     (not boolean — literally true)

// let — TypeScript infers the WIDENED type
// because a let CAN be reassigned, TypeScript widens to the general type
let port2 = 3000;            // inferred: number   (could be reassigned to 9999)
let method2 = "GET";         // inferred: string   (could be reassigned to "POST")
let isActive2 = true;        // inferred: boolean  (could be reassigned to false)
```

**Why this matters:**

```ts
const method = "GET";
// method is type "GET" — the literal — so it can only ever be "GET"

let method2 = "GET";
// method2 is type string — so it can be "GET", "POST", "DELETE", or "anything"

type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

function sendRequest(url: string, method: HttpMethod) { /* ... */ }

const m1 = "GET";        // type: "GET" — assignable to HttpMethod ✅
sendRequest("/users", m1);  // ✅

let m2 = "GET";          // type: string — not assignable to HttpMethod ❌
sendRequest("/users", m2);  // ❌ Argument of type 'string' is not assignable to parameter of type 'HttpMethod'
```

### Rule 3: Inference in objects

```ts
// TypeScript infers the shape of object literals:
const user = {
  id: 1,               // inferred: number
  name: "Parsh",       // inferred: string
  isAdmin: false,      // inferred: boolean
  createdAt: new Date(), // inferred: Date
};
// Full inferred type: { id: number; name: string; isAdmin: boolean; createdAt: Date }

user.id.toFixed();      // ✅ TypeScript knows id is number
user.name.split(" ");   // ✅ TypeScript knows name is string
user.missing;           // ❌ Property 'missing' does not exist

// Object property types are WIDENED (not literal) — because object properties can be reassigned:
user.id = 999;          // ✅ id is number — not literal 1
user.name = "Alice";    // ✅ name is string — not literal "Parsh"
```

### Rule 4: Inference from function return values

```ts
// TypeScript infers the return type from what the function actually returns:
function add(a: number, b: number) {
  return a + b;   // TypeScript infers return type: number
}

function getStatus(code: number) {
  if (code === 200) return "OK";          // string
  if (code === 404) return "Not Found";   // string
  return null;                            // null
}
// TypeScript infers return type: string | null

function buildUser(id: number, name: string) {
  return { id, name, createdAt: new Date() };
}
// TypeScript infers return type: { id: number; name: string; createdAt: Date }
```

### Rule 5: Inference in arrays

```ts
// TypeScript looks at all elements to determine the array type:
const ids = [1, 2, 3];           // inferred: number[]
const names = ["Parsh", "Alice"]; // inferred: string[]
const mixed = [1, "hello"];      // inferred: (string | number)[]
const empty = [];                // inferred: never[] — TypeScript can't tell yet!

// empty array problem — annotate empty arrays:
const userIds: number[] = [];    // ✅ now TypeScript knows it's a number array
userIds.push(42);                // ✅
userIds.push("hello");           // ❌ string not assignable to number
```

---

## When to annotate vs when to let TypeScript infer

This is the key practical question. Here's the complete rule set:

| Situation | Annotate? | Example |
|-----------|-----------|---------|
| Variable with immediate value | ❌ No | `const port = 3000` |
| Variable declared without value | ✅ Yes | `let token: string` |
| Empty array | ✅ Yes | `const ids: number[] = []` |
| Function parameter | ✅ Always | `function fn(userId: number)` |
| Function return type | ✅ Recommended | `function fn(): User \| null` |
| Object with complex shape | ✅ Use a type alias | `const config: ServerConfig = { ... }` |
| Generic functions | ✅ Sometimes | TypeScript often infers T |
| Class properties | ✅ Yes | `private userId: number` |

```ts
// ✅ Don't annotate — TypeScript infers correctly:
const maxRetries = 3;
const baseUrl = "https://api.myapp.com";
const user = { id: 1, name: "Parsh" };
const tags = ["admin", "user"];

// ✅ DO annotate — TypeScript can't infer or inference is wrong:
let authToken: string;                         // no initial value
const userIds: number[] = [];                  // empty array
async function findUser(id: number): Promise<User | null> { /* ... */ }  // return type
```

---

## Type widening — when inference is "too loose"

TypeScript sometimes infers a wider type than you want. This is called **type widening**:

```ts
// Widening example — TypeScript widens to string, not the literal "admin"
let role = "admin";   // inferred: string — NOT "admin"

// This means:
role = "anything";    // ✅ TypeScript allows this — role is string, not "admin"

// If you wanted to lock it to only "admin":
const role2 = "admin";  // inferred: "admin" (literal) — use const
// or
let role3: "admin" = "admin";  // explicit literal type
```

### Preventing widening with `as const`

```ts
// as const freezes the entire structure to its literal types:
const config = {
  method: "GET",    // without as const: inferred as string
  port: 3000,       // without as const: inferred as number
};

const configFrozen = {
  method: "GET",    // with as const: inferred as "GET" (literal)
  port: 3000,       // with as const: inferred as 3000 (literal)
} as const;

// This matters for union type checking:
type HttpMethod = "GET" | "POST";
function request(method: HttpMethod) { /* ... */ }

request(config.method);       // ❌ Argument of type 'string' is not assignable to 'HttpMethod'
request(configFrozen.method); // ✅ "GET" is assignable to HttpMethod
```

---

## Contextual typing — inference flowing backwards

TypeScript can also infer types from **context** — the type flows backwards from where a value is used:

```ts
// TypeScript knows the callback parameter type from the array method's definition:
const userIds = [1, 2, 3];
userIds.forEach(id => {
  // 'id' is inferred as number — TypeScript knows forEach passes numbers
  id.toFixed();   // ✅
});

const names = ["Parsh", "Alice"];
names.map(name => name.toUpperCase());  // 'name' inferred as string

// Event handlers in Express:
import { Request, Response } from 'express';
app.get('/users', (req, res) => {
  // req is inferred as Request, res as Response — no annotation needed
  res.json({ users: [] });   // ✅ TypeScript knows res has json()
});
```

---

## Example 1 — basic

```ts
// Let TypeScript infer — add annotations only where needed

// ✅ Inferred correctly — no annotation needed:
const serverName = "MyAPI";         // string
const defaultPort = 3000;           // number (but 3000 as literal with const!)
const isProduction = false;         // boolean (but false as literal with const!)

// ✅ Must annotate — no initial value:
let currentUserId: number;
let sessionToken: string;
currentUserId = 42;
sessionToken = "Bearer eyJ...";

// ✅ Must annotate — empty array:
const activeConnections: string[] = [];
activeConnections.push("conn-1");   // ✅
activeConnections.push(123);        // ❌ caught — string array only

// Return type inferred from function body:
function buildGreeting(name: string) {
  return `Hello, ${name}!`;         // return type inferred as string
}

const greeting = buildGreeting("Parsh");  // inferred as string
greeting.toUpperCase();   // ✅

// Inferred object shape:
const requestConfig = {
  method: "POST",
  timeout: 5000,
  retries: 3,
};
// TypeScript infers: { method: string; timeout: number; retries: number }

requestConfig.method = "GET";   // ✅ string is fine
requestConfig.timeout = "5s";   // ❌ string not assignable to number
```

---

## Example 2 — real world backend use case

```ts
import express from 'express';

// TypeScript infers the Express app type from the return value of express()
const app = express();   // inferred: Express — no annotation needed

// TypeScript infers router type:
const router = express.Router();  // inferred: Router

// Contextual typing in route handlers — no need to annotate req/res:
router.get('/users/:userId', (req, res) => {
  // req inferred as Request, res inferred as Response — from Express types
  const rawId = req.params.userId;   // inferred: string
  const userId = parseInt(rawId, 10); // inferred: number
  res.json({ userId });               // ✅
});

// Inferred return type from async function:
async function fetchUserFromDb(userId: number) {
  const user = await db.query<{ id: number; email: string }>(
    'SELECT id, email FROM users WHERE id = $1',
    [userId]
  );
  return user.rows[0] ?? null;
  // Return type inferred as: { id: number; email: string } | null
}

// Using as const to lock config shape for type-safe lookups:
const HTTP_STATUS = {
  OK: 200,
  CREATED: 201,
  BAD_REQUEST: 400,
  UNAUTHORIZED: 401,
  NOT_FOUND: 404,
  INTERNAL_ERROR: 500,
} as const;
// Inferred type: { readonly OK: 200; readonly CREATED: 201; ... }

function sendStatus(res: express.Response, key: keyof typeof HTTP_STATUS): void {
  res.status(HTTP_STATUS[key]).send();
}

sendStatus(res, "OK");          // ✅
sendStatus(res, "NOT_FOUND");   // ✅
sendStatus(res, "MISSING");     // ❌ "MISSING" not a key of HTTP_STATUS
```

---

## Common mistakes

### Mistake 1 — Over-annotating what TypeScript already infers

```ts
// ❌ WRONG — redundant annotations create visual noise and maintenance burden
const userId: number = 42;
const email: string = "parsh@dev.io";
const isVerified: boolean = true;
const tags: string[] = ["admin", "verified"];
const user: { id: number; name: string } = { id: 1, name: "Parsh" };

// ✅ RIGHT — let TypeScript do its job
const userId = 42;
const email = "parsh@dev.io";
const isVerified = true;
const tags = ["admin", "verified"];
const user = { id: 1, name: "Parsh" };
// All types are exactly the same — just less noise
```

### Mistake 2 — Not annotating empty arrays (the `never[]` trap)

```ts
// ❌ WRONG — TypeScript infers never[] for empty arrays
const userIds = [];             // inferred: never[]
userIds.push(1);                // ❌ Argument of type 'number' is not assignable to type 'never'
userIds.push("hello");          // ❌ same error
// The array can hold nothing at all — never[]

// ✅ RIGHT — annotate empty arrays
const userIds: number[] = [];   // ✅ TypeScript knows these will be numbers
userIds.push(1);                // ✅
userIds.push(2);                // ✅
```

### Mistake 3 — Trusting widened inference where you need literal types

```ts
// ❌ WRONG — using let when const is needed for literal type inference
type RequestMethod = "GET" | "POST" | "PUT" | "DELETE";

function makeRequest(method: RequestMethod, url: string): void {
  console.log(`${method} ${url}`);
}

let method = "GET";          // inferred: string (widened)
makeRequest(method, "/api"); // ❌ Error: Argument of type 'string' is not assignable to type 'RequestMethod'

// ✅ FIX 1 — use const
const method2 = "GET";       // inferred: "GET" (literal)
makeRequest(method2, "/api"); // ✅

// ✅ FIX 2 — explicit annotation
let method3: RequestMethod = "GET";  // locked to the union type
makeRequest(method3, "/api");        // ✅

// ✅ FIX 3 — as const on the value
let method4 = "GET" as const;  // inferred: "GET" (literal)
makeRequest(method4, "/api");  // ✅
```

---

## Practice exercises

### Exercise 1 — easy

Write 10 variable declarations using inference correctly — do NOT add any type annotations. For each one, add a comment saying what TypeScript would infer. Include: a number constant, a string constant using let, a boolean using const, an array of numbers, an array of strings, an object with at least 3 properties, a Date, a function that returns a string, a function that returns a number, and a function that returns an object.

Then write TWO more variables that DO need an annotation and explain why in a comment.

```ts
// Write your code here
```

### Exercise 2 — medium

You're building a typed HTTP client configuration object. Use `as const` to lock in the types:

```ts
const HTTP_METHODS = { ... } as const   // GET, POST, PUT, DELETE, PATCH as literal values
const STATUS_CODES = { ... } as const   // OK:200, CREATED:201, NOT_FOUND:404, etc.
```

Then write a function `buildRequestConfig` that takes `method` (must be a value from `HTTP_METHODS`) and `endpoint: string` and returns an object with `url`, `method`, `headers` (object with `Content-Type: "application/json"`), and `timeout: 5000`. Rely on TypeScript inference for the return type — do NOT write a return type annotation.

Write two calls to `buildRequestConfig`: one valid, one with a hardcoded string like `"get"` (lowercase) to show the error.

```ts
// Write your code here
```

### Exercise 3 — hard

You're writing a mini API response builder. Using only inference where possible (no explicit return type annotations unless you need to), write:

1. A function `createPaginatedResponse` that takes `items` (any array — TypeScript should infer the element type), `page: number`, `pageSize: number`, and `total: number`. It returns an object with `items`, `page`, `pageSize`, `total`, `totalPages` (calculate it), and `hasNextPage` (boolean). TypeScript should infer the entire return type — verify by hovering over the function name in your editor (or reading the inferred type in a comment).

2. A function `createErrorResponse` that takes `message: string` and an optional `details: string[]`. Returns `{ success: false, error: message, details: details ?? [] }`. TypeScript should infer return type as `{ success: false; error: string; details: string[] }`.

3. Use both functions, assign the results to variables WITHOUT annotations, and verify TypeScript infers the correct types by accessing properties (e.g., `response.hasNextPage` should be `boolean`, `errorResp.success` should be `false`).

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Situation | What TypeScript infers |
|-----------|----------------------|
| `const x = 42` | `42` (literal type) |
| `let x = 42` | `number` (widened) |
| `const x = "hello"` | `"hello"` (literal type) |
| `let x = "hello"` | `string` (widened) |
| `const x = true` | `true` (literal type) |
| `const arr = [1, 2, 3]` | `number[]` |
| `const arr = []` | `never[]` — annotate this! |
| `const obj = { a: 1 }` | `{ a: number }` |
| `const obj = { a: 1 } as const` | `{ readonly a: 1 }` — literal type |
| Function return | Inferred from the return statements |
| Callback params in `.map`/`.forEach` | Inferred from array element type |
| Empty array | Always annotate: `const arr: string[] = []` |

| Pattern | Code |
|---------|------|
| Lock object to literals | `{ method: "GET" } as const` |
| Lock variable to literal | `const m = "GET"` or `let m = "GET" as const` |
| Force type on let | `let x: HttpMethod = "GET"` |
| Get type of value | `typeof myVariable` |
| Get keys of object | `keyof typeof myObject` |

## Connected topics

- **11 — Literal types** — the `"GET"` and `3000` literal types you see in inference with `const`.
- **12 — Type aliases** — naming inferred shapes so you can reuse them.
- **32 — Utility types** — `typeof`, `ReturnType<>`, `Parameters<>` — built on inference.
- **47 — keyof and typeof operators** — using inference-derived types programmatically.
