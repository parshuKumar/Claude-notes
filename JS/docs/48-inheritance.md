# 48 — Inheritance

## What is this?

Inheritance is the mechanism that lets one class **acquire the properties and methods of another class**, so you don't have to rewrite shared behaviour. Think of it like biological inheritance: a Dog is a specific kind of Animal — it shares everything an Animal has (name, breathe, eat) but also adds its own Dog-specific things (bark, fetch). In JavaScript, the `extends` keyword wires up the prototype chain between two classes, and `super` is how the child class talks to the parent.

## Why does it matter?

Without inheritance you end up copy-pasting the same code across multiple classes. Real applications have hierarchies everywhere: `AdminUser extends User`, `SavingsAccount extends BankAccount`, `VideoPlayer extends MediaPlayer`. Understanding `extends` and `super` — and their edge cases — is essential for reading any modern JavaScript or TypeScript codebase, and for understanding why certain bugs happen when someone breaks the inheritance chain.

---

## Syntax

```js
class Animal {                          // parent class (superclass / base class)
  constructor(name, sound) {
    this.name  = name;
    this.sound = sound;
  }
  speak() {
    return `${this.name} says ${this.sound}!`;
  }
}

class Dog extends Animal {              // child class (subclass / derived class)
  constructor(name) {
    super(name, "woof");               // MUST call super() before using 'this'
    this.tricks = [];                  // Dog-specific property
  }
  learnTrick(trick) {                  // Dog-specific method
    this.tricks.push(trick);
    return this;
  }
  showTricks() {
    return `${this.name} knows: ${this.tricks.join(", ")}`;
  }
}

const rex = new Dog("Rex");
console.log(rex.speak());              // "Rex says woof!"  — inherited
console.log(rex.learnTrick("sit").learnTrick("roll").showTricks());
// "Rex knows: sit, roll"
```

---

## How it works — line by line

```js
class Dog extends Animal {
```
Sets up two prototype links:
1. `Dog.prototype.__proto__ = Animal.prototype` — instances inherit Animal's methods
2. `Dog.__proto__ = Animal` — Dog itself inherits Animal's static methods

```js
  constructor(name) {
    super(name, "woof");
```
`super(...)` calls the **parent's constructor**. It creates the object and sets up `this`. You **must** call `super()` before any access to `this` in a derived constructor — JS throws a `ReferenceError` if you don't.

```js
    this.tricks = [];
```
After `super()` returns, `this` is available and you can add child-specific properties.

```js
  learnTrick(trick) { ... }
```
A method that only exists on `Dog.prototype` — `Animal` instances don't have it.

---

## The prototype chain after `extends`

```
rex (Dog instance)
├── name: "Rex"         own property (set by Animal's constructor via super)
├── tricks: []          own property (set by Dog's constructor)
└── [[Prototype]] ────► Dog.prototype
                        ├── learnTrick: fn
                        ├── showTricks: fn
                        └── [[Prototype]] ──► Animal.prototype
                                             ├── speak: fn
                                             └── [[Prototype]] ──► Object.prototype
                                                                   └── [[Prototype]] ► null
```

When `rex.speak()` is called: not on `rex` → not on `Dog.prototype` → found on `Animal.prototype`. ✓

---

## Example 1 — basic: method overriding

A child class can **override** a parent method by defining one with the same name:

```js
class Shape {
  constructor(color) {
    this.color = color;
  }
  area() {
    return 0;   // default — subclasses should override
  }
  describe() {
    return `A ${this.color} shape with area ${this.area().toFixed(2)}`;
  }
}

class Circle extends Shape {
  constructor(color, radius) {
    super(color);              // pass color up to Shape's constructor
    this.radius = radius;
  }
  area() {                     // overrides Shape's area()
    return Math.PI * this.radius ** 2;
  }
}

class Rectangle extends Shape {
  constructor(color, width, height) {
    super(color);
    this.width  = width;
    this.height = height;
  }
  area() {                     // overrides Shape's area()
    return this.width * this.height;
  }
}

const c = new Circle("red", 5);
const r = new Rectangle("blue", 4, 6);

console.log(c.describe());   // "A red shape with area 78.54"
console.log(r.describe());   // "A blue shape with area 24.00"

// describe() is inherited but it calls THIS instance's area() — polymorphism
```

