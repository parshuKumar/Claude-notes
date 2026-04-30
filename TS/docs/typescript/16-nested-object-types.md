# 16 — Nested Object Types

## What is this?

Real-world data is never flat. A user has an address. An address has a city. An order has a customer, which has a billing address, which has a country. A database row has metadata, which has an audit trail, which has a list of changes. TypeScript lets you type every level of nesting — either **inline** (write the shape directly where you use it) or **named** (give the nested shape its own `interface` or `type` alias and reference it by name).

This topic covers how to model deeply nested structures, when to keep types inline vs extract them, how to avoid deeply duplicated nesting, and patterns that keep complex shapes readable.

## Why does it matter?

Without typed nested objects:
- You reach into `req.body.user.address.city` and TypeScript has no idea if `address` is there, if `city` exists, or if it could be `null`
- You refactor one level of nesting and TypeScript can't tell you all the places the old shape was used
- You write functions that receive deeply nested objects and have no way to communicate their expected shape to callers

With typed nested objects:
- TypeScript validates every property access at every depth
- Renaming a nested field surfaces all usage sites
- Functions document exactly what shape they need, regardless of how deep it is

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no shape contract at any level of nesting
function formatAddress(user) {
  // Is user.address guaranteed? Is street a string? Is zip required?
  return `${user.address.street}, ${user.address.city}, ${user.address.zip}`;
  // Silently undefined or throws at runtime if any level is missing
}

// Caller has no idea what to pass:
formatAddress({ name: "Parsh" });           // crashes: Cannot read properties of undefined
formatAddress({ address: { city: "NY" } }); // crashes: street is undefined
```

```ts
// TypeScript — every level typed
interface Address {
  street: string;
  city: string;
  state: string;
  zip: string;
  country: string;
}

interface User {
  id: number;
  name: string;
  email: string;
  address: Address;         // named nested type
}

function formatAddress(user: User): string {
  // TypeScript knows exactly what exists at every level:
  return `${user.address.street}, ${user.address.city}, ${user.address.zip}`;
}

formatAddress({ name: "Parsh" } as any);           // ❌ caught at compile time
formatAddress({ address: { city: "NY" } } as any); // ❌ caught at compile time
```

---

## Syntax

```ts
// ── INLINE nested type ────────────────────────────────────────────────────
interface Order {
  id: number;
  customer: {             // inline — shape defined right here
    userId: number;
    name: string;
    email: string;
  };
  total: number;
}

// ── NAMED nested type ─────────────────────────────────────────────────────
interface OrderCustomer {  // extracted to its own named type
  userId: number;
  name: string;
  email: string;
}

interface Order {
  id: number;
  customer: OrderCustomer; // reference by name
  total: number;
}

// ── DEEP nesting — named types at every level ─────────────────────────────
interface Country {
  code: string;         // "US", "IN", "GB"
  name: string;
}

interface Address {
  street: string;
  city: string;
  state: string;
  zip: string;
  country: Country;     // nested inside Address
}

interface ContactInfo {
  email: string;
  phone?: string;
  address: Address;     // Address (which contains Country) nested inside ContactInfo
}

interface UserProfile {
  userId: number;
  displayName: string;
  contact: ContactInfo; // ContactInfo (containing Address containing Country) nested here
}

// Access:
const profile: UserProfile = { /* ... */ };
profile.contact.address.country.code;  // fully typed at every level
```

---

## How it works — depth by depth

### Inline types

Inline types write the object shape directly in the property definition:

```ts
interface ApiResponse {
  success: boolean;
  data: {                     // inline
    users: {                  // inline, nested one deeper
      id: number;
      name: string;
      role: string;
    }[];
    pagination: {             // inline sibling
      total: number;
      page: number;
      pageSize: number;
    };
  };
  meta: {                     // inline sibling
    requestId: string;
    duration: number;
  };
}
```

**When inline is fine:**
- The nested shape is used in **exactly one place** and is unlikely to be reused
- The shape is **small** (2–3 properties)
- It's a **one-off** response shape unlikely to be referenced independently

**When inline becomes a problem:**
- The nested shape is referenced in **multiple interfaces** → you'd have to duplicate it
- The nested shape is **deeply nested itself** (3+ levels of inline = unreadable)
- You want to **use the nested type as a standalone** function parameter type

### Named nested types — extracting for reuse

```ts
// ── PROBLEM: inline duplication ────────────────────────────────────────────
interface CreateUserBody {
  name: string;
  address: {
    street: string;
    city: string;
    zip: string;
  };
}

