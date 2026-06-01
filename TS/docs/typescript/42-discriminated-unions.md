# 42 — Discriminated Unions

## What is this?

A **discriminated union** (also called a *tagged union* or *algebraic data type*) is a union of object types where every member has a **common literal property** — called the **discriminant**, **tag**, or **kind** field — that uniquely identifies which member it is.

```ts
type Shape =
  | { kind: "circle";    radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle";  base: number;  height: number };
```

The `kind` property is the discriminant. Every member has it. Each has a different literal value. When you check `shape.kind === "circle"`, TypeScript narrows `shape` to exactly `{ kind: "circle"; radius: number }` — no other members possible.

## Why does it matter?

Backend systems are full of "things that can be one of several states, each with different data":

- An HTTP response is either success (with `data`) or error (with `error` and `statusCode`)
- A job is `pending` (just queued), `running` (with `startedAt`), `succeeded` (with `result`), or `failed` (with `error` and `retryCount`)
- A payment is `initiated`, `processing`, `settled`, or `refunded` — each state has different fields
- A notification can be an email (with `subject`, `body`), SMS (with `message`), or push (with `title`, `body`, `deviceToken`)

Without discriminated unions, you represent all these with one big type where most fields are optional, and you never know which ones are present. With discriminated unions, each state has exactly the fields it needs — nothing optional, nothing missing.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — one big object, everything optional, constant defensive checks:
function processJobResult(job) {
  if (job.status === "succeeded") {
    console.log(job.result);      // might be undefined — status might lie
    job.result.data.forEach(...)  // potential TypeError at runtime
  }
  if (job.status === "failed") {
    console.log(job.error);       // might be undefined
    if (job.retryCount > 3) { }   // retryCount might not exist
  }
  // No guarantee any of these fields exist
}
```

```ts
// TypeScript discriminated union — exactly the right fields per state:
type Job =
  | { status: "pending";   queuedAt:  Date }
  | { status: "running";   startedAt: Date; workerId: string }
  | { status: "succeeded"; startedAt: Date; completedAt: Date; result: JobResult }
  | { status: "failed";    startedAt: Date; failedAt: Date;    error: string; retryCount: number };

function processJobResult(job: Job): void {
  switch (job.status) {
    case "succeeded":
      // job: { status: "succeeded"; startedAt: Date; completedAt: Date; result: JobResult }
      job.result.data.forEach(/* ... */); // ✅ result is guaranteed present
      break;
    case "failed":
      // job: { status: "failed"; ... error: string; retryCount: number }
      if (job.retryCount > 3) { /* abandon */ } // ✅ retryCount guaranteed present
      break;
  }
}
```

---

## Syntax

```ts
// ── Define the union — each member has a literal discriminant field ─────────
type Result<T> =
  | { ok: true;  data: T }
  | { ok: false; error: string; code: number };

// ── Narrowing via if/else ──────────────────────────────────────────────────
function handle<T>(result: Result<T>): T {
  if (result.ok) {
    return result.data;          // result: { ok: true; data: T }
  }
  throw new Error(result.error); // result: { ok: false; error: string; code: number }
}

// ── Narrowing via switch ───────────────────────────────────────────────────
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" };

function reduce(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT": return state + action.amount; // action.amount available
    case "DECREMENT": return state - action.amount; // action.amount available
    case "RESET":     return 0;                     // no action.amount — it doesn't exist
    default: {
      const _: never = action; // exhaustiveness check
      return state;
    }
  }
}
```

---

## How it works — concept by concept

### Concept 1 — What makes a good discriminant

The discriminant must be a **literal type** (string, number, boolean literal) shared across all union members under the same property name:

```ts
// ✅ Good discriminants — string literals:
type Event =
  | { type: "click";   x: number; y: number }
  | { type: "keydown"; key: string }
  | { type: "scroll";  delta: number };

// ✅ Good discriminant — boolean literal:
type Result<T> =
  | { ok: true;  value: T }
  | { ok: false; error: Error };

