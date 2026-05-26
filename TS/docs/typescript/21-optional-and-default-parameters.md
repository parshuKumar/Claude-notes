# 21 — Optional and Default Parameters

## What is this?

TypeScript gives you two ways to make a function parameter non-mandatory:

1. **Optional parameter** (`?`) — the parameter may be omitted by the caller; TypeScript widens its type to `T | undefined` inside the function
2. **Default parameter** (`= value`) — the parameter has a fallback value; if the caller omits it (or passes `undefined`), the default is used; the type is inferred from the default

These are not new concepts — JavaScript has default parameters (`= value`) since ES6. What TypeScript adds is the `?` syntax (which JavaScript doesn't have), strict checking that required parameters aren't omitted, and precise type inference around defaults.

## Why does it matter?

In JavaScript, every parameter is implicitly optional — you can always call `fn()` with fewer arguments than it declares. TypeScript locks this down: by default, all parameters are **required**. If you omit one, it's a compile error. This means you must explicitly opt into optional/default behavior, which:

- Makes function signatures self-documenting — callers know exactly what's required vs optional
- Forces you to handle the `undefined` case inside the function body
- Prevents silent `undefined` bugs from accidental missing arguments
- Lets you express "this parameter has a sensible default" without runtime `param = param ?? default` boilerplate

## The JavaScript way vs the TypeScript way

```js
// JavaScript — every parameter is silently optional
function createUser(email, name, role, options) {
  const r = role || "viewer";             // runtime default — fragile (falsy trap)
  const opts = options || {};             // runtime default
  const notify = opts.sendWelcomeEmail !== false; // deeply uncertain
  // All four params could be undefined and JS doesn't warn you
}

createUser();               // works — everything is undefined
createUser("a@dev.io");     // works — name is undefined, no warning
createUser("a@dev.io", "Parsh", 0);  // bug! role is 0, falsy, gets defaulted to "viewer"
```

```ts
// TypeScript — required vs optional is explicit and enforced
interface CreateUserOptions {
  sendWelcomeEmail?: boolean;
  notifyAdmins?: boolean;
}

function createUser(
  email: string,                           // required
  name: string,                            // required
  role: "admin" | "editor" | "viewer" = "viewer",  // default — optional to caller
  options: CreateUserOptions = {},         // default — optional to caller
): User {
  const notify = options.sendWelcomeEmail ?? true;
  // ...
  return { id: Date.now(), email, name, role, createdAt: new Date(), updatedAt: new Date() };
}

createUser();                              // ❌ Missing 'email' and 'name'
createUser("a@dev.io");                    // ❌ Missing 'name'
createUser("a@dev.io", "Parsh");           // ✅ role="viewer", options={}
createUser("a@dev.io", "Parsh", "admin"); // ✅
createUser("a@dev.io", "Parsh", "admin", { sendWelcomeEmail: false }); // ✅
createUser("a@dev.io", "Parsh", 0);        // ❌ 0 is not assignable to "admin" | "editor" | "viewer"
```

---

## Syntax

```ts
// ── OPTIONAL PARAMETER (?) ─────────────────────────────────────────────────
function sendEmail(to: string, subject: string, cc?: string): void {
  //                                                  ^^^
  // cc is: string | undefined inside the function body
}

// ── DEFAULT PARAMETER (= value) ────────────────────────────────────────────
function paginate(page: number = 1, pageSize: number = 20): { skip: number; take: number } {
  return { skip: (page - 1) * pageSize, take: pageSize };
}

// ── DEFAULT OBJECT PARAMETER ───────────────────────────────────────────────
interface QueryOptions {
  page?: number;
  pageSize?: number;
  sortBy?: string;
  sortOrder?: "asc" | "desc";
}

function findUsers(filter: string, options: QueryOptions = {}): Promise<User[]> {
  const { page = 1, pageSize = 20, sortBy = "createdAt", sortOrder = "desc" } = options;
  // ...
  return Promise.resolve([]);
}

// ── OPTIONAL IN TYPE ALIAS ────────────────────────────────────────────────
type LogFn = (message: string, level?: "info" | "warn" | "error") => void;

// ── OPTIONAL IN INTERFACE ─────────────────────────────────────────────────
interface EmailService {
  send(to: string, subject: string, body: string, replyTo?: string): Promise<void>;
}
```

---

## How it works — rule by rule

### Required vs optional vs default — the three states

```ts
function example(
  required: string,               // REQUIRED — caller must provide
  optional?: string,              // OPTIONAL — caller may omit; type is string | undefined
  withDefault: string = "hello",  // DEFAULT  — caller may omit; always string inside fn
): void {}

// Inside the function body:
// required    → string              (always defined)
// optional    → string | undefined  (must guard before use)
// withDefault → string              (always defined — default kicks in if omitted)
```

### Optional parameter `?` — must guard before use

```ts
function createRoute(
  method: "GET" | "POST" | "PUT" | "DELETE",
  path: string,
  description?: string,  // string | undefined inside the function
): RouteDefinition {

  // ❌ WRONG — description might be undefined:
  return { method, path, description: description.toUpperCase() };
  // Error: Object is possibly 'undefined'

  // ✅ RIGHT — guard first:
  return { method, path, description: description?.toUpperCase() };

  // ✅ ALSO RIGHT — provide a fallback:
  return { method, path, description: description ?? "No description" };
}

// Caller can omit the parameter entirely:
createRoute("GET", "/api/users");                    // ✅ description = undefined
createRoute("GET", "/api/users", "List all users");  // ✅ description = "List all users"

// Caller can also explicitly pass undefined (same as omitting):
createRoute("GET", "/api/users", undefined);         // ✅ same as omitting
```

### Default parameter — type is inferred from the default value

```ts
// TypeScript infers the type from the default:
function retry(fn: () => Promise<void>, maxAttempts = 3, delayMs = 1000) {
  //                                                 ^^^^^          ^^^^^
  // TypeScript infers: maxAttempts: number, delayMs: number
}

// You can still annotate explicitly (useful when the default is a complex expression):
function buildQuery(
  table: string,
  conditions: Record<string, unknown> = {},  // inferred as Record<string, unknown>
  limit: number = 100,                        // inferred as number
): string {
  // ...
  return "";
}
```

### Default parameter — passing `undefined` triggers the default

This is a subtle but important difference from the `?` syntax:

```ts
function greet(name: string = "World"): string {
  return `Hello, ${name}!`;
}

greet();              // "Hello, World!"  — omitted → default
greet(undefined);     // "Hello, World!"  — undefined → default (same as omitting!)
greet("Parsh");       // "Hello, Parsh!"  — explicit value

// With optional (?), passing undefined keeps it undefined:
function greetOpt(name?: string): string {
  return `Hello, ${name ?? "World"}!`;
}

greetOpt();           // "Hello, World!"   — omitted → undefined → fallback
greetOpt(undefined);  // "Hello, World!"   — explicitly undefined → fallback
greetOpt("Parsh");    // "Hello, Parsh!"
```

Both behave the same at the call site — but inside the function, a `?` parameter is `T | undefined` and you must handle it, while a default parameter gives you `T` directly.

### Ordering rules

```ts
// ✅ CORRECT — required parameters come first, optional/default at the end:
function createPost(
  title: string,           // required
  content: string,         // required
  authorId: number,        // required
  status = "draft",        // default — can be omitted
  tags: string[] = [],     // default — can be omitted
  publishedAt?: Date,      // optional — can be omitted
): Post { /* ... */ return {} as Post; }

// ❌ WRONG — required parameter after optional:
function badFn(a?: string, b: string): void {}
// Error: A required parameter cannot follow an optional parameter

// ✅ EXCEPTION — default parameters CAN appear before required ones,
// but you must pass undefined explicitly to skip them:
function tricky(a: number = 0, b: string): string {
  return `${a}: ${b}`;
}
tricky(undefined, "hello");  // ✅ a = 0, b = "hello"
tricky(5, "hello");          // ✅ a = 5, b = "hello"
// (This is rarely done intentionally — keep optionals at the end)
```

### Destructured object parameters with defaults

A common pattern in backend code — accept an options object with defaults at the destructure level:

```ts
interface PaginateOptions {
  page?: number;
  pageSize?: number;
  sortBy?: string;
  sortOrder?: "asc" | "desc";
}

// ── Pattern 1: default the whole object, then destructure with defaults ─────
function getUsers({ page = 1, pageSize = 20, sortBy = "id", sortOrder = "asc" }: PaginateOptions = {}): string {
  return `SELECT * FROM users ORDER BY ${sortBy} ${sortOrder} LIMIT ${pageSize} OFFSET ${(page - 1) * pageSize}`;
}

getUsers();                             // ✅ all defaults
getUsers({ page: 2 });                  // ✅ page=2, others default
getUsers({ page: 3, pageSize: 50 });    // ✅

// ── Pattern 2: separate options param with default ─────────────────────────
function queryTable(
  table: string,
  where: Record<string, unknown>,
  options: PaginateOptions = {},
): string {
  const { page = 1, pageSize = 20, sortBy = "id", sortOrder = "asc" } = options;
  return `SELECT * FROM ${table} ORDER BY ${sortBy} ${sortOrder} LIMIT ${pageSize}`;
}
```

### Optional + default in callbacks and interfaces

```ts
// Callback with optional parameter — caller can pass a function that ignores the second arg:
type OnSuccess<T> = (data: T, requestId?: string) => void;

function fetchData<T>(url: string, onSuccess: OnSuccess<T>): void {
  // onSuccess can be: (data) => ... OR (data, requestId) => ...
}

// Interface method with optional params:
interface Logger {
  log(message: string, level?: "info" | "warn" | "error", meta?: Record<string, unknown>): void;
}

// Class implementing it must also accept these optional params:
class AppLogger implements Logger {
  log(message: string, level: "info" | "warn" | "error" = "info", meta?: Record<string, unknown>): void {
    // level has a default here — still satisfies the interface
    console.log(`[${level.toUpperCase()}] ${message}`, meta ?? "");
  }
}
```

---

## Example 1 — basic

```ts
// A set of typed utility functions demonstrating optional and default params

// Pagination helper — all parameters optional with sensible defaults:
function buildPaginationQuery(
  page: number = 1,
  pageSize: number = 20,
  maxPageSize: number = 100,
): { offset: number; limit: number; page: number; pageSize: number } {
  const safePageSize = Math.min(pageSize, maxPageSize);
  const safePage = Math.max(1, page);
  return {
    offset: (safePage - 1) * safePageSize,
    limit: safePageSize,
    page: safePage,
    pageSize: safePageSize,
  };
}

buildPaginationQuery();           // offset:0, limit:20, page:1, pageSize:20
buildPaginationQuery(3);          // offset:40, limit:20, page:3, pageSize:20
buildPaginationQuery(1, 50);      // offset:0,  limit:50, page:1, pageSize:50
buildPaginationQuery(1, 500, 100); // offset:0, limit:100 (capped), page:1, pageSize:100

// HTTP response builder — status is optional (defaults to 200):
function buildApiResponse<T>(
  data: T,
  statusCode: number = 200,
  meta?: { requestId: string; duration: number },
): { statusCode: number; body: { success: boolean; data: T; meta?: { requestId: string; duration: number } } } {
  return {
    statusCode,
    body: { success: statusCode < 400, data, meta },
  };
}

buildApiResponse({ userId: 1 });                               // statusCode: 200, no meta
buildApiResponse({ userId: 1 }, 201);                         // statusCode: 201
buildApiResponse(null, 404);                                   // statusCode: 404
buildApiResponse({ userId: 1 }, 200, { requestId: "abc", duration: 12 }); // with meta

// Retry helper with defaults:
async function withRetry<T>(
  operation: () => Promise<T>,
  maxAttempts: number = 3,
  delayMs: number = 500,
  backoffMultiplier: number = 2,
): Promise<T> {
  let lastError: Error = new Error("No attempts made");

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (err) {
      lastError = err instanceof Error ? err : new Error(String(err));
      if (attempt < maxAttempts) {
        await new Promise(resolve => setTimeout(resolve, delayMs * Math.pow(backoffMultiplier, attempt - 1)));
      }
    }
  }

  throw lastError;
}

// Callers only specify what differs from defaults:
await withRetry(() => fetchUser(1));                   // 3 attempts, 500ms delay, 2x backoff
await withRetry(() => fetchUser(1), 5);                // 5 attempts
await withRetry(() => fetchUser(1), 5, 1000);          // 5 attempts, 1s delay
await withRetry(() => fetchUser(1), 5, 1000, 1.5);     // full custom
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response } from 'express';

// ── Query builder with all-optional options object ─────────────────────────

interface FindUsersOptions {
  page?: number;
  pageSize?: number;
  role?: "admin" | "editor" | "viewer";
  search?: string;
  sortBy?: "name" | "email" | "createdAt";
  sortOrder?: "asc" | "desc";
  includeDeleted?: boolean;
}

async function findUsers(options: FindUsersOptions = {}): Promise<{ users: User[]; total: number }> {
  const {
    page = 1,
    pageSize = 20,
    role,
    search,
    sortBy = "createdAt",
    sortOrder = "desc",
    includeDeleted = false,
  } = options;

  // Build query (pseudocode):
  let query = `SELECT * FROM users WHERE 1=1`;
  if (!includeDeleted) query += ` AND deleted_at IS NULL`;
  if (role)   query += ` AND role = '${role}'`;
  if (search) query += ` AND (name ILIKE '%${search}%' OR email ILIKE '%${search}%')`;
  query += ` ORDER BY ${sortBy} ${sortOrder}`;
  query += ` LIMIT ${pageSize} OFFSET ${(page - 1) * pageSize}`;

  return { users: [], total: 0 };  // stub
}

// ── Route handler — reads optional query params, calls findUsers ───────────
async function listUsersHandler(req: Request, res: Response): Promise<void> {
  const page     = req.query.page     ? parseInt(req.query.page as string, 10)     : undefined;
  const pageSize = req.query.pageSize ? parseInt(req.query.pageSize as string, 10) : undefined;
  const role     = req.query.role as FindUsersOptions["role"] | undefined;
  const search   = req.query.search as string | undefined;

  const result = await findUsers({ page, pageSize, role, search });
  res.json({ success: true, ...result });
}

// ── Email sending with optional CC, BCC, reply-to ─────────────────────────

interface SendEmailOptions {
  cc?: string[];
  bcc?: string[];
  replyTo?: string;
  priority?: "high" | "normal" | "low";
  trackOpens?: boolean;
}

async function sendEmail(
  to: string,                           // required
  subject: string,                      // required
  htmlBody: string,                     // required
  options: SendEmailOptions = {},       // optional with default empty object
): Promise<{ messageId: string }> {
  const {
    cc = [],
    bcc = [],
    replyTo,
    priority = "normal",
    trackOpens = false,
  } = options;

  console.log(`Sending to: ${to}, cc: ${cc.join(", ")}, priority: ${priority}`);
  return { messageId: `msg_${Date.now()}` };
}

// Callers only provide what they need:
await sendEmail("user@dev.io", "Welcome!", "<h1>Welcome!</h1>");
await sendEmail("user@dev.io", "Alert!", "<p>Your plan expires.</p>", { priority: "high" });
await sendEmail(
  "user@dev.io",
  "Invoice",
  "<p>See attached.</p>",
  { cc: ["billing@company.io"], bcc: ["archive@company.io"], trackOpens: true },
);

// ── JWT generation with optional expiry and claims ────────────────────────

interface JwtClaims {
  [key: string]: unknown;
}

function generateToken(
  userId: number,
  role: "admin" | "editor" | "viewer",
  expiresInSeconds: number = 3600,        // default 1 hour
  additionalClaims: JwtClaims = {},       // default empty
): string {
  const payload = {
    sub: String(userId),
    role,
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + expiresInSeconds,
    ...additionalClaims,
  };
  // In reality: return jwt.sign(payload, secret);
  return Buffer.from(JSON.stringify(payload)).toString("base64");
}

generateToken(1, "admin");                           // 1hr expiry, no extra claims
generateToken(1, "admin", 86400);                    // 24hr expiry
generateToken(1, "admin", 86400, { org: "acme" });   // 24hr, with org claim
```

---

## Common mistakes

### Mistake 1 — Optional `?` but not guarding inside the function

```ts
function sendNotification(userId: number, message: string, channel?: string): void {
  // ❌ WRONG — channel is string | undefined, can't call .toUpperCase() blindly:
  console.log(`[${channel.toUpperCase()}] User ${userId}: ${message}`);
  // Error: Object is possibly 'undefined'

  // ✅ RIGHT — guard with optional chaining:
  console.log(`[${channel?.toUpperCase() ?? "DEFAULT"}] User ${userId}: ${message}`);

  // ✅ ALSO RIGHT — provide a default at the top of the function body:
  const ch = channel ?? "email";
  console.log(`[${ch.toUpperCase()}] User ${userId}: ${message}`);
}
```

### Mistake 2 — Putting required parameters after optional ones

```ts
// ❌ Error — required 'name' after optional 'role':
function registerUser(email: string, role?: "admin" | "viewer", name: string): void {}
// Error: A required parameter cannot follow an optional parameter

// ✅ RIGHT — required params first, optional/default last:
function registerUser(email: string, name: string, role?: "admin" | "viewer"): void {}

// ✅ ALSO RIGHT — use an options object to group all optional params:
interface RegisterOptions { role?: "admin" | "viewer"; sendWelcome?: boolean; }
function registerUser(email: string, name: string, options?: RegisterOptions): void {}
```

### Mistake 3 — Using `||` for defaults instead of `??` — the falsy trap

```ts
function createSession(userId: number, ttlSeconds?: number): SessionData {
  // ❌ WRONG — || treats 0 as falsy, so ttlSeconds=0 gets replaced with 3600:
  const ttl = ttlSeconds || 3600;

  // If caller passes ttlSeconds=0 intending "no expiry" or "immediate expiry", it's ignored!

  // ✅ RIGHT — ?? only falls back on null/undefined, not 0 or "":
  const ttl = ttlSeconds ?? 3600;

  // ✅ EVEN BETTER — use a default parameter so you don't have to write the fallback:
  return { userId, expiresAt: Date.now() + ttl * 1000 };
}

// ── Same trap with boolean options:
function search(query: string, caseSensitive?: boolean): string[] {
  // ❌ WRONG — caseSensitive=false gets treated as falsy, defaults to true:
  const sensitive = caseSensitive || true;

  // ✅ RIGHT:
  const sensitive = caseSensitive ?? false;
  return [];
}
```

---

## Practice exercises

### Exercise 1 — easy

Write these functions with proper optional and default parameters:

1. `formatDate(date: Date, locale: string = "en-US", format?: "short" | "long" | "iso"): string`
   - If `format` is `"iso"`, return `date.toISOString()`
   - If `format` is `"long"`, return a verbose locale date string
   - If `format` is `"short"` or omitted, return a short locale date string

2. `buildUrl(base: string, path: string, queryParams?: Record<string, string>): string`
   - If `queryParams` is provided and non-empty, append `?key=value&key2=value2`
   - Otherwise return `base + path`

3. `sleep(ms: number = 1000): Promise<void>` — returns a Promise that resolves after `ms` milliseconds

For each, write 3 call examples: omitting the optional param, passing undefined, and passing an explicit value.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed HTTP client utility with optional/default options:

```ts
interface HttpRequestOptions {
  method?: "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
  headers?: Record<string, string>;
  body?: unknown;
  timeoutMs?: number;
  retries?: number;
  retryDelayMs?: number;
}

interface HttpResponse<T> {
  status: number;
  data: T;
  headers: Record<string, string>;
  durationMs: number;
}
```

Write a function `httpRequest<T>(url: string, options: HttpRequestOptions = {}): Promise<HttpResponse<T>>` that:
- Defaults: `method = "GET"`, `timeoutMs = 5000`, `retries = 0`, `retryDelayMs = 200`
- Simulates a request (no real fetch needed — just return a stub response after a `sleep`)
- Retries on failure up to `retries` times with `retryDelayMs` delay between attempts

Write convenience wrappers built on top:
- `get<T>(url: string, headers?: Record<string, string>): Promise<HttpResponse<T>>`
- `post<T>(url: string, body: unknown, headers?: Record<string, string>): Promise<HttpResponse<T>>`
- `del<T>(url: string, headers?: Record<string, string>): Promise<HttpResponse<T>>`

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed query builder for a REST API. The goal is to produce URL query strings from a typed options object.

Define these interfaces:
```ts
interface SortOptions {
  field: string;
  direction?: "asc" | "desc";   // default "asc"
}

interface FilterOptions {
  field: string;
  operator?: "eq" | "ne" | "gt" | "gte" | "lt" | "lte" | "like" | "in";  // default "eq"
  value: string | number | boolean | (string | number)[];
}

interface QueryBuilderOptions {
  page?: number;           // default 1
  pageSize?: number;       // default 20
  sort?: SortOptions[];
  filters?: FilterOptions[];
  fields?: string[];       // field selection — include only these fields
  search?: string;         // full-text search term
  includeTotal?: boolean;  // default true — include X-Total-Count
}
```

Write a `buildQueryString(options: QueryBuilderOptions = {}): string` function that:
- Applies all defaults
- Converts filters to format: `filter[field][operator]=value`
- Converts sort to format: `sort[0][field]=name&sort[0][direction]=asc`
- Converts field selection to: `fields=id,name,email`
- Appends `page`, `pageSize`, `search`, `includeTotal` when set
- Returns a properly encoded query string starting with `?` (or `""` if no params)

Write a `buildApiUrl(baseUrl: string, endpoint: string, options?: QueryBuilderOptions): string` that composes the full URL.

Write 5 varied call examples, each using different combinations of optional parameters, and `console.log` the resulting URLs.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Caller can omit? | Type inside fn | Explicit `undefined` triggers default? |
|--------|-----------------|----------------|---------------------------------------|
| `param: T` | ❌ Required | `T` | N/A |
| `param?: T` | ✅ | `T \| undefined` | No — stays `undefined` |
| `param: T = default` | ✅ | `T` | Yes — `undefined` → default |
| `param?: T = default` | ✅ | `T` | Yes (same as `= default`) |

| Rule | Notes |
|------|-------|
| Required params first | Optional/default params must come after required ones |
| `?` requires undefined guard | Always use `?.` or `?? fallback` before using an optional param |
| `??` not `\|\|` for defaults | `\|\|` is falsy — treats `0`, `""`, `false` as missing |
| Default inferred from value | TypeScript infers the type from the default expression |
| Passing `undefined` = omitting | For default params, `fn(undefined)` → default kicks in |
| Options object pattern | Group many optionals into one `options: Opts = {}` param |

## Connected topics

- **20 — Function types** — the foundation: typing parameters and return values.
- **22 — Rest parameters and spread** — `...args: T[]` — the third way to handle variable-arity parameters.
- **09 — Union types** — optional `?` makes the type `T | undefined` — a union.
- **11 — Literal types** — `"asc" | "desc"` as default parameter types.
- **16 — Nested object types** — the options-object pattern with deeply nested defaults.
