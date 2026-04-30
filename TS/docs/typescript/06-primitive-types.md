# 06 — Primitive Types

## What is this?

Primitive types are the most basic building blocks in TypeScript: `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, and `bigint`. These are the same primitives that exist in JavaScript — TypeScript just lets you declare them explicitly and enforces them at compile time. Understanding exactly how each behaves in TypeScript (especially `null` and `undefined`) prevents an entire class of runtime bugs.

## Why does it matter?

In JavaScript, primitives silently convert into each other or become `NaN`, `undefined`, or `null` with no warning. TypeScript treats each primitive as a distinct, non-interchangeable type. The biggest wins are around `null` and `undefined` — in JS these cause more runtime crashes than almost anything else. With TypeScript's strict null checks, you can never accidentally use a value that might be `null` or `undefined` without handling it first.

## The JavaScript way vs the TypeScript way

### JavaScript — primitives are loose

```js
// JS doesn't distinguish — all of these are accepted
function setPort(port) {
  return port;
}

setPort(3000);         // fine
setPort("3000");       // fine — but now it's a string, not a number
setPort(null);         // fine — but will crash when you do arithmetic
setPort(undefined);    // fine — but port is now undefined
setPort(true);         // fine — JS treats true as 1 in arithmetic

// No warning, no error — bugs hide until runtime
```

### TypeScript — each primitive is its own type

```ts
function setPort(port: number): number {
  return port;
}

setPort(3000);         // ✅ correct
setPort("3000");       // ❌ Argument of type 'string' is not assignable to parameter of type 'number'
setPort(null);         // ❌ Argument of type 'null' is not assignable to parameter of type 'number'
setPort(undefined);    // ❌ Argument of type 'undefined' is not assignable to parameter of type 'number'
setPort(true);         // ❌ Argument of type 'boolean' is not assignable to parameter of type 'number'
```

Every wrong call is caught before it runs.

---

## The seven primitive types

### 1. `string`

```ts
// Holds any text — single quotes, double quotes, or template literals
let serverHost: string = "localhost";
let welcomeMessage: string = 'Hello, Parsh';
let connectionString: string = `mongodb://${serverHost}:27017/mydb`;

// TypeScript knows all string methods are available
const upperHost = serverHost.toUpperCase();   // ✅ string method, TypeScript approves

// What TypeScript prevents:
serverHost = 42;        // ❌ Type 'number' is not assignable to type 'string'
serverHost = null;      // ❌ Type 'null' is not assignable to type 'string' (with strictNullChecks)
```

### 2. `number`

```ts
// TypeScript has ONE number type — no int/float/double distinction (same as JS)
let userId: number = 101;
let price: number = 29.99;
let negativeBalance: number = -500;
let hexColor: number = 0xff00ff;     // hexadecimal — still type number
let binaryFlag: number = 0b1010;     // binary — still type number

// Special number values — these ARE type number in TypeScript
let notANumber: number = NaN;        // ✅ NaN is a number type
let infinity: number = Infinity;     // ✅ Infinity is a number type

// What TypeScript prevents:
userId = "101";   // ❌ cannot assign string to number
userId = true;    // ❌ cannot assign boolean to number

// Watch out: NaN and Infinity passing type checks
// They are valid 'number' values but often bugs in disguise:
function divide(a: number, b: number): number {
  return a / b;    // b = 0 → returns Infinity; TypeScript doesn't catch this
}
```

### 3. `boolean`

```ts
// Only two possible values: true or false
let isActive: boolean = true;
let isEmailVerified: boolean = false;

// TypeScript prevents JS's "truthy/falsy" confusion
function setAdminStatus(isAdmin: boolean): void {
  console.log(`Admin: ${isAdmin}`);
}

setAdminStatus(true);      // ✅
setAdminStatus(1);         // ❌ Type 'number' is not assignable to type 'boolean'
setAdminStatus("yes");     // ❌ Type 'string' is not assignable to type 'boolean'
setAdminStatus(null);      // ❌ Type 'null' is not assignable to type 'boolean'

// In JS: if (1) is truthy, so you could pass 1 as a boolean — TS stops this
```

### 4. `null`

```ts
// null is its own type in TypeScript — it means "intentionally no value"
// With strictNullChecks: true, null is NOT assignable to string/number/boolean

