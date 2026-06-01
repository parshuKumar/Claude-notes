# 41 — Type Guards

## What is this?

A **type guard** is a function that performs a runtime check and tells TypeScript's type system: "if this check passes, the value is of type `T`." The return type uses the special syntax `value is Type` — called a **type predicate**.

```ts
function isString(value: unknown): value is string {
  return typeof value === "string";
}

const input: unknown = getRequestParam();

if (isString(input)) {
  // input: string — TypeScript narrows here because isString returned true
  input.toUpperCase(); // ✅
}
```

Topic 40 covered the *built-in* narrowing mechanisms (typeof, instanceof, in, etc.). Type guards extend that power to **arbitrary runtime logic** — any check you can write in code, you can wrap in a type guard and have TypeScript trust it.

## Why does it matter?

Built-in narrowing only covers simple, single-expression checks. The moment your validation logic spans multiple lines — checking nested properties, verifying array contents, validating formats — you need to extract it into a function. Without type predicates, that function returns `boolean` and TypeScript forgets the narrowing. With `value is Type`, the narrowing flows through the function call.

Common backend use cases:
- Validating `unknown` request bodies into typed DTOs
- Narrowing `unknown` errors in catch blocks
- Checking if a database row has the required fields
- Discriminating union members by checking multiple properties at once

## The JavaScript way vs the TypeScript way

```js
// JavaScript — the function exists, but callers get no type narrowing:
function isUser(value) {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof value.id    === "number" &&
    typeof value.email === "string"
  );
}

const data = JSON.parse(body);
if (isUser(data)) {
  data.email.toLowerCase(); // no autocomplete, no type safety — data is still `any`
}
```

```ts
// TypeScript — the same function with a type predicate:
interface User { id: number; email: string; role: string; }

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id"    in value && typeof (value as User).id    === "number" &&
    "email" in value && typeof (value as User).email === "string" &&
    "role"  in value && typeof (value as User).role  === "string"
  );
}

const data: unknown = JSON.parse(body);
if (isUser(data)) {
  // data: User — full autocomplete, all User methods and properties available
  data.email.toLowerCase();  // ✅
  data.role.toUpperCase();   // ✅
}
```

---

## Syntax

```ts
// Basic type predicate function:
function isFoo(value: unknown): value is Foo {
  // return a boolean — true means "value is Foo"
  return /* runtime check */;
}

// Arrow function form:
const isBar = (value: unknown): value is Bar => /* check */;

// Method form inside a class:
class Validator {
  isEmail(value: unknown): value is string {
    return typeof value === "string" && value.includes("@");
  }
}

// Asserts form (assertion function — throws instead of returning false):
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError(`Expected string, got ${typeof value}`);
  }
}
// After assertIsString(x), TypeScript knows x is string for the rest of the scope.

// Narrowing for class hierarchies:
function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError;
}
```

---

## How it works — concept by concept

### Concept 1 — The `value is Type` return type

The return type `value is Type` is a **type predicate**. It tells TypeScript: "this function returns `true` if and only if `value` is of type `Type`." When TypeScript sees `if (isUser(data))`, it narrows `data` to `User` in the true branch.

```ts
// Without type predicate — narrowing is lost:
function checkIsUser(value: unknown): boolean {
  return typeof value === "object" && value !== null && "id" in value;
}

const data: unknown = {};
if (checkIsUser(data)) {
  data.id; // ❌ Error: data is still `unknown` — TypeScript didn't narrow
}

// With type predicate — narrowing works:
function isUser(value: unknown): value is { id: number } {
  return typeof value === "object" && value !== null && "id" in value;
}

const data: unknown = {};
if (isUser(data)) {
  data.id; // ✅ data: { id: number }
}
```

### Concept 2 — Type guards for union discrimination

When you have a union type and want to narrow in a function, type guards are the tool:

```ts
type SuccessResponse<T> = { ok: true;  data: T;      status: number; };
type ErrorResponse      = { ok: false; error: string; status: number; };
type ApiResponse<T>     = SuccessResponse<T> | ErrorResponse;

function isSuccess<T>(response: ApiResponse<T>): response is SuccessResponse<T> {
  return response.ok === true;
}

function isError<T>(response: ApiResponse<T>): response is ErrorResponse {
  return response.ok === false;
}

async function fetchUser(userId: number): Promise<void> {
  const response = await apiClient.get<User>(`/users/${userId}`);

  if (isSuccess(response)) {
    // response: SuccessResponse<User>
    console.log(response.data.email);   // ✅
  } else {
    // response: ErrorResponse
    console.error(response.error);     // ✅
    // response.data would be a type error — not on ErrorResponse
  }
}
```