---

## Example 2 — calling parent method with `super.method()`

`super` isn't just for the constructor — you can use `super.methodName()` to call the **parent's version** of an overridden method:

```js
class User {
  constructor(name, email) {
    this.name  = name;
    this.email = email;
  }
  getInfo() {
    return `${this.name} <${this.email}>`;
  }
  hasPermission(action) {
    return false;   // regular users have no permissions by default
  }
}

class AdminUser extends User {
  constructor(name, email, adminLevel) {
    super(name, email);
    this.adminLevel = adminLevel;
    this.permissions = new Set();
  }
  grantPermission(action) {
    this.permissions.add(action);
    return this;
  }
  hasPermission(action) {
    return this.permissions.has(action);  // overrides — checks the Set
  }
  getInfo() {
    const base = super.getInfo();          // calls User's getInfo(), then extends it
    return `${base} [Admin Level ${this.adminLevel}]`;
  }
}

const alice = new User("Alice", "alice@example.com");
const bob   = new AdminUser("Bob", "bob@example.com", 2);
bob.grantPermission("delete").grantPermission("ban");

console.log(alice.getInfo());           // "Alice <alice@example.com>"
console.log(bob.getInfo());             // "Bob <bob@example.com> [Admin Level 2]"
console.log(alice.hasPermission("delete"));  // false
console.log(bob.hasPermission("delete"));    // true
console.log(bob.hasPermission("sudo"));      // false

console.log(alice instanceof User);      // true
console.log(bob   instanceof AdminUser); // true
console.log(bob   instanceof User);      // true  — all the way up the chain
```

---

## Example 3 — real world: multi-level inheritance

```js
class Vehicle {
  constructor(make, model, year) {
    this.make    = make;
    this.model   = model;
    this.year    = year;
    this.mileage = 0;
  }
  drive(miles) {
    if (miles < 0) throw new RangeError("Miles cannot be negative.");
    this.mileage += miles;
    return this;
  }
  describe() {
    return `${this.year} ${this.make} ${this.model} (${this.mileage} miles)`;
  }
}

class Car extends Vehicle {
  constructor(make, model, year, doors) {
    super(make, model, year);
    this.doors = doors;
  }
  describe() {
    return `${super.describe()} — ${this.doors}-door car`;
  }
}

class ElectricCar extends Car {
  constructor(make, model, year, doors, batteryKwh) {
    super(make, model, year, doors);
    this.batteryKwh  = batteryKwh;
    this.chargeLevel = 100;          // percent
  }
  charge(percent) {
    this.chargeLevel = Math.min(100, this.chargeLevel + percent);
    return this;
  }
  describe() {
    return `${super.describe()}, battery: ${this.batteryKwh}kWh (${this.chargeLevel}% charged)`;
  }
}

const tesla = new ElectricCar("Tesla", "Model 3", 2024, 4, 82);
tesla.drive(150).drive(50);

console.log(tesla.describe());
// "2024 Tesla Model 3 (200 miles) — 4-door car, battery: 82kWh (100% charged)"

console.log(tesla instanceof ElectricCar);  // true
console.log(tesla instanceof Car);          // true
console.log(tesla instanceof Vehicle);      // true
console.log(tesla instanceof Object);       // true
```

---

## Static method inheritance

Static methods are also inherited:

```js
class Animal {
  static kingdom() { return "Animalia"; }
  static create(name, sound) { return new this(name, sound); }
  // 'new this' = new Animal when called on Animal,
  //              new Dog when called on Dog (polymorphic factory)
}

class Dog extends Animal {
  constructor(name, sound) {
    super();
    this.name  = name;
    this.sound = sound;
  }
  static species() { return "Canis lupus familiaris"; }
}

console.log(Animal.kingdom());  // "Animalia"
console.log(Dog.kingdom());     // "Animalia"  — inherited static method
console.log(Dog.species());     // "Canis lupus familiaris"

// 'new this' in create() — polymorphic:
const d = Dog.create("Rex", "woof");
console.log(d instanceof Dog);  // true  — 'this' was Dog when we called Dog.create()
```