interface UpdateUserBody {
  name?: string;
  address?: {           // duplicated! Same shape repeated
    street: string;
    city: string;
    zip: string;
  };
}

// ── SOLUTION: extract and name ─────────────────────────────────────────────
interface Address {
  street: string;
  city: string;
  zip: string;
}

interface CreateUserBody {
  name: string;
  address: Address;     // reference — no duplication
}

interface UpdateUserBody {
  name?: string;
  address?: Address;    // same reference
}

// Now changing Address propagates to both automatically
```

### Optional nested objects

```ts
interface UserProfile {
  userId: number;
  bio?: string;            // optional scalar — string | undefined
  socialLinks?: {          // optional nested object — entire object may be absent
    twitter?: string;
    github?: string;
    linkedin?: string;
  };
}

// Accessing optional nested objects — you MUST guard:
function getGitHub(profile: UserProfile): string | null {
  // ❌ WRONG — socialLinks might be undefined:
  return profile.socialLinks.github ?? null;
  // Error: Object is possibly 'undefined'

  // ✅ RIGHT — optional chaining:
  return profile.socialLinks?.github ?? null;
}
```

### Arrays of nested objects

```ts
interface OrderItem {
  productId: number;
  productName: string;
  quantity: number;
  unitPrice: number;
}

interface Order {
  id: number;
  customerId: number;
  items: OrderItem[];       // array of a named nested type
  shippingAddress: Address;
  billingAddress: Address;
  status: "pending" | "processing" | "shipped" | "delivered" | "cancelled";
  createdAt: Date;
}

// TypeScript fully types array access:
const order: Order = { /* ... */ };
order.items[0].productName;    // string ✅
order.items[0].nonExistent;    // ❌ property doesn't exist on OrderItem
```

### Recursive types

```ts
// A category tree where each category can have subcategories:
interface Category {
  id: number;
  name: string;
  slug: string;
  parentId: number | null;
  children: Category[];    // recursive — Category contains Category[]
}

// A comment thread:
interface Comment {
  id: number;
  authorId: number;
  content: string;
  createdAt: Date;
  replies: Comment[];      // recursive — Comment contains Comment[]
}

// A menu tree:
interface MenuItem {
  label: string;
  href?: string;
  icon?: string;
  children?: MenuItem[];   // optional recursive — may or may not have sub-items
}
```

### Typing API response nesting — the real pattern

```ts
// Most REST APIs return nested structures like this:
// {
//   "success": true,
//   "data": { "user": { ... }, "permissions": [...] },
//   "meta": { "requestId": "...", "duration": 12 }
// }

interface ResponseMeta {
  requestId: string;
  duration: number;
  timestamp: Date;
}

interface ApiResponse<T> {
  success: boolean;
  data: T;
  meta: ResponseMeta;
  error?: {
    code: string;
    message: string;
    details?: Record<string, string>;
  };
}

interface UserWithPermissions {
  user: User;
  permissions: string[];
  sessionExpiry: Date;
}

// The response type for the /auth/login endpoint:
type LoginResponse = ApiResponse<UserWithPermissions>;

// TypeScript fully resolves the nesting:
// loginResponse.data.user.email         — string ✅
// loginResponse.data.permissions        — string[] ✅
// loginResponse.meta.requestId          — string ✅
// loginResponse.error?.details          — Record<string,string> | undefined ✅
```

---

## Example 1 — basic

```ts
// A typed product catalog with nested category, dimensions, and inventory