### Concept 3 — `asserts value is Type` — assertion functions

An **assertion function** throws if the check fails, and narrows for all code *after* the call (not just inside an `if`):

```ts
function assertIsString(value: unknown, label = "value"): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError(`Expected ${label} to be a string, got: ${typeof value}`);
  }
}

function assertNonNull<T>(value: T | null | undefined, label = "value"): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error(`${label} must not be null or undefined`);
  }
}

function processRequest(req: unknown): void {
  assertIsString((req as any)?.method, "req.method");
  // From here: TypeScript narrowed (req as any).method to string — but the real
  // power is when asserting on a local variable:

  let userId: number | null = getUserIdFromToken(req);
  assertNonNull(userId, "userId");
  // userId: number — null ruled out for the rest of this function
  userId.toFixed(); // ✅
}
```

### Concept 4 — Combining multiple type guards for deep validation

Type guards compose: call one inside another to build nested validators:

```ts
interface Address {
  street: string;
  city:   string;
  zip:    string;
}

interface CreateOrderDto {
  userId:        number;
  productIds:    number[];
  shippingAddress: Address;
  paymentToken:  string;
}

function isAddress(value: unknown): value is Address {
  return (
    typeof value === "object" &&
    value !== null &&
    "street" in value && typeof (value as Address).street === "string" &&
    "city"   in value && typeof (value as Address).city   === "string" &&
    "zip"    in value && typeof (value as Address).zip    === "string"
  );
}

function isNumberArray(value: unknown): value is number[] {
  return Array.isArray(value) && value.every(item => typeof item === "number");
}

function isCreateOrderDto(value: unknown): value is CreateOrderDto {
  if (typeof value !== "object" || value === null) return false;

  const v = value as Record<string, unknown>;

  return (
    typeof v.userId       === "number"  &&
    isNumberArray(v.productIds)         &&  // reuse the array guard
    isAddress(v.shippingAddress)        &&  // reuse the address guard
    typeof v.paymentToken === "string"
  );
}

// Usage in a route handler:
async function createOrder(req: Request, res: Response): Promise<void> {
  if (!isCreateOrderDto(req.body)) {
    throw ApiError.badRequest("Invalid order payload");
  }
  // req.body: CreateOrderDto — fully typed, all nested shapes available
  const order = await orderService.create(req.body);
  res.status(201).json(order);
}
```

### Concept 5 — Generic type guards

Type guards can be generic — parameterized by the type they check for:

```ts
// Generic non-null guard:
function isDefined<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}

const userIds: (number | null)[] = [1, null, 3, null, 5];
const validIds: number[] = userIds.filter(isDefined);  // ✅ type: number[] — nulls filtered out

// Generic array filter:
function filterMap<T, U>(
  array: T[],
  guard: (item: T) => item is U,
): U[] {
  return array.filter(guard) as U[];
}

const mixed: (string | number)[] = ["hello", 1, "world", 2, 3];
const strings: string[] = filterMap(mixed, (x): x is string => typeof x === "string");
// strings: ["hello", "world"]
```

### Concept 6 — Type guards vs `as` casts — when to use which

```ts
const data: unknown = JSON.parse(req.body);

// ❌ as cast — no runtime check, silently wrong if structure mismatches:
const user = data as User;
user.email.toLowerCase(); // might be undefined at runtime — no safety

// ✅ Type guard — runtime check, TypeScript-narrowed, safe:
if (isUser(data)) {
  data.email.toLowerCase(); // safe — isUser verified the shape
}

// ✅ Assertion function — throws on bad data, narrows permanently:
assertIsUser(data);
data.email.toLowerCase(); // safe after the assertion
```

Use `as` only when:
1. You are certain the structure is correct (e.g. data from a fully trusted, typed API)
2. The type is too complex to guard at runtime (e.g. branded types — covered in later topics)

In all other cases, prefer type guards or assertion functions.

### Concept 7 — Type guards in `Array.filter` and `Array.find`

TypeScript struggles to narrow the return type of `array.filter()` without a type guard because the callback returns `boolean`, not a predicate:

```ts
const items: (User | null)[] = [user1, null, user2, null];

// Without type guard — result is still (User | null)[]:
const users1 = items.filter(item => item !== null);  // type: (User | null)[] — not narrowed!

// With type guard — result is User[]:
const users2 = items.filter((item): item is User => item !== null);  // type: User[] ✅

// Or use isDefined:
function isDefined<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}
const users3 = items.filter(isDefined);  // type: User[] ✅
```