// ✅ Good discriminant — numeric literal:
type HttpCategory =
  | { category: 2; data: unknown }
  | { category: 4; clientError: string }
  | { category: 5; serverError: string };

// ❌ Bad discriminant — string (not literal):
type Bad =
  | { type: string; a: number }  // type: string — not a literal, too wide to discriminate
  | { type: string; b: number };
```

### Concept 2 — Exhaustiveness checking with `never`

After narrowing every member of a union, the remaining type is `never`. Assign it to a `never`-typed variable to enforce that all cases are handled — adding a new union member without handling it becomes a **compile error**:

```ts
type PaymentStatus =
  | "initiated"
  | "processing"
  | "settled"
  | "refunded"
  | "failed";

function getStatusLabel(status: PaymentStatus): string {
  switch (status) {
    case "initiated":  return "Payment initiated";
    case "processing": return "Processing payment";
    case "settled":    return "Payment settled";
    case "refunded":   return "Refunded";
    case "failed":     return "Payment failed";
    default: {
      // If you add "disputed" to PaymentStatus and forget a case, this line is a compile error:
      const _exhaustive: never = status;
      throw new Error(`Unhandled payment status: ${_exhaustive}`);
    }
  }
}

// ✅ Helper to make the pattern reusable:
function assertNever(value: never, message?: string): never {
  throw new Error(message ?? `Unexpected value: ${JSON.stringify(value)}`);
}

function getStatusLabel(status: PaymentStatus): string {
  switch (status) {
    case "initiated":  return "Payment initiated";
    case "processing": return "Processing payment";
    case "settled":    return "Payment settled";
    case "refunded":   return "Refunded";
    case "failed":     return "Payment failed";
    default:           return assertNever(status);  // ← compile error if a case is missing
  }
}
```

### Concept 3 — Union members can share some fields but differ on others

```ts
// All members share requestId and timestamp — they differ on the rest:
type WebhookEvent =
  | { type: "user.created";  requestId: string; timestamp: Date; userId: number; email: string }
  | { type: "user.deleted";  requestId: string; timestamp: Date; userId: number; deletedBy: number }
  | { type: "order.placed";  requestId: string; timestamp: Date; orderId: number; totalCents: number }
  | { type: "payment.failed"; requestId: string; timestamp: Date; paymentId: string; reason: string };

function logEvent(event: WebhookEvent): void {
  // Shared fields — available without narrowing:
  console.log(`[${event.timestamp.toISOString()}] ${event.type} (req: ${event.requestId})`);

  // Type-specific fields — only after narrowing:
  if (event.type === "user.created") {
    console.log(`  → New user ${event.userId}: ${event.email}`);  // ✅ email available
  }

  if (event.type === "payment.failed") {
    console.log(`  → Payment ${event.paymentId} failed: ${event.reason}`);  // ✅ paymentId available
  }
}
```

### Concept 4 — Discriminated unions with object member types

The union members can be full interface types — the discriminant just needs to be a literal:

```ts
interface PendingOrder {
  status:    "pending";
  orderId:   number;
  userId:    number;
  items:     OrderItem[];
  createdAt: Date;
}

interface ConfirmedOrder {
  status:       "confirmed";
  orderId:      number;
  userId:       number;
  items:        OrderItem[];
  createdAt:    Date;
  confirmedAt:  Date;
  paymentId:    string;
}

interface ShippedOrder {
  status:          "shipped";
  orderId:         number;
  userId:          number;
  items:           OrderItem[];
  createdAt:       Date;
  confirmedAt:     Date;
  shippedAt:       Date;
  trackingNumber:  string;
  carrier:         string;
}

interface CancelledOrder {
  status:       "cancelled";
  orderId:      number;
  userId:       number;
  items:        OrderItem[];
  createdAt:    Date;
  cancelledAt:  Date;
  reason:       string;
  refundId:     string | null;
}

type Order = PendingOrder | ConfirmedOrder | ShippedOrder | CancelledOrder;

