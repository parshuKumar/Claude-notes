# 126 — Design a Food Delivery App (Zomato/Swiggy)
## Category: LLD Case Study

---

## What is this?

A **food delivery app** connects three parties: a hungry **customer**, a **restaurant** with a menu, and a **delivery agent** on a scooter. The app lets you browse restaurants, place an order, gets that order cooked and picked up, assigns the nearest free rider, and shows you a status bar creeping from "Order placed" to "Delivered."

Think of it as an **air-traffic controller for lunch**. Planes (orders) request to take off (get placed), a tower (the service) assigns each one a runway and a pilot (a delivery agent), and every plane moves through a strict sequence of states — taxiing, taking off, cruising, landing — that it is not allowed to skip.

This is a classic broad **LLD (low-level design)** interview problem: lots of nouns, a real state machine, and two spots where a swappable policy (a Strategy) fits perfectly.

---

## Why does it matter?

**Interview angle:** "Design Swiggy/Zomato/DoorDash" is a favourite because it is *deliberately open-ended*. There is no single right answer, so the interviewer watches **how you decompose**: Do you find the right objects? Do you notice the order lifecycle is a state machine? Do you spot that "pick a delivery agent" is the same swappable-policy problem as surge pricing? A candidate who blurts out one giant `Order` class with a 200-line `process()` method fails. A candidate who names State and Strategy and explains *why* passes.

**Real-work angle:** Every marketplace app you will ever build — ride-hailing, grocery, courier, freelancer matching — is this same skeleton: a request object with a strict lifecycle, a matching engine that assigns a provider, and a pricing engine. Get this design vocabulary right once and you reuse it for a decade. The two hardest real bugs in these systems are **illegal state transitions** (an order marked DELIVERED that was never PICKED_UP) and **bad assignment** (a rider 8 km away gets the order while a free one sits next door). Both are prevented by the design below.

---

## The core idea — explained simply

### The Restaurant Kitchen Pass Analogy

Walk into a busy restaurant kitchen. There's a metal rail above the pass called the **ticket rail**. Every order is a paper **ticket** clipped onto it, and each ticket physically moves left to right through fixed stations: it's *placed* at the till, *confirmed* by the head chef, *being prepared* at the stove, *ready* under the heat lamp, *picked up* by a runner, and finally *delivered* to the table. A ticket **cannot** teleport from the till straight to the table — it must pass every station in order. That physical constraint is a **state machine**, and it is the heart of an `Order`.

Now, who carries the ready plate out? The floor manager doesn't grab a random waiter. She uses a **rule**: "give it to the closest free waiter." Tomorrow she might switch the rule to "give it to the highest-rated waiter" or "give it to whoever's carrying the fewest plates." The *rule is swappable* without changing anything about the tickets or the kitchen. That swappable rule is a **Strategy**.

And the bill? Item prices, plus a delivery charge that depends on distance, times a surge multiplier when it's raining. That calculation is *also* a swappable rule — another **Strategy**.

| Analogy | Technical concept |
|---------|------------------|
| The paper ticket | An `Order` object |
| Ticket stations (till → table) | `OrderStatus` states |
| Ticket can't skip a station | Valid-transition rules in `updateStatus()` |
| The ticket rail (all live tickets) | The service's order list |
| Floor manager's "closest free waiter" rule | `AssignmentStrategy` (Strategy pattern) |
| Switching to "highest-rated waiter" | Swapping the strategy at runtime |
| Working out the bill | `PricingStrategy` (Strategy pattern) |
| The floor manager herself | `FoodDeliveryService` orchestrator |

Hold this picture. Every class below is one object from this kitchen.

---

## Key concepts inside this topic

We follow the **7-step LLD method**: requirements → nouns → behaviours → class diagram → code → patterns → extensions.

### 1. Requirements & clarifying questions

Before writing a single class, pin down scope. In an interview you say these out loud.

**Functional requirements (what it must do):**
- Browse **restaurants** and their **menus** (only *open* restaurants, only *available* items).
- **Place an order** of one or more menu items from a single restaurant.
- **Assign a delivery agent** to a placed order — pick a good one automatically.
- **Track the order** as it moves through its lifecycle: `PLACED → CONFIRMED → PREPARING → READY → PICKED_UP → DELIVERED`, or `CANCELLED`.
- **Price** an order: item total + distance-based delivery fee + optional surge multiplier.
- Support **ratings** for restaurants and agents (drives future assignment/ranking).

**Non-functional (qualities):** low-latency assignment, correctness of the status lifecycle (never skip a state), and an assignment policy we can change without a rewrite.

**Clarifying questions to ask the interviewer** (asking these earns points):
- *Scheduled orders?* — "Deliver at 8 PM" vs. "now." We'll design so it's an easy add (Extension 5), but build for now-orders.
- *Multiple restaurants in one order?* — Real apps mostly say no (one order = one restaurant). We assume **single restaurant per order** to keep the model clean.
- *Payment handling?* — Out of scope for the core model; we assume price is computed and payment is a separate service.
- *Real-time GPS tracking?* — We model discrete **status** changes now; live lat/lng streaming is Extension 3.
- *Cancellation window?* — Cancel allowed until `PREPARING`. After that, food is cooking.