---

## Example 1 — basic

```ts
// A reusable guard library for common backend validation needs

// ── Primitive guards ───────────────────────────────────────────────────────

function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isNumber(value: unknown): value is number {
  return typeof value === "number" && !Number.isNaN(value);
}

function isInteger(value: unknown): value is number {
  return Number.isInteger(value);
}

function isBoolean(value: unknown): value is boolean {
  return typeof value === "boolean";
}

function isDefined<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}

function isNonEmptyString(value: unknown): value is string {
  return typeof value === "string" && value.trim().length > 0;
}

// ── Object / array guards ──────────────────────────────────────────────────

function isPlainObject(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

function isArrayOf<T>(value: unknown, guard: (item: unknown) => item is T): value is T[] {
  return Array.isArray(value) && value.every(guard);
}

// ── Domain guards ──────────────────────────────────────────────────────────

interface UserId    { readonly _brand: "UserId";    readonly value: number; }
interface ProductId { readonly _brand: "ProductId"; readonly value: number; }

function isPositiveInt(value: unknown): value is number {
  return typeof value === "number" && Number.isInteger(value) && value > 0;
}

function hasField<K extends string>(
  obj: unknown,
  key: K,
): obj is Record<K, unknown> {
  return isPlainObject(obj) && key in obj;
}

// ── Assertion functions ────────────────────────────────────────────────────

function assertDefined<T>(
  value: T | null | undefined,
  message = "Value must not be null or undefined",
): asserts value is T {
  if (value === null || value === undefined) throw new Error(message);
}

function assertString(value: unknown, label = "value"): asserts value is string {
  if (typeof value !== "string") throw new TypeError(`${label} must be a string`);
}

function assertPositiveInt(value: unknown, label = "value"): asserts value is number {
  if (!isPositiveInt(value)) throw new TypeError(`${label} must be a positive integer`);
}

// ── Usage ──────────────────────────────────────────────────────────────────

const rawIds: unknown[] = [1, 2, null, "3", 4, undefined, 5];

// Filter down to valid positive integers:
const validIds: number[] = rawIds.filter(isPositiveInt);
// [1, 2, 4, 5] — "3" (string), null, undefined filtered out

// Array of strings:
const rawTags: unknown[] = ["backend", "typescript", 42, null, "node"];
const tags: string[] = rawTags.filter(isNonEmptyString);
// ["backend", "typescript", "node"]

// Assertion in a route handler:
function handleDeleteUser(req: Request): void {
  assertPositiveInt(req.params.id, "userId param");
  // After assertion: req.params.id is number — safe to use
  // (in practice you'd parse the string first, but shows the pattern)
}
```

---

## Example 2 — real world backend use case