interface ProductDimensions {
  width: number;   // cm
  height: number;  // cm
  depth: number;   // cm
  weight: number;  // kg
}

interface ProductCategory {
  id: number;
  name: string;
  slug: string;
}

interface StockInfo {
  quantity: number;
  reserved: number;
  available: number;     // quantity - reserved
  warehouseLocation: string;
  restockDate: Date | null;
}

interface ProductImage {
  url: string;
  altText: string;
  isPrimary: boolean;
  width: number;
  height: number;
}

interface Product {
  id: number;
  sku: string;
  name: string;
  description: string;
  price: number;
  category: ProductCategory;
  dimensions: ProductDimensions;
  stock: StockInfo;
  images: ProductImage[];
  tags: string[];
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

function formatProductSummary(product: Product): string {
  const primaryImage = product.images.find(img => img.isPrimary);
  return [
    `[${product.sku}] ${product.name}`,
    `Category: ${product.category.name}`,
    `Price: $${product.price.toFixed(2)}`,
    `Stock: ${product.stock.available} available`,
    `Dimensions: ${product.dimensions.width}×${product.dimensions.height}×${product.dimensions.depth} cm`,
    `Image: ${primaryImage?.url ?? "none"}`,
  ].join("\n");
}
```

---

## Example 2 — real world backend use case

```ts
// Typed database model for an e-commerce order with full nesting

interface Address {
  fullName: string;
  line1: string;
  line2?: string;
  city: string;
  state: string;
  postalCode: string;
  country: string;
  phone?: string;
}

interface Money {
  amount: number;   // in cents — always integers to avoid float math
  currency: "USD" | "EUR" | "GBP" | "INR";
}

interface OrderLineItem {
  lineItemId: number;
  productId: number;
  productName: string;    // denormalised — snapshot at time of order
  productSku: string;
  quantity: number;
  unitPrice: Money;
  totalPrice: Money;
  discount: Money;
}

interface PaymentDetails {
  paymentId: string;
  provider: "stripe" | "paypal" | "razorpay";
  method: "card" | "upi" | "bank_transfer" | "wallet";
  status: "pending" | "authorized" | "captured" | "failed" | "refunded";
  capturedAt: Date | null;
  last4?: string;         // last 4 digits of card if card payment
  transactionRef?: string;
}

interface ShipmentInfo {
  shipmentId: string | null;
  carrier: string | null;
  trackingNumber: string | null;
  estimatedDelivery: Date | null;
  shippedAt: Date | null;
  deliveredAt: Date | null;
}

interface OrderAuditEntry {
  timestamp: Date;
  actorId: number | null;   // null = system
  action: string;
  previousStatus: string;
  newStatus: string;
  note?: string;
}

interface Order {
  readonly id: number;
  readonly orderNumber: string;   // human-readable: "ORD-2026-00423"
  customerId: number;

  items: OrderLineItem[];

  subtotal: Money;
  shippingCost: Money;
  taxAmount: Money;
  discountTotal: Money;
  grandTotal: Money;

  shippingAddress: Address;
  billingAddress: Address;

  payment: PaymentDetails;
  shipment: ShipmentInfo;

  status: "pending" | "confirmed" | "processing" | "shipped" | "delivered" | "cancelled" | "refunded";

  auditLog: OrderAuditEntry[];   // array of nested objects