// Narrowing through the discriminant:
function getOrderTimeline(order: Order): string[] {
  const lines: string[] = [`Created: ${order.createdAt.toISOString()}`];

  if (order.status === "confirmed" || order.status === "shipped") {
    // order: ConfirmedOrder | ShippedOrder — both have confirmedAt
    lines.push(`Confirmed: ${order.confirmedAt.toISOString()}`);
  }

  if (order.status === "shipped") {
    // order: ShippedOrder
    lines.push(`Shipped: ${order.shippedAt.toISOString()} via ${order.carrier}`);
    lines.push(`Tracking: ${order.trackingNumber}`);
  }

  if (order.status === "cancelled") {
    // order: CancelledOrder
    lines.push(`Cancelled: ${order.cancelledAt.toISOString()} — ${order.reason}`);
    if (order.refundId) lines.push(`Refund: ${order.refundId}`);
  }

  return lines;
}
```

### Concept 5 — Result/Either pattern

A very common discriminated union in backend code is a `Result<T, E>` type — success or failure without throwing:

```ts
type Result<T, E = Error> =
  | { ok: true;  value: T }
  | { ok: false; error: E };

// Helper constructors:
const ok   = <T>(value: T): Result<T, never>  => ({ ok: true,  value });
const fail = <E>(error: E): Result<never, E>  => ({ ok: false, error });

// Usage — no try/catch, fully typed:
async function findUser(userId: number): Promise<Result<User, "not_found" | "db_error">> {
  try {
    const user = await db.findById(userId);
    if (!user) return fail("not_found" as const);
    return ok(user);
  } catch {
    return fail("db_error" as const);
  }
}

// Caller handles both cases explicitly:
const result = await findUser(userId);
if (result.ok) {
  result.value.email; // ✅ User available
} else {
  switch (result.error) {
    case "not_found": res.status(404).json({ message: "User not found" }); break;
    case "db_error":  res.status(503).json({ message: "Database unavailable" }); break;
    default: assertNever(result.error);
  }
}
```

### Concept 6 — Narrowing multiple union members at once

You can narrow to a subset of the union — TypeScript intersects the narrowed members:

```ts
type JobState =
  | { status: "queued";    queuedAt:    Date }
  | { status: "running";   startedAt:   Date; workerId: string }
  | { status: "succeeded"; completedAt: Date; result: unknown }
  | { status: "failed";    failedAt:    Date; error: string; retries: number }
  | { status: "cancelled"; cancelledAt: Date };

type ActiveJob = Extract<JobState, { status: "running" | "succeeded" | "failed" }>;
// = { status: "running"; ... } | { status: "succeeded"; ... } | { status: "failed"; ... }

function isTerminalState(job: JobState): job is Extract<JobState, { status: "succeeded" | "failed" | "cancelled" }> {
  return job.status === "succeeded" || job.status === "failed" || job.status === "cancelled";
}

function getCompletionDate(job: JobState): Date | null {
  if (job.status === "succeeded") return job.completedAt;
  if (job.status === "failed")    return job.failedAt;
  if (job.status === "cancelled") return job.cancelledAt;
  return null;
}
```

---

## Example 1 — basic

```ts
// HTTP response wrapper — the most common discriminated union pattern in APIs

type ApiResponse<T> =
  | { ok: true;  data: T;      statusCode: 200 | 201 | 204 }
  | { ok: false; error: string; statusCode: 400 | 401 | 403 | 404 | 409 | 422 | 500; code: string };

// Helper constructors:
function success<T>(data: T, statusCode: 200 | 201 | 204 = 200): ApiResponse<T> {
  return { ok: true, data, statusCode };
}

function failure(error: string, statusCode: 400 | 401 | 403 | 404 | 409 | 422 | 500, code: string): ApiResponse<never> {
  return { ok: false, error, statusCode, code };
}

// Service functions return ApiResponse — no throwing:
async function getUserById(userId: number): Promise<ApiResponse<User>> {
  if (userId <= 0)  return failure("Invalid userId", 400, "INVALID_INPUT");
  const user = await db.findUserById(userId);
  if (!user)        return failure("User not found", 404, "NOT_FOUND");
  return success(user);
}