```ts
// Webhook payload validator — incoming payloads are unknown, must be narrowed before processing

import { randomUUID } from "crypto";

// ── Event types ─────────────────────────────────────────────────────────────

interface BaseWebhookEvent {
  eventId:   string;
  timestamp: string;
  version:   "1.0";
}

interface UserRegisteredEvent extends BaseWebhookEvent {
  type:      "user.registered";
  userId:    number;
  email:     string;
  source:    "web" | "mobile" | "api";
}

interface OrderCreatedEvent extends BaseWebhookEvent {
  type:        "order.created";
  orderId:     number;
  userId:      number;
  totalCents:  number;
  lineItems:   Array<{ productId: number; quantity: number; unitPriceCents: number }>;
}

interface PaymentProcessedEvent extends BaseWebhookEvent {
  type:        "payment.processed";
  paymentId:   string;
  orderId:     number;
  amountCents: number;
  provider:    "stripe" | "paypal";
  status:      "succeeded" | "failed";
}

type KnownWebhookEvent = UserRegisteredEvent | OrderCreatedEvent | PaymentProcessedEvent;

// ── Type guards ──────────────────────────────────────────────────────────────

function isBaseWebhookEvent(value: unknown): value is BaseWebhookEvent {
  if (typeof value !== "object" || value === null) return false;
  const v = value as Record<string, unknown>;
  return (
    typeof v.eventId   === "string" &&
    typeof v.timestamp === "string" &&
    v.version          === "1.0"
  );
}

function isLineItem(value: unknown): value is { productId: number; quantity: number; unitPriceCents: number } {
  if (typeof value !== "object" || value === null) return false;
  const v = value as Record<string, unknown>;
  return (
    Number.isInteger(v.productId)      && (v.productId as number) > 0 &&
    Number.isInteger(v.quantity)       && (v.quantity as number) > 0 &&
    Number.isInteger(v.unitPriceCents) && (v.unitPriceCents as number) >= 0
  );
}

function isUserRegisteredEvent(value: unknown): value is UserRegisteredEvent {
  if (!isBaseWebhookEvent(value)) return false;
  if ((value as any).type !== "user.registered") return false;
  const v = value as Record<string, unknown>;
  return (
    Number.isInteger(v.userId)   && (v.userId as number) > 0 &&
    typeof v.email  === "string" && (v.email as string).includes("@") &&
    (v.source === "web" || v.source === "mobile" || v.source === "api")
  );
}

function isOrderCreatedEvent(value: unknown): value is OrderCreatedEvent {
  if (!isBaseWebhookEvent(value)) return false;
  if ((value as any).type !== "order.created") return false;
  const v = value as Record<string, unknown>;
  return (
    Number.isInteger(v.orderId)    && (v.orderId as number) > 0 &&
    Number.isInteger(v.userId)     && (v.userId as number) > 0 &&
    Number.isInteger(v.totalCents) && (v.totalCents as number) >= 0 &&
    Array.isArray(v.lineItems)     && (v.lineItems as unknown[]).every(isLineItem)
  );
}

function isPaymentProcessedEvent(value: unknown): value is PaymentProcessedEvent {
  if (!isBaseWebhookEvent(value)) return false;
  if ((value as any).type !== "payment.processed") return false;
  const v = value as Record<string, unknown>;
  return (
    typeof v.paymentId   === "string" &&
    Number.isInteger(v.orderId)       && (v.orderId as number) > 0 &&
    Number.isInteger(v.amountCents)   && (v.amountCents as number) >= 0 &&
    (v.provider === "stripe" || v.provider === "paypal") &&
    (v.status   === "succeeded" || v.status === "failed")
  );
}

function isKnownWebhookEvent(value: unknown): value is KnownWebhookEvent {
  return (
    isUserRegisteredEvent(value)   ||
    isOrderCreatedEvent(value)     ||
    isPaymentProcessedEvent(value)
  );
}

// ── Webhook handler ──────────────────────────────────────────────────────────

interface WebhookHandlerResult {
  eventId:   string;
  processed: boolean;
  message:   string;
}

async function handleWebhook(rawBody: unknown): Promise<WebhookHandlerResult> {
  if (!isKnownWebhookEvent(rawBody)) {
    throw ApiError.badRequest("Unrecognised or malformed webhook payload");
  }

  // rawBody: KnownWebhookEvent — base fields available
  const { eventId, timestamp } = rawBody;

  // Discriminate by type:
  switch (rawBody.type) {
    case "user.registered": {
      // rawBody: UserRegisteredEvent
      await userService.onRegistered(rawBody.userId, rawBody.email, rawBody.source);
      return { eventId, processed: true, message: `User ${rawBody.userId} registered via ${rawBody.source}` };
    }

    case "order.created": {
      // rawBody: OrderCreatedEvent
      await orderService.onCreated(rawBody.orderId, rawBody.userId, rawBody.lineItems);
      return { eventId, processed: true, message: `Order ${rawBody.orderId} created (${rawBody.lineItems.length} items)` };
    }

    case "payment.processed": {
      // rawBody: PaymentProcessedEvent
      await paymentService.onProcessed(rawBody.paymentId, rawBody.orderId, rawBody.status);
      return { eventId, processed: rawBody.status === "succeeded", message: `Payment ${rawBody.paymentId}: ${rawBody.status}` };
    }

    default: {
      const _exhaustive: never = rawBody;
      throw new Error(`Unhandled event type: ${(_exhaustive as BaseWebhookEvent).type}`);
    }
  }
}
```

---

## Common mistakes

### Mistake 1 — Returning `boolean` instead of the type predicate

```ts
// ❌ Returns boolean — TypeScript cannot use this for narrowing:
function isActiveUser(user: User | null): boolean {
  return user !== null && user.status === "active";
}

const user: User | null = getUser();
if (isActiveUser(user)) {
  user.email; // ❌ Error: user is still User | null — not narrowed
}

// ✅ Type predicate — narrowing works:
function isActiveUser(user: User | null): user is User {
  return user !== null && user.status === "active";
}

if (isActiveUser(user)) {
  user.email; // ✅ user: User
}
```