let selectedUserId: number | null = null;    // explicitly allow null
selectedUserId = 42;                          // ✅
selectedUserId = null;                        // ✅ null is allowed because we said so
selectedUserId = undefined;                   // ❌ undefined ≠ null in TypeScript

// TypeScript forces you to handle null before using the value:
function getUserEmail(userId: number | null): string {
  if (userId === null) {
    return "no user selected";               // ✅ handled the null case
  }
  return `email for user ${userId}`;         // ✅ TypeScript knows userId is number here
}

// Without the null check:
function getUserEmailBroken(userId: number | null): string {
  return `email for user ${userId.toString()}`;  // ❌ Error: Object is possibly 'null'
}
```

### 5. `undefined`

```ts
// undefined = a value that hasn't been assigned yet
// null = intentionally empty
// TypeScript treats these as separate types (unlike many JS codebases that mix them)

let authToken: string | undefined = undefined;  // not assigned yet
authToken = "Bearer eyJhbGci...";               // ✅ assigned later

// Optional object properties produce undefined when accessed:
type User = {
  id: number;
  name: string;
  bio?: string;    // the ? makes bio optional — it's string | undefined
};

const user: User = { id: 1, name: "Parsh" };
const bio: string | undefined = user.bio;    // TypeScript knows bio might be undefined

// Must handle undefined before using:
const bioLength = user.bio?.length;          // ✅ optional chaining — undefined if bio is missing
// const bioLength = user.bio.length;        // ❌ Object is possibly 'undefined'
```

### `null` vs `undefined` — the practical distinction

```ts
// Convention in TypeScript / backend code:
// null  → intentional absence of a value (e.g., "no user found in DB")
// undefined → value not provided / missing / not yet set

type ApiResponse<T> = {
  data: T | null;          // null = "there is no data" (intentional)
  error: string | null;    // null = "there is no error"
};

type User = {
  id: number;
  middleName?: string;     // undefined = "might not be provided" (optional)
};
```

---

### 6. `symbol`

```ts
// Symbols are unique, immutable identifiers — rarely used in typical backends
// But TypeScript handles them as their own type

const requestId: symbol = Symbol("requestId");
const sessionId: symbol = Symbol("sessionId");

// Every Symbol() call creates a UNIQUE symbol — even with the same description
console.log(Symbol("id") === Symbol("id"));   // false — always unique

// Real use case in backend: unique property keys to avoid collisions
const USER_KEY = Symbol("user");
const ROLE_KEY = Symbol("role");

// Used as Map keys for guaranteed uniqueness:
const sessionStore = new Map<symbol, { userId: number }>();
sessionStore.set(requestId, { userId: 42 });

// TypeScript type:
const mySymbol: symbol = Symbol("test");        // general symbol type
const myUniqueSymbol: unique symbol = Symbol(); // even more specific — this exact symbol
```

### 7. `bigint`

```ts
// bigint holds integers larger than Number.MAX_SAFE_INTEGER (2^53 - 1)
// Used when working with very large IDs, cryptography, financial calculations

const maxSafeNumber: number = Number.MAX_SAFE_INTEGER;   // 9007199254740991
const bigUserId: bigint = 9007199254740993n;             // the 'n' suffix makes it bigint

// TypeScript keeps number and bigint STRICTLY separate — no implicit conversion
const regularNumber: number = 42;
const bigNumber: bigint = 42n;

// Cannot mix:
const result = regularNumber + bigNumber;   // ❌ Operator '+' cannot be applied to types 'number' and 'bigint'

// Must convert explicitly:
const result2 = regularNumber + Number(bigNumber);   // ✅
const result3 = BigInt(regularNumber) + bigNumber;   // ✅