async function createUser(dto: CreateUserDto): Promise<ApiResponse<User>> {
  const existing = await db.findUserByEmail(dto.email);
  if (existing)    return failure("Email already registered", 409, "CONFLICT");
  const user = await db.createUser(dto);
  return success(user, 201);
}

// Route handler — send the response from the ApiResponse:
async function handleGetUser(req: Request, res: Response): Promise<void> {
  const userId = parseInt(req.params.id, 10);
  const result = await getUserById(userId);

  if (result.ok) {
    // result: { ok: true; data: User; statusCode: 200 | 201 | 204 }
    res.status(result.statusCode).json({ data: result.data });
  } else {
    // result: { ok: false; error: string; statusCode: ...; code: string }
    res.status(result.statusCode).json({ error: result.error, code: result.code });
  }
}

// Generic response sender:
function sendResponse<T>(res: Response, result: ApiResponse<T>): void {
  if (result.ok) {
    res.status(result.statusCode).json({ ok: true, data: result.data });
  } else {
    res.status(result.statusCode).json({ ok: false, error: result.error, code: result.code });
  }
}
```

---

## Example 2 — real world backend use case

```ts
// Order state machine — discriminated union as a finite state machine with typed transitions

// ── State types ──────────────────────────────────────────────────────────────

interface DraftOrder {
  readonly status:    "draft";
  readonly orderId:   string;
  readonly userId:    number;
  readonly items:     readonly OrderItem[];
  readonly createdAt: Date;
}

interface PendingPaymentOrder {
  readonly status:      "pending_payment";
  readonly orderId:     string;
  readonly userId:      number;
  readonly items:       readonly OrderItem[];
  readonly createdAt:   Date;
  readonly submittedAt: Date;
  readonly totalCents:  number;
}

interface ProcessingOrder {
  readonly status:       "processing";
  readonly orderId:      string;
  readonly userId:       number;
  readonly items:        readonly OrderItem[];
  readonly createdAt:    Date;
  readonly submittedAt:  Date;
  readonly totalCents:   number;
  readonly paymentId:    string;
  readonly paidAt:       Date;
}

interface FulfilledOrder {
  readonly status:         "fulfilled";
  readonly orderId:        string;
  readonly userId:         number;
  readonly items:          readonly OrderItem[];
  readonly createdAt:      Date;
  readonly submittedAt:    Date;
  readonly totalCents:     number;
  readonly paymentId:      string;
  readonly paidAt:         Date;
  readonly fulfilledAt:    Date;
  readonly trackingNumber: string;
  readonly carrier:        string;
}

interface RefundedOrder {
  readonly status:      "refunded";
  readonly orderId:     string;
  readonly userId:      number;
  readonly items:       readonly OrderItem[];
  readonly createdAt:   Date;
  readonly submittedAt: Date;
  readonly totalCents:  number;
  readonly paymentId:   string;
  readonly paidAt:      Date;
  readonly refundId:    string;
  readonly refundedAt:  Date;
  readonly reason:      string;
}

type Order =
  | DraftOrder
  | PendingPaymentOrder
  | ProcessingOrder
  | FulfilledOrder
  | RefundedOrder;

// ── Transition functions — each accepts the REQUIRED prior state only ────────

// TypeScript enforces you can only submit a draft order (not a fulfilled one, etc.)
function submitOrder(order: DraftOrder, totalCents: number): PendingPaymentOrder {
  return {
    ...order,
    status:      "pending_payment",
    submittedAt: new Date(),
    totalCents,
  };
}

function confirmPayment(order: PendingPaymentOrder, paymentId: string): ProcessingOrder {
  return {
    ...order,
    status:    "processing",
    paymentId,
    paidAt:    new Date(),
  };
}

function fulfillOrder(order: ProcessingOrder, trackingNumber: string, carrier: string): FulfilledOrder {
  return {
    ...order,
    status:         "fulfilled",
    fulfilledAt:    new Date(),
    trackingNumber,
    carrier,
  };
}