### Mistake 2 — Type predicates lie — TypeScript trusts you completely

TypeScript cannot verify that your type guard is correct — it trusts the return value. A wrong guard silently corrupts the type:

```ts
// ❌ This guard lies — it says value is User even when it isn't:
function isUser(value: unknown): value is User {
  return typeof value === "object" && value !== null; // NOT sufficient — any object passes
}

const data: unknown = { foo: "bar" }; // not a User
if (isUser(data)) {
  // TypeScript thinks data is User — but it's actually { foo: "bar" }
  data.email.toLowerCase(); // Runtime TypeError — email is undefined
}

// ✅ Check every required field and its type:
function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) return false;
  const v = value as Record<string, unknown>;
  return (
    typeof v.id    === "number" &&
    typeof v.email === "string" &&
    typeof v.role  === "string"
  );
}
```

### Mistake 3 — Using `asserts` when you need the narrowing inside an `if`

```ts
// asserts narrows AFTER the call — for the rest of the function scope:
function assertIsString(v: unknown): asserts v is string {
  if (typeof v !== "string") throw new TypeError("Expected string");
}

// Fine for unconditional narrowing:
assertIsString(rawValue);
rawValue.toUpperCase(); // ✅ narrowed for rest of function

// ❌ But if you want narrowing ONLY inside an if-branch, use a regular type guard:
// assertIsString inside an if makes no sense — it throws, not returns false
if (assertIsString(rawValue)) { // ❌ Nonsensical — asserts returns void, not boolean
  // ...
}

// ✅ Use a type predicate function for conditional narrowing:
function isString(v: unknown): v is string {
  return typeof v === "string";
}
if (isString(rawValue)) {
  rawValue.toUpperCase(); // ✅ narrowed only inside this branch
}
```

---

## Practice exercises

### Exercise 1 — easy

Build a reusable type guard library for backend request validation. Implement these functions with proper type predicates:

```ts
// 1. isEmail(value: unknown): value is string
//    — must be a string, contain exactly one @, have at least one char before and after @

// 2. isPositiveInteger(value: unknown): value is number
//    — must be an integer > 0

// 3. isISODateString(value: unknown): value is string
//    — must be a string that parses to a valid Date (use Date.parse)

// 4. isOneOf<T extends string>(value: unknown, options: readonly T[]): value is T
//    — generic guard: returns true if value is one of the given literal options
//    — usage: isOneOf(req.query.sort, ["asc", "desc"] as const)

// 5. isArrayOf<T>(value: unknown, guard: (item: unknown) => item is T): value is T[]
//    — true if value is an array AND every element passes the guard

// 6. isDefined<T>(value: T | null | undefined): value is T
//    — standard non-null guard (use in Array.filter)
```

Then demonstrate each one:
```ts
isEmail("user@api.dev.io");      // true
isEmail("notanemail");           // false
isPositiveInteger(5);            // true
isPositiveInteger(0);            // false
isPositiveInteger(3.14);         // false
isISODateString("2026-05-28");   // true
isISODateString("not-a-date");   // false
isOneOf("asc", ["asc", "desc"] as const); // true
isOneOf("random", ["asc", "desc"] as const); // false
const nums = [1, null, 2, undefined, 3];
nums.filter(isDefined); // [1, 2, 3]
```

```ts
// Write your code here
```

### Exercise 2 — medium

Build a fully guarded route body parser for these three DTOs:

```ts
interface CreateUserDto {
  email:    string;  // valid email
  password: string;  // min 8 chars
  role:     "admin" | "user" | "moderator";
  age?:     number;  // optional, positive integer 13–120
}

interface UpdateUserDto {
  email?:   string;
  role?:    "admin" | "user" | "moderator";
  age?:     number;
}

interface CreateProductDto {
  name:        string;    // 1–200 chars
  priceCents:  number;    // positive integer
  stock:       number;    // non-negative integer
  categoryId:  number;    // positive integer
  tags?:       string[];  // optional, each tag max 50 chars
}
```

Requirements:
- `isCreateUserDto(value: unknown): value is CreateUserDto`
- `isUpdateUserDto(value: unknown): value is UpdateUserDto` — all fields optional, but if present, must be the right shape
- `isCreateProductDto(value: unknown): value is CreateProductDto`
- Assertion variants: `assertCreateUserDto`, `assertUpdateUserDto`, `assertCreateProductDto` — throw `ApiError.badRequest(message)` with a descriptive message on failure
- Use your guard library from Exercise 1 where applicable

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **typed event bus** that uses type guards to ensure handlers only receive the correct event payload:

```ts
// Define a catalogue of all events the system can emit:
interface EventCatalogue {
  "user.created":       { userId: number; email: string; role: string; timestamp: Date };
  "user.deleted":       { userId: number; deletedBy: number; timestamp: Date };
  "order.placed":       { orderId: number; userId: number; totalCents: number; itemCount: number };
  "order.shipped":      { orderId: number; trackingNumber: string; carrier: string };
  "payment.succeeded":  { paymentId: string; orderId: number; amountCents: number };
  "payment.failed":     { paymentId: string; orderId: number; reason: string; retryable: boolean };
}

type EventType = keyof EventCatalogue;
type EventPayload<T extends EventType> = EventCatalogue[T];
```

Build `TypedEventBus` class with:
- `on<T extends EventType>(type: T, handler: (payload: EventPayload<T>) => void | Promise<void>): () => void`
  — returns an unsubscribe function
- `emit<T extends EventType>(type: T, payload: EventPayload<T>): Promise<void>`
  — calls all registered handlers for that type; handlers run concurrently; errors are caught and logged
- `once<T extends EventType>(type: T, handler: (payload: EventPayload<T>) => void | Promise<void>): () => void`
  — fires once then automatically unsubscribes
- `off<T extends EventType>(type: T, handler: (payload: EventPayload<T>) => void | Promise<void>): void`
- `listenerCount(type: EventType): number`

**Type guard requirement**: Add a `createPayloadGuard<T extends EventType>(type: T): (value: unknown) => value is EventPayload<T>` factory that generates a runtime-checking type guard for any event type. It must check all fields of the payload exist and have the right primitive types. This guard should be used inside `emit` to validate the payload before passing it to handlers.

Demonstrate:
```ts
const bus = new TypedEventBus();

const unsub = bus.on("user.created", (payload) => {
  // payload: { userId: number; email: string; role: string; timestamp: Date }
  console.log(`User created: ${payload.email}`);
});

await bus.emit("user.created", {
  userId:    1,
  email:     "dev@api.dev.io",
  role:      "user",
  timestamp: new Date(),
});

// TypeScript prevents wrong payload type:
// bus.emit("user.created", { orderId: 1 }); // ❌ compile error

unsub(); // unsubscribe
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Regular type guard — use in if conditions:
function isFoo(value: unknown): value is Foo {
  return /* runtime check */;
}
if (isFoo(x)) { /* x: Foo */ }

// Assertion function — throws on failure, narrows permanently:
function assertIsFoo(value: unknown): asserts value is Foo {
  if (!/* check */) throw new Error("Not a Foo");
}
assertIsFoo(x);
x; // x: Foo for the rest of the scope

// Generic type guard:
function isDefined<T>(v: T | null | undefined): v is T { return v != null; }
array.filter(isDefined); // removes null/undefined, narrows element type

// Array type guard:
function isArrayOf<T>(v: unknown, g: (x: unknown) => x is T): v is T[] {
  return Array.isArray(v) && v.every(g);
}

// In Array.filter — return type predicate in callback:
const strings: string[] = mixed.filter((x): x is string => typeof x === "string");

// Important rules:
// 1. Predicate parameter name must match the function parameter name
// 2. TypeScript TRUSTS the predicate — if you lie, runtime crashes are possible
// 3. Use `as` only inside the guard function, never at the call site
// 4. assertX throws → narrows after the call; isX returns bool → narrows in if-branch
```

| | Type predicate (`is`) | Assertion function (`asserts`) |
|---|---|---|
| Return type | `boolean` | `void` |
| Narrowing scope | Inside the `if` branch | Rest of the current scope |
| On failure | Returns `false` | Throws |
| Use in `if` condition | ✅ | ❌ (returns void) |
| Use for mandatory requirements | ❌ | ✅ |

---

## Connected topics

- **40 — Type narrowing in depth** — the built-in mechanisms (`typeof`, `instanceof`, `in`) that type guards extend.
- **42 — Discriminated unions** — type guards are the perfect tool for discriminating union members at runtime.
- **09 — Union types** — what you're narrowing from; union types are the most common input to type guards.
- **07 — Special types** — `unknown` is the safe "untrusted input" type; type guards are the correct way to work with it.
- **32 — Utility types** — `NonNullable<T>` at the type level vs `isDefined` at runtime — two sides of the same coin.
- **44 — Conditional types** — type-level counterpart: `T extends Foo ? A : B` mirrors what `isFoo(value)` does at runtime.