// Real use case: database record IDs in PostgreSQL can exceed JS safe integer range
// When using pg/postgres, use bigint for large serial IDs:
const recordId: bigint = BigInt("9007199254740993");
```

---

## How TypeScript differs from JavaScript for each primitive

| Primitive | JS behavior | TS difference |
|-----------|------------|---------------|
| `string` | Accepts anything | Only actual strings allowed |
| `number` | `NaN`, `Infinity` pass silently | `NaN`/`Infinity` are valid `number` — TypeScript can't detect bad division |
| `boolean` | Truthy/falsy used as boolean | Only `true`/`false` — no `1` or `"yes"` |
| `null` | Assignable to anything | Separate type; only allowed where explicitly declared with `| null` |
| `undefined` | Assignable to anything | Separate type; only allowed where explicitly declared with `| undefined` or `?` |
| `symbol` | No types | `symbol` and `unique symbol` types |
| `bigint` | No types, silent coercion | Completely separate from `number`, cannot mix |

---

## Example 1 — basic

```ts
// All seven primitives in one place
let userName: string = "Parsh";
let userAge: number = 25;
let isPremium: boolean = true;
let deletedAt: Date | null = null;          // null = not deleted yet
let referralCode: string | undefined;       // undefined = not provided yet
const requestTraceId: symbol = Symbol("trace");
const walletBalance: bigint = 1000000000000000000n;  // very large amount

// Function that uses several primitives together
function buildUserSummary(
  name: string,
  age: number,
  isPremium: boolean,
  referralCode: string | undefined
): string {
  const premiumLabel = isPremium ? "[PREMIUM]" : "[FREE]";
  const referral = referralCode ?? "none";           // ?? = nullish coalescing
  return `${premiumLabel} ${name}, age ${age}, referral: ${referral}`;
}

console.log(buildUserSummary("Parsh", 25, true, undefined));
// "[PREMIUM] Parsh, age 25, referral: none"

console.log(buildUserSummary("Alice", 30, false, "SAVE20"));
// "[FREE] Alice, age 30, referral: SAVE20"
```

---

## Example 2 — real world backend use case

```ts
// Typing a user record as it moves through a backend pipeline

// Raw data from DB — some fields might be null (not set yet)
type UserRecord = {
  id: number;
  email: string;
  hashedPassword: string;
  isEmailVerified: boolean;
  lastLoginAt: Date | null;        // null = never logged in
  deletedAt: Date | null;          // null = not deleted
  profileBio: string | null;       // null = no bio set
  referralCode: string | undefined; // undefined = not in old records
};

// Parsing raw DB output into typed UserRecord
function parseUserRow(row: {
  id: number;
  email: string;
  hashed_password: string;
  is_email_verified: boolean;
  last_login_at: string | null;    // DB returns dates as strings or null
  deleted_at: string | null;
  profile_bio: string | null;
  referral_code: string | undefined;
}): UserRecord {
  return {
    id: row.id,
    email: row.email,
    hashedPassword: row.hashed_password,
    isEmailVerified: row.is_email_verified,
    lastLoginAt: row.last_login_at ? new Date(row.last_login_at) : null,
    deletedAt: row.deleted_at ? new Date(row.deleted_at) : null,
    profileBio: row.profile_bio,
    referralCode: row.referral_code,
  };
}

// A guard function that checks if the user is allowed to log in
function canUserLogin(user: UserRecord): boolean {
  if (user.deletedAt !== null) return false;          // TypeScript knows: Date, not null
  if (!user.isEmailVerified) return false;
  return true;
}

// Using optional chaining on nullable fields
function getLastLoginDisplay(user: UserRecord): string {
  return user.lastLoginAt?.toLocaleDateString() ?? "Never logged in";
  // lastLoginAt is Date | null — ?. handles null safely
}
```

---

## Common mistakes

### Mistake 1 — Confusing `null` and `undefined`

```ts
// ❌ WRONG — using null and undefined interchangeably
type UserSession = {
  userId: number | null | undefined;   // which one do you mean?? Inconsistent
};

// When reading this: does null mean "logged out"? Does undefined mean "not checked"?
// Ambiguous and error-prone.

// ✅ RIGHT — be explicit and consistent
type UserSession = {
  userId: number | null;   // null = intentionally no user (logged out)
};

type UserDraft = {
  userId?: number;          // undefined via ? = field not yet provided
};
```

### Mistake 2 — Thinking `number` catches NaN

```ts
// ❌ DANGEROUS — TypeScript says this is fine but it isn't
function parseUserId(rawId: string): number {
  return parseInt(rawId, 10);   // TypeScript thinks this always returns number
  // But parseInt("abc", 10) returns NaN — and NaN is type number!
}

const userId = parseUserId("abc");
userId.toFixed(2);   // ✅ TypeScript says fine — but at runtime: NaN.toFixed(2) = "NaN"