---

## Mixins — composing behaviour without deep inheritance

JavaScript only supports **single inheritance** (one `extends` target). Mixins are a pattern to compose behaviour from multiple sources without deep chains:

```js
// A mixin is just a function that adds methods to a class
const Serializable = (Base) => class extends Base {
  serialize() {
    return JSON.stringify(this);
  }
  static deserialize(json) {
    return Object.assign(new this(), JSON.parse(json));
  }
};

const Validatable = (Base) => class extends Base {
  validate() {
    return Object.values(this).every(v => v !== null && v !== undefined);
  }
};

class Entity {
  constructor(id) { this.id = id; }
}

// Apply both mixins
class User extends Serializable(Validatable(Entity)) {
  constructor(id, name, email) {
    super(id);
    this.name  = name;
    this.email = email;
  }
}

const user = new User(1, "Alice", "alice@example.com");
console.log(user.validate());    // true
console.log(user.serialize());   // '{"id":1,"name":"Alice","email":"alice@example.com"}'
```

---

## Tricky things

### 1. `super()` must come before `this` in the constructor

```js
class Child extends Parent {
  constructor(name) {
    this.name = name;  // ReferenceError: Must call super constructor before accessing 'this'
    super();
  }
}

// Fix:
class Child extends Parent {
  constructor(name) {
    super();           // always first
    this.name = name;
  }
}
```

### 2. Forgetting `super()` entirely when you define a constructor

```js
class Parent {
  constructor() { this.value = 42; }
}

class Child extends Parent {
  constructor() {
    // forgot super()!
  }
  // ReferenceError when instantiated — 'this' is never initialised
}

// Fix: always call super() in derived constructors
// OR omit the constructor entirely if you have nothing extra to set up:
class Child extends Parent {
  // no constructor = parent constructor is called automatically
}
const c = new Child();
console.log(c.value);  // 42 — works fine
```

### 3. Omitting the constructor in a derived class

If you don't write a `constructor` in a child class, JS generates this automatically:

```js
constructor(...args) {
  super(...args);   // forwards all arguments to parent
}
```

This is fine when the child needs no additional setup.

### 4. `super` in method calls — not available in arrow functions at class level

```js
class Parent {
  greet() { return "Hello from Parent"; }
}

class Child extends Parent {
  greet() {
    // This works:
    return super.greet() + " and Child";
  }

  // Arrow function defined as class field — super does NOT work here
  greetArrow = () => {
    // super.greet()  — SyntaxError or unexpected behaviour
  };
}
```

### 5. Overriding a method but forgetting to call `super` when you need it

```js
class Logger {
  log(message) {
    console.log(`[LOG] ${message}`);
    this.lastMessage = message;   // side effect that subclasses might need
  }
}

class TimestampLogger extends Logger {
  log(message) {
    // WRONG — forgot super.log(), so this.lastMessage is never set
    console.log(`[${new Date().toISOString()}] ${message}`);
  }

  // RIGHT:
  log(message) {
    super.log(message);   // sets this.lastMessage + base logging
    console.log(`Timestamp: ${new Date().toISOString()}`);
  }
}
```

### 6. `instanceof` checks the prototype chain — can surprise you

```js
class A {}
class B extends A {}
class C extends B {}

const c = new C();
console.log(c instanceof C);  // true
console.log(c instanceof B);  // true  — B is in the chain
console.log(c instanceof A);  // true  — A is in the chain
console.log(c instanceof Object); // true — always
```

---

## Common mistakes

### Mistake 1 — Using `this` before `super()` in derived constructor

```js
// WRONG
class PremiumUser extends User {
  constructor(name, email, plan) {
    this.plan = plan;   // ReferenceError — super() not called yet
    super(name, email);
  }
}

// RIGHT
class PremiumUser extends User {
  constructor(name, email, plan) {
    super(name, email); // always first
    this.plan = plan;
  }
}
```

### Mistake 2 — Deep inheritance chains for code reuse (prefer composition)