Locking scope like this is *the* skill the interviewer is testing.

### 2. Identify the core objects (nouns)

Underline the nouns in the requirements — each becomes a class:

- **`Customer`** — who orders. Has an id, name, and an `Address`.
- **`Restaurant`** — has a menu (`MenuItem`s), a `Location`, and an `isOpen` flag.
- **`MenuItem`** — name, price, `available` flag.
- **`Order`** — the star. Owns its list of `OrderItem`s, the customer, the restaurant, the assigned agent, its amounts, and its **status** (the lifecycle).
- **`OrderItem`** — a line: which `MenuItem` and what quantity. (We snapshot the price so a later menu change doesn't rewrite history.)
- **`DeliveryAgent`** — a rider: id, name, `Location`, `status` (AVAILABLE/BUSY), rating.
- **`Location`** (and `Address`) — lat/lng, with a **distance helper**.
- **`AssignmentStrategy`** — the swappable rule for choosing an agent.
- **`PricingStrategy`** — the swappable rule for computing the bill.
- **`FoodDeliveryService`** — the orchestrator that ties it all together.

**Enums** (fixed sets, frozen so nobody mutates them):
- **`OrderStatus`**: `PLACED, CONFIRMED, PREPARING, READY, PICKED_UP, DELIVERED, CANCELLED`.
- **`AgentStatus`**: `AVAILABLE, BUSY`.

### 3. Behaviours (verbs) + who owns them

The single most important LLD skill: put each behaviour on the object that **owns the data it touches**. Don't dump everything in one god-class.

| Behaviour | Lives on | Why (owns the data) |
|-----------|----------|---------------------|
| `browseRestaurants()` | `FoodDeliveryService` | It holds the catalogue of restaurants |
| `updateStatus(newStatus)` | **`Order`** | The order owns its own lifecycle — only it can validate a transition |
| `calculateAmount()` | `Order` (line totals) | It owns its items |
| `assign(order, agents)` | **`AssignmentStrategy`** | Assignment *policy* is a swappable rule |
| `calculate(order, ...)` | **`PricingStrategy`** | Pricing *policy* is a swappable rule |
| `distanceTo(other)` | `Location` | It owns lat/lng |
| `placeOrder / assignAgent / advanceOrder` | `FoodDeliveryService` | It orchestrates across objects |

Three ownership decisions carry the whole design:
- **The `Order` owns its status lifecycle.** `updateStatus()` enforces the legal transitions and *rejects illegal jumps* like `PLACED → DELIVERED`. This is the **STATE** idea — recall [46 — State Pattern](./46-pattern-state.md). (We implement it as a compact transition table inside `Order`; a fuller State pattern would give each status its own class — noted in Extensions.)
- **The `DeliveryService` owns agent assignment via a STRATEGY** — recall [42 — Strategy Pattern](./42-pattern-strategy.md). Today: nearest available agent. Tomorrow: highest-rated, or fewest active orders. Swap the object, not the service.
- **A `PricingStrategy` owns fee calculation** — also a Strategy, so surge/flat/free-delivery are drop-in.

### 4. Class diagram (ASCII)

```
                        ┌─────────────────────────┐
                        │    FoodDeliveryService   │  (orchestrator)
                        │  restaurants[]           │
                        │  agents[]                │
                        │  orders[]                │
                        │  assignmentStrategy ─────┼───► AssignmentStrategy (interface)
                        │  pricingStrategy ────────┼──┐          ▲
                        │  placeOrder()            │  │          │ implements
                        │  assignAgent()           │  │   NearestAgentStrategy
                        │  advanceOrder()          │  │   HighestRatedStrategy
                        └───────┬─────────┬────────┘  │   FewestActiveStrategy
                                │         │           │
                 ┌──────────────┘         └──────┐    └──► PricingStrategy (interface)
                 ▼                                ▼                ▲ implements
        ┌────────────────┐              ┌──────────────────┐  StandardPricing
        │    Customer    │              │  DeliveryAgent   │  SurgePricing
        │  id, name      │              │  id, name        │
        │  address ──────┼──► Address   │  location ───────┼──► Location {lat,lng
        └────────┬───────┘   (Location) │  status:AgentSt. │      distanceTo()}
                 │                       │  rating          │
                 │ places                └──────────────────┘
                 ▼
        ┌────────────────────────────────────────────┐
        │                  Order                      │
        │  id, customer, restaurant, agent            │
        │  items: OrderItem[] ──► OrderItem{item,qty} │
        │  status: OrderStatus                        │
        │  itemTotal, deliveryFee, surge, total       │
        │  updateStatus(newStatus)  ◄── STATE MACHINE │
        └────────────────────────────────────────────┘
                 │ from
                 ▼
        ┌────────────────┐        ┌──────────────────┐
        │   Restaurant   │───────►│    MenuItem      │
        │  isOpen, loc   │  menu  │  name, price     │
        │                │        │  available       │
        └────────────────┘        └──────────────────┘
```

**The order state machine** (what `updateStatus` enforces):

```
   ┌────────┐   ┌───────────┐   ┌───────────┐   ┌────────┐   ┌────────────┐   ┌───────────┐
   │ PLACED │──►│ CONFIRMED │──►│ PREPARING │──►│ READY  │──►│ PICKED_UP  │──►│ DELIVERED │
   └───┬────┘   └────┬──────┘   └────┬──────┘   └────────┘   └────────────┘   └───────────┘
       │             │               │
       ▼             ▼               ▼
   ┌───────────────────────────────────────┐
   │              CANCELLED                 │   (allowed only up to PREPARING)
   └───────────────────────────────────────┘

   DELIVERED and CANCELLED are terminal — no outgoing transitions.
   PLACED ──► DELIVERED is ILLEGAL (skips states) and is rejected.
```

### 5. Full JavaScript implementation

Real, complete, runnable Node.js. Save as `food-delivery.js` and run `node food-delivery.js`.

```javascript
'use strict';

// ─────────────────────────────────────────────────────────────
// ENUMS — fixed sets of values, frozen so nobody can mutate them
// ─────────────────────────────────────────────────────────────
const OrderStatus = Object.freeze({
  PLACED: 'PLACED',
  CONFIRMED: 'CONFIRMED',
  PREPARING: 'PREPARING',
  READY: 'READY',
  PICKED_UP: 'PICKED_UP',
  DELIVERED: 'DELIVERED',
  CANCELLED: 'CANCELLED',
});

const AgentStatus = Object.freeze({
  AVAILABLE: 'AVAILABLE',
  BUSY: 'BUSY',
});

// ─────────────────────────────────────────────────────────────
// Location — lat/lng plus a simple distance helper.
// We use the equirectangular approximation: good enough for a
// city-scale delivery radius and easy to read. Returns kilometres.
// ─────────────────────────────────────────────────────────────
class Location {
  constructor(lat, lng) {
    this.lat = lat;
    this.lng = lng;
  }

  distanceTo(other) {
    const R = 6371; // Earth radius in km
    const toRad = (d) => (d * Math.PI) / 180;
    const x = toRad(other.lng - this.lng) * Math.cos(toRad((this.lat + other.lat) / 2));
    const y = toRad(other.lat - this.lat);
    return Math.sqrt(x * x + y * y) * R;
  }
}

// An Address is a Location with a human-readable label.
class Address extends Location {
  constructor(label, lat, lng) {
    super(lat, lng);
    this.label = label;
  }
}

// ─────────────────────────────────────────────────────────────
// Menu / Restaurant
// ─────────────────────────────────────────────────────────────
class MenuItem {
  constructor(name, price, available = true) {
    this.name = name;
    this.price = price;
    this.available = available;
  }
}

class Restaurant {
  constructor(id, name, location, isOpen = true) {
    this.id = id;
    this.name = name;
    this.location = location;
    this.isOpen = isOpen;
    this.menu = new Map(); // itemName -> MenuItem
  }

  addMenuItem(item) {
    this.menu.set(item.name, item);
    return this;
  }

  getItem(name) {
    return this.menu.get(name);
  }

  availableItems() {
    return [...this.menu.values()].filter((i) => i.available);
  }
}

// ─────────────────────────────────────────────────────────────
// Customer
// ─────────────────────────────────────────────────────────────
class Customer {
  constructor(id, name, address) {
    this.id = id;
    this.name = name;
    this.address = address; // an Address (a Location)
  }
}

// ─────────────────────────────────────────────────────────────
// DeliveryAgent — a rider with a live location and status.
// ─────────────────────────────────────────────────────────────
class DeliveryAgent {
  constructor(id, name, location, rating = 4.5) {
    this.id = id;
    this.name = name;
    this.location = location;
    this.status = AgentStatus.AVAILABLE;
    this.rating = rating;
    this.activeOrders = 0; // used by the "fewest active" strategy
  }

  isAvailable() {
    return this.status === AgentStatus.AVAILABLE;
  }
}

// ─────────────────────────────────────────────────────────────
// OrderItem — a line item. We SNAPSHOT the price at order time so
// a later menu price change never rewrites past orders.
// ─────────────────────────────────────────────────────────────
class OrderItem {
  constructor(menuItem, quantity) {
    this.name = menuItem.name;
    this.unitPrice = menuItem.price; // snapshot
    this.quantity = quantity;
  }

  get lineTotal() {
    return this.unitPrice * this.quantity;
  }
}

// ─────────────────────────────────────────────────────────────
// Order — owns its own status lifecycle (the STATE machine).
// A transition table declares which states may follow which.
// updateStatus() REJECTS illegal jumps like PLACED -> DELIVERED.
// ─────────────────────────────────────────────────────────────
const VALID_TRANSITIONS = Object.freeze({
  [OrderStatus.PLACED]:    [OrderStatus.CONFIRMED, OrderStatus.CANCELLED],
  [OrderStatus.CONFIRMED]: [OrderStatus.PREPARING, OrderStatus.CANCELLED],
  [OrderStatus.PREPARING]: [OrderStatus.READY, OrderStatus.CANCELLED],
  [OrderStatus.READY]:     [OrderStatus.PICKED_UP],
  [OrderStatus.PICKED_UP]: [OrderStatus.DELIVERED],
  [OrderStatus.DELIVERED]: [], // terminal
  [OrderStatus.CANCELLED]: [], // terminal
});

let ORDER_SEQ = 1000;

class Order {
  constructor(customer, restaurant, orderItems) {
    this.id = `ORD-${++ORDER_SEQ}`;
    this.customer = customer;
    this.restaurant = restaurant;
    this.items = orderItems; // OrderItem[]
    this.status = OrderStatus.PLACED;
    this.agent = null;

    // Pricing fields, filled in by a PricingStrategy.
    this.itemTotal = this.items.reduce((sum, it) => sum + it.lineTotal, 0);
    this.deliveryFee = 0;
    this.surgeMultiplier = 1;
    this.total = this.itemTotal;

    this.history = [OrderStatus.PLACED]; // audit trail
  }

  // The heart of the STATE machine.
  updateStatus(newStatus) {
    const allowed = VALID_TRANSITIONS[this.status];
    if (!allowed.includes(newStatus)) {
      throw new Error(
        `Illegal transition: ${this.status} -> ${newStatus} ` +
          `(allowed from ${this.status}: ${allowed.join(', ') || 'none — terminal'})`
      );
    }
    this.status = newStatus;
    this.history.push(newStatus);
    return this;
  }

  assignAgent(agent) {
    this.agent = agent;
    return this;
  }
}

// ─────────────────────────────────────────────────────────────
// STRATEGY #1 — AssignmentStrategy (recall 42 — Strategy).
// Base class documents the contract; subclasses swap the policy.
// This is the SAME class of problem as Uber driver matching (127).
// ─────────────────────────────────────────────────────────────
class AssignmentStrategy {
  // returns a DeliveryAgent or null
  assign(order, agents) {
    throw new Error('assign() must be implemented by a subclass');
  }
}

// Pick the closest AVAILABLE agent to the restaurant.
class NearestAgentStrategy extends AssignmentStrategy {
  assign(order, agents) {
    const from = order.restaurant.location;
    const free = agents.filter((a) => a.isAvailable());
    if (free.length === 0) return null;
    return free.reduce((best, a) =>
      from.distanceTo(a.location) < from.distanceTo(best.location) ? a : best
    );
  }
}

// Pick the AVAILABLE agent with the highest rating.
class HighestRatedStrategy extends AssignmentStrategy {
  assign(order, agents) {
    const free = agents.filter((a) => a.isAvailable());
    if (free.length === 0) return null;
    return free.reduce((best, a) => (a.rating > best.rating ? a : best));
  }
}

// Pick the AVAILABLE agent currently carrying the fewest orders (load balancing).
class FewestActiveStrategy extends AssignmentStrategy {
  assign(order, agents) {
    const free = agents.filter((a) => a.isAvailable());
    if (free.length === 0) return null;
    return free.reduce((best, a) => (a.activeOrders < best.activeOrders ? a : best));
  }
}

// ─────────────────────────────────────────────────────────────
// STRATEGY #2 — PricingStrategy (recall 42 — Strategy).
// total = itemTotal + distance-based deliveryFee, times surge.
// ─────────────────────────────────────────────────────────────
class PricingStrategy {
  calculate(order) {
    throw new Error('calculate() must be implemented by a subclass');
  }
}

class StandardPricing extends PricingStrategy {
  constructor({ baseFee = 20, perKm = 8 } = {}) {
    super();
    this.baseFee = baseFee; // flat pickup fee
    this.perKm = perKm;     // rupees per km
  }

  calculate(order) {
    const dist = order.restaurant.location.distanceTo(order.customer.address);
    order.deliveryFee = Math.round(this.baseFee + this.perKm * dist);
    order.surgeMultiplier = 1;
    order.total = Math.round((order.itemTotal + order.deliveryFee) * order.surgeMultiplier);
    return order.total;
  }
}

// Same as standard, but multiplies the whole bill during peak/rain.
class SurgePricing extends StandardPricing {
  constructor(opts = {}, surge = 1.5) {
    super(opts);
    this.surge = surge;
  }

  calculate(order) {
    const dist = order.restaurant.location.distanceTo(order.customer.address);
    order.deliveryFee = Math.round(this.baseFee + this.perKm * dist);
    order.surgeMultiplier = this.surge;
    order.total = Math.round((order.itemTotal + order.deliveryFee) * order.surgeMultiplier);
    return order.total;
  }
}

// ─────────────────────────────────────────────────────────────
// FoodDeliveryService — the orchestrator. Holds the catalogue and
// the fleet, and depends on the two strategies via composition so
// they can be swapped at runtime.
// ─────────────────────────────────────────────────────────────
class FoodDeliveryService {
  constructor(assignmentStrategy, pricingStrategy) {
    this.restaurants = [];
    this.agents = [];
    this.orders = [];
    this.assignmentStrategy = assignmentStrategy;
    this.pricingStrategy = pricingStrategy;
  }

  registerRestaurant(r) { this.restaurants.push(r); return this; }
  registerAgent(a) { this.agents.push(a); return this; }

  // Swap policies at runtime — the point of Strategy.
  setAssignmentStrategy(s) { this.assignmentStrategy = s; return this; }
  setPricingStrategy(s) { this.pricingStrategy = s; return this; }

  browseRestaurants() {
    return this.restaurants.filter((r) => r.isOpen);
  }

  // Build the order from {name, qty} lines, validate availability, price it.
  placeOrder(customer, restaurant, lines) {
    if (!restaurant.isOpen) throw new Error(`${restaurant.name} is closed`);

    const orderItems = lines.map(({ name, qty }) => {
      const menuItem = restaurant.getItem(name);
      if (!menuItem) throw new Error(`No such item: ${name}`);
      if (!menuItem.available) throw new Error(`${name} is unavailable`);
      return new OrderItem(menuItem, qty);
    });

    const order = new Order(customer, restaurant, orderItems);
    this.pricingStrategy.calculate(order); // fill in fees + total
    this.orders.push(order);
    return order;
  }

  assignAgent(order) {
    const agent = this.assignmentStrategy.assign(order, this.agents);
    if (!agent) throw new Error('No delivery agent available');
    order.assignAgent(agent);
    agent.status = AgentStatus.BUSY;
    agent.activeOrders += 1;
    return agent;
  }

  // Advance the order one step; the Order itself validates legality.
  advanceOrder(order, newStatus) {
    order.updateStatus(newStatus);
    // Free the agent once the food is delivered.
    if (newStatus === OrderStatus.DELIVERED && order.agent) {
      order.agent.status = AgentStatus.AVAILABLE;
      order.agent.activeOrders -= 1;
    }
    return order;
  }
}

// ─────────────────────────────────────────────────────────────
// main() — a runnable demo proving assignment, pricing, and the
// status lifecycle all work.
// ─────────────────────────────────────────────────────────────
function main() {
  const money = (n) => `Rs.${n}`;

  // --- Set up restaurants with menus (located around a city) ---
  const spice = new Restaurant('R1', 'Spice Villa', new Location(12.9716, 77.5946))
    .addMenuItem(new MenuItem('Paneer Butter Masala', 260))
    .addMenuItem(new MenuItem('Garlic Naan', 60))
    .addMenuItem(new MenuItem('Gulab Jamun', 90, /* available */ false));

  const pasta = new Restaurant('R2', 'Pasta Point', new Location(12.9600, 77.6400))
    .addMenuItem(new MenuItem('Alfredo Pasta', 320))
    .addMenuItem(new MenuItem('Garlic Bread', 120));

  // --- Delivery agents at different locations ---
  const agents = [
    new DeliveryAgent('A1', 'Ravi', new Location(12.9750, 77.5980), 4.9),  // near Spice Villa
    new DeliveryAgent('A2', 'Meena', new Location(12.9000, 77.7000), 4.6),  // far away
    new DeliveryAgent('A3', 'Karan', new Location(12.9720, 77.5950), 4.2),  // very near Spice Villa
  ];

  // --- Build the service: nearest-agent assignment + standard pricing ---
  const service = new FoodDeliveryService(new NearestAgentStrategy(), new StandardPricing());
  service.registerRestaurant(spice).registerRestaurant(pasta);
  agents.forEach((a) => service.registerAgent(a));

  console.log('Open restaurants:', service.browseRestaurants().map((r) => r.name).join(', '));

  // --- A customer places an order from Spice Villa ---
  const customer = new Customer('C1', 'Asha', new Address('Home', 12.9820, 77.6010));
  const order = service.placeOrder(customer, spice, [
    { name: 'Paneer Butter Masala', qty: 2 },
    { name: 'Garlic Naan', qty: 3 },
  ]);

  console.log(`\nOrder ${order.id} placed by ${customer.name}`);
  console.log(`  Items total:  ${money(order.itemTotal)}`);
  console.log(`  Delivery fee: ${money(order.deliveryFee)}`);
  console.log(`  Surge x:      ${order.surgeMultiplier}`);
  console.log(`  TOTAL:        ${money(order.total)}`);

  // --- Assign the nearest available agent (should be Karan, ~0.05 km) ---
  const agent = service.assignAgent(order);
  console.log(`\nAssigned nearest agent: ${agent.name} (rating ${agent.rating})`);

  // --- Advance through the legal lifecycle ---
  console.log('\nAdvancing order through its lifecycle:');
  for (const next of [
    OrderStatus.CONFIRMED,
    OrderStatus.PREPARING,
    OrderStatus.READY,
    OrderStatus.PICKED_UP,
    OrderStatus.DELIVERED,
  ]) {
    service.advanceOrder(order, next);
    console.log(`  -> ${order.status}`);
  }
  console.log(`  ${agent.name} is now ${agent.status} again (freed after delivery)`);

  // --- Prove illegal transitions are rejected ---
  console.log('\nTrying an ILLEGAL jump on a fresh order (PLACED -> DELIVERED):');
  const bad = service.placeOrder(customer, spice, [{ name: 'Garlic Naan', qty: 1 }]);
  try {
    bad.updateStatus(OrderStatus.DELIVERED);
  } catch (e) {
    console.log('  Rejected:', e.message);
  }

  // --- Swap the pricing Strategy to surge and re-price (Extension demo) ---
  console.log('\nSwapping to SurgePricing (x1.8) and re-pricing a new order:');
  service.setPricingStrategy(new SurgePricing({}, 1.8));
  const rainy = service.placeOrder(customer, spice, [{ name: 'Paneer Butter Masala', qty: 1 }]);
  console.log(`  Order total under surge: ${money(rainy.total)} (surge x${rainy.surgeMultiplier})`);

  // --- Swap the assignment Strategy to highest-rated (Extension demo) ---
  console.log('\nSwapping to HighestRatedStrategy; assigning the surge order:');
  service.setAssignmentStrategy(new HighestRatedStrategy());
  const chosen = service.assignAgent(rainy);
  console.log(`  Chosen agent: ${chosen.name} (rating ${chosen.rating}) <- highest rated free agent`);
}

main();
```

**Expected output (abridged):**

```
Open restaurants: Spice Villa, Pasta Point

Order ORD-1001 placed by Asha
  Items total:  Rs.700
  Delivery fee: Rs.32
  ...
Assigned nearest agent: Karan (rating 4.2)

Advancing order through its lifecycle:
  -> CONFIRMED
  -> PREPARING
  -> READY
  -> PICKED_UP
  -> DELIVERED
  Karan is now AVAILABLE again (freed after delivery)

Trying an ILLEGAL jump on a fresh order (PLACED -> DELIVERED):
  Rejected: Illegal transition: PLACED -> DELIVERED ...
```

Notice: Karan (nearest) got the first order; when we swapped to `HighestRatedStrategy`, Ravi (rating 4.9) was chosen instead — **the service code never changed**. That is Strategy earning its keep.

### 6. Which design patterns used and WHY

Two patterns are the stars of this design. Naming them out loud is what passes the interview.

**STATE — the order lifecycle** (recall [46 — State Pattern](./46-pattern-state.md)).
An `Order` behaves differently depending on which state it's in, and only certain transitions are legal. We encoded the state machine as a **transition table** inside `Order.updateStatus()`, which rejects illegal jumps (`PLACED → DELIVERED`). *Why here?* Because the lifecycle is the one piece of this system where a bug is catastrophic — marking an order DELIVERED that was never picked up means a customer is charged for food they never got. Centralising the rules in one guarded method makes that bug structurally impossible. (A textbook State pattern gives each status its own class with its own `next()` method; the table is the lightweight cousin and is usually enough — see Extensions.)

**STRATEGY — agent assignment AND pricing** (recall [42 — Strategy Pattern](./42-pattern-strategy.md)).
Both "which agent?" and "what price?" are **policies that change without changing the workflow**. We defined `AssignmentStrategy` (nearest / highest-rated / fewest-active) and `PricingStrategy` (standard / surge) as swappable objects the service holds by composition. *Why here?* Because a delivery company changes these rules constantly — surge on rainy days, load-balanced assignment at peak — and each change must be a *new class*, not an edit to a working, tested orchestrator. The demo proves it: swapping the strategy object changed the outcome with zero edits to `FoodDeliveryService`.

**Connection to Uber matching ([127 — LLD Uber Matching](./127-lld-uber-matching.md)):** picking the nearest available delivery agent is *exactly the same class of problem* as matching a rider to the nearest driver. Same shape — a pool of providers, a proximity/scoring function, a swappable matching policy. If you can design one, you can design the other; the only difference is the noun (agent vs. driver). Mention this link and you show the interviewer you see the underlying pattern, not just this one app.

(Minor supporting choices: **composition** for wiring strategies into the service, and a lightweight **snapshot** in `OrderItem` so historical prices never change.)

### 7. Extensions ("now add X") and how the design absorbs them

Interviewers always push with follow-ups. A good design absorbs each with a *small, local* change.

- **Add surge pricing** → **already done** via a pricing Strategy. `new SurgePricing({}, 1.8)` and `setPricingStrategy(...)`. `FoodDeliveryService` is untouched. This is the payoff of having made pricing a Strategy up front.
- **Assign by rating or load instead of distance** → swap the assignment Strategy: `service.setAssignmentStrategy(new HighestRatedStrategy())` or `new FewestActiveStrategy()`. Both already exist in the code above and the demo shows the swap. Zero changes to the workflow.
- **Real-time GPS tracking** → the `DeliveryAgent` already has a mutable `location`. Add an `updateLocation(loc)` method the rider's phone calls every few seconds, and let the customer subscribe (Observer pattern) to `order`/`agent` location changes. The status machine is unaffected — status and GPS are separate concerns.
- **Batch multiple orders to one agent** → drop the `agent.status = BUSY` hard flip; instead let a strategy consider `agent.activeOrders` and a capacity limit (say, 3). `FewestActiveStrategy` already reads `activeOrders`; a `BatchingStrategy` would assign a second nearby order to a rider already headed that way. Only a new Strategy class is needed.
- **Add scheduled orders** → add a `scheduledFor` timestamp to `Order` and a new *initial* state `SCHEDULED` that transitions to `PLACED` when the clock arrives. Because the lifecycle is a single transition table, you add one row — `[SCHEDULED]: [PLACED, CANCELLED]` — and nothing else moves.

Every extension is either a **new Strategy subclass** or **one row in the transition table**. That is the sign of a design that decomposed along the right seams.

---

## Visual / Diagram description

The two diagrams above are the ones to reproduce on a whiteboard:

1. **The class diagram** — draw `FoodDeliveryService` at the top as the orchestrator. From it, draw two dashed arrows to the *interfaces* `AssignmentStrategy` and `PricingStrategy` (label them "swappable — Strategy"), each with concrete subclasses hanging beneath. From the service also draw solid "has-a" arrows down to `Customer`, `DeliveryAgent`, and `Order`. `Order` composes `OrderItem`s, points to its `Restaurant` (which composes `MenuItem`s), its `Customer`, and its assigned `DeliveryAgent`. `Address` and `DeliveryAgent` both hold a `Location` with a `distanceTo()` helper — that helper is what the `NearestAgentStrategy` and the pricing use.

2. **The order state machine** — a left-to-right chain `PLACED → CONFIRMED → PREPARING → READY → PICKED_UP → DELIVERED`, with a downward branch to `CANCELLED` available only from the first three states. Mark `DELIVERED` and `CANCELLED` as terminal (double box). Draw a big red X through an imaginary arrow `PLACED → DELIVERED` to remind yourself that `updateStatus()` throws on it.

The mental model: **objects flow through the class diagram; the `Order` alone walks the state machine.**

---

## Real world examples

### Swiggy / Zomato (India)

Represent the same three-sided marketplace: customer, restaurant partner, delivery partner. Their order lifecycle is conceptually a state machine much like ours (placed → accepted by restaurant → food prep → rider assigned → picked up → delivered). Assignment is a **matching engine** that considers rider proximity, current load, and predicted prep time — a far richer version of our swappable `AssignmentStrategy`. Surge/dynamic delivery fees during rain or peak hours are a real, live example of a pricing Strategy. (Details representative; internals are not public.)

### DoorDash (US)

Publicly discusses its **"Deep Red" / dispatch** system, which does batching (one Dasher carries multiple nearby orders) and optimizes assignment against estimated ready-times — precisely our "batch multiple orders" extension taken to production scale. The core abstraction — a pool of couriers scored by a swappable policy — matches this design's `AssignmentStrategy`. (Representative summary of publicly described concepts.)

### Uber Eats

Reuses Uber's rider-matching infrastructure for food couriers — which is the whole point of the [127 — Uber Matching](./127-lld-uber-matching.md) connection: the *same* dispatch engine assigns cars to riders and couriers to meals. One matching abstraction, two products. That is the real-world proof that "assignment" deserves to be its own swappable strategy object rather than code buried in an order handler.

---

## Trade-offs

**Transition table (our choice) vs. full State-pattern classes:**

| Approach | Pros | Cons |
|----------|------|------|
| Transition table in `Order` | Tiny, all rules in one place, easy to read | Behaviour per state (not just legality) gets awkward |
| One class per state (full State) | Each state owns its own behaviour + next() | More classes; overkill when states only gate transitions |

**Strategy for assignment/pricing vs. `if/else` inside the service:**

| Approach | Pros | Cons |
|----------|------|------|
| Strategy objects | New policy = new class; testable in isolation; swap at runtime | A little more upfront structure |
| `if (mode === 'surge')` branches | Fast to write first time | Every new policy edits a working file; branches multiply; untestable |

**Auto-assign nearest vs. broadcast-and-accept (like ride-hailing):**

| Approach | Pros | Cons |
|----------|------|------|
| Service picks nearest (ours) | Deterministic, simple, no rider "declines" to handle | Rider can't refuse; ignores traffic/prep time |
| Offer to riders, first to accept wins | Respects rider choice; models reality | Race conditions, timeouts, re-offer logic |

**The sweet spot:** a **transition table** for the lifecycle plus **Strategy objects** for assignment and pricing. It is the smallest design that still lets every interviewer follow-up land as a new class or a new table row — which is exactly what "extensible" means in an LLD interview.

---

## Common interview questions on this topic

### Q1: "How do you stop an order being marked DELIVERED when it was never picked up?"
**Hint:** The `Order` owns a **transition table**; `updateStatus()` checks that `newStatus` is in the allowed set for the current status and throws otherwise. `PLACED → DELIVERED` isn't in any allowed set, so it's rejected. Correctness lives in one guarded method, not scattered across the codebase — this is the STATE idea (46).

### Q2: "Tomorrow we want to assign the highest-rated rider instead of the nearest. How much changes?"
**Hint:** One line. `service.setAssignmentStrategy(new HighestRatedStrategy())`. The `FoodDeliveryService` and `Order` are untouched. That's the whole reason assignment is a Strategy (42) rather than an `if` inside `placeOrder`.

### Q3: "Where does distance get computed, and why not put it on the service?"
**Hint:** On `Location.distanceTo()`, because `Location` owns lat/lng — behaviour goes on the object that owns the data. Both the pricing (distance-based fee) and `NearestAgentStrategy` reuse it. Putting it on the service would duplicate geo-math and couple the service to coordinates it shouldn't know about.

### Q4: "This looks a lot like Uber. Is it the same problem?"
**Hint:** Yes — assignment is the *same class of problem* as driver matching (127): a pool of providers, a scoring function, a swappable policy. Uber Eats literally reuses Uber's dispatch engine. Saying this shows you see the pattern under the app.

### Q5: "Add batching — one rider carries three orders. Does your design break?"
**Hint:** No. Replace the hard `AVAILABLE → BUSY` flip with a capacity check on `agent.activeOrders`, and add a `BatchingStrategy` (a new `AssignmentStrategy` subclass) that assigns a nearby second order to a rider already en route. Only a new Strategy class; the lifecycle and service are unchanged.

---

## Practice exercise

**Build a `FewestActiveStrategy` demo and a cancellation flow (~30 min).**

Starting from the code above:
1. Give three agents different `activeOrders` counts (e.g., 0, 2, 1) but keep them all AVAILABLE.
2. Swap the service to `new FewestActiveStrategy()` and place two orders, assigning both. Confirm the *least-loaded* agent is chosen first, and that assignment increments `activeOrders`.
3. Place a fresh order and call `advanceOrder(order, OrderStatus.CANCELLED)` while it's still `PLACED`. Confirm it succeeds.
4. Advance another order to `PREPARING`, then try to cancel it. Confirm your transition table rejects it (food is already cooking) — adjust the table so `CANCELLED` is only reachable from `PLACED`, `CONFIRMED`, `PREPARING`, and verify the terminal states stay terminal.

**Produce:** console output proving (a) the least-loaded agent was picked, (b) an early cancel works, and (c) a late cancel is rejected with a clear error message. If all three print correctly, you have internalised both Strategy and the state machine.

---

## Quick reference cheat sheet

- **Two patterns are the stars:** STATE (order lifecycle) + STRATEGY (assignment & pricing). Name both out loud.
- **`Order` owns its lifecycle.** `updateStatus()` + a transition table reject illegal jumps like `PLACED → DELIVERED`.
- **Lifecycle:** `PLACED → CONFIRMED → PREPARING → READY → PICKED_UP → DELIVERED`; `CANCELLED` only early; last two are terminal.
- **Assignment is a Strategy:** nearest / highest-rated / fewest-active — swap the object, never edit the service.
- **Pricing is a Strategy:** `itemTotal + distance-based deliveryFee` then `× surge`. Surge = a subclass.
- **Behaviour goes where the data lives:** `distanceTo()` on `Location`, `updateStatus()` on `Order`.
- **Enums** via `Object.freeze` so nobody mutates the value set.
- **`OrderItem` snapshots price** so a later menu change never rewrites past orders.
- **Same problem as Uber matching (127):** provider pool + scoring + swappable policy.
- **Every extension** is a new Strategy subclass or one new row in the transition table — the sign of good seams.
- **One order = one restaurant** (scope decision); ask about scheduled/multi-restaurant/payment up front.
- **Composition over inheritance:** the service *has-a* strategy; it doesn't *inherit* pricing logic.

---

## Connected topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Previous** | [42 — Strategy Pattern](./42-pattern-strategy.md) | The pattern behind swappable assignment and pricing — learn it before wielding it here |
| **Next** | [127 — LLD Uber Matching](./127-lld-uber-matching.md) | Agent assignment IS driver matching — the same problem at ride-hailing scale |
| **Related** | [46 — State Pattern](./46-pattern-state.md) | The order lifecycle is a state machine; formalise it with per-state classes |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (requirements → nouns → verbs → diagram → code → patterns → extensions) we followed here |
| **Related** | [121 — LLD Splitwise](./121-lld-splitwise.md) | Another broad LLD case study with swappable Strategy policies (split algorithms) — compare the decomposition |