function refundOrder(order: ProcessingOrder | FulfilledOrder, refundId: string, reason: string): RefundedOrder {
  return {
    status:     "refunded",
    orderId:    order.orderId,
    userId:     order.userId,
    items:      order.items,
    createdAt:  order.createdAt,
    submittedAt: order.submittedAt,
    totalCents: order.totalCents,
    paymentId:  order.paymentId,
    paidAt:     order.paidAt,
    refundId,
    refundedAt: new Date(),
    reason,
  };
}

// ── Query functions — work on the full union ─────────────────────────────────

function getOrderSummary(order: Order): string {
  switch (order.status) {
    case "draft":
      return `Order ${order.orderId}: Draft (${order.items.length} items)`;

    case "pending_payment":
      return `Order ${order.orderId}: Awaiting payment ($${(order.totalCents / 100).toFixed(2)})`;

    case "processing":
      return `Order ${order.orderId}: Processing (Payment: ${order.paymentId})`;

    case "fulfilled":
      return `Order ${order.orderId}: Fulfilled — ${order.carrier} tracking: ${order.trackingNumber}`;

    case "refunded":
      return `Order ${order.orderId}: Refunded — ${order.reason} (Refund: ${order.refundId})`;

    default:
      return assertNever(order);  // exhaustiveness check
  }
}

function canBeRefunded(order: Order): order is ProcessingOrder | FulfilledOrder {
  return order.status === "processing" || order.status === "fulfilled";
}

function hasBeenPaid(order: Order): order is ProcessingOrder | FulfilledOrder | RefundedOrder {
  return order.status === "processing" || order.status === "fulfilled" || order.status === "refunded";
}

// ── State machine orchestrator ────────────────────────────────────────────────

class OrderStateMachine {
  constructor(
    private readonly orderRepo:    OrderRepository,
    private readonly paymentSvc:   PaymentService,
    private readonly fulfillSvc:   FulfillmentService,
    private readonly logger:       Logger,
  ) {}

  async processPayment(orderId: string, paymentToken: string): Promise<Order> {
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw ApiError.notFound("Order");

    if (order.status !== "pending_payment") {
      throw ApiError.conflict(`Cannot process payment for order in status: ${order.status}`);
    }
    // order: PendingPaymentOrder — TypeScript narrows after the check above

    const paymentId = await this.paymentSvc.charge(paymentToken, order.totalCents);
    const processing = confirmPayment(order, paymentId);
    await this.orderRepo.save(processing);
    this.logger.info("Order payment confirmed", { orderId, paymentId });
    return processing;
  }

  async requestRefund(orderId: string, reason: string): Promise<Order> {
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw ApiError.notFound("Order");

    if (!canBeRefunded(order)) {
      throw ApiError.conflict(`Cannot refund order in status: ${order.status}`);
    }
    // order: ProcessingOrder | FulfilledOrder — TypeScript narrowed by the type guard

    const refundId = await this.paymentSvc.refund(order.paymentId, order.totalCents);
    const refunded = refundOrder(order, refundId, reason);
    await this.orderRepo.save(refunded);
    this.logger.info("Order refunded", { orderId, refundId, reason });
    return refunded;
  }
}

// Helper used above:
function assertNever(value: never): never {
  throw new Error(`Unhandled discriminated union member: ${JSON.stringify(value)}`);
}
```

---

## Common mistakes

### Mistake 1 — Using a wide type (not a literal) as the discriminant

```ts
// ❌ 'type' is string — not a literal — TypeScript cannot narrow from it:
type Event =
  | { type: string; userId: number }   // too wide
  | { type: string; orderId: number };

function handle(event: Event): void {
  if (event.type === "user.created") {
    event.userId; // ❌ Error: userId does not exist on type Event — not narrowed
  }
}

// ✅ Use string literal types:
type Event =
  | { type: "user.created";  userId:  number }
  | { type: "order.created"; orderId: number };

function handle(event: Event): void {
  if (event.type === "user.created") {
    event.userId; // ✅ narrowed to { type: "user.created"; userId: number }
  }
}
```

### Mistake 2 — Forgetting the exhaustiveness check when adding new union members

```ts
type Status = "active" | "inactive" | "suspended";