// ✅ RIGHT — validate the output
function parseUserId(rawId: string): number | null {
  const parsed = parseInt(rawId, 10);
  if (isNaN(parsed)) return null;     // explicitly handle NaN
  return parsed;
}
```

### Mistake 3 — Assigning `null` without enabling it in the type

```ts
// ❌ WRONG — trying to assign null without declaring it in the type
let activeRequestId: number = 42;
activeRequestId = null;   // ❌ Error: Type 'null' is not assignable to type 'number'

// This is actually TypeScript PROTECTING you — a plain number should never be null.
// If null is a valid state, you must be explicit about it:

// ✅ RIGHT — declare the null possibility up front
let activeRequestId: number | null = 42;
activeRequestId = null;    // ✅ intentionally cleared
```

---

## Practice exercises

### Exercise 1 — easy

Declare variables for each of the seven primitive types with realistic backend names. Then write a function `describeServer` that takes `host: string`, `port: number`, `isSecure: boolean` and returns a string like `"https://api.myapp.com:443"` (if `isSecure` is true) or `"http://api.myapp.com:3000"` (if false). Annotate all parameters and the return type.

```ts
// Write your code here
```

### Exercise 2 — medium

Write a function `parseRequestParam` that takes a single parameter `rawValue: string | null | undefined` and returns `number | null`. Rules:
- If `rawValue` is `null` or `undefined`, return `null`.
- If `rawValue` is a string that parses to a valid number (not `NaN`), return that number.
- If `rawValue` is a string that is not a valid number, return `null`.

Then write a second function `buildQueryFilter` that takes `userId: number | null` and `minAge: number | null` and returns a string like `"WHERE user_id = 42 AND age >= 18"`, `"WHERE user_id = 42"`, `"WHERE age >= 18"`, or `""` (empty string if both are null). All annotations must be explicit.

```ts
// Write your code here
```

### Exercise 3 — hard

You're building a user profile update handler for an Express route. The incoming request body has these fields — all potentially missing or null from the client:

```ts
type UpdateProfileBody = {
  displayName: string | undefined;
  bio: string | null | undefined;
  age: number | undefined;
  isPublic: boolean | undefined;
};
```

Write a function `buildProfileUpdatePayload` that takes `body: UpdateProfileBody` and returns a `Partial<UpdateProfileBody>` — an object containing ONLY the fields that were explicitly provided (not `undefined`). Fields that are `null` should be included (null means "clear this field"). Fields that are `undefined` should be omitted entirely.

Example:
- Input: `{ displayName: "Parsh", bio: null, age: undefined, isPublic: true }`
- Output: `{ displayName: "Parsh", bio: null, isPublic: true }` (age is omitted because it was undefined)

Then write a function `validateAge` that takes `age: number | undefined` and returns `string | null` — `null` if valid (or undefined), or an error message string if the age is a number but outside 13–120. Do not flag `undefined` as an error — it just means not provided.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Type | Values | Notes |
|------|--------|-------|
| `string` | `"hello"`, `'world'`, `` `template` `` | All text |
| `number` | `42`, `3.14`, `NaN`, `Infinity` | NaN and Infinity are valid number — validate separately |
| `boolean` | `true`, `false` only | No truthy/falsy coercion |
| `null` | `null` only | Intentional absence of value |
| `undefined` | `undefined` only | Unassigned / missing |
| `symbol` | `Symbol("desc")` | Unique, rarely used in backends |
| `bigint` | `9007199254740993n` | Large integers, cannot mix with number |

| Pattern | Syntax |
|---------|--------|
| Allow null | `type \| null` |
| Allow undefined | `type \| undefined` or optional `?` |
| Allow both | `type \| null \| undefined` |
| Nullish coalescing | `value ?? "default"` |
| Optional chaining | `obj?.prop` |
| Null check before use | `if (x !== null) { ... }` |
| NaN check | `if (isNaN(n)) { ... }` |

## Connected topics

- **07 — Special types** — `any`, `unknown`, `never`, `void` — the non-primitive TS types.
- **08 — Type inference** — how TypeScript guesses the primitive type from a value.
- **09 — Union types** — combining primitives like `string | number | null`.
- **62 — Strict null checks** — deep dive into how `null` and `undefined` are handled safely.