  notes?: string;
  readonly createdAt: Date;
  updatedAt: Date;
}

// Functions that work on parts of the nested structure:
function calculateOrderRefund(order: Order): Money {
  // Only needs the Money parts — TypeScript ensures we don't reach for wrong fields
  return {
    amount: order.grandTotal.amount - order.discountTotal.amount,
    currency: order.grandTotal.currency,
  };
}

function appendAuditEntry(
  order: Order,
  actorId: number | null,
  action: string,
  newStatus: Order["status"],  // pulls the union type directly from Order — stays in sync
): void {
  const entry: OrderAuditEntry = {
    timestamp: new Date(),
    actorId,
    action,
    previousStatus: order.status,
    newStatus,
  };
  order.auditLog.push(entry);
  order.status = newStatus;
  order.updatedAt = new Date();
}

// Note: Order["status"] is an indexed access type — pulls the type of the 'status'
// property directly from the Order interface. If you add a new status to Order,
// appendAuditEntry automatically accepts it without any change needed here.
```

---

## Common mistakes

### Mistake 1 — Deeply nested inline types that should be extracted

```ts
// ❌ BAD — 4 levels of inline nesting, unreadable and unreusable:
interface UserRecord {
  id: number;
  profile: {
    name: string;
    contact: {
      email: string;
      address: {
        street: string;
        city: string;
        location: {
          lat: number;
          lng: number;
        };
      };
    };
  };
}

// ✅ GOOD — each level extracted and named:
interface GeoLocation {
  lat: number;
  lng: number;
}

interface PostalAddress {
  street: string;
  city: string;
  location: GeoLocation;
}

interface ContactInfo {
  email: string;
  address: PostalAddress;
}

interface UserProfileData {
  name: string;
  contact: ContactInfo;
}

interface UserRecord {
  id: number;
  profile: UserProfileData;
}
// Now each part is independently usable and testable
```

### Mistake 2 — Forgetting to guard optional nested objects before deep access

```ts
interface ServerConfig {
  host: string;
  tls?: {
    certPath: string;
    keyPath: string;
    caPath?: string;
  };
}

function getTlsCert(config: ServerConfig): string {
  // ❌ WRONG — tls might be undefined:
  return config.tls.certPath;
  // Error: Object is possibly 'undefined'

  // ✅ RIGHT — guard with optional chaining:
  return config.tls?.certPath ?? "";

  // ✅ ALSO RIGHT — explicit guard:
  if (!config.tls) throw new Error("TLS not configured");
  return config.tls.certPath;  // after the guard, TypeScript knows it's defined
}
```

### Mistake 3 — Duplicating nested object shapes instead of naming and reusing them

```ts
// ❌ BAD — same shape defined three times:
interface CreateOrderBody {
  customerId: number;
  shippingAddress: {
    street: string;
    city: string;
    postalCode: string;
    country: string;
  };
}

interface UpdateOrderBody {
  shippingAddress?: {
    street: string;    // duplicated
    city: string;
    postalCode: string;
    country: string;
  };
}

interface OrderRecord {
  id: number;
  shippingAddress: {   // duplicated again
    street: string;
    city: string;
    postalCode: string;
    country: string;
  };
}

// ✅ GOOD — one definition, referenced everywhere:
interface ShippingAddress {
  street: string;
  city: string;
  postalCode: string;
  country: string;
}

interface CreateOrderBody {
  customerId: number;
  shippingAddress: ShippingAddress;
}

interface UpdateOrderBody {
  shippingAddress?: ShippingAddress;
}

interface OrderRecord {
  id: number;
  shippingAddress: ShippingAddress;
}
// Change ShippingAddress once → all three types update automatically
```

---

## Practice exercises

### Exercise 1 — easy

Define fully typed interfaces for a blog post system with the following nested structure:

- `Author` — `id`, `name`, `avatarUrl` (optional)
- `Tag` — `id`, `name`, `slug`
- `PostStats` — `views`, `likes`, `comments`, `shares`
- `Post` — `id`, `title`, `slug`, `content`, `author` (use `Author`), `tags` (array of `Tag`), `stats` (use `PostStats`), `publishedAt` (`Date | null`), `createdAt`, `updatedAt`

Write a function `getPostSummary(post: Post): string` that returns: `"[slug] title by authorName — views views"`.

Write a function `isPublished(post: Post): boolean`.

```ts
// Write your code here
```

### Exercise 2 — medium

Model a typed e-commerce shopping cart with three levels of nesting:

```
CartItem:
  - productId: number
  - name: string (snapshot)
  - quantity: number
  - unitPrice: { amount: number; currency: "USD" | "EUR" }
  - discount?: { type: "percent" | "fixed"; value: number }

Cart:
  - cartId: string
  - userId: number
  - items: CartItem[]
  - appliedCoupon?: { code: string; discountPercent: number; expiresAt: Date }
  - createdAt: Date
  - updatedAt: Date
```

Write these functions (all fully typed):
1. `addItem(cart: Cart, item: CartItem): Cart` — returns a new cart (don't mutate)
2. `removeItem(cart: Cart, productId: number): Cart` — returns a new cart
3. `calculateSubtotal(cart: Cart): number` — sum of `unitPrice.amount * quantity` for all items
4. `applyCoupon(cart: Cart, coupon: Cart["appliedCoupon"]): Cart` — returns new cart with coupon applied

Note: `Cart["appliedCoupon"]` uses indexed access type to pull the coupon type from `Cart` directly.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a fully typed configuration system for a multi-service backend application:

```
DatabaseConfig:
  - host, port, name, user
  - password (string)
  - pool: { min: number; max: number; idleTimeout: number }
  - ssl?: { certPath: string; keyPath: string; rejectUnauthorized: boolean }

RedisConfig:
  - host, port
  - password?: string
  - db: number (0-15)
  - tls?: { enabled: boolean; certPath?: string }
  - retryStrategy?: { maxRetries: number; retryDelay: number }

LoggingConfig:
  - level: "debug" | "info" | "warn" | "error"
  - format: "json" | "pretty"
  - outputs: Array<{ type: "console" | "file" | "http"; destination?: string; minLevel?: "debug" | "info" | "warn" | "error" }>
  - redactFields: string[]

ServiceConfig:
  - name: string
  - version: string
  - port: number
  - environment: "development" | "staging" | "production"
  - database: DatabaseConfig
  - redis: RedisConfig
  - logging: LoggingConfig
  - features: Record<string, boolean>   // feature flags
  - cors?: { origins: string[]; methods: string[]; allowCredentials: boolean }
```

Write these functions:
1. `validateConfig(config: ServiceConfig): string[]` — returns array of validation error messages (empty array = valid). Check: port between 1–65535, database pool min < max, redis db between 0–15, at least one log output defined.
2. `maskSensitiveFields(config: ServiceConfig): ServiceConfig` — returns a new config with `database.password` replaced by `"***"` and `redis.password` (if present) replaced by `"***"`. Must not mutate the original.
3. `isProductionReady(config: ServiceConfig): boolean` — returns true only if: environment is "production", ssl is configured in database, tls is enabled in redis, log level is "info" or "warn" or "error" (not "debug").

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Pattern | Syntax |
|---------|--------|
| Inline nested shape | `property: { key: type; }` |
| Named nested type | Define separately, reference by name |
| Optional nested object | `property?: NestedType` — guard with `?.` before access |
| Array of nested objects | `property: NestedType[]` |
| Recursive type | `children: Category[]` inside `Category` |
| Indexed access type | `Order["status"]` — pull a nested property's type |
| Optional chaining | `user?.address?.city` — safe access through optional nesting |
| Nullish coalescing | `user?.address?.city ?? "Unknown"` — fallback for undefined |

| Rule | Notes |
|------|-------|
| Inline when | Single-use, 2–3 properties, unlikely to be standalone |
| Extract when | Reused in 2+ places, deeply nested itself, needs standalone use |
| Always guard optional nested | `?.` or explicit `if` before accessing nested optional object |
| Never duplicate nested shapes | Extract, name, and reference — change once, updates everywhere |
| Indexed access | `T["key"]` pulls the exact type, stays in sync with T |

## Connected topics

- **14 — Interfaces** — all the building blocks used throughout this topic.
- **09 — Union types** — used inside nested shapes for status fields, currency enums, etc.
- **32 — Utility types** — `Partial<T>`, `Pick<T, K>`, `Omit<T, K>` work on nested interfaces.
- **40 — Mapped types** — transform the entire shape of nested types programmatically.
- **43 — Indexed access types** — `Order["status"]`, `User["address"]["city"]` — pulling types from nested structures.
- **22 — Optional chaining in types** — how TypeScript narrows types after `?.` access.