```js
// WRONG — fragile hierarchy
class Animal {}
class Mammal extends Animal {}
class Pet extends Mammal {}
class Dog extends Pet {}
class GoldenRetriever extends Dog {}
class MyDog extends GoldenRetriever {}   // any change up top breaks everything

// BETTER — shallow hierarchy + mixins / composition for shared behaviour
class Dog extends Animal { ... }
// Add capabilities via mixins or helper functions, not 5 levels of extends
```

### Mistake 3 — Overriding a method and breaking the parent contract

```js
// Parent's method returns a number
class Calculator {
  compute(a, b) { return a + b; }   // always returns number
}

// WRONG — override changes the return type
class LoggingCalculator extends Calculator {
  compute(a, b) {
    console.log(`Computing ${a} + ${b}`);
    // forgot to return!
  }
}

const lc = new LoggingCalculator();
const result = lc.compute(2, 3);   // undefined — breaks any code expecting a number

// RIGHT
class LoggingCalculator extends Calculator {
  compute(a, b) {
    console.log(`Computing ${a} + ${b}`);
    return super.compute(a, b);  // preserves the return value
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

Create a parent class `Animal` with `name` and `type` properties and a `describe()` method returning `"[name] is a [type]."`. Create two child classes `Cat` and `Dog`, each calling `super` with the correct type string. `Cat` adds a method `purr()` returning `"Purrr..."`. `Dog` adds a method `fetch(item)` returning `"[name] fetches the [item]!"`. Verify that both `instanceof Animal` checks return `true`.

```js
// Write your code here
```

### Exercise 2 — medium

Create a class `Account` with `owner`, `balance`, `deposit(amount)`, and `withdraw(amount)`. Extend it with `SavingsAccount` that adds:
- A `interestRate` property (e.g. 0.05 for 5%)
- A `applyInterest()` method that increases balance by `balance * interestRate`
- Override `withdraw(amount)` to enforce a **minimum balance of $100** — throw an error if a withdrawal would drop below it
- Override `describe()` (add one to Account too) so SavingsAccount includes the interest rate

Create one Account and one SavingsAccount, run deposits and withdrawals on both, and test that the minimum balance rule is enforced only on SavingsAccount.

```js
// Write your code here
```

### Exercise 3 — hard

Build a three-level class hierarchy for a notification system:

1. `Notification` (base) — `id` (auto-increment static counter), `title`, `message`, `createdAt` (Date), `isRead = false`. Methods: `markRead()`, `toString()` returning `"[id] [title]: [message]"`.

2. `AlertNotification extends Notification` — adds `severity` (`"low"`, `"medium"`, `"critical"`). Override `toString()` to prepend `"[SEVERITY]"`. Add `isCritical()` returning `true` if severity is `"critical"`.

3. `ActionableNotification extends AlertNotification` — adds `actionLabel` and `actionUrl`. Add `execute()` that returns `"Navigating to [actionUrl]"` and marks the notification as read. Override `toString()` to append `" → [actionLabel]"`.

Create one of each type, call all their methods, verify the `instanceof` chain for the deepest instance.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Syntax / Rule |
|---|---|
| Define child class | `class Dog extends Animal { ... }` |
| Call parent constructor | `super(arg1, arg2)` — must be **first line** in derived constructor |
| Call parent method | `super.methodName(args)` — inside any overridden method |
| Omit constructor | Safe — JS auto-generates `constructor(...args) { super(...args); }` |
| Override a method | Define method with same name in child class |
| Check ancestry | `obj instanceof ParentClass` — walks entire prototype chain |
| Static inheritance | Child class inherits parent static methods automatically |
| `super` in static | `static method() { return super.staticMethod(); }` — works too |
| `this` before `super` | `ReferenceError` — always call `super()` first |
| Single inheritance only | One `extends` target — use mixins for multiple sources |
| Prototype chain | `DogInstance → Dog.prototype → Animal.prototype → Object.prototype → null` |
| Class chain | `Dog.__proto__ === Animal` (the classes themselves are also linked) |

---

## Connected topics

- **47 — ES6 Classes** — Inheritance is a direct extension of class syntax. That topic is the prerequisite.
- **46 — Prototypes and prototype chain** — `extends` works entirely by setting up prototype links. Knowing the chain explains every `instanceof` check.
- **49 — Static methods and properties** — Static members are also inherited via `extends`, but the rules have nuances worth their own topic.