function handleStatus(status: Status): string {
  if (status === "active")   return "User is active";
  if (status === "inactive") return "User is inactive";
  return "unknown"; // ← silently wrong when "suspended" is added — no compiler warning
}

// ✅ Add the exhaustiveness check:
function handleStatus(status: Status): string {
  if (status === "active")    return "User is active";
  if (status === "inactive")  return "User is inactive";
  if (status === "suspended") return "User is suspended";

  // If "banned" is added to Status later, this line becomes a compile error:
  const _: never = status;
  throw new Error(`Unhandled status: ${_}`);
}
```

### Mistake 3 — Accessing a member-specific field without narrowing first

```ts
type Notification =
  | { channel: "email"; to: string; subject: string; body: string }
  | { channel: "sms";   to: string; message: string }
  | { channel: "push";  deviceToken: string; title: string; body: string };

function sendNotification(n: Notification): void {
  // ❌ Cannot access channel-specific fields without narrowing:
  console.log(n.subject);       // Error: Property 'subject' does not exist on type 'Notification'
  console.log(n.deviceToken);   // Error: same

  // ✅ Narrow first:
  if (n.channel === "email") {
    console.log(n.subject, n.body);          // ✅
  } else if (n.channel === "sms") {
    console.log(n.message);                  // ✅
  } else {
    console.log(n.deviceToken, n.title);     // ✅ n: push notification
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

Model a customer support ticket system using a discriminated union. A ticket can be:

- `open` — `{ ticketId, userId, subject, body, createdAt }`
- `assigned` — all of open + `{ assigneeId, assignedAt }`
- `in_progress` — all of assigned + `{ startedAt, notes: string[] }`
- `resolved` — all of in_progress + `{ resolvedAt, resolution: string }`
- `closed` — all of resolved + `{ closedAt, satisfactionScore: 1 | 2 | 3 | 4 | 5 | null }`

Then write:

1. `getTicketAge(ticket: Ticket): number` — ms since creation
2. `getAssignee(ticket: Ticket): number | null` — returns assigneeId if assigned, null if open
3. `isResolved(ticket: Ticket): ticket is ResolvedTicket | ClosedTicket` — type guard
4. `getStatusLabel(ticket: Ticket): string` — exhaustive switch returning a human label
   — adding a new status without handling it must be a compile error

```ts
// Write your code here
```

### Exercise 2 — medium

Build a complete `Result<T, E>` type and helper library:

```ts
// The union:
type Result<T, E = Error> =
  | { ok: true;  value: T }
  | { ok: false; error: E };

// Implement these functions:
// ok<T>(value: T): Result<T, never>
// fail<E>(error: E): Result<never, E>
// map<T, U, E>(result: Result<T, E>, fn: (value: T) => U): Result<U, E>
// flatMap<T, U, E>(result: Result<T, E>, fn: (value: T) => Result<U, E>): Result<U, E>
// mapError<T, E, F>(result: Result<T, E>, fn: (error: E) => F): Result<T, F>
// getOrThrow<T>(result: Result<T, unknown>): T   — throws if ok is false
// getOrElse<T>(result: Result<T, unknown>, fallback: T): T
// isOk<T, E>(result: Result<T, E>): result is { ok: true; value: T }
// isFail<T, E>(result: Result<T, E>): result is { ok: false; error: E }
// collect<T, E>(results: Result<T, E>[]): Result<T[], E>
//   — if all ok: returns ok(values[]); if any fail: returns fail(first error)
```

Then model a database query pipeline using these helpers — no try/catch anywhere in the pipeline itself:

```ts
async function getUserWithOrders(userId: number): Promise<Result<{ user: User; orders: Order[] }, string>> {
  // Use map, flatMap, collect to chain the async operations
  // Return ok({ user, orders }) or fail("not_found") | fail("db_error") etc.
}
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **finite state machine** framework using discriminated unions where the valid transitions between states are enforced by the type system:

```ts
// A generic FSM where:
// - States is a discriminated union (the tag field is "status")
// - Transitions map "from status" → "to status" and carry transition-specific data
// - You cannot call an invalid transition at compile time

// Model a subscription lifecycle:
type SubscriptionState =
  | { status: "trial";     userId: number; startedAt: Date; trialEndsAt: Date }
  | { status: "active";    userId: number; startedAt: Date; plan: "basic" | "pro" | "enterprise"; renewsAt: Date; paymentMethodId: string }
  | { status: "past_due";  userId: number; startedAt: Date; plan: "basic" | "pro" | "enterprise"; dueAt: Date; paymentMethodId: string }
  | { status: "cancelled"; userId: number; startedAt: Date; cancelledAt: Date; reason: string; endsAt: Date }
  | { status: "expired";   userId: number; startedAt: Date; expiredAt: Date };

// Transition functions — TypeScript must enforce the valid prior state:
// activate(sub: TrialState, plan, paymentMethodId): ActiveState
// failPayment(sub: ActiveState): PastDueState
// retryPayment(sub: PastDueState, newPaymentMethodId): ActiveState
// cancel(sub: TrialState | ActiveState | PastDueState, reason): CancelledState
// expire(sub: TrialState | PastDueState | CancelledState): ExpiredState

// Then build a SubscriptionService class that:
// - Uses the state machine transitions internally
// - Exposes methods: startTrial, activate, cancelSubscription, processPaymentFailed, processPaymentRetried
// - Has getSubscriptionSummary(sub: SubscriptionState): string — exhaustive switch
// - Has a static assertNever helper for exhaustiveness
// - Persists state transitions by saving the new state via subscriptionRepo

// Show that calling activate() on a CancelledState is a compile error.
// Show that the exhaustive switch catches missing cases.
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Define a discriminated union:
type MyUnion =
  | { kind: "a"; dataA: string }
  | { kind: "b"; dataB: number }
  | { kind: "c"; dataC: boolean };

// Narrow with if:
if (x.kind === "a") { x.dataA; } // x: { kind: "a"; dataA: string }

// Narrow with switch + exhaustiveness:
switch (x.kind) {
  case "a": x.dataA; break;
  case "b": x.dataB; break;
  case "c": x.dataC; break;
  default: assertNever(x); // compile error if a case is missing
}

// assertNever helper:
function assertNever(value: never): never {
  throw new Error(`Unhandled union member: ${JSON.stringify(value)}`);
}

// Result/Either pattern:
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };
const ok   = <T>(value: T): Result<T, never>  => ({ ok: true,  value });
const fail = <E>(error: E): Result<never, E>  => ({ ok: false, error });

// Utility types for discriminated unions:
type ExtractKind<T, K> = Extract<T, { kind: K }>;  // get one member
// Extract<MyUnion, { kind: "a" }> → { kind: "a"; dataA: string }
```

| Rule | Detail |
|---|---|
| Discriminant must be a **literal** | `"active"`, `true`, `200` — not `string`, `boolean`, `number` |
| All union members need the same discriminant key | `status`, `kind`, `type`, `ok`, `tag` — pick one |
| After switch default → `never` | Use `assertNever(x)` to enforce exhaustiveness |
| Shared fields accessible without narrowing | Only discriminant-unique fields need narrowing |
| Transition functions take the specific prior state | Not the full union — prevents invalid transitions |

---

## Connected topics

- **09 — Union types** — discriminated unions are a pattern built on top of regular union types.
- **40 — Type narrowing** — the `===` equality narrowing mechanism that powers discriminant checks.
- **41 — Type guards** — type predicates like `isTerminalState(order): order is X | Y` for subset narrowing.
- **32 — Utility types** — `Extract<Union, { kind: "a" }>` and `Exclude` for slicing union members.
- **44 — Conditional types** — `T extends { kind: "a" } ? ... : ...` — the type-level counterpart to runtime narrowing.
- **43 — Mapped types** — building `Record<EventKind, Handler>` from a discriminated union's tag values.
